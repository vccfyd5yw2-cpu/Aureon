# Tasks: AI-Powered Gold Market Intelligence Platform

**Input**: Design documents from `specs/001-ai-gold-platform/`

**Prerequisites**: [plan.md](plan.md) · [spec.md](spec.md) · [research.md](research.md) · [data-model.md](data-model.md) · [contracts/](contracts/) · [quickstart.md](quickstart.md)

**Stack**: Java 21 · Spring Boot 3.3 · PostgreSQL 16 · Redis 7 · React 18 · Vite 5 · Docker Compose

---

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel with other [P] tasks in the same phase (different files, no shared state)
- **[Story]**: User story this task belongs to (US1–US6)
- Exact file paths are listed in every task description

---

## Phase 1: Project Setup

**Purpose**: Initialize all three sub-projects, wire Docker Compose, and establish CI conventions.

- [ ] T001 Create root directory layout: `backend/`, `frontend/`, `infra/` at repo root; add root `.gitignore` covering `infra/.env`, build outputs, and IDE files
- [ ] T002 [P] Initialize Spring Boot 3.3 project in `backend/` via Spring Initializr or `gradle init`; configure `build.gradle.kts` with dependencies: `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-data-redis`, `spring-boot-starter-security`, `spring-boot-starter-oauth2-client`, `spring-boot-starter-webflux`, `spring-ai-openai-spring-boot-starter`, `flyway-core`, `lombok`, `mapstruct`; set Java toolchain to 21
- [ ] T003 [P] Initialize React + Vite + TypeScript project in `frontend/` via `npm create vite@latest`; install: `axios`, `@tanstack/react-query`, `zustand`, `lightweight-charts`, `recharts`, `tailwindcss`, `react-router-dom`; configure `vite.config.ts` with proxy `/api → http://localhost:8080`
- [ ] T004 [P] Create `infra/docker-compose.yml` defining services: `postgres` (image: postgres:16), `redis` (image: redis:7-alpine), `backend` (build: ../backend), `frontend` (build: ../frontend), `nginx` (build: ./nginx); create `infra/docker-compose.dev.yml` override exposing postgres:5432 and redis:6379 for local dev
- [ ] T005 [P] Write `infra/.env.example` with all required keys: `POSTGRES_PASSWORD`, `GOLD_API_KEY`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `JWT_SECRET`, `OPENAI_API_KEY`, `SPRING_AI_OPENAI_MODEL`; add `infra/.env` to `.gitignore`
- [ ] T006 [P] Create `infra/nginx/nginx.conf`: proxy `/api/` to `http://backend:8080/`, serve React SPA from `/` with `try_files $uri /index.html`
- [ ] T007 Write `backend/Dockerfile`: multi-stage — `gradle:8-jdk21` build stage → `eclipse-temurin:21-jre-alpine` runtime stage; copy fat JAR; set non-root user
- [ ] T008 [P] Write `frontend/Dockerfile`: multi-stage — `node:20-alpine` build stage (`npm ci && npm run build`) → `nginx:alpine` runtime stage serving `dist/`
- [ ] T009 [P] Configure `backend/src/main/resources/application.yml`: datasource (via `${POSTGRES_*}` env), Redis (`${REDIS_*}`), JPA (`ddl-auto: validate`, `show-sql: false`), Flyway (`enabled: true`), Spring AI (`${OPENAI_API_KEY}`, `${SPRING_AI_OPENAI_MODEL}`); add `application-dev.yml` with relaxed CORS; **set `spring.task.scheduling.pool.size=2`** so `PricePollingScheduler` and `AlertCheckScheduler` each get a dedicated thread and cannot block each other (G7 fix)
- [ ] T010 [P] Set up Tailwind CSS in `frontend/`: create `tailwind.config.ts`, add `@tailwind` directives to `frontend/src/index.css`
- [ ] T011-MSW [P] Install and configure MSW for frontend development isolation: `npm install --save-dev msw`; create `frontend/src/mocks/handlers.ts` with request handlers stubbing all 7 contract endpoints (prices, charts, chat, auth, watchlist, portfolio, alerts); create `frontend/src/mocks/browser.ts` (dev) and `frontend/src/mocks/server.ts` (Vitest); enable in `vite.config.ts` dev mode via `msw/vite` plugin — allows frontend to run and be tested fully without backend (G4 fix)

**Checkpoint**: `docker compose up --build` starts all services; `http://localhost` returns frontend placeholder; `http://localhost/api/v1/health` returns 200

---

## Phase 2: Foundation (Blocking Prerequisites)

**Purpose**: Domain model, persistence layer, cache, security skeleton, and all Flyway migrations. Nothing from Phase 3+ can start until this phase is complete.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

### Database Migrations

- [ ] T011 Write Flyway migration `backend/src/main/resources/db/migration/V1__create_users.sql` — `users` table per data-model.md
- [ ] T012 [P] Write `V2__create_gold_prices.sql` — `gold_prices` table with composite index `(symbol, recorded_at DESC)` and dedup unique index
- [ ] T013 [P] Write `V3__create_watchlist_items.sql` — `watchlist_items` table with unique constraint `(user_id, symbol)`
- [ ] T014 [P] Write `V4__create_holdings.sql` — `holdings` table with check constraints
- [ ] T015 [P] Write `V5__create_price_alerts.sql` — `price_alerts` table including `alert_direction` and `alert_status` enum types
- [ ] T016 [P] Write `V6__create_notifications.sql` — `notifications` table with `is_read BOOLEAN` column (not `read`, which is a reserved word in JPA/HQL) and partial index on `(user_id, is_read) WHERE is_read = false`

### Domain Model (pure Java records, zero framework imports)

- [ ] T017 Create `backend/src/main/java/com/aureon/domain/model/GoldPrice.java` — Java record: `symbol`, `price`, `currency`, `source`, `recordedAt`
- [ ] T018 [P] Create `domain/model/User.java` — record: `id`, `googleSub`, `email`, `displayName`, `avatarUrl`, `createdAt`
- [ ] T019 [P] Create `domain/model/WatchlistItem.java` — record: `id`, `userId`, `symbol`, `label`, `createdAt`
- [ ] T020 [P] Create `domain/model/Holding.java` — record: `id`, `userId`, `symbol`, `quantity`, `purchasePrice`, `purchaseCurrency`, `purchaseDate`, `notes`
- [ ] T021 [P] Create `domain/model/PriceAlert.java` — record with `AlertDirection` and `AlertStatus` enums
- [ ] T022 [P] Create `domain/model/Notification.java` — record: `id`, `userId`, `type`, `referenceId`, `title`, `body`, `read`, `createdAt`
- [ ] T023 [P] Create `domain/model/ChatMessage.java` — record: `role` (enum USER/ASSISTANT), `content`, `timestamp`, `dataGrounded`

### Outbound Ports (interfaces in `domain/port/out/`)

- [ ] T024 Create `GoldPriceRepository.java` — methods: `save`, `findLatestBySymbol`, `findBySymbolBetween`, `findLatestBySymbolBefore(symbol, instant)` (used to compute 24 h price change)
- [ ] T025 [P] Create `UserRepository.java` — methods: `findByGoogleSub`, `findById`, `upsert`
- [ ] T026 [P] Create `WatchlistRepository.java` — methods: `findByUserId`, `save`, `deleteByUserIdAndSymbol`, `existsByUserIdAndSymbol`
- [ ] T027 [P] Create `HoldingRepository.java` — methods: `findByUserId`, `findByIdAndUserId`, `save`, `deleteByIdAndUserId`
- [ ] T028 [P] Create `AlertRepository.java` — methods: `findByUserId`, `findActiveBySymbol`, `save`, `updateStatus`
- [ ] T029 [P] Create `NotificationRepository.java` — methods: `findByUserId`, `save`, `markRead`
- [ ] T030 [P] Create `PriceFeedPort.java` — method: `fetchLive(String symbol): GoldPrice`
- [ ] T031 [P] Create `LlmPort.java` — method: `chat(String systemPrompt, List<ChatMessage> history, String userMessage): Flux<String>`
- [ ] T032 [P] Create `PriceCachePort.java` — methods: `getLivePrice`, `setLivePrice`, `getChart`, `setChart`, `getIndicator`, `setIndicator`

### Inbound Ports (interfaces in `domain/port/in/`)

- [ ] T033 Create all inbound use-case interfaces: `GetLivePriceUseCase`, `GetPriceHistoryUseCase`, `GetIndicatorsUseCase`, `AskChatbotUseCase`, `ManageWatchlistUseCase`, `ManageHoldingsUseCase`, `ManageAlertsUseCase`, `GetNotificationsUseCase` — in `domain/port/in/`

### Infrastructure Adapters (JPA entities, repositories)

- [ ] T034 Create JPA `@Entity` classes in `infrastructure/persistence/entity/`: `UserEntity`, `GoldPriceEntity`, `WatchlistItemEntity`, `HoldingEntity`, `PriceAlertEntity`, `NotificationEntity` — annotated with `@Table`, `@Column`, mapped to migration table names
- [ ] T035 [P] Create Spring Data JPA interfaces in `infrastructure/persistence/repository/` for each entity; implement outbound port adapters in `infrastructure/persistence/adapter/` using MapStruct mappers

### Security Framework

- [ ] T036 Create `infrastructure/security/SecurityConfig.java` — Spring Security filter chain: permit `GET /api/v1/prices/**`, `GET /api/v1/charts/**`, `POST /api/v1/chat/**`, `GET /api/v1/chat/**`; require auth for `/api/v1/watchlist/**`, `/api/v1/portfolio/**`, `/api/v1/alerts/**`, `/api/v1/notifications/**`, `/api/v1/auth/me`; configure OAuth2 login with Google; disable CSRF for API routes; add `JwtAuthenticationFilter`
- [ ] T037 [P] Create `infrastructure/security/JwtService.java` — generate (HS256, 7d expiry from `${JWT_SECRET}`) and validate JWT; never log the secret
- [ ] T038 [P] Create `infrastructure/security/JwtAuthenticationFilter.java` — extract `Authorization: Bearer` header, validate token, set `SecurityContext`
- [ ] T039 [P] Create `infrastructure/security/OAuth2UserServiceImpl.java` — implements `OAuth2UserService`; on Google callback: upsert `User` record via `UserRepository`; generate JWT; redirect to SPA with token in URL fragment

### Redis Adapter & Cache

- [ ] T040 Create `infrastructure/cache/RedisCacheAdapter.java` implementing `PriceCachePort`; use Lettuce; serialize/deserialize with Jackson; apply TTL strategy from research.md (live price 65 s, chart TTL by range)

### Error Handling

- [ ] T041 Create `presentation/rest/GlobalExceptionHandler.java` (`@RestControllerAdvice`) — map domain exceptions and Spring validation failures to `ErrorResponse` DTO; return RFC 7807-style JSON; never expose stack traces in production; **add guard in `JwtAuthenticationFilter`: after validating JWT signature, load user from `UserRepository.findById(userId)`; if absent (deleted/deactivated account), reject with 401 — prevents deactivated accounts from using valid tokens for up to 7 days** (G6 fix)

**Checkpoint**: Backend starts cleanly with all migrations applied; `GET /api/v1/health` returns `{"status":"UP"}`; Spring Security denies unauthenticated requests to protected routes

---

## Phase 3: US1 — Real-Time Gold Price Dashboard (Priority: P1) 🎯 MVP

**Goal**: Live XAU/USD and local gold prices visible on the dashboard with ≤ 60 s staleness, no login required.

**Independent Test**: Open `http://localhost` → dashboard shows current XAU/USD price, change %, and last-updated timestamp; price refreshes within 60 s; "data unavailable" banner appears when feed is mocked offline.

### Backend — Price Feed & Live API

- [ ] T042 Create `infrastructure/pricefeed/GoldApiAdapter.java` implementing `PriceFeedPort` — `WebClient`-based HTTP call to GoldAPI.io; parse response into `GoldPrice`; map HTTP errors to domain exception; inject `${GOLD_API_KEY}` via constructor (never hardcoded)
- [ ] T043 Create `infrastructure/scheduler/PricePollingScheduler.java` — `@Scheduled(fixedDelay = 55_000)`; call `PriceFeedPort.fetchLive` for `XAU/USD` and `XAU/VND`; write result to `PriceCachePort` and persist to `GoldPriceRepository`; log warn on failure but do not crash
- [ ] T044 Create `application/price/GetLivePriceService.java` implementing `GetLivePriceUseCase` — read latest price from `PriceCachePort` first; fall back to `GoldPriceRepository.findLatestBySymbol`; attach `stale: true` flag when cache miss; **compute `changeAmount` and `changePercent` (required fields in `contracts/prices.yaml`) by querying the closing price from ~24 h earlier via `GoldPriceRepository.findLatestBySymbolBefore(symbol, now - 24h)` and computing the delta; if no 24 h-old data exists, return change fields as 0 with `stale` semantics preserved** (C-002 fix)
- [ ] T045 Create `presentation/rest/PriceController.java` — `GET /api/v1/prices/live?symbol=` mapping to `GetLivePriceUseCase`; response: `GoldPriceResponse` DTO per `contracts/prices.yaml`
- [ ] T046 Create `presentation/sse/PriceSseController.java` — `GET /api/v1/prices/live/stream` returns `Flux<ServerSentEvent<GoldPriceResponse>>`; emits every 30 s by polling cache; set `Content-Type: text/event-stream`

### Integration Tests (backend)

- [ ] T047 [P] Write Testcontainers integration test `backend/src/test/integration/PriceFeedIntegrationTest.java` — spin up Redis; assert `PricePollingScheduler` writes price to cache and DB; WireMock stub for GoldAPI
- [ ] T048 [P] Write REST-assured contract test `backend/src/test/contract/PricesContractTest.java` — assert `GET /api/v1/prices/live` returns 200 with correct schema; assert 503 schema when feed unavailable

### Frontend — Dashboard Page

- [ ] T049 Create `frontend/src/api/pricesApi.ts` — Axios client for `GET /api/v1/prices/live`; SSE hook `useLivePriceStream` using `EventSource`
- [ ] T050 [P] Create `frontend/src/components/dashboard/LivePriceCard.tsx` — displays symbol, price, change amount, change %, currency; Tailwind styled; green/red color for change direction
- [ ] T051 [P] Create `frontend/src/components/dashboard/PriceChangeBadge.tsx` — inline badge showing +/− percentage; accessible color contrast
- [ ] T052 [P] Create `frontend/src/components/dashboard/DataStaleBanner.tsx` — shows warning bar when `stale: true` with last-known timestamp
- [ ] T053 Create `frontend/src/pages/Dashboard.tsx` — compose `LivePriceCard` × 2 (XAU/USD, XAU/VND); subscribe to SSE via `useLivePriceStream`; render `DataStaleBanner` conditionally
- [ ] T054 [P] Write Vitest unit test `frontend/tests/LivePriceCard.test.tsx` — render with mock data; assert price and badge text; assert `DataStaleBanner` renders when `stale=true`

**Checkpoint**: US1 complete — live price dashboard fully functional, accessible without login

---

## Phase 4: US2 — Historical Charts with Technical Indicators (Priority: P2)

**Goal**: Interactive OHLCV chart for any supported range with all five indicators togglable.

**Independent Test**: Navigate to Charts page → select 3M range → enable all 5 indicators simultaneously → verify all render without error → hover data point → tooltip shows price + all indicator values.

### Backend — History & Indicators

- [ ] T055 Create `domain/service/TechnicalIndicatorCalculator.java` — pure domain service implementing SMA, EMA, RSI, MACD (12,26,9), Bollinger Bands (configurable period/stddev); takes `List<Double>` closes; returns typed result objects; fully unit-testable with no Spring context
- [ ] T056 Write unit tests `backend/src/test/unit/TechnicalIndicatorCalculatorTest.java` — assert SMA/EMA values against hand-calculated expected results for a 20-point series; assert RSI bounds (0–100); assert Bollinger upper > middle > lower
- [ ] T057 Create `application/chart/GetPriceHistoryService.java` implementing `GetPriceHistoryUseCase` — query `GoldPriceRepository.findBySymbolBetween` using range enum to derive start date; map to OHLCV candles (aggregate snapshots into appropriate resolution per range); check chart cache first; **set `dataComplete: false` in the response when the oldest available data point is newer than the requested range start (e.g. only 3 months of data exists but 5Y was requested)** — satisfies edge case in spec.md and `dataComplete` field in `contracts/charts.yaml` (G5 fix)
- [ ] T058 Create `application/chart/GetIndicatorsService.java` implementing `GetIndicatorsUseCase` — load close prices for range; invoke `TechnicalIndicatorCalculator`; cache result with composite key per research.md; return `Map<IndicatorType, List<DataPoint>>`
- [ ] T059 Create `presentation/rest/ChartController.java` — `GET /api/v1/charts/history` and `GET /api/v1/charts/indicators` per `contracts/charts.yaml`; validate `range` enum; return 400 on invalid params

### Integration & Contract Tests

- [ ] T060 [P] Write Testcontainers test `ChartDataIntegrationTest.java` — seed 90 days of gold_prices; assert history endpoint returns correct candle count for 1M range
- [ ] T061 [P] Write contract test `ChartsContractTest.java` — assert history and indicators response schemas match `contracts/charts.yaml`

### Frontend — Chart Page

- [ ] T062 Create `frontend/src/api/chartsApi.ts` — React Query hooks: `usePriceHistory(symbol, range)`, `useIndicators(symbol, range, types, params)`
- [ ] T063 Create `frontend/src/components/chart/PriceChart.tsx` — Lightweight Charts `createChart`; renders OHLCV candlestick series; handles zoom/pan; tooltip via `subscribeCrosshairMove`
- [ ] T064 [P] Create `frontend/src/components/chart/IndicatorOverlay.tsx` — overlays SMA/EMA line series on main chart; renders RSI, MACD, Bollinger in Recharts sub-panels below main chart
- [ ] T065 [P] Create `frontend/src/components/chart/ChartToolbar.tsx` — range selector buttons (1D/1W/1M/3M/1Y/5Y); indicator toggle checkboxes with period inputs (SMA period, EMA period, BB period/stddev)
- [ ] T066 Create `frontend/src/pages/Charts.tsx` — compose `PriceChart`, `ChartToolbar`, `IndicatorOverlay`; manage selected range and indicator state with `useState`; pass to React Query hooks
- [ ] T067 [P] Write Vitest tests `frontend/tests/ChartToolbar.test.tsx` — assert range buttons change active state; assert indicator checkbox toggles fire correct callbacks

**Checkpoint**: US2 complete — all 5 indicators render on any time range; charts fully interactive

---

## Phase 5: US3 — AI Market Analysis Chatbot (Priority: P2)

**Goal**: Conversational AI grounded in platform price data; streaming responses; session history.

**Independent Test**: Open chat panel → ask "What has the gold price done over the last 30 days?" → response streams in, references actual stored price stats → ask follow-up "Is that bullish?" → response is contextually aware of previous turn.

### Backend — LLM Integration & Chat

- [ ] T068 Create `infrastructure/llm/SpringAiLlmAdapter.java` implementing `LlmPort` — use `ChatClient` (Spring AI); inject `OPENAI_API_KEY` and model via env; call `stream()` returning `Flux<String>`; timeout at 12 s; on timeout emit fallback message rather than error
- [ ] T069 Create `infrastructure/cache/ChatSessionAdapter.java` — store/retrieve `List<ChatMessage>` in Redis key `chat:{sessionId}` with 30 min TTL; serialize with Jackson; enforce max 10-turn sliding window
- [ ] T070 Create resource file `backend/src/main/resources/prompts/market-analysis.st` — Spring AI StringTemplate; system prompt instructs: "You are a gold market analyst. Use ONLY the data provided in the context block to answer. If data is insufficient, state so clearly. Do not invent prices or dates."; includes `{context}` placeholder for injected price data
- [ ] T071 Create `application/chat/AskChatbotService.java` implementing `AskChatbotUseCase` — (1) retrieve session history from Redis; (2) query `GoldPriceRepository` for last 90-day stats (min, max, avg, current, 7/30/90-day change); (3) format as structured context block; (4) render `market-analysis.st` with context; (5) call `LlmPort.chat`; (6) persist user + assistant messages to Redis; (7) return `Flux<String>` for SSE streaming
- [ ] T072 Create `presentation/rest/ChatController.java` — **`POST /api/v1/chat/message`** returns `text/event-stream` (SSE) delegating to `AskChatbotUseCase`; **`GET /api/v1/chat/history?sessionId=`** returns `ChatHistoryResponse` by reading the session list from Redis via `ChatSessionAdapter` — both per `contracts/chat.yaml`; rate-limit 20 req/min per session via Redis counter; reject empty questions with 400

### Integration & Contract Tests

- [ ] T073 [P] Write `ChatbotIntegrationTest.java` — Testcontainers (Postgres + Redis); WireMock stub returning streamed tokens; assert response contains seeded price data reference; assert session history persists after first turn; **fire 5 varied question types (trend, current price, historical comparison, out-of-scope, follow-up) and assert the injected context block is non-empty in all gold-market queries; assert total response time for each request is < 12 s; record median and assert < 5 s** (G2+G3 fix)
- [ ] T074 [P] Write contract test `ChatContractTest.java` — assert POST schema, SSE content-type header, and history GET schema match `contracts/chat.yaml`

### Frontend — Chat Panel

- [ ] T075 Create `frontend/src/api/chatApi.ts` — `sendMessage(sessionId, question)` via `fetch` with SSE (`ReadableStream`); `getChatHistory(sessionId)` via Axios; generate and persist `sessionId` UUID to `localStorage` on first use
- [ ] T076 Create `frontend/src/components/chatbot/MessageBubble.tsx` — renders USER (right-aligned) and ASSISTANT (left-aligned) messages; ASSISTANT messages support markdown rendering; show `dataGrounded` indicator badge
- [ ] T077 [P] Create `frontend/src/components/chatbot/ChatWindow.tsx` — scrollable message list using `MessageBubble`; auto-scrolls to bottom on new message; shows typing indicator during streaming
- [ ] T078 [P] Create `frontend/src/components/chatbot/InputBar.tsx` — text input + send button; Enter-to-send; disable during in-flight request; 1000-char max with counter
- [ ] T079 Create `frontend/src/pages/Chat.tsx` — compose `ChatWindow` + `InputBar`; stream tokens from SSE and append to assistant message progressively; handle stream completion and error states
- [ ] T080 [P] Write Vitest test `frontend/tests/MessageBubble.test.tsx` — assert USER vs ASSISTANT alignment; assert `dataGrounded` badge renders when flag is true

**Checkpoint**: US3 complete — chatbot returns data-grounded streaming responses; follow-up questions work

---

## Phase 6: US4 — Google OAuth Authentication & Watchlist (Priority: P3)

**Goal**: Optional sign-in with Google; authenticated users can save/restore a watchlist.

**Independent Test**: Sign in with Google → add XAU/USD and XAU/VND to watchlist → sign out → sign back in → both watchlist items restored. Verify that all Phase 3/4/5 features still work without signing in.

### Backend — Auth Completion & Watchlist

- [ ] T081 Verify OAuth2 callback flow end-to-end: `GET /oauth2/authorization/google` → Google → `/login/oauth2/code/google` → `OAuth2UserServiceImpl` upserts user → JWT issued → redirect to SPA `/#token=<jwt>`; add integration test `OAuth2FlowIntegrationTest.java` using WireMock to simulate Google token endpoint
- [ ] T081b Create `presentation/rest/AuthController.java` — **`GET /api/v1/auth/me`** returns the authenticated `UserProfile` from `SecurityContext` (401 if unauthenticated); **`POST /api/v1/auth/logout`** returns 204 (stateless — client discards JWT) — both per `contracts/auth.yaml`; consumed by frontend T087 (C-001 fix)
- [ ] T082 Create `application/watchlist/ManageWatchlistService.java` implementing `ManageWatchlistUseCase` — `add` checks duplicate via `WatchlistRepository.existsByUserIdAndSymbol`, throws `DuplicateWatchlistItemException` on conflict; `remove` checks ownership; `list` returns user's items with current live price attached
- [ ] T083 Create `presentation/rest/WatchlistController.java` — `GET/POST /api/v1/watchlist`, `DELETE /api/v1/watchlist/{symbol}` per `contracts/watchlist.yaml`; extract `userId` from `SecurityContext`; map `DuplicateWatchlistItemException` → 409

### Integration & Contract Tests

- [ ] T084 [P] Write `WatchlistIntegrationTest.java` — Testcontainers; assert add/list/remove cycle; assert duplicate returns 409; assert sign-out → sign-in restores list
- [ ] T085 [P] Write contract test `WatchlistContractTest.java` — assert all schemas per `contracts/watchlist.yaml`; assert 401 on unauthenticated requests

### Frontend — Auth Flow & Watchlist Panel

- [ ] T086 Create `frontend/src/store/authStore.ts` — Zustand slice: `{ user, token, setAuth, clearAuth }`; `setAuth` writes token to memory (not localStorage); on app init, check URL fragment for `#token=` and call `setAuth`
- [ ] T087 [P] Create `frontend/src/api/authApi.ts` — `getCurrentUser()` → `GET /api/v1/auth/me`; `logout()` → `POST /api/v1/auth/logout`; `initiateGoogleLogin()` → redirect to `/oauth2/authorization/google`
- [ ] T088 [P] Create `frontend/src/components/watchlist/WatchlistPanel.tsx` — lists watchlist items with current price (from live price cache); "Add" button opens symbol input; remove button per item; empty state with sign-in prompt
- [ ] T089 Create `frontend/src/api/watchlistApi.ts` — React Query hooks: `useWatchlist`, `useAddToWatchlist` (mutation), `useRemoveFromWatchlist` (mutation); attach JWT from `authStore` in Axios interceptor
- [ ] T090 [P] Create login/logout button — **depends on T114 (`AppHeader.tsx` skeleton)**; add sign-in/avatar/logout controls to the header component created in Phase 9; this task may be implemented as a placeholder stub in Phase 6 and completed in Phase 9 once T114 exists (D1 fix)
- [ ] T091 [P] Write Vitest test `frontend/tests/WatchlistPanel.test.tsx` — mock authenticated state; assert items render; assert add mutation fires; assert unauthenticated state shows sign-in prompt

**Checkpoint**: US4 complete — Google OAuth works end-to-end; watchlist persists across sessions; unauthenticated access to all prior features unaffected

---

## Phase 7: US5 — Portfolio Tracking (Priority: P3)

**Goal**: Authenticated users record gold holdings and see live gain/loss calculations.

**Independent Test**: Sign in → add holding (1 oz XAU/USD @ $2200, purchased 2025-01-15) → portfolio shows current market value, gain/loss amount and % → edit quantity → values recalculate → delete holding → portfolio summary updates.

### Backend — Portfolio

- [ ] T092 Create `application/portfolio/ManageHoldingsService.java` implementing `ManageHoldingsUseCase` — `add` validates purchaseDate not in future; `update` checks ownership; on `list`, enrich each holding with current live price from cache to compute `currentValue`, `gainLossAmount`, `gainLossPercent`; aggregate totals
- [ ] T093 Create `presentation/rest/PortfolioController.java` — `GET/POST /api/v1/portfolio/holdings`, `PUT/DELETE /api/v1/portfolio/holdings/{id}` per `contracts/portfolio.yaml`; extract `userId` from `SecurityContext`

### Integration & Contract Tests

- [ ] T094 [P] Write `PortfolioIntegrationTest.java` — seed a holding; assert list response includes computed gain/loss; assert ownership enforcement (user A cannot delete user B's holding)
- [ ] T095 [P] Write contract test `PortfolioContractTest.java` — assert all schemas per `contracts/portfolio.yaml`

### Frontend — Portfolio Page

- [ ] T096 Create `frontend/src/api/portfolioApi.ts` — React Query hooks: `useHoldings`, `useAddHolding`, `useUpdateHolding`, `useDeleteHolding`
- [ ] T097 Create `frontend/src/components/portfolio/HoldingRow.tsx` — table row showing symbol, quantity, purchase price, current price, gain/loss with color coding
- [ ] T098 [P] Create `frontend/src/components/portfolio/PortfolioSummary.tsx` — header card with total current value, total gain/loss amount and percentage
- [ ] T099 Create `frontend/src/pages/Portfolio.tsx` — compose `PortfolioSummary` + holding list + add/edit form (modal or inline); form validates quantity > 0, price > 0, date not in future
- [ ] T100 [P] Write Vitest test `frontend/tests/HoldingRow.test.tsx` — assert gain renders green, loss renders red; assert values are correctly formatted

**Checkpoint**: US5 complete — portfolio tracking fully functional; gain/loss calculated from live prices

---

## Phase 8: US6 — Price Alerts & In-App Notifications (Priority: P4)

**Goal**: Authenticated users set price threshold alerts; in-app notification fires within 2 min of threshold crossing.

**Independent Test**: Sign in → create ABOVE alert for XAU/USD at $99 999 → seed a price snapshot above threshold → verify notification appears in bell icon within 2 min → mark notification as read → alert status shows TRIGGERED.

### Backend — Alerts & Notifications

- [ ] T101 Create `application/alert/ManageAlertsService.java` implementing `ManageAlertsUseCase` — `create` validates direction enum; `cancel` checks ownership; `list` filters by status param
- [ ] T102 Create `infrastructure/scheduler/AlertCheckScheduler.java` — `@Scheduled(fixedDelay = 30_000)`; acquire Redis distributed lock `alert:check:lock` (90 s TTL) to prevent duplicate checks; load all `ACTIVE` alerts grouped by symbol; **detect threshold crossing using gap-aware logic: compare the previous cached price against the current cached price (store previous price in Redis key `price:prev:{symbol}` each poll cycle). An ABOVE alert fires when `current >= threshold` even if the price gapped past it (e.g. jumped from 2000 to 2500 over a 2300 threshold); a BELOW alert fires when `current <= threshold`** — satisfies the gap-crossing edge case in spec.md; for each triggered alert: update status to `TRIGGERED`, record `triggeredAt` and `triggerPrice`, create `Notification` record (`is_read = false`); release lock (C-005 fix)
- [ ] T103 Create `application/alert/GetNotificationsService.java` implementing `GetNotificationsUseCase` — `list(userId, unreadOnly)`; `markRead(userId, notificationId)` checks ownership
- [ ] T104 Create `presentation/rest/AlertController.java` — `GET/POST /api/v1/alerts`, `DELETE /api/v1/alerts/{id}`, **`PATCH /api/v1/alerts/{id}/cancel`** (sets status to `CANCELLED`, returns updated `PriceAlertResponse`; returns 409 if already TRIGGERED or CANCELLED), `GET /api/v1/notifications`, `PATCH /api/v1/notifications/{id}/read` per `contracts/alerts.yaml` (C1+I3 fix)

### Integration & Contract Tests

- [ ] T105 [P] Write `AlertCheckIntegrationTest.java` — Testcontainers; seed ACTIVE alert; seed price above threshold; run `AlertCheckScheduler.checkAlerts()` directly; assert alert status = TRIGGERED; assert Notification created; assert idempotent (second run does not double-trigger)
- [ ] T106 [P] Write contract test `AlertsContractTest.java` — assert all alert and notification schemas per `contracts/alerts.yaml`

### Frontend — Alerts & Notification Bell

- [ ] T107 Create `frontend/src/api/alertsApi.ts` — React Query hooks: `useAlerts`, `useCreateAlert`, `useDeleteAlert`, `useNotifications`, `useMarkNotificationRead`
- [ ] T108 Create `frontend/src/components/alerts/AlertForm.tsx` — form: symbol selector, threshold price input, direction toggle (ABOVE/BELOW), submit; client-side validation
- [ ] T109 [P] Create `frontend/src/components/alerts/AlertList.tsx` — list of alerts with status badge (ACTIVE green, TRIGGERED amber, CANCELLED grey); delete button for active alerts
- [ ] T110 [P] Create `frontend/src/components/layout/NotificationBell.tsx` — badge with unread count; dropdown of recent notifications; click marks as read; polls `GET /api/v1/notifications?unreadOnly=true` every 60 s when authenticated; **depends on T114 (`AppHeader.tsx` skeleton) — compose into header there** (D1 fix)
- [ ] T111 Create `frontend/src/pages/Alerts.tsx` — compose `AlertForm` + `AlertList`
- [ ] T112 [P] Write Vitest test `frontend/tests/AlertForm.test.tsx` — assert ABOVE/BELOW toggle; assert submit mutation fires with correct payload; assert validation blocks empty price

**Checkpoint**: US6 complete — alerts fire and in-app notifications appear within 2 min of threshold crossing

---

## Phase 9: Navigation, Routing & App Shell

**Purpose**: Wire all pages together with React Router; app shell, header, sidebar/nav.

- [ ] T113 Create `frontend/src/App.tsx` — `BrowserRouter`; define routes: `/` → Dashboard, `/charts` → Charts, `/chat` → Chat, `/portfolio` → Portfolio (require auth redirect), `/alerts` → Alerts (require auth redirect); `ProtectedRoute` wrapper component
- [ ] T114 [P] Create `frontend/src/components/layout/AppHeader.tsx` — Aureon logo/brand, nav links, auth button (sign in / avatar+logout), notification bell; responsive collapse to hamburger on mobile
- [ ] T115 [P] Create `frontend/src/components/layout/ProtectedRoute.tsx` — checks `authStore.token`; if absent redirects to Dashboard with `?authRequired=true` query param which triggers sign-in prompt
- [ ] T116 [P] Configure React Query `QueryClient` in `frontend/src/main.tsx` with default `staleTime: 30_000` and `retry: 1`; configure Axios interceptor to inject `Authorization: Bearer` from `authStore`

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: Hardening, performance, and production-readiness across all user stories.

- [ ] T117 Add `spring-boot-starter-actuator` to backend; expose `/api/v1/health` and `/api/v1/info`; configure `management.endpoints.web.exposure.include=health,info` only
- [ ] T118 [P] Add structured logging to all Spring services using SLF4J MDC: include `userId` (if available), `symbol`, and `requestId` in every log line; ensure no PII or tokens are logged
- [ ] T119 [P] Add HTTP security response headers to Nginx config: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security`, `Content-Security-Policy` (restrictive; allow `connect-src` for SSE endpoints)
- [ ] T120 [P] Add HikariCP configuration to `application.yml`: `maximum-pool-size: 10`, `minimum-idle: 2`, `connection-timeout: 3000`, `idle-timeout: 300000`; add Redis pool config
- [ ] T121 [P] Add global React `ErrorBoundary` component in `frontend/src/components/layout/ErrorBoundary.tsx`; wrap each page route; display user-friendly error card instead of blank screen
- [ ] T122 [P] Add `frontend/src/utils/formatters.ts` — currency formatter (`Intl.NumberFormat`), date formatter, percentage formatter; used consistently across all components
- [ ] T123 [P] Ensure all interactive frontend components have `aria-label` or `aria-live` attributes; verify keyboard navigation for form inputs and chart toolbar
- [ ] T124 Run `quickstart.md` validation checklist: `docker compose up --build` from clean state; verify all 6 user stories against acceptance scenarios; confirm JWT rotation procedure works by changing `JWT_SECRET` and restarting backend
- [ ] T125 [P] Write end-to-end smoke test script `infra/smoke-test.sh` — curl-based: assert `/api/v1/prices/live` returns 200, `/api/v1/charts/history?range=1M` returns non-empty candles, POST to `/api/v1/chat/message` returns SSE stream
- [ ] T126 [P] Write cache performance assertion test `backend/src/test/integration/ChartPerformanceTest.java` — Testcontainers; pre-warm Redis chart cache for `XAU/USD` 5Y range; assert `GET /api/v1/charts/history?symbol=XAU/USD&range=5Y` responds within 3 000 ms (REST-assured `response.time()`); assert `dataComplete` field is present in response body; validates SC-002 (G1 fix)
- [ ] T127 [P] Add cancel alert button to `frontend/src/components/alerts/AlertList.tsx` — alongside existing delete button for ACTIVE alerts; calls `PATCH /api/v1/alerts/{id}/cancel`; on success updates alert status badge to CANCELLED (grey) without removing the row from history; write Vitest test asserting cancel mutation fires and row status updates (C1 frontend fix)

---

## Dependencies & Execution Order

### Phase Dependencies

```
Phase 1 (Setup)
  └── Phase 2 (Foundation)  ← BLOCKS all user stories
        ├── Phase 3 (US1 – Live Prices)
        ├── Phase 4 (US2 – Charts)        ← can start in parallel with Phase 3
        ├── Phase 5 (US3 – Chatbot)       ← can start in parallel with Phase 3/4
        ├── Phase 6 (US4 – Auth/Watchlist)
        ├── Phase 7 (US5 – Portfolio)     ← requires Phase 6 (auth)
        └── Phase 8 (US6 – Alerts)        ← requires Phase 6 (auth)
              └── Phase 9 (Routing)       ← requires all pages to exist
                    └── Phase 10 (Polish)
```

### User Story Dependencies

| Story | Depends on | Can parallelize with |
|-------|-----------|---------------------|
| US1 (Live Prices) | Phase 2 | US2, US3 |
| US2 (Charts) | Phase 2 | US1, US3 |
| US3 (Chatbot) | Phase 2 | US1, US2 |
| US4 (Auth/Watchlist) | Phase 2 | US1, US2, US3 | T090 depends on T114 |
| US5 (Portfolio) | US4 (auth infrastructure) | US6 |
| US6 (Alerts) | US4 (auth infrastructure) | US5 | T110, T127 depend on T114 |

### Within Each User Story

Tasks marked `[P]` within the same phase can run in parallel. Backend and frontend tasks for the same story are independent once the API contract is agreed — backend and frontend teams can work simultaneously using MSW mocks on the frontend.

---

## Implementation Strategy

**Recommended sequence for a solo developer**:

1. **Phase 1 + 2** (foundation): ~2 days — all infrastructure, migrations, domain model, security skeleton
2. **Phase 3** (US1, P1): ~1 day — fastest path to a working demo (live price ticker)
3. **Phase 4** (US2, P2): ~2 days — charts unlock analytical value
4. **Phase 5** (US3, P2): ~2 days — AI chatbot is the key differentiator
5. **Phase 6** (US4, P3): ~1.5 days — OAuth + watchlist; unlocks US5 and US6
6. **Phase 7 + 8** (US5/US6, P3/P4): ~2 days — portfolio and alerts
7. **Phase 9 + 10** (routing + polish): ~1 day — wire everything together

**For a team (parallel tracks)**:
- Backend engineer: Phases 1 → 2 → US1 backend → US2 backend → US3 backend
- Frontend engineer: Phase 1 setup → US1 frontend (using MSW mocks) → US2 frontend → US3 frontend
- Merge tracks at Phase 6 (auth requires full-stack coordination)

**Total task count**: 129 tasks across 10 phases (125 original + T011-MSW, T081b, T126, T127)
