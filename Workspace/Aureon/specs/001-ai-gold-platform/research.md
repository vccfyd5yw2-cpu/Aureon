# Research: AI-Powered Gold Market Intelligence Platform

**Phase**: 0 (Pre-Design) | **Date**: 2026-06-20 | **Plan**: [plan.md](plan.md)

---

## 1. Gold Price Data Feed

### Decision: External REST API (polling, 55 s interval)

**Options evaluated**:

| Option | Pro | Con | Decision |
|--------|-----|-----|----------|
| GoldAPI.io | Simple REST, free tier, XAU/USD + local prices | Rate limited on free tier (1 req/min) | **Selected** — fits 55 s poll; swap via port |
| Metals-API (via Fixer) | Well-documented, historical data endpoint | Paid for historical beyond 1 year | Acceptable; same adapter interface |
| WebSocket feed (e.g., Oanda) | True real-time push | Complex auth, not needed for 60 s SLA | Rejected — over-engineered for SLA |

**Architecture**:
- `PriceFeedPort` (domain interface) → `GoldApiAdapter` (infrastructure)
- `PricePollingScheduler` runs every 55 s using `@Scheduled`
- On each tick: fetch → write to Redis (TTL 65 s) → persist snapshot to `gold_prices` table
- SSE endpoint streams Redis-cached value to connected browser clients every 30 s

**Local gold price**: Vietnamese market uses VND/lượng (tael). Price feed must support `XAU/VND` conversion or a supplementary endpoint. Assumption: `price_vnd = price_usd × usd_vnd_rate` where USD/VND rate is cached separately.

---

## 2. Technical Indicators — Computation Strategy

### Decision: Server-side computation, cached results

**Rationale**: Indicators computed server-side keep the frontend thin, enable caching, and allow future server-push of indicator updates. Client-side JS calculation is acceptable for prototypes but creates duplicated logic across clients.

**Indicator formulas**:

| Indicator | Formula summary | Input | Output |
|-----------|----------------|-------|--------|
| SMA(n) | Sum(close[i], n) / n | close prices, period | single line series |
| EMA(n) | α·close + (1−α)·EMA_prev, α=2/(n+1) | close prices, period | single line series |
| RSI(14) | 100 − 100/(1 + RS), RS = avg_gain/avg_loss | close prices, period | oscillator 0–100 |
| MACD(12,26,9) | EMA(12)−EMA(26); signal=EMA(9) of MACD | close prices | MACD line + signal + histogram |
| Bollinger(20,2) | middle=SMA(20); upper/lower = middle ± 2σ | close prices, period, stddev multiplier | three band series |

**Implementation**: `TechnicalIndicatorCalculator` domain service; pure Java, zero framework imports, fully unit-testable.

**Caching**: Computed indicator series cached in Redis with a composite key `indicator:{type}:{period}:{range}:{currency}` and TTL matching the chart range (1D → 5 min TTL; 5Y → 24 h TTL).

---

## 3. AI Chatbot — LLM Integration

### Decision: Spring AI with OpenAI-compatible API; RAG via price context injection

**Options evaluated**:

| Option | Pro | Con | Decision |
|--------|-----|-----|----------|
| OpenAI GPT-4o | High quality, well-supported Spring AI starter | Cost, data residency concern | **Default** — env-configurable |
| Anthropic Claude | Strong reasoning, Spring AI support | Slightly more expensive | Supported via Spring AI; swap via config |
| Ollama (local) | Free, privacy-preserving | Hardware requirements, lower quality | Supported for dev/test via Spring AI |

**Grounding strategy** (RAG-lite, no vector DB for v1):
1. User submits question
2. Application layer queries PostgreSQL for relevant price data (last 90 days of daily closes, current price, 7/30/90-day stats)
3. Data is formatted as a structured context block and injected into the system prompt
4. LLM generates response referencing the provided data
5. Response tagged with `data_grounded: true` if context was non-empty

**Conversation history**: Last 10 turns stored in Redis (`chat:{sessionId}`, TTL 30 min). Anonymous sessions use a browser-generated UUID stored in `localStorage`.

**Prompt template** (system): stored as `src/main/resources/prompts/market-analysis.st` (Spring AI StringTemplate format).

**Cost control**: Max 4 096 output tokens per request; context window budget 8 000 tokens (context block + history + system prompt).

---

## 4. Authentication — Google OAuth 2.0

### Decision: Spring Security OAuth2 Login + stateless JWT for API calls

**Flow**:
1. Frontend redirects to `GET /oauth2/authorization/google`
2. Spring Security handles Google callback at `GET /login/oauth2/code/google`
3. On success: `OAuth2UserService` upserts `User` record; issues signed JWT (HS256, 7 d expiry)
4. JWT returned to frontend via redirect with token in URL fragment (SPA pattern)
5. Frontend stores JWT in memory (not `localStorage`) and attaches as `Authorization: Bearer` header
6. `JwtAuthenticationFilter` validates token on every protected request

**Security considerations**:
- JWT signing key injected via env var `JWT_SECRET` (≥ 256-bit random)
- PKCE enabled on OAuth2 client to prevent authorization code interception
- CSRF disabled for stateless JWT API; enabled for the OAuth2 redirect endpoints
- All API responses include `X-Content-Type-Options: nosniff` and `X-Frame-Options: DENY`

---

## 5. Caching Architecture (Redis)

| Key pattern | TTL | Content |
|-------------|-----|---------|
| `price:live:{symbol}` | 65 s | Latest `GoldPriceDTO` (JSON) |
| `price:rate:usd_vnd` | 300 s | USD/VND exchange rate |
| `chart:{symbol}:{range}` | varies (5 min – 24 h) | OHLCV array (JSON) |
| `indicator:{type}:{period}:{symbol}:{range}` | same as chart | Computed indicator series |
| `chat:{sessionId}` | 30 min | List of `ChatMessage` (JSON) |
| `alert:check:lock` | 90 s | Distributed lock for alert checking scheduler |

---

## 6. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Price API rate limit hit | Medium | High (stale prices) | 55 s poll < 1-req/min limit; Redis cache absorbs all UI reads |
| LLM API latency > 10 s | Medium | Medium (SC-004 breach) | Streaming response via SSE; timeout guard at 12 s with fallback message |
| LLM hallucinates price data | Medium | High (trust damage) | Context injection with actual DB data; prompt instructs "cite provided data only" |
| PostgreSQL connection pool exhaustion | Low | High | HikariCP pool size tuned; Testcontainers tests verify under concurrency |
| JWT secret exposure | Low | Critical | Injected via env var; secret never logged; rotation procedure documented in quickstart |
| Price alert gap scenario | Medium | Low | Alert checker runs every 30 s; gap-crossing detected by comparing previous and current prices |

---

## 7. Resolved Design Decisions

| Question | Decision | Rationale |
|----------|----------|-----------|
| Build tool: Maven vs Gradle | Gradle (Kotlin DSL) | Faster incremental builds; better multi-module support for future |
| ORM strategy | Spring Data JPA (Hibernate) with explicit SQL for bulk reads | JPA for CRUD entities; native queries for time-series bulk reads to avoid N+1 |
| Frontend chart library | Lightweight Charts (TradingView) for OHLCV; Recharts for indicator sub-panels | Lightweight Charts is purpose-built for financial charts; Recharts handles RSI/MACD oscillator sub-panels cleanly |
| State management | Zustand for auth/UI state; React Query (TanStack) for all server data | React Query eliminates manual cache management for API data; Zustand is minimal for global UI state |
| CSS approach | Tailwind CSS | Utility-first; no runtime CSS-in-JS overhead; excellent with Vite |
| API versioning | URL prefix `/api/v1/` | Simple, explicit; allows v2 routes without breaking existing clients |
| Historical data storage | Dedicated `gold_prices` table with `(symbol, timestamp)` composite index | Time-series query pattern suits indexed relational table for this data volume (< 50k rows/year) |
