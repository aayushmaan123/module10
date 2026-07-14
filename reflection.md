# Module 10 Reflection — Secure User Model, Validation, DB Testing & Docker Deployment

## Overview

This module extended the FastAPI application with a secure user model backed by
SQLAlchemy, request/response validation with Pydantic, password hashing, a full
suite of unit + integration tests running against a real PostgreSQL database, and
a GitHub Actions CI/CD pipeline that tests, scans, and pushes a Docker image to
Docker Hub.

## What I Built / Learned

### Secure User Model (SQLAlchemy)
- Defined the `User` model with `username`, `email`, and `password` (stored as a
  bcrypt hash), each with the correct uniqueness constraints on `email` and
  `username`.
- Added lifecycle timestamps (`created_at`, `updated_at`) and account flags
  (`is_active`, `is_verified`, `last_login`), using a UUID primary key.
- Learned that the DB uniqueness constraint is the real guarantee — application
  checks are a UX convenience, but the constraint is what prevents duplicates
  under concurrency.

### Pydantic Validation
- `UserCreate` accepts new-user input (names, email, username, password) and
  enforces password rules (length, upper/lower/digit) plus `EmailStr` validation.
- `UserResponse` returns user data while **omitting the password hash**, which is
  the whole point of separating input and output schemas.
- `from_attributes=True` (ORM mode) lets a response schema be built directly from
  a SQLAlchemy row.

### Password Hashing
- Used `passlib` with bcrypt: `hash_password` on write, `verify_password` on
  login. Plaintext passwords are never stored or logged.

### Testing (the hardest part)
- Unit tests cover hashing and schema validation in isolation.
- Integration tests run against a **real Postgres container**, exercising
  uniqueness violations, invalid emails, and full register/authenticate flows.
- The session-scoped autouse fixture drops and recreates tables for a clean
  slate; per-test truncation keeps tests isolated. Understanding fixture scope
  was key to avoiding cross-test data leaks.

### CI/CD & Docker
- The GitHub Actions workflow runs three stages: **test** (with a Postgres
  service container), **security** (Trivy image scan, fails on CRITICAL/HIGH),
  and **deploy** (build + push multi-arch image to Docker Hub on `main`).
- Docker Hub credentials are injected via `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN`
  repository secrets — never hardcoded.

## Challenges Faced

1. **Test database wiring.** `conftest.py` builds the engine from
   `settings.DATABASE_URL` at import time, so tests fail immediately if Postgres
   isn't reachable. Running a local Postgres container matching the default URL
   fixed this; in CI the Postgres service container plays the same role.

2. **Personalizing the deploy target.** The starter workflow pushed to the
   instructor's Docker Hub repo. I had to repoint the image tags to my own
   Docker Hub namespace and add my own repository secrets, or the deploy step
   would fail (or push nowhere I control).

3. **Schema vs. model duplication.** Password rules live in the Pydantic mixin,
   while length is also checked in `User.register`. Keeping validation layered
   (schema first, DB constraint last) rather than duplicated ad hoc took some
   thought.

4. **Trivy gate.** A CRITICAL/HIGH vulnerability in a base image or dependency
   blocks deploy by design. This reinforced keeping the base image slim and
   dependencies current.

## Takeaways

- Separate input/output schemas are a security control, not boilerplate.
- Integration tests against a real database catch what mocks hide (constraint
  violations, type coercion, migration drift).
- A CI pipeline that gates deploy on tests **and** a vulnerability scan turns
  "it works on my machine" into a repeatable, auditable release.
