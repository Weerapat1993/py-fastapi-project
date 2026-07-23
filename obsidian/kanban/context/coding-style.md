# Coding Style

## Backend (Python / FastAPI)
- Follow Ruff's rule set in `backend/pyproject.toml` (`E`, `W`, `F`, `I`, `B`, `C4`, `UP`, `ARG001`, `T201`)
- Classes use PascalCase; functions/variables use snake_case
- Table names via SQLModel are plural (match existing `Item`, `User` conventions)
- Every schema change needs a reversible Alembic migration
- Keep route handlers thin — no complex query building or business logic inline; push reusable logic into `app/crud.py` or a dedicated module
- No raw SQL unless there's no reasonable SQLModel/SQLAlchemy way to express it
- Type everything; `mypy --strict` and `ty check` must pass
- Run after every backend change:
  ```sh
  cd backend
  bash scripts/format.sh   # ruff check --fix + ruff format
  bash scripts/lint.sh     # mypy, ty check, ruff check, ruff format --check
  ```

## Frontend (React / TypeScript)
- Components use PascalCase, colocated under `frontend/src/components/<Feature>/`
- Routes live in `frontend/src/routes` (TanStack Router file-based) — never hand-edit `routeTree.gen.ts`
- Hooks start with `use`, camelCase, under `frontend/src/hooks`
- Props/types must be explicit; avoid `any`
- Extract shared types into their own file when reused across components
- Never hardcode API URLs or hand-roll fetch — use the generated client in `frontend/src/client` (`npm run generate-client` after backend model/schema changes)
- Use TanStack Query for server state; react-hook-form + zod for forms
- UI priority: existing `src/components/ui` primitives (Radix-based) → compose with Tailwind CSS v4 utility classes
- Run after every frontend change:
  ```sh
  cd frontend
  bun run lint   # biome check --write --unsafe
  bun run build  # tsc -p tsconfig.build.json && vite build
  ```

## General
- Name files by what they contain, not by ticket or task
- Delete unused code instead of commenting it out
- Avoid duplicate logic; extract a shared helper when the same logic appears 3+ times
- Comment only non-obvious *why* (workarounds, invariants); no docstring padding
- Don't create documentation files unless explicitly asked

## Testing
- Backend: Pytest under `backend/tests/`, mirroring `backend/app/` structure (`tests/api/routes/`, `tests/crud/`)
  ```sh
  cd backend
  bash scripts/test.sh              # coverage run + report + html
  uv run pytest tests/ -k test_name # single test
  ```
- Frontend E2E: Playwright under `frontend/tests/`
  ```sh
  cd frontend
  bun run test      # bunx playwright test
  bun run test:ui   # interactive UI mode
  ```
- Every behavior change needs a matching test; don't delete a test without approval
- Full local stack (Docker Compose, mirrors CI):
  ```sh
  docker compose watch
  ```

## Review Code
Use the `code-review` skill as the review gate before moving a ticket to Done.

Read:
- spec/context referenced in the ticket
- related files (backend routes/models/crud, frontend routes/components, migrations)
- tests covering the change
- acceptance criteria

Report:
- Verdict: PASS / NEEDS FIX / BLOCKED
- Findings ordered by severity
- Missing tests, if any
- Commands run (lint/type-check/tests/build)
- Whether the ticket is ready to move to Done
