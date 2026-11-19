# Boomerang — Production Cloud Deployment Guide (Render + Neon)

## 1. Cloud Architecture Overview

Boomerang's live production environment is hosted on **Render** as a web service backed by a managed serverless **Neon PostgreSQL 16** database instance.

```mermaid
flowchart LR
    subgraph PublicInternet ["Public Internet"]
        Users["Campus Users (HTTPS)"]
    end

    subgraph RenderPlatform ["Render Cloud Platform"]
        LoadBalancer["SSL / Reverse Proxy\n(ProxyFix: TRUSTED_PROXY_COUNT=1)"]
        WebService["Boomerang Web Service\n(Gunicorn WSGI / Python 3.11)\nHealth Check: /health"]
    end

    subgraph StorageServices ["Image Storage Options"]
        Cloudinary["Cloudinary (Render Free-Tier Persistent Storage)\nIMAGE_STORAGE_BACKEND=cloudinary"]
        R2["Cloudflare R2 (S3-Compatible Object Storage)\nIMAGE_STORAGE_BACKEND=r2"]
        LocalDisk["Ephemeral Container Disk (Reset on redeploy)\nIMAGE_STORAGE_BACKEND=local"]
    end

    subgraph NeonCloud ["Neon Serverless Cloud"]
        PgBouncer["Connection Pooler (Port 5432 / SSL require)"]
        PostgresDB[("PostgreSQL 16 Engine\nAdvisory Locks & Immutability Triggers")]
    end

    Users -->|HTTPS| LoadBalancer
    LoadBalancer -->|WSGI| WebService
    WebService -->|psycopg3 pooled connection| PgBouncer
    PgBouncer --> PostgresDB
    WebService -.->|Uploads & Signed Images| Cloudinary
```

---

## 2. Environment Variables Configuration

The following environment variables configure the production instance on Render:

| Variable Key | Required | Value / Example | Description |
| :--- | :---: | :--- | :--- |
| `BOOMERANG_ENV` | Yes | `production` | Enables production hardening (forces DEBUG off, enables strict secrets) |
| `SECRET_KEY` | Yes | `<64-character-random-hex>` | Must be non-default and at least 16 characters for cryptographic session security |
| `DATABASE_URL` | Yes | `postgresql://user:pass@ep-xyz.neon.tech/boomerang?sslmode=require` | Neon PostgreSQL URI (automatically normalized to `postgresql+psycopg://`) |
| `PORT` | Auto | `10000` (Injected by Render) | Dynamic port evaluated by Gunicorn listener |
| `IMAGE_STORAGE_BACKEND` | Yes | `cloudinary` (or `local` / `r2`) | Storage backend strategy for uploaded item photos |
| `TRUSTED_HOSTS` | Yes | `boomerang-9gwe.onrender.com` | Rejects unauthorized Host headers to prevent cache poisoning |
| `TRUSTED_PROXY_COUNT` | Yes | `1` | Configures Werkzeug ProxyFix to safely extract HTTPS scheme and client IP |
| `SESSION_COOKIE_SECURE` | Yes | `true` | Enforces HTTPS-only delivery of session cookies |
| `CLOUDINARY_CLOUD_NAME` | If cld | `<cloud-name>` | Cloudinary account identifier for persistent storage |
| `CLOUDINARY_API_KEY` | If cld | `<api-key>` | Cloudinary API key |
| `CLOUDINARY_API_SECRET` | If cld | `<api-secret>` | Cloudinary API secret |
| `CLOUDINARY_FOLDER` | If cld | `boomerang` | Namespace folder for uploaded campus photos |
| `RATE_LIMIT_ENABLED` | No | `true` | Enables proxy-aware sliding-window rate limiting |

---

## 3. Database Schema Initialization & Migrations

Boomerang utilizes an **atomic application startup schema initializer** (`initialize_database_schema` in `models.py`) rather than requiring manual migration CLIs (such as Alembic or Flask-Migrate).

### Startup Schema Flow
1. **Advisory Lock Acquisition:** On PostgreSQL, the application acquires a transaction-scoped advisory lock:
   ```sql
   SELECT pg_advisory_xact_lock(POSTGRES_INIT_LOCK_ID);
   ```
   This prevents concurrent Gunicorn workers from executing racing DDL operations during container startup.
2. **Table Creation:** Executes SQLAlchemy `metadata.create_all(bind=conn, checkfirst=True)`.
3. **Idempotent Column Migrations:** Executes safe, non-destructive schema migrations:
   ```sql
   ALTER TABLE item ADD COLUMN IF NOT EXISTS contact_number VARCHAR(30);
   ```
4. **Immutability Trigger Installation:** Installs database-level triggers enforcing permanent immutability on completed `exchange_record` transactions.

---

## 4. Web Service Build & Launch Specification

- **Build Command:**
  ```bash
  pip install --upgrade pip && pip install -r requirements.txt
  ```
- **Start Command:**
  ```bash
  gunicorn --config gunicorn.conf.py app:app
  ```
- **Health Check Path:**
  ```
  /health
  ```
  *(Render continuously probes `/health`, which executes a database `SELECT 1` ping. Traffic is routed only when HTTP 200 is returned)*

---

## 5. College Expo Demonstration Runbook (Cloudflare Quick Tunnel)

For campus demos where local laptop databases and images must be shared without deploying to the cloud:
1. Ensure both Docker containers are running:
   ```powershell
   docker compose up -d --build
   ```
2. Verify local health check:
   ```powershell
   Invoke-RestMethod http://localhost:8000/health
   # Expected: @{status=ok}
   ```
3. Launch Cloudflare Quick Tunnel in a separate terminal:
   ```powershell
   cloudflared tunnel --url http://localhost:8000
   ```
4. Access the generated `https://<random-words>.trycloudflare.com` URL across demo laptops on campus Wi-Fi.
