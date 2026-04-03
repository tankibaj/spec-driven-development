# Workspace Bootstrap Standard

**Status:** Active
**Last updated:** 2026-04-04
**Applies to:** All workspaces registered in `registry/routes.yaml`

Every workspace (backend service or frontend app) MUST be scaffolded to this standard before
any Work Package implementation begins. The agent creates this scaffold as the first act of
Phase 4 when the workspace directory is empty.

---

## 1. Backend Workspace (Python)

### 1.1 Toolchain

| Concern | Tool | Version |
|---|---|---|
| Framework | FastAPI | `^0.111` |
| ASGI server | Uvicorn | `^0.29` |
| ORM | SQLAlchemy (async) | `^2.0` |
| DB driver | asyncpg (PostgreSQL) | `^0.29` |
| Migrations | Alembic | `^1.13` |
| Validation | Pydantic v2 | `^2.7` |
| HTTP client | httpx | `^0.27` |
| Package manager | uv | latest |
| Linter / formatter | Ruff | `^0.4` |
| Type checker | mypy | `^1.10` |
| Test runner | pytest + pytest-asyncio | `^8.x` |
| Test HTTP client | httpx (AsyncClient) | — |
| Contract tests | Schemathesis | `^3.x` |

### 1.2 Project Structure

```
workspaces/{service-name}/
├── src/
│   ├── __init__.py
│   ├── main.py                  # FastAPI app factory, lifespan, middleware
│   ├── config.py                # Settings via pydantic-settings + .env
│   ├── api/
│   │   ├── __init__.py
│   │   ├── router.py            # Mounts all route groups
│   │   ├── health.py            # GET /health and GET /ready
│   │   └── v1/                  # Versioned route groups
│   ├── services/                # Business logic — no DB calls here
│   ├── repositories/            # DB access — one file per aggregate root
│   ├── models/                  # SQLAlchemy ORM models
│   ├── schemas/                 # Pydantic request / response schemas
│   └── dependencies.py          # FastAPI dependency injection
├── tests/
│   ├── conftest.py              # Shared fixtures (app, db, client)
│   ├── unit/                    # Pure logic tests — no I/O
│   ├── integration/             # DB + service layer tests (real DB in Docker)
│   └── contract/                # Schemathesis + Pact tests
├── alembic/
│   ├── env.py
│   └── versions/
├── Dockerfile
├── docker-compose.yml           # Local dev: app + postgres + redis
├── pyproject.toml
├── .env.example
└── README.md
```

### 1.3 `pyproject.toml` template

```toml
[project]
name = "{service-name}"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
  "fastapi>=0.111",
  "uvicorn[standard]>=0.29",
  "sqlalchemy[asyncio]>=2.0",
  "asyncpg>=0.29",
  "alembic>=1.13",
  "pydantic>=2.7",
  "pydantic-settings>=2.2",
  "httpx>=0.27",
  "prometheus-fastapi-instrumentator>=6.1",
  "python-json-logger>=2.0",
]

[dependency-groups]
dev = [
  "pytest>=8",
  "pytest-asyncio>=0.23",
  "ruff>=0.4",
  "mypy>=1.10",
  "schemathesis>=3",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]

[tool.mypy]
strict = true
plugins = ["pydantic.mypy"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

### 1.4 `Dockerfile` (multi-stage)

```dockerfile
# ── Build stage ───────────────────────────────────────────────────────────────
FROM python:3.12-slim AS builder
WORKDIR /app

RUN pip install uv

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-editable

# ── Runtime stage ─────────────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime
WORKDIR /app

# Non-root user for security
RUN useradd --create-home --shell /bin/bash appuser

COPY --from=builder /app/.venv /app/.venv
COPY src/ ./src/

ENV PATH="/app/.venv/bin:$PATH"
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD python -c "import httpx; httpx.get('http://localhost:8000/health').raise_for_status()"

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 1.5 `docker-compose.yml` (local dev)

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

### 1.6 `.env.example`

```dotenv
# Application
APP_ENV=development
LOG_LEVEL=INFO
SECRET_KEY=change-me-in-production

# Database
DATABASE_URL=postgresql+asyncpg://app:app@localhost:5432/app

# Redis
REDIS_URL=redis://localhost:6379/0

# Tenant
DEFAULT_TENANT_ID=00000000-0000-0000-0000-000000000001

# External services (set to mock URLs in development)
INVENTORY_SERVICE_URL=http://localhost:8001
NOTIFICATION_SERVICE_URL=http://localhost:8002
```

### 1.7 GitHub Actions CI (`.github/workflows/ci.yml`)

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    name: Lint & type-check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with:
          version: "latest"
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run mypy src/

  test:
    name: Unit & integration tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: app
          POSTGRES_DB: app
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U app"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --frozen
      - run: uv run alembic upgrade head
        env:
          DATABASE_URL: postgresql+asyncpg://app:app@localhost:5432/app
      - run: uv run pytest tests/unit tests/integration -v --tb=short
        env:
          DATABASE_URL: postgresql+asyncpg://app:app@localhost:5432/app
          REDIS_URL: redis://localhost:6379/0

  contract:
    name: Contract validation
    runs-on: ubuntu-latest
    needs: [test]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --frozen
      - name: Start service
        run: uv run uvicorn src.main:app --host 0.0.0.0 --port 8000 &
        env:
          DATABASE_URL: postgresql+asyncpg://app:app@localhost:5432/app
          REDIS_URL: redis://localhost:6379/0
      - name: Wait for service
        run: sleep 5
      - name: Run Schemathesis
        run: |
          uv run schemathesis run ../../contracts/api/{service}.openapi.yaml \
            --base-url http://localhost:8000 \
            --checks all \
            --hypothesis-max-examples 50

  build:
    name: Docker build
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 2. Frontend Workspace (TypeScript / React)

### 2.1 Toolchain

| Concern | Tool | Version |
|---|---|---|
| Build tool | Vite | `^5` |
| Framework | React | `^18` |
| Language | TypeScript | `^5.4` |
| Linter / formatter | Biome | `^1.7` |
| Test runner | Vitest | `^1.6` |
| Component tests | Testing Library (React) | `^15` |
| API mocking (dev + test) | MSW (Mock Service Worker) | `^2` |
| API client | openapi-fetch | `^0.9` |
| State management | Zustand | `^4` |
| Routing | React Router v6 | `^6` |
| HTTP types | Generated from OpenAPI via `openapi-typescript` | — |

### 2.2 Project Structure

```
workspaces/{app-name}/
├── src/
│   ├── main.tsx                 # Entry point
│   ├── App.tsx                  # Root component + router
│   ├── api/
│   │   ├── client.ts            # openapi-fetch instance + auth headers
│   │   └── types.ts             # Generated from OpenAPI spec (do not hand-edit)
│   ├── components/              # Shared UI components
│   ├── features/                # Feature-scoped modules
│   │   └── {feature}/
│   │       ├── index.ts
│   │       ├── components/
│   │       ├── hooks/
│   │       └── store.ts
│   ├── mocks/
│   │   ├── browser.ts           # MSW browser setup
│   │   ├── server.ts            # MSW Node setup (for tests)
│   │   └── handlers/            # One file per API route group
│   └── types/                   # Shared TypeScript types
├── tests/
│   ├── setup.ts                 # Vitest + Testing Library global setup
│   ├── unit/                    # Pure logic and hook tests
│   └── integration/             # Component tests with MSW
├── public/
│   └── mockServiceWorker.js     # MSW service worker (generated)
├── Dockerfile
├── nginx.conf                   # Production static file serving
├── package.json
├── vite.config.ts
├── tsconfig.json
├── biome.json
├── .env.example
└── README.md
```

### 2.3 `package.json` (key scripts)

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "biome check .",
    "format": "biome format --write .",
    "typecheck": "tsc --noEmit",
    "generate:types": "openapi-typescript ../../contracts/api/{service}.openapi.yaml -o src/api/types.ts"
  }
}
```

### 2.4 `Dockerfile` (multi-stage)

```dockerfile
# ── Build stage ───────────────────────────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# ── Runtime stage (nginx) ─────────────────────────────────────────────────────
FROM nginx:1.27-alpine AS runtime

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:80/health || exit 1
```

### 2.5 `nginx.conf`

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Health check endpoint (no app logic needed)
    location /health {
        return 200 '{"status":"ok"}';
        add_header Content-Type application/json;
    }

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header Referrer-Policy strict-origin-when-cross-origin;
}
```

### 2.6 GitHub Actions CI (`.github/workflows/ci.yml`)

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    name: Lint & type-check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci
      - run: npx biome check .
      - run: npm run typecheck

  test:
    name: Unit & component tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci
      - run: npm test -- --reporter=verbose

  build:
    name: Production build
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci
      - run: npm run build
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 3. Agent Instructions for Workspace Bootstrap

When entering Phase 4 for a WP whose workspace directory is empty:

1. Read `registry/routes.yaml` to confirm the workspace `type` (`backend` or `frontend`).
2. Apply the relevant scaffold above — create all directories, copy all template files, substituting `{service-name}` or `{app-name}` with the actual workspace ID.
3. In the GitHub Actions CI `schemathesis` step, substitute `{service}` with the correct OpenAPI filename from `contracts/api/`.
4. In the frontend `generate:types` script, substitute `{service}` with the relevant API spec.
5. Commit the scaffold as a single commit: `chore: bootstrap {workspace-name} workspace`.
6. Then begin implementing the WP on top of the scaffold.

Do not deviate from the toolchain above without an ADR. Consistency across workspaces is more valuable than local optimisation.
