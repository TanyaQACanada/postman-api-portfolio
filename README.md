# Postman API Testing Portfolio

A senior-level Postman project demonstrating API test design, automation, documentation, and CI readiness. Uses the public [ReqRes](https://reqres.in) API so the project runs without private services.

---

## Project purpose

- Showcase **API test design**: positive, negative, boundary, security, contract, and E2E flows.
- Demonstrate **Postman best practices**: collection/environment variables, pre-request scripts, test scripts, correlation (create → store ID → reuse → cleanup).
- Provide **CI-ready** execution via Newman and sample GitHub Actions.
- Serve as a **portfolio** artifact for QA automation roles.

---

## API under test

**Base URL:** `https://reqres.in` (public, free, no API key).

| Area        | Assumption / behavior |
|------------|------------------------|
| **Auth**   | `POST /api/login`, `POST /api/register` return `{ "token": "..." }` on success; 400 with `error` on invalid/missing fields. |
| **Users**  | `GET /api/users` (list with `page`, `per_page`), `GET /api/users/:id`, `POST /api/users` (create), `PUT`/`PATCH`/`DELETE` on `/api/users/:id`. |
| **Persistence** | ReqRes does **not** persist created users. `GET /api/users/:id` only returns predefined users (e.g. id 1–12). E2E “Create → Get by ID” may get **404** for the created ID; the flow still demonstrates correlation and cleanup. |
| **Rate limits** | ReqRes does not expose 429 in normal use. The “Rate Limits” folder validates **contract**: if status is 429, then Retry-After or X-RateLimit-* headers should be present. |
| **Idempotency** | ReqRes does not enforce idempotency keys; the folder sends the same body twice with `Idempotency-Key` and asserts 200/201 and consistent response shape. |

---

## Setup

1. **Import in Postman**
   - **Collection:** `File → Import` → select `collection/API-Testing-Portfolio.postman_collection.json`.
   - **Environments:** Import `environments/local.postman_environment.json` and `environments/staging.postman_environment.json`.

2. **Select environment**
   - Choose **Local** (or **Staging** if you point `baseUrl` to your own backend).

3. **Variables (no hardcoding)**
   - **Collection variables:** `authToken`, `createdUserId`, `e2eUserId`, etc. are set by scripts.
   - **Environment variables:** `baseUrl`, `loginEmail`, `loginPassword`, `userName`, `userJob`. See below for .env-like mapping.

### .env-like mapping (placeholders only)

Do not commit real secrets. Example mapping for local/staging:

```env
# baseUrl - API root (e.g. https://reqres.in or https://staging-api.example.com)
BASE_URL=https://reqres.in

# Auth (placeholders)
LOGIN_EMAIL=eve.holt@reqres.in
LOGIN_PASSWORD=your_password_placeholder

# Test user for create/update
USER_NAME=morpheus
USER_JOB=leader
```

Map these to Postman environment keys: `baseUrl`, `loginEmail`, `loginPassword`, `userName`, `userJob`.

---

## Test strategy

| What is covered | What is not |
|-----------------|-------------|
| Auth (login/register success and failure, missing fields). | Real token refresh (ReqRes has no refresh endpoint; we re-login if needed). |
| Core CRUD (list, get, create, update PUT/PATCH, delete). | Actual persistence of created users (ReqRes is mock). |
| Negative/validation (missing fields, invalid JSON, wrong content-type, 404, invalid ID). | True rate-limit triggering (we only assert contract when 429 occurs). |
| Pagination (page, per_page, metadata). | Real concurrency/race (Postman runs sequentially; see Concurrency folder note). |
| Idempotency (same body + Idempotency-Key sent twice). | Load/performance (use k6 or Artillery). |
| Security (injection strings, unauthorized/no-token, invalid token). | |
| Contract (response schema for list, get, create). | |
| Smoke (login + list users). | |
| Regression (critical path). | |
| E2E chained flows (Create→Get→Delete; Create→Update→Get; Login→Create). | |
| Data-driven (CSV for create user and login). | |

**Risks:** Dependency on ReqRes availability; E2E “Get by created ID” may 404 on this API. Runs from some CI or restricted networks may get 403 (e.g. Cloudflare); run locally or from an allowed IP if that occurs.

---

## Folder structure

| Folder | Purpose |
|--------|---------|
| **Auth** | Login/register success; invalid credentials; missing password/email. |
| **Core CRUD** | List users, get by ID, create, update (PUT/PATCH), delete. |
| **Negative - Validation** | Missing name, empty body, invalid JSON, wrong content-type, 404, invalid ID, missing email. |
| **Pagination - Filtering - Sorting** | List with `page`/`per_page`, metadata and boundary. |
| **Rate Limits - Throttling** | Contract: if 429, expect Retry-After or rate-limit headers. |
| **Idempotency** | Two identical POSTs with same Idempotency-Key; assert 200/201 and id in response. |
| **Concurrency** | Three create requests; documents that Postman runs sequentially (true concurrency needs external tool). |
| **Security** | SQL injection / XSS payloads; no token; invalid token. |
| **Contract checks** | Schema assertions for list, get, create. |
| **Smoke suite** | Login + list users. |
| **Regression suite** | Critical path (e.g. list users). |
| **E2E Chained flows** | Create→Get→Delete; Create→Update→Get; Login→Create (token reused). |
| **Data-driven (run with CSV)** | Create user and login parameterized by CSV for Collection Runner. |

---

## Test data strategy

- **Dynamic:** `authToken`, `createdUserId`, `e2eUserId`, `e2eUpdateId`, `idempotencyKey` are set in scripts from responses.
- **Environment:** `baseUrl`, `loginEmail`, `loginPassword`, `userName`, `userJob` — no hardcoding in requests.
- **Data-driven:** Use `data/data-driven-users.csv` and `data/data-driven-login.csv` in Collection Runner; columns map to `{{name}}`, `{{job}}`, `{{email}}`, `{{password}}`, `{{expectedStatus}}`.

---

## Running with Newman

Install Newman (and optional reporter):

```bash
npm install -g newman newman-reporter-htmlextra
```

**Mac / Linux:**

```bash
# Smoke only (Smoke suite folder)
newman run collection/API-Testing-Portfolio.postman_collection.json \
  --environment environments/local.postman_environment.json \
  --folder "Smoke suite"

# Full collection
newman run collection/API-Testing-Portfolio.postman_collection.json \
  --environment environments/local.postman_environment.json

# With HTML report
newman run collection/API-Testing-Portfolio.postman_collection.json \
  --environment environments/local.postman_environment.json \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export reports/newman-report.html

# Data-driven: run "Data-driven (run with CSV)" folder with CSV (5 iterations for users)
newman run collection/API-Testing-Portfolio.postman_collection.json \
  --environment environments/local.postman_environment.json \
  --folder "Data-driven (run with CSV)" \
  --iteration-data data/data-driven-users.csv
```

**Windows (PowerShell):**

```powershell
# Smoke only
newman run collection/API-Testing-Portfolio.postman_collection.json --environment environments/local.postman_environment.json --folder "Smoke suite"

# Full collection
newman run collection/API-Testing-Portfolio.postman_collection.json --environment environments/local.postman_environment.json

# With HTML report (create reports folder first)
mkdir reports -Force
newman run collection/API-Testing-Portfolio.postman_collection.json --environment environments/local.postman_environment.json --reporters cli,htmlextra --reporter-htmlextra-export reports/newman-report.html
```

**Sample output expectations:**

- **Smoke:** 2 requests, 0 failures.
- **Full run:** Multiple requests; some negative tests may show “expected 400” style assertions; E2E “Get by created ID” can 404 on ReqRes (known).

**JUnit report (for CI):**

```bash
newman run collection/API-Testing-Portfolio.postman_collection.json \
  --environment environments/local.postman_environment.json \
  --reporters cli,junit \
  --reporter-junit-export reports/newman-junit.xml
```

---

## CI sample: GitHub Actions

Create `.github/workflows/postman.yml`:

```yaml
name: Postman API Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  newman:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Newman
        run: npm install -g newman newman-reporter-htmlextra

      - name: Run Postman collection
        run: |
          mkdir -p reports
          newman run collection/API-Testing-Portfolio.postman_collection.json \
            --environment environments/local.postman_environment.json \
            --reporters cli,htmlextra,junit \
            --reporter-htmlextra-export reports/newman-report.html \
            --reporter-junit-export reports/newman-junit.xml

      - name: Upload HTML report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: newman-html-report
          path: reports/newman-report.html

      - name: Publish JUnit
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: reports/newman-junit.xml
```

---

## Reporting

- **CLI:** Default Newman output (pass/fail, assertions).
- **htmlextra:** `--reporters htmlextra --reporter-htmlextra-export path/report.html`.
- **junit:** `--reporters junit --reporter-junit-export path/junit.xml` for CI dashboards.

---

## Test plan

### Smoke suite

- **Entry:** Build and environment available; `baseUrl` and credentials set.
- **Scope:** Login (200 + token), List users (200 + data array).
- **Exit:** All smoke tests pass → proceed to regression. Any failure → block release.

### Regression suite

- **Entry:** Smoke passed.
- **Scope:** Auth, Core CRUD, Negative/Validation, Pagination, Contract, Security, Idempotency, E2E flows (as applicable).
- **Exit:** All regression tests pass → release candidate. Failures → defect triage.

### Entry/exit criteria

- **Entry:** Environment selected; no real secrets in repo; ReqRes (or target API) reachable.
- **Exit:** Newman run completes; failures reviewed (including known ReqRes limitations).

---

## Bug examples (hypothetical)

These are sample bug reports based on *hypothetical* API failures, for portfolio documentation only.

---

**Bug 1: Login accepts empty password**

- **Summary:** `POST /api/login` returns 200 when `password` is empty string.
- **Steps:** Send `{ "email": "eve.holt@reqres.in", "password": "" }`.
- **Expected:** 400 with error message (e.g. “Missing password”).
- **Actual:** 200 and token returned.
- **Impact:** Security; weak validation.
- **Test:** “Login - Missing password” (negative test) would catch this if API were to regress.

---

**Bug 2: Create user returns 500 for long name**

- **Summary:** `POST /api/users` with `name` length 10,001 characters returns 500.
- **Steps:** Create user with `name` of 10,001 characters, `job` valid.
- **Expected:** 400 (validation error) or 201 with truncated/sanitized value.
- **Actual:** 500 Internal Server Error.
- **Impact:** No clear validation; server error exposed to client.
- **Test:** Add boundary test “Create User - Very long name” with 10,001-char string.

---

**Bug 3: List users pagination metadata inconsistent**

- **Summary:** `GET /api/users?page=2&per_page=5` returns `total_pages: 0` while `data` has 5 items.
- **Steps:** Call list with page=2, per_page=5.
- **Expected:** `total_pages` and `total` consistent with `data.length` and total count.
- **Actual:** `total_pages: 0`, `total: 0`, but `data` contains 5 items.
- **Impact:** Clients may hide valid data or paginate incorrectly.
- **Test:** “List Users - Page 2” and contract check “List Users - Schema” plus assertion on `total_pages`/`total` vs `data`.

---

## Summary

- **Collection:** `collection/API-Testing-Portfolio.postman_collection.json` — Auth, CRUD, Negative, Pagination, Rate-limit contract, Idempotency, Concurrency (simulation), Security, Contract, Smoke, Regression, E2E, Data-driven.
- **Environments:** `environments/local.postman_environment.json`, `environments/staging.postman_environment.json`.
- **Data:** `data/data-driven-users.csv`, `data/data-driven-login.csv`.
- **Run:** Newman with `--environment` and optional `--folder` and `--iteration-data`.
- **CI:** Use the provided GitHub Actions YAML and JUnit/htmlextra reporters.

No real secrets are stored in the repo; use placeholders and environment variables only.
