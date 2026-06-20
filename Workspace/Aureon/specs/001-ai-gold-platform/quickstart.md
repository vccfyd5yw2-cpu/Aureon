# Quickstart: AI-Powered Gold Market Intelligence Platform

**Date**: 2026-06-20 | **Plan**: [plan.md](plan.md)

---

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Docker Desktop | 25+ | https://docs.docker.com/desktop/ |
| Docker Compose | v2 (bundled with Desktop) | — |
| Java | 21 (for backend dev only) | `sdk install java 21-tem` via SDKMAN |
| Node.js | 20 LTS (for frontend dev only) | https://nodejs.org or `nvm install 20` |
| Git | any recent | — |

---

## 1. Clone & Configure

```bash
git clone <repo-url> aureon
cd aureon
cp infra/.env.example infra/.env
```

Edit `infra/.env` and fill in required values:

```dotenv
# ── Required ──────────────────────────────────────────────────────────────────
POSTGRES_PASSWORD=changeme_dev_only

# Gold price feed (get free key at https://www.goldapi.io)
GOLD_API_KEY=your_goldapi_key_here

# Google OAuth (create credentials at https://console.cloud.google.com)
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret

# JWT signing secret — minimum 32 random characters
JWT_SECRET=replace_with_32plus_char_random_string

# LLM provider (OpenAI by default; change SPRING_AI_OPENAI_BASE_URL for Anthropic/Ollama)
OPENAI_API_KEY=your_openai_api_key

# ── Optional overrides ────────────────────────────────────────────────────────
POSTGRES_DB=aureon
POSTGRES_USER=aureon
REDIS_PASSWORD=
SPRING_AI_OPENAI_MODEL=gpt-4o-mini
```

> **Security**: Never commit `infra/.env`. It is in `.gitignore`.

---

## 2. Full Stack — Docker Compose

```bash
cd infra
docker compose up --build
```

Services started:

| Service | Port | Notes |
|---------|------|-------|
| `backend` | 8080 | Spring Boot API |
| `frontend` | 3000 | React SPA (served by Nginx) |
| `postgres` | 5432 | PostgreSQL 16 |
| `redis` | 6379 | Redis 7 |
| `nginx` | 80 | Reverse proxy (`/api` → backend, `/` → frontend) |

Open `http://localhost` in your browser.

API base URL: `http://localhost/api/v1`

---

## 3. Backend — Development (Hot Reload)

```bash
cd backend
./gradlew bootRun
```

Requires a running PostgreSQL and Redis. Use the dev compose override:

```bash
cd infra
docker compose -f docker-compose.yml -f docker-compose.dev.yml up postgres redis
```

Backend runs at `http://localhost:8080`.

**Run tests** (requires Docker for Testcontainers):

```bash
./gradlew test
```

**Run only unit tests** (no Docker required):

```bash
./gradlew test --tests "com.aureon.*" -x integrationTest
```

---

## 4. Frontend — Development (Vite HMR)

```bash
cd frontend
npm install
npm run dev
```

Frontend runs at `http://localhost:5173` with proxy to backend at `http://localhost:8080`.

**Run tests**:

```bash
npm run test
```

---

## 5. OAuth2 Callback URL

Register the following redirect URI in your Google Cloud Console OAuth2 credentials:

```
http://localhost/login/oauth2/code/google
```

For local backend-only dev (no Nginx):

```
http://localhost:8080/login/oauth2/code/google
```

---

## 6. Database Migrations

Flyway runs automatically on Spring Boot startup. To inspect migration state:

```bash
./gradlew flywayInfo
```

To repair a failed migration:

```bash
./gradlew flywayRepair
```

Migration files live at:

```
backend/src/main/resources/db/migration/
├── V1__create_users.sql
├── V2__create_gold_prices.sql
├── V3__create_watchlist_items.sql
├── V4__create_holdings.sql
├── V5__create_price_alerts.sql
└── V6__create_notifications.sql
```

---

## 7. Using an Alternative LLM (Ollama for local dev)

```bash
# Pull and run Ollama locally
docker run -d -p 11434:11434 --name ollama ollama/ollama
docker exec -it ollama ollama pull llama3.2
```

Update `infra/.env`:

```dotenv
SPRING_AI_OPENAI_BASE_URL=http://host.docker.internal:11434/v1
SPRING_AI_OPENAI_API_KEY=ollama
SPRING_AI_OPENAI_MODEL=llama3.2
```

---

## 8. Directory Reference

```
aureon/
├── backend/        Spring Boot application (Java 21)
├── frontend/       React SPA (TypeScript, Vite)
├── infra/          Docker Compose, Nginx config, .env.example
└── specs/          Spec-Kit planning artifacts
    └── 001-ai-gold-platform/
        ├── spec.md         Feature specification
        ├── plan.md         This implementation plan
        ├── research.md     Technology decisions
        ├── data-model.md   Database schema + domain model
        ├── quickstart.md   This file
        ├── contracts/      OpenAPI YAML contracts
        └── tasks.md        Implementation tasks (created by /speckit.tasks)
```

---

## 9. Common Issues

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `gold_prices` table empty after startup | `GOLD_API_KEY` not set | Set key in `.env` and restart backend |
| OAuth login fails with redirect_uri_mismatch | Wrong URI in Google Console | Ensure `http://localhost/login/oauth2/code/google` is registered |
| LLM returns empty response | `OPENAI_API_KEY` invalid or quota exceeded | Verify key; switch to Ollama for dev |
| `JWT_SECRET` error on startup | Secret too short | Use 32+ character random string |
| Testcontainers tests fail | Docker not running | Start Docker Desktop |
