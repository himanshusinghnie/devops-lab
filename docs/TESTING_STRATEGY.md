# Boomerang — Testing Strategy & Quality Assurance Framework

## 1. Testing Philosophy & Test Suite Architecture

The Boomerang test suite is designed to guarantee high reliability, prevent privacy regressions, and ensure rock-solid claim authorization. It employs **Pytest** with modular fixtures, in-memory SQLite isolation for fast local and CI unit execution, and a dedicated PostgreSQL 16 service container for full relational integration testing.

---

## 2. Algorithmic Matching vs Claim Verification Specifications

A critical architectural distinction in Boomerang is the separation between **Candidate Matching** and **Claim Verification**:

### 2.1 Algorithmic Matching Engine (`matching.py`)
- Discover potential matches between opposite item types (`lost` vs `found`).
- Evaluates pairs using `MatchFeatureBuilder` extracting a 6-dimensional feature vector:
  1. `text_similarity`: TF-IDF cosine similarity between title and description.
  2. `image_similarity`: Cosine similarity of ResNet-50 visual embeddings (with deterministic color/spatial fallback when PyTorch is unavailable).
  3. `category_match`: Exact category match binary indicator ($1.0$ if matching, $0.0$ otherwise).
  4. `location_similarity`: Token-based Jaccard similarity and substring containment check ($\ge 0.6$ if substring matches).
  5. `date_diff_days`: Normalized date difference $\min(\text{days}, 30) / 30.0$.
  6. `has_both_images`: Binary flag ($1.0$ if both items possess image embeddings, $0.0$ otherwise).
- **Classification Model:** `RandomForestClassifier(n_estimators=100, max_depth=6, random_state=42)`.
- **Match Filtering Threshold:** `min_confidence = 0.35` (35% probability threshold). Candidate items scoring $\ge 35\%$ confidence are ranked in descending order, returning the top $N$ candidates (default 5).

### 2.2 Claim Verification & Dynamic 60% Threshold (`verification.py`)
- Once a claimant submits a claim, the finder evaluates the claimant's answers against 6 specific verification questions.
- **Dynamic 60% Threshold Formula:**
  $$\text{threshold\_required} = \lceil 0.60 \times \text{total\_questions} \rceil$$
  *(Implemented as `math.ceil(0.60 * total_questions)`)*
  - 6 questions: $\lceil 0.60 \times 6 \rceil = \lceil 3.6 \rceil = 4$ correct answers required.
  - 7 questions: $\lceil 0.60 \times 7 \rceil = \lceil 4.2 \rceil = 5$ correct answers required.
- Evaluates finder review marks: `meets_threshold = (correct_answers_count >= threshold_required)`.

---

## 3. Test Suites Breakdown (184 Total Collected Tests)

The automated test suite spans 15 specialized test modules:

| Test File | Tests | Focus Area |
| :--- | :---: | :--- |
| `test_app.py` | 13 | App factory, health endpoint (`/health`), port binding, and database URI normalization |
| `test_auth.py` | 14 | Registration, USN uniqueness, password hashing, and role isolation |
| `test_claims.py` | 23 | 60% threshold formula, finder claim visibility fix, and mutual consent disclosures |
| `test_cloudinary_storage.py`| 14 | Cloudinary upload, error handling, signed URL generation, and fallback isolation |
| `test_csrf.py` | 5 | CSRF tokens on all POST forms and protection across student workflows |
| `test_hardening_corrections.py`| 7 | Permanent exchange showcase authorization and trigger installation idempotency |
| `test_items.py` | 23 | Lost/Found item reporting, mandatory contact number normalization, and catalog filtering |
| `test_matching.py` | 21 | Random Forest classifier training, feature vector extraction, and candidate ranking |
| `test_negative_and_edge_cases.py`| 12 | End-to-end claim rejection, one-sided consent states, and handed-over item locks |
| `test_phase12_bugfixes.py` | 8 | Regression tests for critical workflow bugfixes |
| `test_postgres_integration.py` | 3 | Real PostgreSQL 16 transaction advisory locks, triggers, and migrations (`@pytest.mark.postgres`) |
| `test_private_matching.py` | 14 | Private candidate matching, zero duplicate notifications, and contact confidentiality |
| `test_production_hardening.py` | 18 | Rate limiting, ProxyFix, trusted hosts, safe 500 pages, and Gemini vision parsing |
| `test_storage.py` | 14 | Local disk and Cloudflare R2 object storage drivers and path traversal guards |
| `test_timestamps.py` | 7 | UTC to IST timezone conversions, midnight boundary handling, and template filters |

---

## 4. Test Execution Reference

### Running the Standard SQLite Test Suite
```bash
python -m pytest -m "not postgres" -v
```
*Result: `181 passed, 3 deselected in 79.6s`*

### Running Targeted Claims Tests
```bash
python -m pytest tests/test_claims.py -v
```
*Result: `23 passed in 2.1s`*

### Running Code Style & Hygiene Check
```bash
python -m flake8 .
```
*Result: `0 violations across all modules`*
