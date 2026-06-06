---
name: service-test-runner
description: >-
  Translate a manual test plan (the HTML produced by the test-case-writer skill, or its exported Markdown) into executable pytest tests, run them against the implemented service, and report PASS/FAIL/SKIP keyed back to each TC-id. Bridges the gap test-case-writer leaves — a manual checklist becomes real assertions (httpx/TestClient hitting endpoints, asserting status + body against each "Expect"). Delegates the actual run to the `test-runner` subagent. Use this skill ONLY when explicitly invoked — `/service-test-runner`, "auto test the service from the test plan", "chạy test plan", "test service theo test plan", "run the test plan against the service", "sinh pytest từ test plan rồi chạy". Does NOT auto-trigger after implementing a feature. Pairs with: test-case-writer (input) and impl-status (feature folder). For the "don't claim done until green" gate, combine with the built-in `verify` skill.
---

# Service Test Runner

Turn a **manual** test plan into **automated** service tests, run them, and reconcile results by `TC-id`.

```
test plan (HTML/md from test-case-writer)
   │  Step 1 parse → cases [{id, pri, tag, desc, expect, depth, autoClass, oracle, agentHint}]
   ▼
classify auto | agent | manual (Step 3 — read autoClass, self-classify only if absent)
   │  Step 4 generate: auto → static pytest · agent → call+oracle+judgement · manual → skip
   ▼
docs/features/<feature>/tests/test_plan_<slug>.py
   │  Step 5 run — DELEGATE to `test-runner` subagent (capture evidence for agent cases)
   ▼
pytest output + agent evidence
   │  Step 6 reconcile → PASS / 🤖 agent-judged / FAIL / SKIP / ERROR per TC-id
   ▼
docs/features/<feature>/test_results_<slug>.md
```

## When this skill applies

**Explicit trigger only** (see description). Do **not** fire automatically after an implement task. Once invoked, stay active for the session until the user says stop.

## Core rules (read first)

- **TC-id traceability is the whole point.** Every generated test function name + docstring carries its `TC-id`. A failure must map to exactly one plan case. Never merge two cases into one test.
- **Never fabricate a pass.** A TC is `PASS` only if its assertion actually ran and passed. If the service wasn't reachable, the dependency was missing, or the case can't be automated → mark `SKIP` or `ERROR` with the reason. No green without a real green.
- **Three tiers, no fudging between them.** `auto` = static pytest assertion. `agent` = needs an oracle beyond the response or semantic judgement → agent calls the service, checks the oracle, judges **with cited evidence**, result labelled `🤖 agent-judged` (never folded into a plain PASS). `manual` = visual/UX/exploratory → `pytest.skip("manual: <reason>")`, listed explicitly. Don't pretend to test manual cases; don't let an agent-judged verdict masquerade as a deterministic PASS.
- **Don't pollute the repo's CI suite.** Generated tests live under the feature folder (`docs/features/<feature>/tests/`), run by explicit path — NOT under `<service>/tests/` which CI collects.
- **Report real output.** Paste the pytest summary + each failing assertion into the report. The user trusts this to decide ship/no-ship.

## Workflow

### Step 1 — Locate + parse the test plan

1. Find the plan. Prefer `docs/features/<feature>/` (same folder impl-status/test-case-writer use). Glob `**/test_plan_*.html`. If the user named a feature, target that folder; if several plans, ask which.
2. Parse the structured data. The test-case-writer HTML embeds a JS array whose schema is the **shared contract** — defined in `test-case-writer/references/case-schema.md`:
   ```js
   const plan = [ { id:'S1', title:'...', cases:[
     {id:'TC-01', title, pri, tag, desc, expect,
      depth, autoClass, oracle, agentHint}, ...   // contract fields — may be absent on older/shallow plans
   ] } ];
   ```
   Extract that array verbatim — it is the source of truth (richer than the exported md). **Read the contract fields if present**: `autoClass` tells you how to test each case (Step 3), `oracle`+`agentHint` ground the agent-assisted tier (Step 4–5). If a case lacks `autoClass` (old plan), self-classify it in Step 3. If only an exported `test_report_*.md` exists, parse its per-section tables instead (columns `TC | Pri | Title | Status | Note`) and self-classify everything.
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
- **White-box oracle access** — any `auto`-with-side-channel-oracle or `agent` case asserts against a DB row / event / log, not just the API. Black-box (API-only) can't reach those. Get the oracle's access (DB DSN, broker bootstrap, log path) from repo `.env` / `docker-compose.yml`, exposed as a fixture (creds via `os.getenv`). **If the oracle sink is unreachable, the case is `SKIP (blocked)` or `ERROR` — never a response-only PASS dressed up as a verdict.** That is the line that keeps "never fabricate a pass" intact under the new tier.

### Step 3 — Classify each case: `auto` | `agent` | `manual`

**Prefer the plan's `autoClass` field** (set by test-case-writer's deep pass) — it was decided with full proposal + code context. Only self-classify cases that lack it (old/shallow plans), using:

**The discriminator: can a deterministic assertion capture the truth, given the right fixture?** Yes → `auto` (even if the oracle is a DB row / event / log — that's white-box `auto`, still reproducible). No → `agent`.

| Signal in case | Class |
|----------------|-------|
| Hits an HTTP endpoint, deterministic status/body/error expectation | `auto` (response oracle) |
| Data-model / validation / boundary (required, null, type, length, constraint) | `auto` |
| Auth/ACL (expect 401/403/200 by key) | `auto` |
| State machine, formula, invariant, idempotency, multi-step flow, side-effect — truth in a DB row / event / log / recomputed value | `auto` (**white-box** oracle — needs DB/broker/log fixture) |
| Output is fuzzy NL / ranking / recommendation — **no deterministic oracle exists**, correctness needs judgement | `agent` |
| Visual, layout, "looks like Figma", UX feel, exploratory | `manual` |
| Needs human judgement / external system not in scope | `manual` |

The three tiers map to how the case is executed:
- `auto` → static pytest assertion (Step 4). May be **white-box** (asserts a DB/event/log oracle) — still fully reproducible.
- `agent` → **agent-assisted tier** (Step 4–5): only when no assertion can decide correctness even with DB access. Agent calls the service, reasons, emits a `🤖 agent-judged` verdict with cited evidence. Never folded into a plain pytest PASS.
- `manual` → `pytest.skip("manual: <reason>")`, listed in the report.

**Push down, not up.** A DB/event oracle does NOT make a case `agent` — it stays `auto` with a richer fixture. Only stay `agent` when no static assertion captures the truth even given full white-box access. The agent tier is softer (non-reproducible, costs tokens) — keep it rare.

State the split to the user (`N auto, M agent, K manual`) before generating.

### Step 4 — Generate pytest

Write to `docs/features/<feature>/tests/test_plan_<slug>.py` (+ a `conftest.py` in the same dir). One function per `auto` TC:

**Where concrete paths + payloads come from (critical).** The test plan cases are prose — they carry the *intent* and *expected outcome*, NOT exact endpoint paths or request bodies. Source those from, in order:
1. `proposal.html` **§3 API surface** (Method + Path) and **§2 data model** (field names, types, required/optional) — the contract.
2. The **implemented handler code** in the service (route signature, Pydantic request/response models) — ground truth; read it to get the real path, required fields, and valid example values.
3. Only if both are missing → ask the user, or emit the test marked `pytest.skip("blocked: endpoint/schema unknown")` rather than guessing a path that 404s.

Never ship a placeholder `json={...}`. If you can't determine a real payload, the case is `SKIP (blocked)`, not a fabricated test.

**White-box `auto` cases (oracle beyond the response).** Most core-logic cases land here — the oracle is a DB row / event / log / recomputed value, which *is* deterministic. Write a normal pytest function: call the endpoint(s) (path/payload from handler code), then assert the oracle via a fixture (`db`, broker consumer, log reader). Setup steps come from `agentHint`; multi-step flow ⇒ one test = the whole chain. These report as ordinary PASS/FAIL — they are NOT agent-judged. If the oracle sink is unreachable ⇒ `SKIP (blocked)` / `ERROR`, never a response-only PASS.

**Agent-assisted cases (`autoClass: 'agent'`).** ONLY when no deterministic oracle exists even with DB access — fuzzy NL output, "is this ranking reasonable," recommendation sensibility. Keep the verdict grounded (never a bare "looks right"):

1. **The call and its evidence are real.** Hit the endpoint, capture request + response. Assert whatever deterministic slice exists (e.g. `200`, schema shape).
2. **`agentHint` is guidance, not a spec.** It carries setup + the rule under test — NOT exact paths/payloads. Resolve those from proposal §3 + handler code. A vague hint never justifies a guessed path; unresolved ⇒ `SKIP (blocked)`.
3. **The correctness verdict is the agent's judgement** — and it must cite evidence (request, response, what was compared). No evidence ⇒ `ERROR`, not a pass.
4. **Labelled `🤖 agent-judged`**, kept distinct from static PASS in the report (Step 6).

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

# white-box oracle fixture — only for auto-with-oracle / agent cases that assert a DB row.
# DSN from env; if unset/unreachable, those cases are SKIP(blocked)/ERROR, never a response-only PASS.
@pytest.fixture(scope="session")
def db():
    dsn = os.getenv("DATABASE_URL")
    if not dsn:
        pytest.skip("blocked: no DATABASE_URL — oracle sink unreachable")
    import psycopg                                       # adapt to the service's driver
    with psycopg.connect(dsn, row_factory=psycopg.rows.dict_row) as conn:
        yield conn
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

# auto WHITE-BOX: oracle is a DB row + recomputed formula — fully deterministic, so it's `auto`
# (NOT agent), it just needs the `db` fixture. This is the common shape for core-logic cases.
def test_tc_03_cancel_within_24h_refunds_90pct(client, db):
    """TC-03 (P0, auto white-box): cancel <24h ⇒ 10% penalty.
    oracle: recompute refund = price*0.9; bookings.status==CANCELLED.
    agentHint: book+pay first, then cancel."""
    bid = _setup_paid_booking(client, price=1000)        # setup from agentHint
    r = client.post(f"/api/v1/bookings/{bid}/cancel")
    assert r.status_code == 200, r.text
    assert r.json()["refund_amount"] == 900              # oracle: recomputed formula
    row = db.execute("select status from bookings where id=%s", (bid,)).fetchone()
    assert row["status"] == "CANCELLED"                   # oracle: persisted state, not just response

# agent-assisted: ONLY when no deterministic oracle exists. The call is real and its
# evidence (request/response) is captured; the correctness verdict is the agent's judgement,
# reported as 🤖 agent-judged with that evidence — never auto-stamped PASS.
def test_tc_11_summary_is_faithful(client):
    """TC-11 (P1, agent): summary must reflect the input bullets, no hallucinated facts.
    No assertion can decide 'faithful' — the runner agent judges the response text
    against the input and records its reasoning + the response as evidence."""
    r = client.post("/api/v1/summarize", json={"bullets": ["a", "b", "c"]})
    assert r.status_code == 200, r.text                   # the only deterministic part
    pytest.skip("agent-judged: faithfulness assessed by runner agent — see report")
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

**Agent-assisted cases need their evidence captured, not just pass/fail.** For each `agent` case, the runner subagent must return the concrete evidence: the request sent, the response, and the oracle query result (DB row / recomputed value / event read). That evidence backs the `🤖 agent-judged` verdict in the report — a verdict without evidence is `ERROR`, not a pass. The deterministic call+oracle assertions run as normal pytest; only the residual semantic call is the agent's judgement.

### Step 6 — Reconcile + report

Write `docs/features/<feature>/test_results_<slug>.md`:

```markdown
# Test Results — <feature> / <phase>
Run: <timestamp> · mode: live (http://localhost:8000) · service: backend

| Result | Count |
|---|---|
| PASS (auto) | x |
| 🤖 PASS (agent-judged) | a |
| FAIL | y |
| SKIP (manual) | z |
| ERROR | e |

| TC | Pri | Class | Result | Detail / Evidence |
|----|-----|-------|--------|--------|
| TC-01 | P0 | auto | ✅ PASS | 201, listing_id present |
| TC-03 | P0 | auto (white-box) | ✅ PASS | refund==900 (=price*0.9); `bookings.status==CANCELLED` |
| TC-04 | P0 | auto | ❌ FAIL | expected 400, got 500 — <assertion / traceback head> |
| TC-07 | P2 | manual | ⏭️ SKIP | manual: visual vs Figma |
| TC-11 | P1 | agent | 🤖 PASS | summary faithful to 3 input bullets, no hallucination (req/resp shown); agent reasoning attached |
| TC-12 | P1 | agent | ⚠️ ERROR | no evidence captured — response not reached |
```

`🤖 PASS (agent-judged)` rows MUST carry their evidence inline (request/response + oracle result). An agent verdict without evidence is `ERROR`, not PASS.

Then a verdict using the plan's exit criteria, **coverage-aware**: state both pass/fail AND how it was covered — e.g. "all P0 PASS (4 auto, 2 🤖 agent-judged — review the 2 before ship), 3 manual not run". **all P0 PASS + no open critical** → ship-ok; else block, list blockers. Flag if many P0 are `agent`/`manual` (soft coverage) rather than `auto`. Same `TC-id`s as the plan so the user cross-references the HTML checklist directly.

### Step 7 — Gate (optional)

If the user wants a hard "done" gate, combine with the built-in `verify` skill: do not declare the feature done while any P0 is FAIL/ERROR. State blockers plainly. Treat a P0 resting on `🤖 agent-judged` (not a deterministic `auto` PASS) as **needs-human-review**, not auto-green — surface it so the user signs off on the soft-covered criticals before shipping.

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
