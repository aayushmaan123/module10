# Module 10 — Secure User Model, Pydantic Validation, DB Testing & Docker Deployment

A FastAPI application with a secure SQLAlchemy `User` model, Pydantic
request/response schemas, bcrypt password hashing, a full unit + integration
test suite backed by PostgreSQL, and a GitHub Actions CI/CD pipeline that tests,
scans, and pushes a Docker image to Docker Hub.

## Docker Hub

- **Image:** [`aayushrox007/module10_is601`](https://hub.docker.com/r/aayushrox007/module10_is601)

Pull and run:

```bash
docker pull aayushrox007/module10_is601:latest
docker run -p 8000:8000 aayushrox007/module10_is601:latest
```

App serves at http://localhost:8000 (interactive docs at `/docs`).

---

## Features

| Area | What it does |
|------|--------------|
| SQLAlchemy `User` model | `username`, `email`, bcrypt `password`, UUID PK, unique constraints on email + username, `created_at`/`updated_at` timestamps |
| Pydantic schemas | `UserCreate` (input, validates email + password rules), `UserResponse` (output, omits password hash), `Token`, `UserLogin` |
| Password security | `passlib` bcrypt — `hash_password` / `verify_password`; plaintext never stored |
| Auth | JWT access-token creation + verification |
| Tests | Unit (hashing, schema validation) + integration against a real Postgres DB (uniqueness, invalid email, register/authenticate) |
| CI/CD | GitHub Actions: test → Trivy scan → push to Docker Hub |

---

## Running Tests Locally

### 1. Prerequisites
- Python 3.10+
- Docker (for a local Postgres) **or** a local PostgreSQL server

### 2. Create a virtual environment & install deps

```bash
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Start PostgreSQL

Easiest — a throwaway container:

```bash
docker run -d --name m10_pg \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=fastapi_db \
  -p 5432:5432 postgres:latest
```

> If port 5432 is already used by a local Postgres, map `-p 5433:5432` and set
> `DATABASE_URL` to use port `5433` (see below).

### 4. Point the app at the database

The app reads `DATABASE_URL` (default in `app/config.py`:
`postgresql://postgres:postgres@localhost:5432/fastapi_db`).

```bash
export DATABASE_URL="postgresql://postgres:postgres@localhost:5432/fastapi_db"
# Windows PowerShell: $env:DATABASE_URL="postgresql://postgres:postgres@localhost:5432/fastapi_db"
```

### 5. Run the tests

```bash
# Unit + integration
pytest tests/unit tests/integration

# Everything (adds Playwright e2e — requires: playwright install)
pytest

# With coverage report (configured in pytest.ini)
pytest --cov=app --cov-report=term-missing
```

Unit + integration tests do **not** require Playwright. The e2e suite
(`tests/e2e/`) needs browsers installed via `playwright install`.

---

## Running the App Locally

```bash
# With Docker Compose (app + Postgres + pgAdmin)
docker compose up

# Or directly (Postgres must be running)
uvicorn main:app --reload
```

---

## CI/CD Pipeline

`.github/workflows/test.yml` runs on every push/PR to `main`:

1. **test** — spins a Postgres service container, installs deps, runs unit,
   integration, and e2e tests.
2. **security** — builds the image and scans it with Trivy (fails on
   CRITICAL/HIGH vulnerabilities).
3. **deploy** — on `main`, logs into Docker Hub and pushes a multi-arch image.

### Required repository secrets
Add these under **Settings → Secrets and variables → Actions**:

| Secret | Value |
|--------|-------|
| `DOCKERHUB_USERNAME` | your Docker Hub username |
| `DOCKERHUB_TOKEN` | a Docker Hub access token (Account Settings → Security) |

---

## Reflection

See [`reflection.md`](./reflection.md).
