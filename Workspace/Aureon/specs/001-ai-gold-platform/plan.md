# Implementation Plan: AI-Powered Gold Market Intelligence Platform

**Branch**: `001-ai-gold-platform` | **Date**: 2026-06-20 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-ai-gold-platform/spec.md`

---

## Summary

Build a full-stack gold market intelligence platform that delivers real-time XAU/USD and local gold price tracking, interactive historical charts with five technical indicators, and an AI chatbot grounded in platform data. Optional Google OAuth authentication unlocks persistent watchlists, portfolio tracking, and price alerts. The system is implemented as a Java 21 / Spring Boot backend with a React SPA frontend, backed by PostgreSQL and Redis, and deployed via Docker Compose following clean architecture principles.

---

## Technical Context

**Language/Version**: Java 21 (backend), TypeScript / Node 20 LTS (frontend)

**Primary Dependencies**:
- Backend: Spring Boot 3.3, Spring Security (OAuth2), Spring Data JPA, Spring WebFlux (SSE for live prices), Spring AI (LLM integration), Flyway (DB migrations), Lombok, MapStruct
- Frontend: React 18, Vite 5, Recharts / Lightweight Charts (TradingView), Axios, React Query (TanStack), Zustand, Tailwind CSS
- Infrastructure: PostgreSQL 16, Redis 7, Docker Compose

**Storage**:
- PostgreSQL 16 — persistent data (users, watchlists, portfolios, alerts, price history)
- Redis 7 — live price cache (TTL 55 s), rate-limit counters, chat session state

**Testing**:
- Backend: JUnit 5, Mockito, Testcontainers (PostgreSQL + Redis), Spring Boot Test, REST-assured
- Frontend: Vitest, React Testing Library, MSW (API mocking)

**Target Platform**: Linux server (Docker Compose, single-host)

**Project Type**: Web service (REST API backend) + Single-Page Application (frontend)

**Performance Goals**:
- Live price refresh latency ≤ 60 s end-to-end (FR-001)
- Chart data API response ≤ 3 s for any supported range
- AI chatbot response ≤ 10 s p95 (FR-013)

**Constraints**:
- No mobile native apps; responsive web only
- Authentication is optional — all read-only features must work unauthenticated
- Single-host Docker Compose deployment (no Kubernetes)
- All secrets injected via environment variables; no hardcoded credentials

**Scale/Scope**: Single-host deployment, estimated < 1 000 concurrent users initially; architecture supports horizontal scaling of the backend service later

---

## Constitution Check

*Constitution file is a template (not yet filled). Applying general clean-architecture and security gates.*

| Gate | Status | Notes |
|------|--------|-------|
| Clean layer separation (domain / application / infrastructure / presentation) | PASS | Enforced by package structure; domain has zero framework imports |
| No secrets in source code | PASS | All credentials via env vars / Docker secrets |
| All public endpoints documented in contracts/ | PASS | OpenAPI contract files produced in Phase 1 |
| Authentication optional (unauthenticated access to read-only features) | PASS | Security filter chain permits GET /api/prices/**, /api/charts/**, /api/chat/** without auth |
| Testable without external services | PASS | Testcontainers for DB/Redis; WireMock for price API and LLM API |
| No implementation details leak into spec | PASS | spec.md is technology-agnostic |

*Post-Phase-1 re-check*: All gates pass after design. No violations requiring justification.

---

## Project Structure

### Documentation (this feature)

```text
specs/001-ai-gold-platform/
├── plan.md              # This file
├── research.md          # Phase 0 — technology decisions and risk analysis
├── data-model.md        # Phase 1 — entity schema and relationships
├── quickstart.md        # Phase 1 — local dev setup guide
├── contracts/           # Phase 1 — OpenAPI YAML contracts per domain
│   ├── prices.yaml
│   ├── charts.yaml
│   ├── chat.yaml
│   ├── auth.yaml
│   ├── watchlist.yaml
│   ├── portfolio.yaml
│   └── alerts.yaml
└── tasks.md             # Phase 2 — created by /speckit.tasks
```

### Source Code (repository root)

```text
backend/                                   # Spring Boot application
├── src/
│   ├── main/
│   │   ├── java/com/aureon/
│   │   │   ├── domain/                    # Pure domain — entities, value objects, ports
│   │   │   │   ├── model/                 # GoldPrice, User, Holding, WatchlistItem, PriceAlert, ChatMessage
│   │   │   │   ├── port/
│   │   │   │   │   ├── in/                # Use-case interfaces (e.g., GetLivePrice, AskChatbot)
│   │   │   │   │   └── out/               # Repository / external service ports
│   │   │   │   └── service/               # Domain services (e.g., TechnicalIndicatorCalculator)
│   │   │   ├── application/               # Use-case implementations (orchestration only)
│   │   │   │   ├── price/
│   │   │   │   ├── chart/
│   │   │   │   ├── chat/
│   │   │   │   ├── watchlist/
│   │   │   │   ├── portfolio/
│   │   │   │   └── alert/
│   │   │   ├── infrastructure/            # Framework-specific adapters
│   │   │   │   ├── persistence/           # Spring Data JPA repositories + JPA entities
│   │   │   │   ├── cache/                 # Redis adapter (Lettuce)
│   │   │   │   ├── pricefeed/             # External price API client (WebClient)
│   │   │   │   ├── llm/                   # Spring AI adapter (LLM API calls)
│   │   │   │   ├── security/              # Spring Security config, OAuth2 user service, JWT filter
│   │   │   │   └── scheduler/             # Price polling scheduler
│   │   │   └── presentation/              # Spring MVC / WebFlux controllers + DTOs
│   │   │       ├── rest/
│   │   │       └── sse/                   # Server-Sent Events for live price push
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       └── db/migration/              # Flyway SQL migrations (V1__init.sql, V2__..., …)
│   └── test/
│       ├── unit/                          # Pure unit tests (no Spring context)
│       ├── integration/                   # Testcontainers-backed integration tests
│       └── contract/                      # REST-assured contract tests
├── build.gradle (or pom.xml)
└── Dockerfile

frontend/                                  # React SPA
├── src/
│   ├── api/                               # Axios clients + React Query hooks per domain
│   ├── components/
│   │   ├── chart/                         # PriceChart, IndicatorOverlay, ChartToolbar
│   │   ├── chatbot/                       # ChatWindow, MessageBubble, InputBar
│   │   ├── dashboard/                     # LivePriceCard, PriceChangeBadge, DataStaleBanner
│   │   ├── portfolio/                     # HoldingRow, PortfolioSummary
│   │   ├── watchlist/                     # WatchlistPanel, WatchlistItem
│   │   └── alerts/                        # AlertForm, AlertList
│   ├── pages/                             # Dashboard, Charts, Chat, Portfolio, Alerts, Login
│   ├── store/                             # Zustand slices (auth, ui)
│   └── utils/                             # indicator math helpers, formatters
├── tests/
├── index.html
├── vite.config.ts
└── Dockerfile

infra/                                     # Container orchestration
├── docker-compose.yml                     # Full stack: backend + frontend + postgres + redis
├── docker-compose.dev.yml                 # Dev override (hot reload, exposed ports)
├── .env.example                           # Template for required environment variables
├── nginx/
│   └── nginx.conf                         # Reverse proxy: /api → backend, / → frontend SPA
└── postgres/
    └── init/                              # Any seed SQL (optional)
```

**Structure Decision**: Option 2 (web application) with explicit `infra/` layer for Docker Compose and Nginx. `backend/` follows hexagonal (ports & adapters) clean architecture. `frontend/` is feature-slice organized by domain capability, not by technical type.

---

## Complexity Tracking

| Design choice | Why Needed | Simpler Alternative Rejected Because |
|---------------|------------|--------------------------------------|
| Hexagonal architecture (ports & adapters) | Enables swapping price feed, LLM provider, and DB without touching domain logic | Layered N-tier: couples use cases to Spring/JPA, making provider swaps expensive |
| Redis cache layer | Price feed API has rate limits; 55 s TTL prevents excessive outbound calls and gives sub-second dashboard loads | Direct DB reads for live prices: DB not suited for sub-minute write/read churn; no TTL semantics |
| SSE endpoint for live prices | Pushes updates to browser without polling overhead | WebSocket: heavier protocol; SSE sufficient for unidirectional server→client price stream |
| Spring AI abstraction | Decouples LLM provider (OpenAI, Anthropic, Ollama) via a single port; swappable without code changes | Direct OpenAI SDK: vendor lock-in; harder to test with local/mock LLM |
| Flyway migrations | Schema versioning ensures reproducible environments across dev/test/prod containers | Hibernate `ddl-auto=update`: unsafe in production; no rollback or audit trail |
