# AGENTS.md

OpenAlgo is a self-hosted algorithmic trading platform: Flask backend (Python 3.12+), React 19 frontend, single-user per deployment, ~36 broker plugins sharing one broker session and WebSocket feed. Trading money is involved — the invariants below came from real position-reversing bugs.

## Read first

- **`CLAUDE.md`** — the definitive reference. It carries invariants, runtime constraints, security/deployment model, and architecture that are *not discoverable from code*. OpenCode does not load it automatically; read it before touching order/streaming/DB code.
- **`docs/INDEX.md`** — map of all docs; open only the area you need (API, strategy module & RMS, broker integration, user guide).
- **`.claude/skills/`** — load on demand: `fd-audit` (leaks), `version-bump`, `broker-integration`, `chart-indicator`, `flow-builder`, `security-audit`, `verify`.
- **`docs/` is the single source of truth and a map, not a copy.** Edit a doc in place; never restate it somewhere else.

## Commands

Always `uv` — never global Python or a hand-managed venv. Frontend work goes through `npm` in `frontend/`.

```bash
uv sync                      # install deps (fully pinned in uv.lock, 239 packages)
uv run app.py                # dev server — standard threading, NOT eventlet
uv run gunicorn --worker-class eventlet -w 1 app:app   # production shape
uv run pytest test/ -v       # full suite (addopts include --timeout=60)
uv run pytest test/test_x.py::test_y -v   # one test
uv run ruff check . --fix && uv run ruff format .   # backend lint/format
```

```bash
cd frontend
npm install
npm run build    # runs tsc -b && vite build  (this IS the typecheck)
npm run dev      # Vite dev server
npm run lint     # biome
npm run test:run -- src/path/file.test.tsx
npm run e2e      # Playwright (needs `npx playwright install chromium`)
```

Config: copy `.sample.env` → `.env`. `VALID_BROKERS` in the env gates which broker plugins load (discovered at startup). `.secrets.baseline` is the detect-secrets baseline — no committed secrets.

## Runtime constraints — production vs dev server

Production is `gunicorn --worker-class eventlet -w 1`. **Dev (`uv run app.py`) uses standard threading.** Code must work in both. This is the #1 way a change passes locally and fails on deploy:

- **No `asyncio` in the production path.** Eventlet monkey-patches the stdlib; `asyncio.run()`/`async`/`await`/`get_event_loop()` silently work on the dev server and break under eventlet. `uv run app.py` never patches anything, so nothing eventlet-related is caught locally.
- **Eventlet tests intentionally run in subprocesses** (`test/test_eventlet_cross_thread_locks.py`, `test/test_sqlite_lock_cooperative.py`) — `monkey_patch()` is global. Add to them rather than starting a third.
- A handful of threads are genuinely real (asyncio loop in `services/websocket_client.py`, Telegram bot / Kaleido renderer, broker snapshot-feed threads). For anything both worlds touch, use `utils/real_threading` (`Lock`, `Event`, `Condition`, `Queue`, `Thread`, `wait_for`) instead of `threading.*`. Never hand a result across with `run_coroutine_threadsafe` — use `WebSocketClient._run_on_loop`. Never wait on a C-served timeout (e.g. `PRAGMA busy_timeout`) holding a write lock.

## Invariants — do not break (from CLAUDE.md, learned from real defects)

- **ZeroMQ bus is fan-in: the proxy SUB binds, every publisher CONNECTs.** Never make a publisher bind, never scan ports or drift `ZMQ_PORT` — a racing bind silently slides the port and ticks stop arriving while `subscribe` reports success.
- **Risk rules live in `services/risk/` and are never reimplemented.** No I/O of any kind there; consumers translate, they don't decide. `test/risk/vectors.json` is the contract shared with the TS copy in `frontend/src/hooks/useTrailingSL.ts`.
- **An order path decides once, under the lock.** Duplicate-check and the `exit_kind` marker happen in one hold of `state.claim_leg_exit`, `exit_kind` written *before* dispatch (never rely on `exit_order_id`). Match fills to the *order*, not the leg. `force_live=True` is required when executing a position you opened (analyzer toggle would route exits to the sandbox).
- **A 2nd-device login must not tear down the shared broker feed.** `database/auth_db.upsert_auth` only tears down (ZMQ `CACHE_INVALIDATE_ALL` + `cleanup_pools_for_user`) when the token actually changed — compare decrypted plaintext (Fernet ciphertext is non-deterministic), never encrypted blobs.
- **SQLite engines use `NullPool`, never `StaticPool`** (via `database.engine_factory.create_db_engine()`). StaticPool corrupts cursor state under concurrency.
- **Logging:** `logger = get_logger(__name__)` from `utils/logging.py`; use `logger.exception()`. Never `print()`, never `traceback.print_exc()`/`format_exc()`. Read `log/errors.jsonl` first when debugging (truncated to last 1000 at startup).
- **No icons or emojis anywhere** — source, logs, commits, PRs, changelogs.

## Testing quirks

- `test/conftest.py` **neutralizes dotenv and reassigns all DB URLs to `db/*-test.db` and `LOG_DIR` to `log/test`** — these assignments are unconditional exactly so a dev/CI environment with production values cannot pollute the real DBs. Consequences:
  - Running tests writes/overwrites the `*-test.db` files and rewrites `log/test/errors.jsonl`. Read `log/errors.jsonl` only after a test run, and read the repo one, not `log/test/`.
  - Do not rely on the operator's `.env` inside tests — the loader is turned off.
- **The full suite needs broker credentials/live services;** CI runs a curated safe subset (the long `uv run pytest ...` list in `.github/workflows/ci.yml`). For a focused verification, run that subset or the specific test you touched.
- `test/test_bot_web.py`, `test/test_websocket.py`, `test/test_websocket_service.py` are excluded from collection (manual diagnostics).

## Build / frontend dist model

- `frontend/dist/` is **gitignored locally but tracked on `main`** — the `commit-dist` CI job force-adds it after every push. Feature branches CI hasn't built may have stale or missing `dist/`; build locally (`npm run build`) or rebase onto recent `main`. Never commit `.gz`/`.br` pre-compressed assets — `utils/precompress_assets.py` regenerates them at startup (they used to bloat repo history).
- `frontend/package.json` depends on `"openalgo": "file:.."` — the repo root is a local package dependency of the frontend, and `openalgo-charts` is a pinned third-party charting package.
- **Custom `/trading` indicators load at runtime from `strategies/indicators/*.js`**, never bundled — the only tracked example is `open_range_breakout.js`. Use the `chart-indicator` skill; it validates against the real library.

## Conventions that differ from defaults

- **SQLAlchemy ORM, never raw SQL.**
- **Schema changes ship as migration scripts in `upgrade/`**, registered in `upgrade/migrate_all.py`'s `MIGRATIONS` list — a start-up `init_db()` hook is *not* enough for existing installs. Each script is idempotent, supports `--status`, never clobbers customised values, and must be tested against a *populated* DB forced back to the old schema.
- **Adding a page requires three registrations:** the `<Route>` in `frontend/src/App.tsx`, a `serve_react_app()` view in `blueprints/react_app.py` (so direct hits aren't counted as 404s / IP-ban fodder), and the nav entry in `frontend/src/config/navigation.ts`.
- **Commits:** Conventional Commits (`feat:`, `fix:`, `docs:`, `refactor:`, `chore:`). Pre-commit runs ruff, `npx --prefix frontend biome check --write src/`, and detect-secrets.
- **Versions:** the platform version (`openalgoUI` in `pyproject.toml`) and the pinned `openalgo` SDK dependency are *two unrelated numbers*; use the `version-bump` skill.
- **CI lint jobs are `continue-on-error`** for pre-existing broker-module warnings — don't assume broker/ dir is ruff-clean, but write new code clean.

## Architecture map (what the dirs mean)

- `app.py` — Flask entry; WSGI middleware order matters (`traffic` wraps `security`, registered in reverse).
- `blueprints/` — routes/pages/webhooks; `restx_api/` — `/api/v1/` surface; `services/` — one logic file per feature.
- `broker/<name>/{api,mapping,database,streaming,plugin.json}` — plugin layout for each broker.
- `websocket_proxy/` — unified market-data relay (port 8765); `mcp/` — Agent Control Mode surface (stdio server is local-only).
- Market data pipeline: broker adapters → **ZeroMQ bus (5555)** → websocket proxy → SocketIO clients. Ports: app 5000, proxy 8765, ZMQ 5555.
- Six isolated DBs: `openalgo.db`, `logs.db`, `latency.db`, `health.db`, `sandbox.db` (isolated from live trading), `historify.duckdb`.
- Broker tokens expire ~3:00 AM IST daily; session management is aligned to that.