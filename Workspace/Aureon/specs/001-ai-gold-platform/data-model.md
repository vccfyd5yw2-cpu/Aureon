# Data Model: AI-Powered Gold Market Intelligence Platform

**Phase**: 1 | **Date**: 2026-06-20 | **Plan**: [plan.md](plan.md)

---

## Entity Relationship Overview

```
users
  ├── watchlist_items   (1:N)
  ├── holdings          (1:N)
  └── price_alerts      (1:N)

gold_prices             (time-series, no FK — standalone)
notifications           (FK → users)

# Chat history is NOT a database table — it is stored in Redis only
# (key chat:{sessionId}, 30 min TTL). See ChatHistory entity in spec.md.
```

---

## Table Definitions

### `users`

Stores authenticated users created on first Google OAuth sign-in.

```sql
CREATE TABLE users (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    google_sub    VARCHAR(255) NOT NULL UNIQUE,   -- Google OAuth subject identifier
    email         VARCHAR(320) NOT NULL UNIQUE,
    display_name  VARCHAR(255) NOT NULL,
    avatar_url    VARCHAR(1024),
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_google_sub ON users(google_sub);
```

---

### `gold_prices`

Append-only time-series of gold price snapshots collected by the polling scheduler.

```sql
CREATE TABLE gold_prices (
    id          BIGSERIAL    PRIMARY KEY,
    symbol      VARCHAR(16)  NOT NULL,            -- e.g. 'XAU/USD', 'XAU/VND'
    price       NUMERIC(18,6) NOT NULL,
    currency    VARCHAR(8)   NOT NULL,
    source      VARCHAR(64)  NOT NULL,             -- e.g. 'goldapi.io'
    recorded_at TIMESTAMPTZ  NOT NULL DEFAULT now()
);

-- Composite index for time-range queries per symbol
CREATE INDEX idx_gold_prices_symbol_time ON gold_prices(symbol, recorded_at DESC);

-- Unique constraint: prevent duplicate snapshots within the same second
CREATE UNIQUE INDEX idx_gold_prices_dedup ON gold_prices(symbol, date_trunc('second', recorded_at));
```

**Retention**: No hard limit in v1; table expected < 500k rows/year at 55 s polling frequency for 2 symbols.

---

### `watchlist_items`

Instruments a user has bookmarked for quick access.

```sql
CREATE TABLE watchlist_items (
    id          UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    symbol      VARCHAR(16)  NOT NULL,             -- e.g. 'XAU/USD'
    label       VARCHAR(128),                      -- optional user-defined display name
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT now(),

    CONSTRAINT uq_watchlist_user_symbol UNIQUE (user_id, symbol)
);

CREATE INDEX idx_watchlist_user ON watchlist_items(user_id);
```

---

### `holdings`

User's gold portfolio entries. Each row is one lot purchase.

```sql
CREATE TABLE holdings (
    id               UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id          UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    symbol           VARCHAR(16)  NOT NULL,
    quantity         NUMERIC(18,6) NOT NULL CHECK (quantity > 0),
    purchase_price   NUMERIC(18,6) NOT NULL CHECK (purchase_price > 0),
    purchase_currency VARCHAR(8)  NOT NULL,         -- e.g. 'USD', 'VND'
    purchase_date    DATE         NOT NULL,
    notes            TEXT,
    created_at       TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE INDEX idx_holdings_user ON holdings(user_id);
```

---

### `price_alerts`

User-defined price threshold triggers.

```sql
CREATE TYPE alert_direction AS ENUM ('ABOVE', 'BELOW');
CREATE TYPE alert_status    AS ENUM ('ACTIVE', 'TRIGGERED', 'CANCELLED');

CREATE TABLE price_alerts (
    id                UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id           UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    symbol            VARCHAR(16)     NOT NULL,
    threshold_price   NUMERIC(18,6)   NOT NULL,
    currency          VARCHAR(8)      NOT NULL,
    direction         alert_direction NOT NULL,
    status            alert_status    NOT NULL DEFAULT 'ACTIVE',
    created_at        TIMESTAMPTZ     NOT NULL DEFAULT now(),
    triggered_at      TIMESTAMPTZ,
    trigger_price     NUMERIC(18,6)
);

CREATE INDEX idx_alerts_user        ON price_alerts(user_id);
CREATE INDEX idx_alerts_active_sym  ON price_alerts(symbol, status) WHERE status = 'ACTIVE';
```

---

### `notifications` (in-app)

Records triggered alert notifications for display in the UI.

```sql
CREATE TYPE notification_type AS ENUM ('PRICE_ALERT');

CREATE TABLE notifications (
    id            UUID              PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id       UUID              NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type          notification_type NOT NULL,
    reference_id  UUID,                             -- FK to price_alerts.id
    title         VARCHAR(255)      NOT NULL,
    body          TEXT              NOT NULL,
    is_read       BOOLEAN           NOT NULL DEFAULT false,
    created_at    TIMESTAMPTZ       NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_user_unread ON notifications(user_id, is_read) WHERE is_read = false;
```

---

## Domain Model (Java)

### Core Domain Classes

```
com.aureon.domain.model
├── GoldPrice           record — symbol, price, currency, source, recordedAt
├── User                record — id (UUID), googleSub, email, displayName, avatarUrl
├── WatchlistItem       record — id, userId, symbol, label, createdAt
├── Holding             record — id, userId, symbol, quantity, purchasePrice, purchaseCurrency, purchaseDate
├── PriceAlert          record — id, userId, symbol, thresholdPrice, currency, direction, status, triggeredAt
├── Notification        record — id, userId, type, referenceId, title, body, read, createdAt
└── ChatMessage         record — role (USER/ASSISTANT), content, timestamp
```

**Note**: Domain model uses Java records (immutable). Persistence uses separate JPA `@Entity` classes in `infrastructure/persistence/` — mapped via MapStruct. The `Notification` domain field `read` maps to DB column `is_read` via `@Column(name = "is_read")`; the JSON API field remains `read` (per `contracts/alerts.yaml`).

---

## Domain Ports (Interfaces)

### Inbound (use-case interfaces)
```
com.aureon.domain.port.in
├── GetLivePriceUseCase          — getLivePrice(symbol): GoldPrice
├── GetPriceHistoryUseCase       — getHistory(symbol, range): List<GoldPrice>
├── GetIndicatorsUseCase         — getIndicators(symbol, range, types, params): Map<IndicatorType, List<DataPoint>>
├── AskChatbotUseCase            — ask(sessionId, question): Flux<String>   (streaming)
├── ManageWatchlistUseCase       — add/remove/list
├── ManageHoldingsUseCase        — add/update/delete/list
├── ManageAlertsUseCase          — create/cancel/list
└── GetNotificationsUseCase      — list(userId, unreadOnly)
```

### Outbound (repository / adapter interfaces)
```
com.aureon.domain.port.out
├── GoldPriceRepository          — save, findLatestBySymbol, findBySymbolAndTimeRange, findLatestBySymbolBefore
├── UserRepository               — findByGoogleSub, findById, upsert
├── WatchlistRepository          — findByUserId, save, deleteByUserIdAndSymbol
├── HoldingRepository            — findByUserId, findById, save, deleteById
├── AlertRepository              — findByUserId, findActiveBySymbol, save, update
├── NotificationRepository       — findByUserIdUnread, save, markRead
├── PriceFeedPort                — fetchLive(symbol): GoldPrice
├── LlmPort                      — chat(systemPrompt, history, userMessage): Flux<String>
└── PriceCachePort               — getLive, setLive, getChart, setChart, getIndicator, setIndicator
```

---

## Flyway Migrations Order

```
V1__create_users.sql
V2__create_gold_prices.sql
V3__create_watchlist_items.sql
V4__create_holdings.sql
V5__create_price_alerts.sql
V6__create_notifications.sql
```
