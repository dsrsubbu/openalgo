# Copilot instructions for OpenAlgo

## Project overview

OpenAlgo is a self-hosted algorithmic trading platform built with Flask on the backend and React 19 on the frontend. The app combines four surfaces in one deployment: a unified broker API (`/api/v1/`), a Python strategy host (`/python`), a visual Flow builder (`/flow`), and an options trading suite (`/tools`). They share one broker session, one WebSocket feed, and one local deployment.

This repo is not a generic CRUD app: most work crosses backend broker integration, database/session management, and the live trading/websocket stack. Primary references for project context are `README.md`, `CLAUDE.md`, and `CONTRIBUTING.md`.

## Repo layout and entry points

- `app.py`: Flask application entry point.
- `blueprints/`: UI pages, webhooks, and route handlers.
- `restx_api/`: REST API endpoints for `/api/v1/`.
- `services/`: business logic and orchestration.
- `database/`: SQLAlchemy models and per-database initialization.
- `broker/`: broker integrations; each broker follows a shared plugin layout (`api/`, `mapping/`, `database/`, `streaming/`, `plugin.json`).
- `websocket_proxy/`: unified market data relay and client subscription handling.
- `frontend/`: React + Vite SPA; built assets are served by Flask from `frontend/dist/`.
- `test/`: backend tests and fixtures.

## Build, test, and lint commands

### Python / backend

```bash
# install/update project dependencies
pip install uv
uv sync

# run the app in dev mode
uv run app.py

# production mode (Linux/Gunicorn; single worker required)
uv run gunicorn --worker-class eventlet -w 1 app:app

# run the full backend test suite
uv run pytest test/ -v

# run one backend file
uv run pytest test/test_broker.py -v

# run a single backend test
uv run pytest test/test_broker.py::test_function_name -v

# lint and format
uv run ruff check .
uv run ruff check . --fix
uv run ruff format .
```

### Frontend

```bash
cd frontend
npm install

# local dev server
npm run dev

# production build
npm run build

# single frontend test file
npm run test:run -- src/path/to/file.test.tsx

# full frontend test suite
npm run test:run

# lint
npm run lint

# e2e
npm run e2e
```

## Architecture and data flow

### Backend + frontend split

- The Flask app serves the React frontend from `frontend/dist/` in production.
- During local UI work, the Vite dev server (`npm run dev`) can be used alongside the Flask app, with the frontend proxying API requests back to the backend.
- Production deployments do not require a local frontend build unless actively changing React code.

### Broker integration pattern

Each broker in `broker/<broker>/` is built around the same shape:

- `api/` for authentication, orders, market data, and funds
- `mapping/` for translating between OpenAlgo and broker data formats
- `database/` for symbol/master-contract access
- `streaming/` for broker WebSocket adapters
- `plugin.json` for runtime discovery

This plugin architecture is central to the project and should be preserved when adding or modifying brokers.

### Market data and live order flow

The runtime stack is built around a unified feed:

- broker adapters normalize tick data and publish it into a ZeroMQ bus
- the unified WebSocket proxy (`websocket_proxy/`) handles subscriptions and fan-out to browser clients
- order and position flows pass through service layers before broker API calls

This is one of the main “big-picture” patterns in the repo: business logic is separated from broker-specific adapters, while the frontend and broker integrations both consume the shared infra.

### Database model

The repo intentionally uses multiple databases instead of a single monolithic DB:

- `db/openalgo.db`: main application data
- `db/logs.db`: traffic/API logs
- `db/latency.db`: latency data
- `db/health.db`: health monitoring
- `db/sandbox.db`: sandbox trading state
- `db/historify.duckdb`: historical market data

Each database has its own initialization and lifecycle conventions.

## Key codebase conventions

### Python environment and tooling

- Always use `uv run` for Python commands; do not use global Python or manually manage a venv.
- Keep Python dependencies in `pyproject.toml` and refresh the lockfile with `uv sync` after changes.
- The project targets Python 3.12+.

### SQLite and connection handling

- SQLite databases use `NullPool`; do not switch to `StaticPool`.
- The repo explicitly avoids shared SQLite connections because concurrent requests can corrupt cursor state and trigger the known “bad parameter or other API misuse” / “cannot commit - SQL statements in progress” issues.
- Session cleanup and file-descriptor hygiene are important throughout the app, especially around requests, traffic logging, websockets, and broker sessions.

### SQLAlchemy and data access

- Prefer SQLAlchemy ORM queries and parameterized access over raw SQL.
- The repo’s security model assumes parameterized SQL and avoids direct SQL string construction.
- For broker/order logic, stay in the service/database layers rather than embedding broker-specific logic in route handlers.

### Runtime constraints

- Production runs with Gunicorn + `eventlet` using a single worker (`-w 1`).
- Do not use `asyncio.run()` or other asyncio patterns in the production runtime path; eventlet monkey-patching makes them incompatible.
- Local development (`uv run app.py`) runs under the standard threading model and behaves differently from production, so code should be compatible with both environments.

### Frontend conventions

- The frontend uses React 19, TypeScript, Vite, Tailwind, shadcn/ui, TanStack Query, and Zustand.
- Use the existing API/service patterns and leave the generated `frontend/dist/` assets alone unless intentionally building for distribution.
- CI force-adds `frontend/dist/` on `main`, so local dist artifacts are not the source of truth for the project.

### Logging and error-handling expectations

- Use the project logger (`logger = get_logger(__name__)`) and prefer `logger.exception(...)` for failures.
- Do not rely on raw traceback printing in normal code paths.
- This project centralizes logging and error reporting; keep that mechanism intact.

## Practical guidance for changes

- Start by understanding the affected route/service/broker boundary before editing code.
- For broker work, keep the implementation aligned with the plugin contract and mapping conventions used by existing brokers.
- For frontend work, prefer the established React patterns and route structure instead of introducing a parallel app architecture.
- For backend changes touching DB or sockets, check for session/connection cleanup and the single-worker deployment constraints.

This repo rewards changes that respect the existing broker abstraction, the live trading stack, and the backend/frontend split rather than “inlining” logic in one layer.
