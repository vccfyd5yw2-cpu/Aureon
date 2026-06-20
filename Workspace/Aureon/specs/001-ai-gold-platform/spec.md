# Feature Specification: AI-Powered Gold Market Intelligence Platform

**Feature Branch**: `001-ai-gold-platform`

**Created**: 2026-06-20

**Status**: Draft

**Input**: User description: "Build an AI-powered gold market intelligence platform using Java 21, Spring Boot, PostgreSQL, Redis, and React. The system should provide real-time gold price tracking (XAU/USD and local gold prices), historical data visualization with interactive charts and technical indicators (SMA, EMA, RSI, MACD, Bollinger Bands), and an AI chatbot for market analysis powered by an LLM API. Users should be able to ask natural language questions about market trends and receive data-grounded insights. The system must include optional Google OAuth authentication for saving watchlists, portfolios, and price alerts. Architecture should follow clean architecture principles with REST APIs, caching layer (Redis), database persistence (PostgreSQL), and containerized deployment using Docker."

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Real-Time Gold Price Dashboard (Priority: P1)

As a gold market observer, I want to see live XAU/USD and local gold prices on a dashboard so that I can monitor current market conditions at a glance.

**Why this priority**: This is the core value proposition — price visibility is the foundation that all other features depend on. Without live prices, the platform has no utility.

**Independent Test**: Can be fully tested by visiting the dashboard and observing current XAU/USD and local gold prices updating in real time, delivering immediate market awareness without requiring any account.

**Acceptance Scenarios**:

1. **Given** I open the platform without an account, **When** the dashboard loads, **Then** I see the current XAU/USD spot price, percentage change, and at least one local gold price (e.g., VND/tael or local equivalent) refreshed automatically at regular intervals.
2. **Given** the dashboard is displayed, **When** the price source updates, **Then** the displayed price reflects the new value within 60 seconds without requiring a manual page refresh.
3. **Given** the platform is loading prices, **When** the external price feed is temporarily unavailable, **Then** a clear "Data unavailable" indicator is shown and the last known price is displayed with a timestamp.

---

### User Story 2 - Historical Price Charts with Technical Indicators (Priority: P2)

As a gold market analyst, I want to explore historical gold price charts with overlaid technical indicators so that I can identify trends and make informed assessments.

**Why this priority**: Historical visualization with indicators is the primary analytical tool for market participants and differentiates this platform from a simple price ticker.

**Independent Test**: Can be fully tested by selecting a time range on the chart and toggling indicators (SMA, EMA, RSI, MACD, Bollinger Bands) to verify they compute and render correctly over historical data.

**Acceptance Scenarios**:

1. **Given** I am on the price chart view, **When** I select a time range (e.g., 1D, 1W, 1M, 3M, 1Y, 5Y), **Then** the chart renders historical OHLCV candlestick or line data for that period.
2. **Given** a chart is displayed, **When** I enable a technical indicator (any of: SMA, EMA, RSI, MACD, Bollinger Bands), **Then** the indicator is computed and overlaid on the chart with a visible legend and configurable parameters (e.g., SMA period).
3. **Given** multiple indicators are active, **When** I hover over a data point, **Then** a tooltip shows the price and all active indicator values for that timestamp.
4. **Given** I am viewing a chart, **When** I zoom in or pan through time, **Then** the chart redraws smoothly without data loss or distortion.

---

### User Story 3 - AI Chatbot for Market Analysis (Priority: P2)

As a market participant, I want to ask natural language questions about gold market trends and receive data-grounded answers from an AI assistant so that I can quickly understand market dynamics without expertise in chart reading.

**Why this priority**: The AI chatbot is the key differentiator of the platform; it transforms raw data into accessible, conversational insights.

**Independent Test**: Can be fully tested by typing market questions (e.g., "What is the gold price trend over the last month?") and verifying that responses reference actual stored data with reasoning.

**Acceptance Scenarios**:

1. **Given** I open the chat interface, **When** I ask "What is the current gold price trend?", **Then** the AI responds with a human-readable analysis that references actual recent price data from the platform.
2. **Given** I ask a question about historical patterns, **When** the system processes it, **Then** the response cites specific price points or time ranges drawn from the platform's stored data.
3. **Given** I ask a question outside the system's data scope (e.g., about stock markets), **When** the AI responds, **Then** it clearly states the limitation and redirects to gold market topics it can address.
4. **Given** any user (authenticated or not), **When** they use the chatbot, **Then** responses are generated within 10 seconds under normal load.

---

### User Story 4 - User Authentication and Personalized Watchlist (Priority: P3)

As a returning user, I want to sign in with my Google account and save a watchlist of gold instruments so that I can quickly access prices I care about on future visits.

**Why this priority**: Authentication unlocks persistence and personalization, enabling retention and deeper engagement — but the platform is fully usable without it.

**Independent Test**: Can be fully tested by signing in with Google OAuth, saving two instruments to a watchlist, signing out, signing back in, and confirming the watchlist persists.

**Acceptance Scenarios**:

1. **Given** I am not signed in, **When** I click "Sign in with Google", **Then** I am redirected through Google OAuth and returned to the platform as an authenticated user.
2. **Given** I am signed in, **When** I mark a gold instrument as a watchlist item, **Then** it appears in my personal watchlist and persists across sessions.
3. **Given** I have a watchlist, **When** I sign out and sign back in, **Then** my watchlist is fully restored.
4. **Given** I choose not to sign in, **When** I use the platform, **Then** all core features (live prices, charts, technical indicators, AI chat) remain accessible without restriction.

---

### User Story 5 - Portfolio Tracking (Priority: P3)

As an investor, I want to record my gold holdings and see their current value so that I can track my portfolio performance over time.

**Why this priority**: Portfolio tracking adds significant long-term value for investors and encourages repeat visits, but requires authentication and is secondary to core market data features.

**Independent Test**: Can be fully tested by adding a holding (e.g., "10 tael purchased at price X"), observing the current calculated value displayed, and confirming the entry persists after sign-out and sign-in.

**Acceptance Scenarios**:

1. **Given** I am signed in, **When** I add a holding with quantity and purchase price, **Then** it appears in my portfolio with current market value and gain/loss calculated.
2. **Given** I have holdings recorded, **When** the market price updates, **Then** the portfolio value recalculates automatically.
3. **Given** I view my portfolio, **When** I inspect a holding, **Then** I see purchase price, current price, percentage gain/loss, and total value.

---

### User Story 6 - Price Alerts (Priority: P4)

As a price-sensitive trader, I want to set threshold alerts for gold prices so that I am notified when a target price is crossed.

**Why this priority**: Alerts increase engagement and retention for active traders, but are an enhancement layer on top of core market data features.

**Independent Test**: Can be fully tested by creating an alert at a specific price threshold, observing the alert appear in the active alerts list, and verifying notification delivery (in-app) when the threshold is simulated.

**Acceptance Scenarios**:

1. **Given** I am signed in, **When** I create a price alert for XAU/USD at a specific threshold (above or below), **Then** the alert is saved and listed as active.
2. **Given** an active alert exists, **When** the gold price crosses the alert threshold, **Then** I receive an in-app notification indicating which alert was triggered and the price at trigger time.
3. **Given** I have multiple alerts, **When** I view the alerts list, **Then** I can see all active and triggered alerts with their conditions and status.

---

### Edge Cases

- What happens when the external gold price API is unavailable for an extended period (>5 minutes)?
- How does the AI chatbot handle ambiguous or very short queries (e.g., "gold")?
- What happens when a user tries to add duplicate watchlist items?
- How does the system behave when historical data for a requested time range is incomplete?
- What happens if a user's Google account is deactivated after sign-in?
- How are price alerts handled if the price jumps past the threshold without hitting it exactly (gap scenario)?
- What happens when a user submits a portfolio holding with a future purchase date?

---

## Requirements *(mandatory)*

### Functional Requirements

**Real-Time Price Tracking**
- **FR-001**: System MUST display the current XAU/USD spot price, refreshed at least every 60 seconds.
- **FR-002**: System MUST display at least one local gold price denomination (e.g., local market price per standardized unit).
- **FR-003**: System MUST show price change amount and percentage change from the previous 24-hour closing price.
- **FR-004**: System MUST display the timestamp of the last successful price update.
- **FR-005**: System MUST show a clear data-unavailability indicator when the live price feed cannot be reached.

**Historical Data & Charts**
- **FR-006**: System MUST provide historical gold price data viewable across multiple time ranges: 1 day, 1 week, 1 month, 3 months, 1 year, and 5 years.
- **FR-007**: System MUST render price history as an interactive chart (candlestick or line) with zoom and pan support.
- **FR-008**: System MUST support overlay of the following technical indicators on the price chart: Simple Moving Average (SMA), Exponential Moving Average (EMA), Relative Strength Index (RSI), Moving Average Convergence Divergence (MACD), and Bollinger Bands.
- **FR-009**: System MUST allow users to configure indicator parameters (e.g., period for SMA/EMA).
- **FR-010**: System MUST display a data tooltip on chart hover showing price and active indicator values.

**AI Market Analysis Chatbot**
- **FR-011**: System MUST provide a conversational interface where users can submit natural language questions about the gold market.
- **FR-012**: System MUST ground AI responses in actual price data stored by the platform (responses must reference real data, not generic knowledge alone).
- **FR-013**: System MUST respond to market queries within 10 seconds under normal operating load.
- **FR-014**: System MUST clearly indicate when a query falls outside the platform's knowledge scope (non-gold topics, missing data periods).
- **FR-015**: System MUST maintain conversation history within a session so follow-up questions are contextually aware.

**Authentication & User Accounts**
- **FR-016**: System MUST support optional sign-in via Google OAuth — no account required to access core features.
- **FR-017**: System MUST create and persist a user profile upon first successful Google OAuth sign-in.
- **FR-018**: System MUST issue and validate session tokens to maintain authenticated user state.

**Watchlist**
- **FR-019**: Authenticated users MUST be able to add gold instruments to a personal watchlist.
- **FR-020**: System MUST persist watchlist entries across user sessions.
- **FR-021**: System MUST prevent duplicate entries in a user's watchlist and display an appropriate message when a duplicate is attempted.
- **FR-022**: Authenticated users MUST be able to remove items from their watchlist.

**Portfolio**
- **FR-023**: Authenticated users MUST be able to record gold holdings with quantity, purchase price, and purchase date.
- **FR-024**: System MUST calculate and display current market value and gain/loss for each holding using live prices.
- **FR-025**: Authenticated users MUST be able to edit or delete recorded holdings.

**Price Alerts**
- **FR-026**: Authenticated users MUST be able to create price alerts with a target price and direction (above/below threshold).
- **FR-027**: System MUST deliver an in-app notification when an alert condition is met.
- **FR-028**: System MUST mark alerts as triggered after firing and retain them in alert history.
- **FR-029**: Authenticated users MUST be able to delete or deactivate alerts.

### Key Entities *(include if feature involves data)*

- **GoldPrice**: A recorded price point for a gold instrument at a specific timestamp; attributes include instrument symbol, price, currency, source, and timestamp.
- **User**: A registered account linked to an external OAuth identity; attributes include unique identifier, display name, email, and registration timestamp.
- **WatchlistItem**: An association between a user and a gold instrument they wish to monitor; attributes include user reference, instrument symbol, and creation timestamp.
- **Holding**: A recorded gold position within a user's portfolio; attributes include user reference, instrument, quantity, purchase price, purchase currency, and purchase date.
- **PriceAlert**: A user-defined price trigger; attributes include user reference, instrument, threshold price, direction (above/below), status (active/triggered), creation timestamp, and trigger timestamp.
- **ChatHistory**: A conversational exchange within a user session; maintained as an ordered list of message turns (role + content + timestamp) stored ephemerally in cache with a 30-minute TTL. Not persisted to the database — sessions are anonymous and non-recoverable after expiry.
- **Notification**: A system-generated delivery record created when a price alert fires; attributes include user reference, type, reference to the triggering alert, title, body, read status, and creation timestamp.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The live gold price dashboard displays a current XAU/USD price with no more than 60-second staleness under normal operating conditions.
- **SC-002**: Historical price charts render for any supported time range within 3 seconds on a standard broadband connection.
- **SC-003**: All five technical indicators (SMA, EMA, RSI, MACD, Bollinger Bands) can be enabled and displayed simultaneously on a chart without rendering errors.
- **SC-004**: The AI chatbot responds to a submitted market question within 10 seconds in 95% of requests under normal load.
- **SC-005**: AI chatbot responses reference specific price data points from the platform in at least 80% of market-related queries.
- **SC-006**: Google OAuth sign-in completes within 30 seconds end-to-end (user click to authenticated dashboard) in 95% of attempts.
- **SC-007**: Watchlist entries and portfolio holdings persist correctly across 100% of sign-out/sign-in cycles in acceptance testing.
- **SC-008**: Price alert notifications are delivered within 2 minutes of the threshold being crossed.
- **SC-009**: All core features (live prices, charts, AI chat) remain fully accessible to unauthenticated users.
- **SC-010**: The platform is deployable as a containerized system with a single orchestration command and starts successfully within 3 minutes.

---

## Assumptions

- An external gold price data provider API is available and accessible; the specific API vendor is a technical implementation decision.
- "Local gold price" refers to the primary local denomination relevant to the target market (e.g., VND/tael for Vietnamese market); this can be extended to other locales in future iterations.
- Google OAuth is the sole supported authentication provider for this version; additional providers (e.g., GitHub, Apple) are out of scope.
- In-app notifications are sufficient for price alerts in v1; push notifications (email, SMS, mobile push) are out of scope.
- The AI chatbot uses a third-party LLM API; the choice of provider and model is a technical decision.
- Historical data availability depends on what the price data provider exposes; gaps in historical data should be handled gracefully.
- Containerized deployment targets a single-host Docker Compose setup; multi-host orchestration (Kubernetes) is out of scope for v1.
- Currency conversion for portfolio display uses the same price feed; multi-currency support beyond XAU/USD and one local currency is out of scope for v1.

---

## Out of Scope

- Mobile native applications (iOS/Android) — web-responsive UI only.
- Push notifications via email, SMS, or mobile push.
- Social or sharing features (sharing watchlists, public portfolios).
- Automated trading or order placement integration.
- Support for precious metals other than gold (silver, platinum, etc.) in v1.
- Multi-host or cloud-native orchestration.
- Payment processing or subscription tiers.
