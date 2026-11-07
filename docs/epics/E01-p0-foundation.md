# Epic E01 — P0 Foundation

**Milestone:** P0 (README §19). **Owner:** you. **Reviewer:** Claude (project lead).
**Exit criteria (from the spec):** a clean checkout can start the DB and the API, and the tests pass.

**Estimated effort:** 1–2 focused sessions. Tasks are ordered, and each one builds on the previous one.

---

## How we work

### Workflow per task
1. **Read the task** and its acceptance criteria (AC) before writing code. Ask questions early.
2. **Branch:** `git switch -c e01/t3-liveness` (one branch per task).
3. **Test first where it makes sense.** Write the failing test, then make it pass.
4. **Small commits** with clear messages: `feat(api): add liveness route`, `test: cover readiness 503`, `chore: add ruff config`.
5. **Self-check** against the AC and the review checklist below.
6. **Request a review:** tell me `review E01-T3`. I'll read the diff, run the checks, and reply with:
   - **Must fix**: correctness, security, or a broken AC. This blocks the task.
   - **Should fix**: maintainability or clarity. Fix it now or log a follow-up.
   - **Nit**: style or preference. Optional.
7. **Merge** to `main` locally once all must-fix items are resolved. Nothing gets pushed without an explicit decision.

### Asking for help
- `hint E01-T5`: I give a nudge, not a solution.
- `explain <concept>`: a short explanation tied to this codebase.
- If you're stuck for more than 30 minutes, say so. Being stuck that long is a signal, not a failure.

### Review checklist (applies to every task)
- [ ] All AC are met, and I can verify them with the commands you list.
- [ ] Tests cover the happy path **and** at least one failure path.
- [ ] `ruff check`, `ruff format --check`, `mypy` and `pytest` are clean. No new `# type: ignore` or `noqa` without a comment explaining why.
- [ ] No secrets, `.env` files, or connection strings in code, logs, test output, or commits.
- [ ] Explicit types at boundaries (function signatures, request/response models).
- [ ] No dependencies beyond what the task needs. No later-milestone scaffolding.
- [ ] README commands match reality, and planned commands are marked as planned.
- [ ] You can explain *why* the code is shaped the way it is (I may ask one question).

---

## Tasks

### E01-T0 — Workstation and repository hygiene (~20 min)
**Goal:** a reproducible toolchain without global Python changes.

- Install `uv` (decision pending: user-level installer, or uv inside a conda env).
- `uv python install 3.12` (uv-managed, not system-wide).
- `git init`, with `main` as the default branch.
- Root `.gitignore`: `.venv/`, `__pycache__/`, `.env`, `*.pyc`, `.mypy_cache/`, `.ruff_cache/`, `.pytest_cache/`, `node_modules/`, `dist/`, `*.tfstate*`, `*.tfplan`, `*.pem`, `*.key`.
- Start Docker Desktop.

**AC**
- `uv --version` and `uv run --python 3.12 python --version` both work.
- `git status` shows the repo and `.gitignore` is tracked.
- `git check-ignore .env` prints `.env`.

**Concept:** the difference between a virtual environment and a global install. Where uv stores its Pythons.

---

### E01-T1 — Backend project skeleton (~30 min)
**Goal:** an importable `sentinel` package in a `src/` layout, managed by uv.

- `backend/pyproject.toml` with `requires-python = ">=3.12,<3.13"` and `backend/.python-version` set to `3.12`.
- Package at `backend/src/sentinel/__init__.py`.
- Runtime deps: `fastapi` and `uvicorn` only. Dev deps (dependency group): `pytest`, `httpx`, `ruff`, `mypy`.
- Commit `uv.lock`.

**AC**
- `cd backend && uv sync --locked` succeeds from a fresh clone.
- `uv run python -c "import sentinel"` exits 0.
- `uv.lock` is committed. `.venv/` is not.

**Concepts:** `pyproject.toml` vs `package.json`; why a `src/` layout prevents accidentally importing un-installed code; dependency groups vs runtime deps.

**Hint:** `uv init --package` gets you most of the way. Read what it generated and delete anything you don't need.

---

### E01-T2 — Typed configuration (~40 min)
**Goal:** a single, validated configuration mechanism that fails fast at startup.

- `sentinel/config.py` with a `Settings` class (`pydantic-settings`) containing at least:
  - `environment`: `Literal["dev", "test"]`
  - `database_url`: required, with **no default**
  - `db_connect_timeout_seconds`: int with bounds (e.g. 1–10), default 2
- `.env.example` at the repo root with placeholder values only, never real passwords.
- A `get_settings()` function so tests can override configuration.

**AC**
- Starting with `DATABASE_URL` missing raises a clear validation error. A test asserts this.
- An out-of-range timeout is rejected. A test asserts this.
- Settings are never printed or logged in full. If you add a `__repr__`, the DSN is masked.

**Concepts:** Pydantic v2 validation, `Literal` types, why configuration validation belongs at startup and not at first use. A `.env` file is a dev convenience and not a secret store.

---

### E01-T3 — App factory and liveness route (~40 min)
**Goal:** the first HTTP endpoint and the first passing test.

- `sentinel/main.py` with `create_app() -> FastAPI`.
- `sentinel/api/health.py`: an `APIRouter` mounted at `/api/v1/health`.
- `GET /api/v1/health/live` returns `200 {"status": "ok"}` with no dependency details.
- `tests/unit/test_health.py` using `fastapi.testclient.TestClient`.

**AC**
- `uv run pytest` passes.
- `uv run uvicorn sentinel.main:create_app --factory --reload` serves the route, and `curl localhost:8000/api/v1/health/live` returns the JSON above.
- A request to an unknown route returns 404, covered by a test.
- The response uses an explicit Pydantic response model.

**Concepts:** modules and packages and how imports resolve; the app-factory pattern and why it makes tests easier; `def` vs `async def` in FastAPI (use plain `def` here and be able to explain why).

---

### E01-T4 — Local PostgreSQL with Compose (~30 min)
**Goal:** a disposable local database you can start and stop safely.

- Root `compose.yaml` with a `db` service using `postgres:16`. Pin a specific minor tag.
- Named volume for data. Port bound to **`127.0.0.1:5432`**, not `0.0.0.0`.
- Password supplied from `.env` with **no default in the compose file** (use `${POSTGRES_PASSWORD:?...}` syntax).
- A `healthcheck` using `pg_isready`.

**AC**
- `docker compose up -d db` reaches `healthy` (check with `docker compose ps`).
- `docker compose down` keeps the data, and `docker compose up` again shows it's still there.
- Running without `POSTGRES_PASSWORD` set fails with your error message.
- No password appears in `compose.yaml` or in git.

**Concepts:** containers vs volumes (data lifetime); healthchecks vs "the container started"; why we bind to localhost.

---

### E01-T5 — Readiness route with a real dependency check (~60 min)
**Goal:** honest readiness: report 503 when the DB is unreachable.

- Add runtime dep `psycopg[binary]`. **No SQLAlchemy yet**, which comes in P1.
- A small function such as `check_database(settings) -> bool` that opens a connection with the configured connect timeout, runs `SELECT 1`, and closes it.
- `GET /api/v1/health/ready` returns `200 {"status": "ready"}` or `503 {"status": "unavailable"}`.
- The 503 body and logs must **not** include the DSN, host, password, or the raw exception text.

**AC**
- **Unit tests** (no DB needed): override the check via FastAPI dependency overrides so you can test both 200 and 503 deterministically.
- **Integration test** marked `@pytest.mark.integration`: hits real Postgres from T4 and gets 200. It's skipped by default and run with `pytest -m integration`.
- With the DB stopped, `curl` returns 503 within roughly the connect timeout and doesn't hang.

**Concepts:** liveness vs readiness (and why orchestrators care); dependency injection with FastAPI `Depends`; timeouts as a safety property; unit vs integration tests and pytest markers; why we use a context manager (`with`) for connections.

---

### E01-T6 — Quality gates and Makefile (~40 min)
**Goal:** one command per routine task, and consistent lint and type checks.

- Ruff configuration in `pyproject.toml` (lint and format). Pick a sensible rule set, such as `E`, `F`, `I`, `B`, `UP`, and be able to explain each.
- mypy configuration: `strict = true` for `src/`. If something can't pass, fix it rather than ignoring it.
- Root `Makefile` with **only working targets**: `setup`, `dev` (DB plus API), `test`, `test-integration`, `lint`, `stop`.

**AC**
- `make setup && make lint && make test` passes from a clean clone.
- `make stop` stops containers **without** deleting the volume.
- Each target has a one-line comment, and `make help` (optional) lists them.

**Concepts:** why linters and type checkers catch different bugs; `strict` mypy and gradual typing; Makefile basics (`.PHONY`, tabs).

---

### E01-T7 — Frontend shell (~45 min)
**Goal:** a minimal React/TS app that proves the same-origin `/api` path works.

- `frontend/` created with Vite's `react-ts` template. No UI libraries, router, or state library yet.
- Vite dev-server proxy: `/api` points to `http://127.0.0.1:8000`.
- One page that calls `/api/v1/health/live` and shows **loading**, **ok**, and **error** states.

**AC**
- `npm ci && npm run build` succeeds, and `package-lock.json` is committed.
- With the API running, the page shows "ok". With the API stopped, it shows an error state and doesn't crash.
- No CORS configuration was added to the backend, because the proxy makes it unnecessary. Be able to explain why.

**Concepts:** same-origin vs CORS; why the proxy matters later for cookies (P2).

---

### E01-T8 — Getting-started docs (~20 min)
**Goal:** someone else could set this up from the README alone.

- Add a short "Getting started (P0)" section to the README with prerequisites, `.env` setup from `.env.example`, and the `make` commands.
- Mark the other §20 commands (`migrate`, `seed`, `demo`, `reset-dev`) as **planned**.

**AC**
- Following only the README on a fresh clone gives passing tests and a 200 from `/health/live` (I'll verify).
- No generic tutorials. Keep to what's specific to this project.

---

## Stretch tasks (optional)
- **S1 — Version endpoint:** `GET /api/v1/health/version` returns the package version read via `importlib.metadata`. Test it.
- **S2 — CI draft:** a GitHub Actions workflow running lint, unit tests and integration tests (with a Postgres service). Write it locally only. Pushing and enabling it is a separate decision.
- **S3 — ADR 0001:** a short `docs/decisions/0001-uv-for-python-tooling.md` covering context, alternatives (pip+venv, conda), and consequences.

## Explicitly out of scope for E01
SQLAlchemy, Alembic, tables, seed data, auth, demo services, AI, workers, Terraform, any cloud resources.

## Epic done when
- [ ] T0–T8 merged with all must-fix review items resolved.
- [ ] `make setup && make lint && make test` is green on a clean clone, and `make test-integration` is green with Docker up.
- [ ] You can explain: venv vs global install, app factory, liveness vs readiness, why `def` not `async def`, why `127.0.0.1` binding.

**Next epic preview (E02 / P1):** SQLAlchemy models, the first Alembic migration, tenants, users, resources and incidents, a seed command, and SQL exercises 1–3.
