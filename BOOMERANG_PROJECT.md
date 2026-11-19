# Boomerang — Smart Campus Lost & Found System

## Project Overview

**Boomerang** is a modern, privacy-first web application designed for university campuses to streamline the reporting, matching, verification, and recovery of lost personal belongings. By replacing disorganized campus bulletin boards and chat groups with automated algorithmic matching and zero-knowledge claim verification, Boomerang safeguards student privacy while maximizing recovery rates.

---

## Central Repository & Live Deployment

- **Main Project Repository:** [https://github.com/HNAnirudh06/Boomerang](https://github.com/HNAnirudh06/Boomerang)
- **Live Production URL:** [https://boomerang-9gwe.onrender.com](https://boomerang-9gwe.onrender.com)
- **Upstream Verified Commit:** `0e94b4315d05b51a552d3433d2700905350dacbe`
- **Application Architecture:** Flask 3.0 / Python 3.11 / PostgreSQL 16 (Render & Neon) / Gunicorn WSGI

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
   - Continuous Integration / Continuous Deployment (CI/CD) pipeline design.
   - GitHub Actions multi-stage build matrix: code linting (Flake8), automated testing (Pytest), security audit, and automated deploy hooks.

2. [docs/DEPLOYMENT_GUIDE.md](docs/DEPLOYMENT_GUIDE.md)
   - Step-by-step production deployment guide for Render (PaaS) and Neon Serverless PostgreSQL.
   - Environment variables specification, database connection pooling, and zero-downtime database migration handling with Flask-Migrate / Alembic.

3. [docs/TESTING_STRATEGY.md](docs/TESTING_STRATEGY.md)
   - Comprehensive test engineering framework: Unit tests, integration tests, claim verification logic, and notification deduplication tests.
   - Test suites breakdown: 180 passing automated test cases with 0 Flake8 violations.

4. [.github/workflows/validate-docs.yml](.github/workflows/validate-docs.yml)
   - Automated workflow validating repository hygiene, markdown formatting, and pipeline configuration integrity.

---

## Quality Metrics & Test Summary

| Metric | Target | Status |
| :--- | :--- | :--- |
| **Flake8 Compliance** | 0 violations (PEP 8) | **PASSED** (0 violations) |
| **Unit & Integration Suite** | 100% pass rate | **180 passed**, 3 postgres-specific deselected |
| **Claims Workflow Tests** | Zero visibility bugs | **23 passed** |
| **Matching Engine Tests** | Strict threshold adherence | **21 passed** |
| **Notification Tests** | Duplicate suppression | **PASSED** |
