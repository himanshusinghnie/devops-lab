# Boomerang — Smart Campus Lost & Found System

## Project Overview

**Boomerang** is a modern, privacy-first web application designed for university campuses to streamline the reporting, matching, verification, and recovery of lost personal belongings. By replacing disorganized campus bulletin boards and chat groups with machine learning-driven candidate matching and zero-knowledge claim verification, Boomerang safeguards student privacy while maximizing recovery rates.

---

## Central Repository & Live Deployment

- **Main Project Repository:** [https://github.com/HNAnirudh06/Boomerang](https://github.com/HNAnirudh06/Boomerang)
- **Live Production URL:** [https://boomerang-9gwe.onrender.com](https://boomerang-9gwe.onrender.com)
- **Upstream Verified Commit:** `0e94b4315d05b51a552d3433d2700905350dacbe`
- **Application Framework:** Flask 3.0 / Python 3.11 / PostgreSQL 16 (Render & Neon) / Gunicorn WSGI (`gunicorn.conf.py`)

---

## Academic & Team Context

- **Institution:** The National Institute of Engineering (NIE), Mysuru
- **Department:** Department of Computer Science & Engineering (CSE)
- **Course:** DevOps Laboratory / CDD Minor Project
- **Contributor:** **Himanshu Singh**
- **Module Ownership:** **CI/CD Automation, Testing Framework & Cloud Deployment Guide**

---

## Core Contribution Modules

This repository houses the DevOps automation and verification assets for the Boomerang project:

1. [docs/CICD_PIPELINE.md](docs/CICD_PIPELINE.md)
   - Continuous Integration (CI) pipeline design based on `.github/workflows/ci.yml`.
   - GitHub Actions multi-job architecture: Flake8 linting, SQLite unit test matrix, PostgreSQL 16 integration tests, and Docker runtime validation.

2. [docs/DEPLOYMENT_GUIDE.md](docs/DEPLOYMENT_GUIDE.md)
   - Step-by-step production deployment guide for Render (PaaS) and Neon Serverless PostgreSQL.
   - Atomic database schema initialization (`pg_advisory_xact_lock`), connection pool recycling, environment variables, and Cloudflare Quick Tunnel expo runbook.

3. [docs/TESTING_STRATEGY.md](docs/TESTING_STRATEGY.md)
   - Comprehensive test engineering framework: Unit tests, claim verification logic, Random Forest matching engine features, and storage backends.
   - Test suite breakdown: 181 passing automated tests (3 PostgreSQL-specific tests deselected when running on SQLite) and 0 Flake8 violations.

4. [.github/workflows/validate-docs.yml](.github/workflows/validate-docs.yml)
   - Automated workflow validating repository hygiene, markdown formatting, and pipeline configuration integrity.

---

## Quality Metrics & Test Summary

| Metric | Target | Actual Verified Status |
| :--- | :--- | :--- |
| **Flake8 Compliance** | 0 violations (PEP 8) | **PASSED** (0 violations across all modules) |
| **Non-PostgreSQL Test Suite** | 100% pass rate on SQLite | **181 passed**, 3 deselected in 79.6s |
| **Total Collected Test Cases** | Complete test coverage | **184 total tests** across 15 test suites |
| **Claims & Verification Suite** | Zero visibility bugs | **23 passed** (`test_claims.py`) |
| **Matching Engine Suite** | Random Forest classification | **21 passed** (`test_matching.py`) |
| **Private Matching & Notifications**| Duplicate alert suppression | **14 passed** (`test_private_matching.py`) |
| **Item Catalog & Contact Suite**| Mandatory contact validation | **23 passed** (`test_items.py`) |
