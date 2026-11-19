# Boomerang — Production Cloud Deployment Guide (Render + Neon)

## 1. Cloud Architecture Overview

Boomerang's production environment is hosted on **Render** (Cloud PaaS) backed by a managed serverless **Neon PostgreSQL 16** database instance.

```mermaid
flowchart LR
    subgraph PublicInternet ["Public Internet"]
        Users["Campus Users (HTTPS)"]
    end

    subgraph RenderPlatform ["Render Cloud Platform"]
        LoadBalancer["SSL / CDN Edge\n(Automatic Let's Encrypt Cert)"]
        WebService["Boomerang Web Service\n(Gunicorn WSGI / Python 3.11)\nAuto-scaling & Health Probes"]
        PersistentDisk["Persistent Disk\n(/app/uploads - 10GB)"]
    end

    subgraph NeonCloud ["Neon Serverless Cloud"]
        PgBouncer["Connection Pooler / Proxy\n(SSL mode=require)"]
        PostgresDB[("PostgreSQL 16 Engine\nCompute Auto-suspend / Resume")]
    end

    Users -->|HTTPS / Port 443| LoadBalancer
    LoadBalancer -->|Reverse Proxy| WebService
    WebService -->|Persistent Storage| PersistentDisk
    WebService -->|TCP / SSL Pooler (Port 5432)| PgBouncer
    PgBouncer --> PostgresDB
```

---

## 2. Environment Variables Configuration

Configure the following environment variables within the Render Dashboard under **Environment Settings**:

| Variable Key | Required | Example / Recommended Value | Description |
| :--- | :---: | :--- | :--- |
| `FLASK_ENV` | Yes | `production` | Enables production optimizations and disables debug toolbar |
| `FLASK_APP` | Yes | `wsgi.py` | Entry point WSGI module |
| `SECRET_KEY` | Yes | `<64-character-random-hex>` | Cryptographic secret for signing session cookies & CSRF tokens |
| `DATABASE_URL` | Yes | `postgresql://user:pass@ep-xyz.neon.tech/boomerang?sslmode=require` | Neon PostgreSQL pooled connection URI |
| `PORT` | Yes | `10000` (Injected automatically by Render) | Dynamic port for Gunicorn HTTP listener |
| `DB_POOL_SIZE` | No | `10` | Base SQLAlchemy connection pool size |
| `DB_MAX_OVERFLOW` | No | `20` | Maximum temporary connections above pool size |
| `DB_POOL_TIMEOUT` | No | `30` | Seconds to wait before timing out on connection pool exhaustion |
| `DB_POOL_RECYCLE` | No | `1800` | Recycles connections every 30 minutes to prevent stale Neon sockets |
| `UPLOAD_FOLDER` | Yes | `/app/uploads` | Path to persistent disk for item photo uploads |
| `MAX_CONTENT_LENGTH`| No | `5242880` (5 MB) | Strict upload size ceiling |

---

## 3. Pre-Deploy Command & Database Migrations

Render provides a **Pre-Deploy Command** step that executes inside the freshly built container before routing traffic to it.

Set the Pre-Deploy Command to:
```bash
python -m flask db upgrade
```

### Safe Migration Best Practices with Neon:
1. **Advisory Locking:** Flask-Migrate / Alembic obtains transactional DDL locks to prevent race conditions when multiple workers launch simultaneously.
2. **Backward Compatibility:** All schema changes must follow the non-destructive principle (new columns default to `nullable=True` or have default values) so that current running containers continue functioning until traffic swings to the new container.

---

## 4. Web Service Build & Launch Settings

- **Runtime:** Python 3
- **Build Command:**
  ```bash
  pip install --upgrade pip && pip install -r requirements.txt
  ```
- **Start Command:**
  ```bash
  gunicorn --bind 0.0.0.0:$PORT --workers 3 --threads 2 --timeout 60 wsgi:app
  ```
- **Health Check Path:**
  ```
  /health/ready
  ```
  *(Render will only shift incoming web traffic to the new instance when `/health/ready` returns HTTP 200)*

---

## 5. Security & SSL Enforcement

1. **Automatic TLS / SSL:** Render automatically provisions and renews Let's Encrypt SSL/TLS certificates.
2. **Cookie Security Headers:** In `production` mode, the application automatically enforces:
   - `SESSION_COOKIE_SECURE = True`
   - `SESSION_COOKIE_HTTPONLY = True`
   - `SESSION_COOKIE_SAMESITE = 'Lax'`
3. **HTTP Strict Transport Security (HSTS):** Enforced via response headers to prevent man-in-the-middle protocol downgrade attacks.

---

## 6. Post-Deployment Verification Checklist

Once the deployment completes:
- [ ] Verify HTTP 200 response from `https://boomerang-9gwe.onrender.com/health/ready`.
- [ ] Confirm authentication works (User login / session registration).
- [ ] Test Lost Item submission with mandatory contact number.
- [ ] Test Found Item submission with image upload.
- [ ] Verify email/notification delivery without duplicates.
- [ ] Check Render application logs for database connection pooling warnings.
