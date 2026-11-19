# Boomerang — Continuous Integration (CI) Pipeline Architecture

## 1. CI/CD Architecture Overview

The Boomerang engineering pipeline employs GitHub Actions (`.github/workflows/ci.yml`) to establish an automated, zero-regression software delivery lifecycle. Every push and pull request targeting `main` or `feature/**` branches undergoes three parallel and staged validation jobs.

```mermaid
flowchart TD
    subgraph Trigger ["Code Triggers"]
        PR["Pull Request -> main, feature/**"]
        Push["Push -> main, feature/**"]
    end

    subgraph CIJobs ["GitHub Actions CI Pipeline (.github/workflows/ci.yml)"]
        subgraph Job1 ["Job 1: Lint & Unit Tests (SQLite)"]
            Flake8["Flake8 (PEP 8 Compliance)"]
            PytestSQLite["Pytest: -m 'not postgres' (181 Tests)"]
        end

        subgraph Job2 ["Job 2: PostgreSQL 16 Integration"]
            PGService[("PostgreSQL 16 Service Container\npg_isready healthcheck")]
            PytestPG["Pytest: Full Suite with TEST_DATABASE_URL"]
        end

        subgraph Job3 ["Job 3: Docker Build & Hygiene Verification"]
            DockerBuild["Docker Build (python:3.11-slim)"]
            HygieneCheck["Container Hygiene:\n- .env exclusion verified\n- Non-root user (appuser:1000)\n- Dev tools (pytest/flake8) absent"]
        end
    end

    subgraph Deployment ["Continuous Deployment (Render)"]
        RenderDeploy["Render Web Service (Auto-deploy on main push)\nProduction Container Image & Health Probing"]
    end

    Push --> CIJobs
    PR --> CIJobs
    Push -->|Merged to main| RenderDeploy
```

---

## 2. GitHub Actions Job Specifications

The pipeline is defined in `.github/workflows/ci.yml` and comprises three core jobs:

### Job 1: `lint-and-sqlite-tests`
- **Runner:** `ubuntu-latest` (Python 3.11)
- **Execution Steps:**
  1. Checks out repository using `actions/checkout@v4`.
  2. Sets up Python 3.11 with pip caching on `boomerang/requirements*.txt`.
  3. Installs dependencies from `boomerang/requirements-dev.txt`.
  4. Runs code style enforcement:
     ```bash
     python -m flake8 .
     ```
  5. Executes test suite excluding PostgreSQL-dependent tests:
     ```bash
     python -m pytest -m "not postgres"
     ```

### Job 2: `postgres-integration-tests`
- **Runner:** `ubuntu-latest` with a Dockerized PostgreSQL 16 service container.
- **Service Configuration:**
  - Image: `postgres:16`
  - Health check: `pg_isready -U boomerang_test -d boomerang_test`
- **Execution Steps:**
  - Injects `TEST_DATABASE_URL=postgresql+psycopg://boomerang_test:boomerang_test_password@localhost:5432/boomerang_test`.
  - Executes full test suite verifying transaction advisory locks, PostgreSQL schema triggers, and real relational queries:
    ```bash
    python -m pytest -v
    ```

### Job 3: `docker-build`
- **Objective:** Verifies container buildability and guarantees sensitive environment variables do not leak into images.
- **Execution Steps:**
  1. Generates a temporary local `.env` file to validate `.dockerignore` enforcement.
  2. Builds the Docker runtime image:
     ```bash
     docker build -t boomerang-ci:test -f boomerang/Dockerfile boomerang
     ```
  3. Executes container assertions verifying:
     - `/app/.env` does **not** exist in the container.
     - `/app/.env.example` is present.
     - Process runs as non-root (`test "$(id -u)" -ne 0`).
     - `/app/static/uploads` is writable by `appuser`.
     - Development packages (`pytest`, `flake8`) are absent from the runtime image.
  4. Cleans up the synthetic `.env` file.

---

## 3. GitHub Actions Workflow Configuration

Below is the complete production workflow definition from `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches:
      - main
      - "feature/**"
  pull_request:
    branches:
      - main
      - "feature/**"

permissions:
  contents: read

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint-and-sqlite-tests:
    name: Lint & Unit Tests (SQLite)
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"
          cache-dependency-path: boomerang/requirements*.txt

      - name: Upgrade pip
        run: python -m pip install --upgrade pip

      - name: Install development and runtime dependencies
        working-directory: boomerang
        run: python -m pip install -r requirements-dev.txt

      - name: Run flake8
        working-directory: boomerang
        run: python -m flake8 .

      - name: Run pytest suite (SQLite)
        working-directory: boomerang
        run: python -m pytest -m "not postgres"

  postgres-integration-tests:
    name: PostgreSQL 16 Integration Tests
    runs-on: ubuntu-latest
    timeout-minutes: 15

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: boomerang_test
          POSTGRES_USER: boomerang_test
          POSTGRES_PASSWORD: boomerang_test_password
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U boomerang_test -d boomerang_test"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"
          cache-dependency-path: boomerang/requirements*.txt

      - name: Upgrade pip
        run: python -m pip install --upgrade pip

      - name: Install development and runtime dependencies
        working-directory: boomerang
        run: python -m pip install -r requirements-dev.txt

      - name: Run PostgreSQL 16 integration and full regression tests
        working-directory: boomerang
        env:
          TEST_DATABASE_URL: postgresql+psycopg://boomerang_test:boomerang_test_password@localhost:5432/boomerang_test
        run: python -m pytest -v

  docker-build:
    name: Docker Build & Hygiene Verification
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Create synthetic local .env to test .dockerignore exclusion
        run: |
          printf 'SECRET_KEY=ci-dummy-not-a-secret\n' > boomerang/.env

      - name: Build Docker runtime image (no push)
        run: docker build -t boomerang-ci:test -f boomerang/Dockerfile boomerang

      - name: Verify .dockerignore, non-root user, and clean runtime image
        run: |
          docker run --rm --entrypoint sh boomerang-ci:test -c '
            test ! -f /app/.env
            test -f /app/.env.example
            test "$(id -u)" -ne 0
            test -w /app/static/uploads
            ! python -c "import pytest" 2>/dev/null
            ! python -c "import flake8" 2>/dev/null
          '

      - name: Remove synthetic local .env
        if: always()
        run: rm -f boomerang/.env
```

---

## 4. Production Deployment on Render

Once all CI jobs succeed on the `main` branch:
1. Render automatically triggers a deployment webhook configured for `HNAnirudh06/Boomerang`.
2. The web service builds the container and runs the application.
3. Database schemas and columns are initialized atomically at startup via `initialize_database_schema`.
4. Render monitors the `/health` endpoint until HTTP 200 is confirmed before routing production traffic.
