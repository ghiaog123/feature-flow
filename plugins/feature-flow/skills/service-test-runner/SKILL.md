---
name: service-test-runner
description: >-
  Translate a manual test plan (the HTML produced by the test-case-writer skill, or its exported Markdown) into executable pytest tests, run them against the implemented service, and report PASS/FAIL/SKIP keyed back to each TC-id. Bridges the gap test-case-writer leaves — a manual checklist becomes real assertions (httpx/TestClient hitting endpoints, asserting status + body against each "Expect"). Delegates the actual run to the `test-runner` subagent. Use this skill ONLY when explicitly invoked — `/service-test-runner`, "auto test the service from the test plan", "chạy test plan", "test service theo test plan", "run the test plan against the service", "sinh pytest từ test plan rồi chạy". Does NOT auto-trigger after implementing a feature. Pairs with: test-case-writer (input) and impl-status (feature folder). For the "don't claim done until green" gate, combine with the built-in `verify` skill.
---

# Service Test Runner

Turn a **manual** test plan into **automated** service tests, run them, and reconcile results by `TC-id`.

```
test plan (HTML/md from test-case-writer)
   │  Step 1 parse → cases [{id, pri, tag, desc, expect}]
   ▼
classify auto vs manual (Step 3)
   │  Step 4 generate pytest, one function per auto TC-id
   ▼
docs/features/<feature>/tests/test_plan_<slug>.py
   │  Step 5 run — DELEGATE to `test-runner` subagent
   ▼
pytest output
   │  Step 6 reconcile → PASS/FAIL/SKIP/ERROR per TC-id
   ▼
docs/features/<feature>/test_results_<slug>.md
```

## When this skill applies

**Explicit trigger only** (see description). Do **not** fire automatically after an implement task. Once invoked, stay active for the session until the user says stop.

## Core rules (read first)

- **TC-id traceability is the whole point.** Every generated test function name + docstring carries its `TC-id`. A failure must map to exactly one plan case. Never merge two cases into one test.
- **Never fabricate a pass.** A TC is `PASS` only if its assertion actually ran and passed. If the service wasn't reachable, the dependency was missing, or the case can't be automated → mark `SKIP` or `ERROR` with the reason. No green without a real green.
- **Manual cases stay manual.** Visual/UX/exploratory cases (no deterministic API assertion) are classified `manual` → emitted as `pytest.skip("manual: <reason>")`, listed explicitly in the report. Do not pretend to test them.
- **Don't pollute the repo's CI suite.** Generated tests live under the feature folder (`docs/features/<feature>/tests/`), run by explicit path — NOT under `<service>/tests/` which CI collects.
- **Report real output.** Paste the pytest summary + each failing assertion into the report. The user trusts this to decide ship/no-ship.

## Workflow

### Step 1 — Locate + parse the test plan

1. Find the plan. Prefer `docs/features/<feature>/` (same folder impl-status/test-case-writer use). Glob `**/test_plan_*.html`. If the user named a feature, target that folder; if several plans, ask which.
2. Parse the structured data. The test-case-writer HTML embeds a JS array:
   ```js
   const plan = [ { id:'S1', title:'...', cases:[ {id:'TC-01', title, pri, tag, desc, expect}, ... ] } ];
   ```
   Extract that array verbatim — it is the source of truth (richer than the exported md). If only an exported `test_report_*.md` exists, parse its per-section tables instead (columns `TC | Pri | Title | Status | Note`).
3. If no plan exists → stop and tell the user to run `test-case-writer` first. Do not invent cases here — that defeats the proposal→plan→test chain.

### Step 2 — Identify the service under test + run mode

Inspect the repo to fill these in (ask the user only if genuinely ambiguous):

- **Service**: which one the plan targets (in a monorepo, match by the endpoints/paths named in the plan).
- **Run mode**:
  - **Live (default, black-box)** — service already running; hit a real base URL with `httpx`. Best fit for "after implement, auto-test the running service." Get base URL from the user, repo `.env`, or `docker-compose.yml` (e.g. `http://localhost:8000`).
  - **In-process (`TestClient`)** — only when the user wants no running server. Check the project's existing unit tests for import-time bootstrap quirks (e.g. module-level mocks of external services applied **before** importing the app); replicate that pattern in `conftest.py` or imports may fail.
- **Auth + conventions** — read them, don't guess:
  - Discover the service's auth scheme from its docs / dependency code (header names like `X-API-Key` or `Authorization: Bearer`, which endpoints need which key). Do not assume a convention.
  - Pull key values + base URL from env; never hardcode secrets into the generated file — read from `os.getenv` in `conftest.py`.

### Step 3 — Classify each case: `auto` | `manual`

| Signal in case | Class |
|----------------|-------|
| Hits an HTTP endpoint, deterministic status/body/error expectation | `auto` |
| Data-model / validation / boundary (required, null, type, length, constraint) | `auto` |
| Auth/ACL (expect 401/403/200 by key) | `auto` |
| Visual, layout, "looks like Figma", UX feel, exploratory | `manual` |
| Needs human judgement / external system not in scope | `manual` |

State the split to the user (`N auto, M manual`) before generating.

### Step 4 — Generate pytest

Write to `docs/features/<feature>/tests/test_plan_<slug>.py` (+ a `conftest.py` in the same dir). One function per `auto` TC:

**Where concrete paths + payloads come from (critical).** The test plan cases are prose — they carry the *intent* and *expected outcome*, NOT exact endpoint paths or request bodies. Source those from, in order:
1. `proposal.html` **§3 API surface** (Method + Path) and **§2 data model** (field names, types, required/optional) — the contract.
2. The **implemented handler code** in the service (route signature, Pydantic request/response models) — ground truth; read it to get the real path, required fields, and valid example values.
3. Only if both are missing → ask the user, or emit the test marked `pytest.skip("blocked: endpoint/schema unknown")` rather than guessing a path that 404s.

Never ship a placeholder `json={...}`. If you can't determine a real payload, the case is `SKIP (blocked)`, not a fabricated test.

```python
import os, pytest, httpx

# --- conftest.py ---
@pytest.fixture(scope="session")
def base_url():
    return os.getenv("SERVICE_BASE_URL", "http://localhost:8000")

@pytest.fixture(scope="session")
def client(base_url):
    headers = {"X-API-Key": os.getenv("API_KEY", "")}   # adapt header name to the service's auth scheme; read value from env, never hardcode
    with httpx.Client(base_url=base_url, headers=headers, timeout=30) as c:
        yield c
```

```python
# --- test_plan_<slug>.py ---
# Plan: docs/features/<feature>/test_plan_<slug>.html

# one function per TC. Use @pytest.mark.parametrize("field", [...]) only when a single
# TC genuinely fans out over many invalid inputs — never to merge two different TC-ids.
def test_tc_01_create_listing_returns_201(client):
    """TC-01 (P0): Create listing with valid payload.
    Expect: 201 + body contains listing_id."""
    # path + payload come from proposal.html API surface + data model + the real handler (see Step 4 note)
    r = client.post("/api/v1/listings", json={"title": "Apt", "price": 1000})
    assert r.status_code == 201, r.text
    assert "listing_id" in r.json()

@pytest.mark.skip(reason="manual: visual — compare against Figma, no deterministic assertion")
def test_tc_07_listing_card_matches_design():
    """TC-07 (P2): manual."""
```

Mapping `expect` prose → assertions:

| Expect says | Assertion |
|-------------|-----------|
| "returns 201 / created" | `assert r.status_code == 201` |
| "400 / validation error" | `assert r.status_code == 400` + check error body |
| "401/403 without key" | call with no/invalid header, assert status |
| "body contains X" | `assert X in r.json()` / check field value |
| "rejects null/empty/too long" | parametrize invalid inputs, assert 4xx each |
| numeric/boundary | parametrize min/max/over, assert behavior |

Rules: function name `test_tc_<id>_<short>`, docstring = TC-id + pri + the plan's title/expect. Order so P0 first (run/report priority). Regenerating = overwrite the same `<slug>` file (idempotent); don't accumulate stale copies.

### Step 5 — Run (delegate to `test-runner`)

Do NOT run inline and dump verbose output into the main thread. Spawn the **`test-runner` subagent** (Agent tool, `subagent_type: "test-runner"`) with a command like:

```
cd <service> && <project test command> <abs path to docs/features/<feature>/tests/> -v
```

Use the project's own Python env / runner (`uv run pytest`, `poetry run pytest`, `python -m pytest`, …) so `httpx`/`pytest` resolve. If no `test-runner` subagent is available on this machine, fall back to a general-purpose subagent with the same instructions — or run via Bash as a last resort, keeping output summarized. Ask it to return: per-test pass/fail, the summary line, and the full assertion text for any failure. If live mode and the service isn't up, the agent should report connection errors — those become `ERROR`, not `FAIL`.

### Step 6 — Reconcile + report

Write `docs/features/<feature>/test_results_<slug>.md`:

```markdown
# Test Results — <feature> / <phase>
Run: <timestamp> · mode: live (http://localhost:8000) · service: backend

| Result | Count |
|---|---|
| PASS | x |
| FAIL | y |
| SKIP (manual) | z |
| ERROR | e |

| TC | Pri | Result | Detail |
|----|-----|--------|--------|
| TC-01 | P0 | ✅ PASS | 201, listing_id present |
| TC-04 | P0 | ❌ FAIL | expected 400, got 500 — <assertion / traceback head> |
| TC-07 | P2 | ⏭️ SKIP | manual: visual vs Figma |
| TC-09 | P1 | ⚠️ ERROR | connection refused — service not running |
```

Then a one-line verdict using the plan's exit criteria: **all P0 PASS + no open critical** → ship-ok; else block, list blockers. Same `TC-id`s as the plan so the user cross-references the HTML checklist directly.

### Step 7 — Gate (optional)

If the user wants a hard "done" gate, combine with the built-in `verify` skill: do not declare the feature done while any P0 is FAIL/ERROR. State blockers plainly.

## Output files

```
docs/features/<feature>/
├── proposal.html                 (impl-status)
├── implementation_status.html    (impl-status)
├── test_plan_<slug>.html         (test-case-writer)
├── tests/
│   ├── conftest.py               (this skill — fixtures/auth/base_url)
│   └── test_plan_<slug>.py       (this skill — one fn per auto TC)
└── test_results_<slug>.md        (this skill — PASS/FAIL/SKIP per TC-id)
```

## Why this skill exists

`test-case-writer` produces a **manual** checklist; `test-runner` **runs** existing tests but won't author them. The missing piece is the deterministic, traceable translation **plan → pytest → run → result-per-TC**. Doing it ad-hoc each time gives non-reproducible assertions and no regression trail. This skill fixes the translation and reuses `test-runner` for execution, closing the loop: proposal → test plan → implement → auto-test.
