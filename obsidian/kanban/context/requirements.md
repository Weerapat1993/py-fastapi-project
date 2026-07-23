# Project Requirements: py-fastapi-project (Full Stack FastAPI Template)

## Goal
Full-stack app: FastAPI backend + React admin/dashboard frontend, based on the `fastapi/full-stack-fastapi-template`. Backend exposes JWT-authenticated CRUD API (users, items); frontend is a private dashboard, not a public SEO site.

## Tech Stack

### Backend (`backend/`)
- Python 3.14+, managed with `uv`
- FastAPI (`fastapi[standard]`)
- SQLModel (ORM, built on SQLAlchemy + Pydantic)
- PostgreSQL + Alembic migrations
- Pydantic v2 / pydantic-settings
- JWT auth: `pyjwt` + `pwdlib` (argon2/bcrypt) password hashing
- `emails` + Jinja2 for email templates (password recovery)
- Sentry SDK for error tracking
- Pytest + coverage for tests
- Ruff (lint + format), mypy (strict) and `ty` for type checking

### Frontend (`frontend/`)
- React 19 + TypeScript + Vite
- TanStack Router (typed routing) + TanStack Query (server state) + TanStack Table
- Tailwind CSS v4 + shadcn/ui-style components (Radix UI primitives under `src/components/ui`)
- Auto-generated OpenAPI client (`@hey-api/openapi-ts` → `npm run generate-client`)
- react-hook-form + zod for form validation
- Biome for lint/format
- Playwright for end-to-end tests
- Bun as package manager/runner

### Infra
- Docker Compose (`compose.yml` + `compose.override.yml`) for local dev
- Traefik as reverse proxy / load balancer
- Mailcatcher for local email testing

## Functional Requirements
- JWT login/logout, password recovery via email (backend routes: `login.py`, `users.py`)
- Superuser vs regular user permission model (see `current_user.is_superuser` checks in routes)
- CRUD for `items` scoped to the owning user, superusers see all
- Validation via Pydantic models (`*Create`, `*Update`, `*Public` schema pattern in `app/models.py`)
- Frontend shows loading, error, and success/toast state for every mutation

## Backend Requirements
- Routes live in `backend/app/api/routes/*.py`, registered via `backend/app/api/main.py`
- Route handlers stay thin: query/validate/call, no heavy business logic inline
- Use SQLModel `select()` + `session.exec()`, not raw SQL
- Auth/session deps come from `app/api/deps.py` (`SessionDep`, `CurrentUser`)
- Any DB schema change needs an Alembic migration (`backend/app/alembic/versions/`)
- Errors use `HTTPException(status_code=..., detail=...)`; generic acks use the `Message` schema (`{"message": "..."}`)
- New settings go through `app/core/config.py` (`Settings`, pydantic-settings)

## Frontend Requirements
- Pages/routes live under `frontend/src/routes` (TanStack Router file-based routing, generates `routeTree.gen.ts` — never hand-edit that file)
- Reusable components under `frontend/src/components` (grouped by feature: `Admin`, `Items`, `UserSettings`, `Sidebar`, `Common`, `ui`)
- Component files use PascalCase (e.g. `ItemActionsMenu.tsx`); hooks use camelCase starting with `use`
- Never hand-write fetch calls or hardcode API URLs — use the generated client in `frontend/src/client` (regenerate with `npm run generate-client` / `bun run generate-client` after backend schema changes)
- Use TanStack Query for server state; keep local UI state in component hooks
- UI priority: existing `src/components/ui` (shadcn/Radix-based) primitives first, then compose with Tailwind; avoid pulling in a new UI kit
- Forms: react-hook-form + zod schema validation, matching backend Pydantic constraints
- Keep components focused and short; extract shared logic into `frontend/src/hooks` or `frontend/src/utils.ts`
- Design for responsive layout (this is a dashboard, but still used on smaller screens)

## API Response Shape
- Success: FastAPI `response_model` (a Pydantic/SQLModel schema), returned directly — no custom envelope
- Collections: `{"data": [...], "count": N}` (see `ItemsPublic`)
- Simple acks/errors: `{"message": "..."}` (`Message` schema) or FastAPI's default `{"detail": "..."}` for `HTTPException`
