# CLAUDE.md — packages/fastapi-server

Python counterpart to `apps/server`: FastAPI blog API (users + posts) with JWT auth, async SQLAlchemy 2.0, Alembic and pytest. `README.md` has the full endpoint table and env reference.

## Commands (run in `packages/fastapi-server`, Python 3.13, uv)

```sh
uv sync
cp .env.example .env                 # SECRET_KEY is required; the app won't import without it
uv run fastapi dev main.py           # :8000, Swagger at /docs, /health
uv run pytest                        # all tests
uv run pytest tests/test_posts.py::test_name -v   # single test
uv run ruff check . && uv run ruff format .

uv run alembic revision --autogenerate -m "msg"   # after editing db/models.py
uv run alembic upgrade head
uv run python -m scripts.seed        # demo data, password "password123"
```

`package.json` wraps these for Turborepo (`pnpm dev`, `pnpm test`, `pnpm db:migrate`, ...). Add dependencies with `uv add`, never pip.

## Architecture

Modules are top-level and imported absolutely (`from config import settings`, `from service import user_service`), so run everything from this directory.

```
routers/  ->  service/  ->  db/
  HTTP        rules       SQLAlchemy
```

- **Routers** own path, status code, `response_model`, and auth (`CurrentUser` from `auth.py`, `DbSession = Annotated[AsyncSession, Depends(get_db)]`). One or two lines per handler; no queries.
- **Services** (`service/*_service.py`, module-level async functions) take `db` plus plain args, hold every rule (uniqueness, ownership, pagination, uploads) and **never import FastAPI or raise `HTTPException`**. They raise `ServiceError` subclasses from `exceptions.py`; `main.py` maps them to the JSON envelope `{status, message, path}` via each class's `status_code`. New error kind → new class in `exceptions.py`.
- **Models** live in `db/models.py` and are accessed through the `models` namespace object (`from db.models import models`; `models.User`). A new model must also be added to the `Models` class at the bottom of that file.
- **Schemas** (Pydantic v2, `from_attributes=True` for responses) are in `schema/schema.py`.
- Settings: `config.py` (pydantic-settings, `.env`, env vars win). `ENVIRONMENT=development` runs `create_all` on startup; anything else relies on Alembic.

## Gotchas

- Declare literal paths before parameterised ones in a router (`/me` before `/{user_id}`).
- Login is OAuth2 password form: the `username` field carries the **email**.
- Alembic reads the DB URL from `config.settings`; `render_as_batch=True` keeps column changes working on SQLite.
- SQLite by default; for Postgres `uv add asyncpg` and set `DATABASE_URL=postgresql+asyncpg://...` — no code changes.
- Relationships are async: eager-load with `selectinload` in services instead of touching lazy attributes.

## Tests

`tests/conftest.py` sets `SECRET_KEY`/`ENVIRONMENT` before importing the app, gives each test a fresh in-memory SQLite (`db_session`, `StaticPool`) injected via `app.dependency_overrides`, and provides `client` (httpx `ASGITransport`) and `auth_headers`. `pytest-asyncio` runs in auto mode, so test functions are plain `async def`. Service rules are tested directly in `tests/test_service.py`.
