# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Full Stack FastAPI Template: FastAPI + SQLModel + PostgreSQL backend, React + TypeScript + Vite frontend (TanStack Router/Query, Tailwind CSS + shadcn/ui), Docker Compose + Traefik for deploy. This repo is a clone/fork of `fastapi/full-stack-fastapi-template` and can also be regenerated as a Copier template (`copier.yml`, `.copier/update_dotenv.py`).

## Commands

### Local stack (Docker Compose)

```bash
docker compose watch          # start full stack with live reload
docker compose logs           # all logs
docker compose logs backend   # single service logs
docker compose exec backend bash   # shell inside backend container
```

- Frontend: http://localhost:5173
- Backend + Swagger docs: http://localhost:8000/docs (ReDoc at `/redoc`)
- Adminer (DB admin): http://localhost:8080
- Traefik dashboard: http://localhost:8090
- MailCatcher (captures dev emails): http://localhost:1080

You can stop one Docker service and run its local dev server instead (same ports), e.g. `docker compose stop frontend` then `bun run dev`.

### Backend (`backend/`, run with `uv`)

```bash
cd backend
uv sync                              # install deps
source .venv/bin/activate
fastapi dev app/main.py              # local dev server (if not using Docker)

bash scripts/lint.sh                 # mypy + ty check + ruff check + ruff format --check
bash scripts/format.sh                # ruff check --fix + ruff format
bash scripts/test.sh                  # coverage run + report + html (backend/htmlcov/index.html)
uv run pytest tests/ -k test_name     # run a single test
docker compose exec backend bash scripts/tests-start.sh -x  # run tests against the running stack, stop on first failure
```

Migrations (inside the backend container or venv, after changing `app/models.py`):

```bash
alembic revision --autogenerate -m "Add column last_name to User model"
alembic upgrade head
```

Commit the generated files under `backend/app/alembic/versions/`.

### Frontend (`frontend/`, run with `bun`)

```bash
cd frontend
bun install
bun run dev            # vite dev server
bun run build          # tsc -p tsconfig.build.json && vite build
bun run lint           # biome check --write --unsafe
bun run generate-client   # regenerate OpenAPI client in src/client after backend schema changes
bun run test           # bunx playwright test (needs backend stack running)
bun run test:ui        # playwright UI mode
```

Root-level `bash scripts/generate-client.sh` regenerates the frontend client from the backend's live OpenAPI schema — run this (or let pre-commit run it) whenever backend routes/schemas change.

### Pre-commit

Uses `prek` (not `pre-commit`). Config in `.pre-commit-config.yaml`.

```bash
uv run prek install -f       # install git hook (run from backend/, since prek is a backend dep)
uv run prek run --all-files  # run all hooks manually (ruff, biome, mypy, ty, typos, generate-client, zizmor)
```

## Architecture

### Backend request flow
`app/main.py` builds the FastAPI app and mounts `app/api/main.py`'s `api_router` at `settings.API_V1_STR` (`/api/v1`). That router aggregates per-resource routers from `app/api/routes/` (`login`, `users`, `items`, `utils`, and `private` — the last only mounted when `ENVIRONMENT == "local"`, for test/dev-only endpoints).

- **Settings**: `app/core/config.py` — `Settings` (pydantic-settings), reads the *top-level* `.env` (one directory above `backend/`), not a `backend/.env`.
- **DB engine/session**: `app/core/db.py` creates the engine from `settings.SQLALCHEMY_DATABASE_URI` and seeds the first superuser in `init_db()`. Tables are created via Alembic migrations, not `SQLModel.metadata.create_all()` (that call is commented out — flip it only if you intentionally want to abandon migrations).
- **Auth**: `app/core/security.py` (JWT via `pyjwt`, hashing via `pwdlib`). Route-level auth deps (`SessionDep`, `CurrentUser`) live in `app/api/deps.py`.
- **Models**: all SQLModel table + schema classes live in one file, `app/models.py` — the `*Base` / `*Create` / `*Update` / `*Public` / table-model split pattern is used throughout (e.g. `Item`, `ItemCreate`, `ItemPublic`, `ItemsPublic`). New models must be imported here (or imported before engine init) or SQLModel relationships won't resolve correctly.
- **CRUD**: `app/crud.py` holds the one shared layer of DB-mutating helper functions (`create_user`, `authenticate`, `create_item`, etc.) that routes call into — keep route handlers thin and put reusable persistence logic here.
- **Response shape**: no custom envelope. Success responses are the Pydantic `response_model` directly; collections use `{"data": [...], "count": N}` (`ItemsPublic`-style); simple acks use `Message` (`{"message": "..."}`); errors are plain FastAPI `HTTPException`.
- **Permission model**: superuser vs owner-only visibility is checked inline per route (`current_user.is_superuser`, `owner_id == current_user.id`) — see `app/api/routes/items.py` for the reference pattern.
- **Email**: templates in `app/email-templates/src` (MJML) built to `app/email-templates/build` (HTML), rendered with Jinja2 and sent via `emails`; MailCatcher intercepts SMTP locally.

### Frontend structure
- `src/routes` — TanStack Router file-based routes (pages live under `routes/_layout`); the router auto-generates `src/routeTree.gen.ts` — never hand-edit that file.
- `src/client` — OpenAPI-generated API client (via `@hey-api/openapi-ts`); regenerate instead of hand-writing fetch calls whenever the backend schema changes.
- `src/components` — grouped by feature (`Admin`, `Items`, `UserSettings`, `Sidebar`, `Common`) plus `components/ui` for shadcn/Radix-based primitives — prefer composing from `ui/` before reaching for a new dependency.
- `src/hooks` — shared custom hooks.
- Server state goes through TanStack Query; forms use react-hook-form + zod, matching backend Pydantic constraints.

### Cross-cutting
- The frontend OpenAPI client and the backend schema must stay in sync — any backend route/model change should be followed by regenerating the client (`scripts/generate-client.sh` or `bun run generate-client`), and this is also enforced by a pre-commit hook.
- `compose.yml` is the base stack definition; `compose.override.yml` adds dev-only behavior (source mounted as a volume, `fastapi run --reload`, local Traefik for testing subdomains via `DOMAIN=localhost.tiangolo.com`).
- Removing the frontend entirely (API-only use) means deleting `./frontend` and the `frontend`/`playwright` services from both compose files — supported as a first-class option per `frontend/README.md`.

## Repo-specific notes

- This is a template repo: contributors are asked to open a GitHub Discussion before big/architectural changes (see `CONTRIBUTING.md`); PRs from non-maintainers may not modify `pyproject.toml` / `uv.lock` (supply-chain guard, enforced by `.github/workflows/guard-dependencies.yml`).
- `obsidian/kanban/` holds this repo's local ticket workflow (see `.claude/skills/kanban/SKILL.md` and `obsidian/kanban/context/kanban-flow.md`) — read `obsidian/kanban/context/requirements.md` and `coding-style.md` before starting ticket work, they mirror the stack details above with workflow-specific conventions.
