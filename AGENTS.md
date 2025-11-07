# Repository Guidelines

## Project Structure & Module Organization
Work across three roots: `backend/` (FastAPI service), `frontend/` (Uni-App mini-program), and `docs/` (API, DB, ops specs). `backend/src` splits into `core/` for configuration + DB sessions, `shared/` for utilities and response schema helpers, and `modules/` for domain packages such as `auth`, `users`, `catalog`, `schedule`, and `appointments`, with migrations in `backend/alembic/`. The client mirrors Uni-App conventions: `pages/` covers the booking funnel, `pages_sub/` holds profile/patient utilities, `pages_admin/` serves staff dashboards, and shared assets live in `api/`, `components/`, `store/`, and `static/`.

## Build, Test, and Development Commands
- `cd backend && uv sync --frozen` – install locked Python dependencies.
- `cd backend && uv run uvicorn main:app --reload` – run the API locally with hot reload.
- `cd backend && uv run alembic upgrade head` – keep the PostgreSQL schema current before coding or testing.
- `cd backend && uv run ruff check && uv run mypy && uv run pytest -q` – lint, type-check, and execute suites (target ≥80% coverage).
- `cd frontend && hbx-cli dev` – start the Uni-App dev server; `hbx-cli build mp-weixin --minimize` produces the release bundle consumed by WeChat.

## Coding Style & Naming Conventions
Target Python 3.13 with snake_case modules, async SQLAlchemy sessions, and routers exposed via `APIRouter(prefix="/api/v1/...")`; keep ULID primary keys and shared response envelopes described in `docs/BACKEND.md`. Ruff controls formatting/import order, mypy enforces strict typing, and secrets/configs travel through `pydantic-settings`. Uni-App code should rely on `<script setup>` Composition API, PascalCase components, kebab-case route folders, and Pinia stores declared as `useXStore`, with admin-only screens gated by FastAPI dependencies plus navigation interceptors.

## Testing Guidelines
Pytest is canonical—place files beside their modules or under `backend/tests/`, name them `test_*.py`, and emphasize availability calculations, appointment states, JWT role guards, and alembic migrations using async fixtures to isolate PostgreSQL. Front-end widgets run through Vitest + Vue Test Utils, and E2E flows must cover login → booking → cancellation plus admin overrides, mirroring the checklists in `docs/OPS_AND_TEST.md`.

## Commit & Pull Request Guidelines
Upstream history follows Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`) with short scopes such as `feat(schedule): add overrides`; keep messages imperative, reference tasks, and mention env or migration impacts. Pull requests should summarize behavior changes, link the relevant spec pages, attach UI captures when front-end code shifts, and list verification commands (`ruff`, `mypy`, `pytest`, `hbx-cli build`). Call out follow-up todos or configuration diffs so reviewers can reproduce quickly.
