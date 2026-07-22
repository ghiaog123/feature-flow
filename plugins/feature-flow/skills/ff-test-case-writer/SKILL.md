---
name: ff-test-case-writer
description: Generate interactive HTML test case documents in a standardized format with collapsible sections, pass/fail tracking, progress bar, localStorage persistence, and Markdown export. Derives the test plan from an ff-impl-status proposal (`docs/features/<feature>/proposal.html`) when one exists — mapping API surface + flows → sections, data-model changes → validation cases, risks → priority — otherwise falls back to interviewing the user or a feature/API description. Applies risk-based QA methodology (positive/negative/boundary, priority P0–P3, entry/exit criteria, regression tiers). Use this skill whenever the user asks to write, create, generate, or document test cases, test plans, test checklists, QA plans, or manual test scenarios, or to "write a test plan from the proposal" — even if they phrase it as "document tests", "make a test checklist", or similar. Always prefer this skill over generic HTML generation when test cases are involved.
---

# Test Case Writer Skill

Generate a self-contained, interactive HTML test plan document matching the standardized format.

## Format Overview

The output is a single `.html` file. Key UI components:

1. **Header** — dark navy (`#1a1a2e`), project/service name + phase label on left, "last saved" timestamp on right
2. **Progress bar** — thin green bar below header, fills as test cases are checked PASS
3. **Stats bar** — white bar: total count, pass count (green), fail count (red), plus Export and Reset buttons
4. **Sections** — collapsible cards, each with a numbered badge, title, and `done/total` progress counter
5. **Test cases** — rows inside each section:
   - Checkbox (accent green) → marks PASS
   - FAIL button (red outline, fills red when active) → marks FAIL
   - TC ID + optional tag badge (pdf/docx/xlsx/all, or custom)
   - Title (bold)
   - Description (gray)
   - Expected result (blue left-border box, prefixed **Expect:**)
   - Note textarea (auto-saved)
   - Status badge right-aligned: TODO / PASS / FAIL

## Data Model

Test data is a JS `const plan` array. **The schema is the contract shared with the `ff-service-test-runner` skill — defined once in [`references/case-schema.md`](references/case-schema.md). Read it before generating; do not redefine fields here.**

Quick shape (full semantics in the schema file):

```js
const plan = [
  {
    id: 'S1',
    title: 'Section Title',
    cases: [
      {
        id: 'TC-01',
        title: 'Short descriptive title',
        pri: 'P0',           // P0 | P1 | P2 | P3 (risk-based priority)
        tag: 'pdf',          // optional UI tag
        desc: 'Steps or context. What to do.',
        expect: 'Exact expected outcome.',
        // contract fields — fill from the deep pass (Step 0); optional, omit when unknown:
        depth: 'core',       // 'surface' | 'core' — exercises core business logic?
        autoClass: 'auto',   // 'auto' | 'agent' | 'manual' — deterministic (incl. DB oracle) → auto; only fuzzy/NL → agent
        oracle: 'query bookings.status==CANCELLED; refund_log has 1 row',  // truth beyond the response (white-box)
        agentHint: 'Setup: book+pay. Rule: cancel <24h ⇒ 10% penalty. Verify refund==price*0.9'
      }
    ]
  }
];
```

The four contract fields (`depth`, `autoClass`, `oracle`, `agentHint`) let `ff-service-test-runner` skip re-guessing which cases are hard and how to verify them. **You** (with full proposal + code context here) are the right place to decide that — set them in Step 0. They are optional: omit when a case is plain, or when no code exists yet to ground them (mark provisional in `agentHint`).

## Tag Colors

| tag value | background | text |
|-----------|-----------|------|
| `pdf`     | `#fee2e2` | `#991b1b` |
| `docx`    | `#dbeafe` | `#1e40af` |
| `xlsx`    | `#dcfce7` | `#15803d` |
| `all`     | `#f3e8ff` | `#7e22ce` |
| custom    | `#e5e7eb` | `#374151` |

## Interactive Behavior

- **Checkbox PASS**: marks row green, badge = PASS, clears FAIL state
- **FAIL button**: marks row red, badge = FAIL, unchecks checkbox
- **Note textarea**: auto-saves on `oninput`
- **State**: persisted to `localStorage` under key `<service>_test_v1` (derive from document title)
- **Progress bar**: `(passCount / total) * 100%`
- **Section counter**: `done/total` next to each section title

## Export Function

`exportReport()` generates Markdown with:
- Title + date
- Summary table (Total/Pass/Fail/TODO)
- Per-section table: TC ID | Title | Status (✅/❌/⬜) | Note

Downloads as `test_report_YYYY-MM-DD.md`.

## How to Generate the HTML

### Step 0: Derive from ff-impl-status proposal (preferred input)

Before interviewing, check for a proposal produced by the `ff-impl-status` skill:

1. **Locate.** Look for `docs/features/<feature>/proposal.html` in the working directory (Glob `docs/features/*/proposal.html`). If the user named a feature, target that folder; if multiple proposals exist, ask which one.
2. **If a proposal exists → derive the plan from it.** Read the file and map its sections to test data:

   | proposal.html section | → test plan |
   |------------------------|-------------|
   | `<h2>3. API surface</h2>` (Method/Path/Description table) | one **section** (`Sx`) per endpoint or endpoint group |
   | `<h2>4. Detailed flows</h2>` (numbered steps per API) | **happy-path** cases following each step's expected outcome |
   | `<h2>2. Data model changes</h2>` (Layer/Change table) | **validation + boundary** cases (required/optional, null, type, length, constraint) for each changed field |
   | `<h2>5. Assessment → Risks</h2>` card | **negative/edge** cases; each risk drives a higher priority (a listed risk ⇒ `P0`/`P1`) |
   | `<h2>6. Rollout checklist</h2>` | a **smoke / regression** section covering each rollout item |

3. **Set `pri` from risk.** Default `P2`. Endpoint/flow named in a risk → `P0`. Core happy-path of a new endpoint → `P1`. Cosmetic/optional → `P3`.
4. **Scope gate — agree scope in plain language BEFORE the deep pass.** This is the single scope-confirmation point. Do it now, after you have a candidate scope from the proposal but *before* reading code / authoring cases — so you don't deep-pass areas the user then trims. See "Scope gate" below.
5. **Deep pass — read the implemented code, not just the proposal.** Only over the areas the user kept in scope. The proposal gives the API surface; the *core logic* lives in the handlers/services. When the code exists, read it and derive cases that exercise it (see "Core-logic test design" below), then set the contract fields per case:
   - `depth`: `core` if the case hits a state machine / formula / invariant / idempotency / concurrency / multi-step flow; else `surface`.
   - `autoClass`: `auto` (a deterministic assertion captures it — incl. white-box oracle: DB row / event / log / recomputed value) · `agent` (no deterministic oracle exists — fuzzy NL / ranking / recommendation needing judgement) · `manual` (visual/UX/human). Most `core` cases are `auto` (often white-box).
   - `oracle`: where the truth lives beyond the response (DB row, computed value, emitted event, log) — for white-box `auto` cases and as grounding for `agent` cases.
   - `agentHint`: setup + the business rule under test + what to verify. Guidance only — not exact paths/payloads.

   **If the code is NOT written yet** (TDD-style: test plan before implement) — do not block. Derive `depth`/`autoClass`/`oracle`/`agentHint` from the proposal as best you can and prefix `agentHint` with `provisional:`, or omit the fields. The runner will ground them against real code later.
6. **Append the test plan into the scope doc.** After the deep pass, author the cases and append the plan sections *below* the agreed scope in the same HTML file (see "Scope gate" — one living document: scope on top, plan underneath). No second scope confirmation — scope was already locked at step 4.

**If no proposal exists → fall back to Step 1 (interview / feature description).** Do not block on a missing proposal — many features skip the design doc. The scope gate still applies (run it on whatever the user describes). Still do the deep pass against whatever code exists.

### Step 1: Interview (if needed)

Ask the user for:
- **Service / project name** — goes in `<title>` and header
- **Phase label** — e.g. "Phase 1: Happy Path", "Regression v2.3"
- **Sections** — high-level groupings (e.g. by API endpoint, feature area, role)
- **Test cases per section** — ID scheme, title, description, expected result, optional tag

If the user gives a feature description or API spec, derive the test cases yourself and confirm before generating.

### Scope gate (run before authoring — both paths)

One scope-confirmation point. Runs after you have a candidate scope (from the proposal in Step 0, or from the user's description in Step 1) and **before** the deep pass + case authoring. Two parts: a plain-language agreement, then a living HTML doc that the test plan is later appended into.

**1. Agree scope in plain language — no jargon.** Talk in terms the user (incl. a PO/QC) understands. Keep `depth`/`autoClass`/`oracle`/`agent`/`manual` OUT of this — those are set later in the deep pass. Cover only:
- **What's covered** — which feature / flows / endpoints (by name, in plain words).
- **What's OUT of scope** — explicitly list what you will *not* test (and why: out of this release, owned elsewhere, unchanged, etc.). The out-of-scope list matters as much as the in-scope one.
- **Depth** — quick **smoke** (critical happy paths only) vs **standard** (happy + main negative/boundary) vs **thorough** (full positive/negative/boundary/edge + regression).
- **Environment / data / preconditions** — where it runs, what test data/accounts are needed.

Use `AskUserQuestion` to make this fast: a multiSelect on the candidate areas (which to keep in scope) plus a single-select depth choice. Let the user trim, add, or move items to out-of-scope. **Do not start the deep pass until the user confirms.**

**2. Write the agreed scope into a living HTML doc.** The moment scope is locked, write it as the **top** of the output HTML file (the `.scope-doc` block in `assets/test_plan_template.html` — copy the template at this point, per Step 2): in-scope list, out-of-scope list, depth, environment/data, and a "Scope locked at …" line. This file is the single living document — the test plan sections get appended **below** this scope block in step 6, same file. So the reader sees *what was agreed* first, then *the cases that realize it*. If scope changes later, update the scope block in place and reconcile the sections beneath it.

### Test Design Methodology (apply when deriving cases)

Cover each behavior across these axes — don't only write happy paths:

- **Positive / negative / boundary** — valid input passes; invalid input is rejected with the right error; min/max/empty/null/over-length/wrong-type at every boundary.
- **Priority (`pri`)** is risk-based: `P0` business-critical / security / data-loss / named in a proposal risk · `P1` major feature, common flow · `P2` minor / workaround exists · `P3` cosmetic / rare edge. Order/group sections so P0 sits first.
- **Test types** to mix: functional (business logic), integration (component↔component, API↔DB), regression (existing behavior still works), and where relevant security (authz/injection) and performance.
- **Entry criteria** (note in subtitle or a "Pre-flight" section if useful): env ready, test data seeded, build deployed, dependencies up.
- **Exit criteria**: all P0 pass, ≥90% P1 pass, no open critical bug.
- **Regression tiers** when scope is large: smoke (critical paths, fast) → targeted (changed area) → full. Make smoke its own section.

Anti-patterns to avoid: vague steps with no expected result, missing preconditions, no test data, skipped edge/boundary cases.

### Core-logic test design (the deep pass)

Surface cases (status code, field presence, basic validation) are necessary but not sufficient. Read the implemented handlers/services and write cases that exercise the **core logic** — these carry `depth: 'core'`. Most are still `autoClass: 'auto'` — a deterministic assertion *can* capture them, often via a side-channel `oracle` (DB row, event, log, recomputed formula) that just needs a white-box fixture. Reserve `autoClass: 'agent'` for the rare case where **no deterministic oracle exists** (fuzzy NL output, "is this ranking reasonable"). Examples below:

- **State machines** — every legal transition passes; every illegal one is rejected (e.g. `PAID→CANCELLED` ok, `CANCELLED→PAID` forbidden). Oracle: the persisted status after the call.
- **Computed values / formulas** — recompute the expected number independently and compare (fees, discounts, proration, tax, totals). Oracle: recompute from inputs, compare to response/DB. Don't just assert "a number came back."
- **Invariants** — things that must always hold (balance never negative, sum of splits == total, stock conserved). Oracle: query the aggregate after the operation.
- **Idempotency / retries** — same request twice ⇒ one effect, not two. Oracle: count rows / effects.
- **Multi-step flows** — a sequence where step N depends on N-1's state (create → reserve → pay → refund). One case = the whole chain; `agentHint` describes the setup.
- **Side-effects beyond the response** — emitted events, audit logs, rows in secondary tables, outbound calls. A `200` response does not prove the side-effect happened. Oracle: the side-effect's real sink.

For each, write `oracle` (where truth is) + `agentHint` (setup + rule + what to verify). These are almost all `auto` (white-box): the oracle is a deterministic side-channel the runner asserts against — it just needs DB/broker/log access. If the truth is fully in the response, keep it `auto` with the formula in `expect`. Only mark `agent` when the output is genuinely fuzzy and no assertion can decide it even with full DB access.

### Step 2: Output the full HTML

**Copy the template file — do NOT retype it.** The full HTML/CSS/JS lives at `assets/test_plan_template.html` (relative to this skill's directory). Copy it to the output path with `cp`, then **edit only the placeholders and data** in the copy. Never re-emit the boilerplate CSS/JS through generation, and never simplify the interactivity — the localStorage, progress tracking, export, and reset must all keep working.

Adapt only:
- `STORAGE_KEY` string
- `<title>` and `<h1>` text
- `.meta` subtitle
- `const plan` data
- the `scope` block placeholders

## Placeholders to replace

| Placeholder | What to put |
|-------------|-------------|
| `{{SERVICE_NAME}}` | Service or project name (e.g. "Document Service", "Auth API") |
| `{{PHASE_LABEL}}` | Phase or test type (e.g. "Happy Path", "Regression v2", "Edge Cases") |
| `{{SUBTITLE}}` | Short subtitle shown below h1 (e.g. "document_service · Phase 1: Happy Path") |
| `{{STORAGE_KEY}}` | Unique localStorage key, e.g. `doc_svc_test_v1` |
| `{{SCOPE_LOCKED_AT}}` | Date scope was agreed at the scope gate, e.g. `2026-06-06` |
| `{{SCOPE_DEPTH}}` | `smoke` \| `standard` \| `thorough` (from the scope gate) |
| `{{SCOPE_ENV}}` | Env / test data / preconditions in plain words; `''` to hide |
| `{{REPLACE WITH ACTUAL TEST DATA}}` | The actual `plan` array entries |

Fill `scope.inScope` / `scope.outScope` from the agreed scope gate (plain language). The scope block renders above all sections — it is the living scope doc; the plan sections are appended below it.

Also replace `{{SERVICE_NAME}}` and `{{PHASE_LABEL}}` inside the `exportReport()` function.

## ID Conventions

- Sections: `S1`, `S2`, `S3`, ...
- Test cases: `TC-01`, `TC-02`, ... (zero-padded, sequential across all sections)
- `STORAGE_KEY`: snake_case from service name + `_test_v` + version number

## Output

One living HTML file holds both the agreed scope (top) and the test plan (sections below). Write it in two phases:

1. **At scope lock (scope gate):** write the file with `scope` filled and `plan` empty (or a stub). Show it to the user — this is the scope-agreement doc.
2. **After the deep pass:** fill `plan` and rewrite the same file — the plan now sits beneath the scope block. Same filename, no second file.

- Filename: `test_plan_<phase_slug>.html` (e.g. `test_plan_happy_path.html`, `test_plan_regression.html`)
- Open with: `open <filename>` so the user can see it immediately

Tell the user the filename and that they can open it in any browser — no server needed. If scope changes later, edit the `scope` block in place and reconcile the sections below it.
