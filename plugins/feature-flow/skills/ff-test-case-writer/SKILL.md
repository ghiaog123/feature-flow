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

**2. Write the agreed scope into a living HTML doc.** The moment scope is locked, write it as the **top** of the output HTML file (the `.scope-doc` block — see template): in-scope list, out-of-scope list, depth, environment/data, and a "Scope locked at …" line. This file is the single living document — the test plan sections get appended **below** this scope block in step 6, same file. So the reader sees *what was agreed* first, then *the cases that realize it*. If scope changes later, update the scope block in place and reconcile the sections beneath it.

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

Use the exact CSS and JS patterns from the template below. Do NOT simplify the interactivity — the localStorage, progress tracking, export, and reset must all work.

Adapt only:
- `STORAGE_KEY` string
- `<title>` and `<h1>` text
- `.meta` subtitle
- `const plan` data

## Full HTML Template

Use this template verbatim, replacing only the marked placeholders:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{{SERVICE_NAME}} — {{PHASE_LABEL}}</title>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; background: #f5f5f5; color: #222; }

  header {
    background: #1a1a2e;
    color: #fff;
    padding: 20px 32px;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
  header h1 { font-size: 18px; font-weight: 600; }
  header .meta { font-size: 13px; opacity: .65; }

  .progress-bar-wrap { background: #2a2a4a; height: 6px; }
  .progress-bar { height: 6px; background: #4ade80; transition: width .3s; }

  .stats {
    background: #fff;
    border-bottom: 1px solid #e5e5e5;
    padding: 12px 32px;
    display: flex;
    gap: 24px;
    font-size: 13px;
    align-items: center;
  }
  .stat { display: flex; align-items: center; gap: 6px; }
  .stat .num { font-weight: 700; font-size: 18px; }
  .stat.pass .num { color: #16a34a; }
  .stat.fail .num { color: #dc2626; }
  .stat.total .num { color: #1a1a2e; }

  .actions { margin-left: auto; display: flex; gap: 8px; }
  button {
    padding: 6px 14px; border: none; border-radius: 6px;
    cursor: pointer; font-size: 13px; font-weight: 500;
  }
  .btn-reset { background: #fee2e2; color: #991b1b; }
  .btn-reset:hover { background: #fecaca; }
  .btn-export { background: #dbeafe; color: #1e40af; }
  .btn-export:hover { background: #bfdbfe; }

  main { max-width: 960px; margin: 24px auto; padding: 0 16px 48px; }

  .section {
    background: #fff;
    border: 1px solid #e5e5e5;
    border-radius: 10px;
    margin-bottom: 16px;
    overflow: hidden;
  }
  .section-header {
    padding: 14px 20px;
    background: #f8f8f8;
    border-bottom: 1px solid #e5e5e5;
    display: flex;
    align-items: center;
    gap: 12px;
    cursor: pointer;
    user-select: none;
  }
  .section-header:hover { background: #f0f0f0; }
  .section-num {
    background: #1a1a2e; color: #fff;
    border-radius: 20px; width: 28px; height: 28px;
    display: flex; align-items: center; justify-content: center;
    font-size: 13px; font-weight: 700; flex-shrink: 0;
  }
  .section-title { font-weight: 600; font-size: 15px; flex: 1; }
  .section-prog { font-size: 12px; color: #666; }
  .section-arrow { font-size: 12px; color: #999; transition: transform .2s; }
  .section.collapsed .section-arrow { transform: rotate(-90deg); }
  .section.collapsed .section-body { display: none; }

  .section-body { padding: 0; }

  .tc {
    display: flex;
    align-items: flex-start;
    gap: 12px;
    padding: 12px 20px;
    border-bottom: 1px solid #f0f0f0;
    transition: background .15s;
  }
  .tc:last-child { border-bottom: none; }
  .tc:hover { background: #fafafa; }
  .tc.done { background: #f0fdf4; }
  .tc.done:hover { background: #dcfce7; }
  .tc.fail-mark { background: #fff5f5; }
  .tc.fail-mark:hover { background: #fee2e2; }

  .tc-check {
    display: flex;
    flex-direction: column;
    gap: 4px;
    align-items: center;
    flex-shrink: 0;
    padding-top: 2px;
  }
  .tc-check input[type=checkbox] {
    width: 18px; height: 18px; cursor: pointer;
    accent-color: #16a34a;
  }
  .tc-fail-btn {
    font-size: 10px; padding: 2px 6px;
    border-radius: 4px; border: 1px solid #fca5a5;
    background: #fff; color: #dc2626; cursor: pointer;
    white-space: nowrap;
  }
  .tc-fail-btn.active { background: #dc2626; color: #fff; border-color: #dc2626; }

  .tc-body { flex: 1; }
  .tc-id { font-size: 11px; color: #999; font-weight: 600; margin-bottom: 3px; }
  .tc-title { font-size: 14px; font-weight: 600; margin-bottom: 4px; }
  .tc-desc { font-size: 13px; color: #444; line-height: 1.5; }
  .tc-expect {
    margin-top: 6px; font-size: 12px;
    background: #f0f9ff; border-left: 3px solid #38bdf8;
    padding: 6px 10px; border-radius: 0 4px 4px 0;
    color: #0369a1;
  }
  .tc-expect strong { font-weight: 700; }
  .tc-oracle {
    margin-top: 6px; font-size: 12px;
    background: #fefce8; border-left: 3px solid #eab308;
    padding: 6px 10px; border-radius: 0 4px 4px 0;
    color: #854d0e;
  }
  .tc-hint {
    margin-top: 6px; font-size: 12px;
    background: #f5f3ff; border-left: 3px solid #8b5cf6;
    padding: 6px 10px; border-radius: 0 4px 4px 0;
    color: #5b21b6;
  }
  .tc-oracle strong, .tc-hint strong { font-weight: 700; }
  .tc-note-wrap { margin-top: 8px; }
  .tc-note {
    width: 100%; font-size: 12px; border: 1px solid #e5e5e5;
    border-radius: 6px; padding: 6px 8px; resize: vertical;
    font-family: inherit; color: #333; background: #fafafa;
    min-height: 36px; outline: none;
  }
  .tc-note:focus { border-color: #93c5fd; background: #fff; }

  .tag {
    display: inline-block; font-size: 10px; font-weight: 700;
    padding: 1px 6px; border-radius: 4px; margin-left: 6px;
    vertical-align: middle;
  }
  .tag-pdf { background: #fee2e2; color: #991b1b; }
  .tag-docx { background: #dbeafe; color: #1e40af; }
  .tag-xlsx { background: #dcfce7; color: #15803d; }
  .tag-all { background: #f3e8ff; color: #7e22ce; }
  .tag-custom { background: #e5e7eb; color: #374151; }

  .pri {
    display: inline-block; font-size: 10px; font-weight: 700;
    padding: 1px 6px; border-radius: 4px; margin-left: 6px;
    vertical-align: middle;
  }
  .pri-p0 { background: #fecaca; color: #7f1d1d; }
  .pri-p1 { background: #fed7aa; color: #9a3412; }
  .pri-p2 { background: #fef08a; color: #854d0e; }
  .pri-p3 { background: #e5e7eb; color: #374151; }

  .cls {
    display: inline-block; font-size: 10px; font-weight: 700;
    padding: 1px 6px; border-radius: 4px; margin-left: 6px;
    vertical-align: middle;
  }
  .cls-auto { background: #dcfce7; color: #15803d; }
  .cls-agent { background: #ede9fe; color: #6d28d9; }
  .cls-manual { background: #e5e7eb; color: #374151; }
  .cls-core { background: #ffe4e6; color: #9f1239; }

  .tc-status-badge {
    font-size: 10px; font-weight: 700; padding: 2px 8px;
    border-radius: 10px; margin-left: auto; flex-shrink: 0;
    align-self: flex-start; margin-top: 2px;
  }
  .badge-pass { background: #dcfce7; color: #15803d; }
  .badge-fail { background: #fee2e2; color: #991b1b; }
  .badge-todo { background: #f3f4f6; color: #6b7280; }

  .scope-doc {
    background: #fff; border: 1px solid #e5e5e5; border-radius: 10px;
    margin-bottom: 16px; overflow: hidden;
  }
  .scope-doc .scope-head {
    padding: 14px 20px; background: #eef2ff; border-bottom: 1px solid #e0e7ff;
    display: flex; align-items: center; gap: 12px;
  }
  .scope-doc .scope-head h2 { font-size: 15px; font-weight: 700; color: #3730a3; flex: 1; }
  .scope-doc .scope-locked { font-size: 12px; color: #6366f1; }
  .scope-doc .scope-depth {
    font-size: 11px; font-weight: 700; padding: 2px 8px; border-radius: 10px;
    background: #c7d2fe; color: #3730a3;
  }
  .scope-body { padding: 16px 20px; font-size: 13px; line-height: 1.6; color: #333; }
  .scope-body h3 { font-size: 13px; margin: 12px 0 4px; color: #1a1a2e; }
  .scope-body h3:first-child { margin-top: 0; }
  .scope-body ul { margin: 0 0 4px 20px; }
  .scope-in li::marker { color: #16a34a; }
  .scope-out li::marker { color: #dc2626; }
  .scope-out li { color: #6b7280; }
  .scope-meta { margin-top: 10px; font-size: 12px; color: #555; }

  @media print {
    .actions, .tc-note-wrap, .tc-check { display: none; }
    .tc.done .tc-title::after { content: ' ✓'; color: #16a34a; }
    .tc.fail-mark .tc-title::after { content: ' ✗'; color: #dc2626; }
  }
</style>
</head>
<body>

<header>
  <div>
    <h1>{{SERVICE_NAME}} — {{PHASE_LABEL}}</h1>
    <div class="meta">{{SUBTITLE}}</div>
  </div>
  <div class="meta" id="last-saved"></div>
</header>
<div class="progress-bar-wrap">
  <div class="progress-bar" id="progress-bar"></div>
</div>

<div class="stats">
  <div class="stat total"><span class="num" id="stat-total">0</span> test cases</div>
  <div class="stat pass"><span class="num" id="stat-pass">0</span> passed</div>
  <div class="stat fail"><span class="num" id="stat-fail">0</span> failed</div>
  <div class="actions">
    <button class="btn-export" onclick="exportReport()">Export report</button>
    <button class="btn-reset" onclick="resetAll()">Reset all</button>
  </div>
</div>

<main id="main"></main>

<script>
const STORAGE_KEY = '{{STORAGE_KEY}}';

// Agreed test scope (locked at the scope gate, BEFORE the plan below was authored).
// This is the living scope doc; the `plan` sections underneath realize it.
const scope = {
  locked: '{{SCOPE_LOCKED_AT}}',        // e.g. '2026-06-06'
  depth: '{{SCOPE_DEPTH}}',             // 'smoke' | 'standard' | 'thorough'
  inScope: [
    // 'Plain-language area / flow that IS covered',
  ],
  outScope: [
    // 'Plain-language area that is NOT covered (+ why)',
  ],
  env: '{{SCOPE_ENV}}',                 // env / test data / preconditions, plain words; '' to hide
};

const plan = [
  // {{REPLACE WITH ACTUAL TEST DATA}}
];

// --- state ---
let state = {};

function loadState() {
  try { state = JSON.parse(localStorage.getItem(STORAGE_KEY) || '{}'); } catch { state = {}; }
}
function saveState() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
  const d = new Date();
  document.getElementById('last-saved').textContent =
    `Saved ${d.getHours().toString().padStart(2,'0')}:${d.getMinutes().toString().padStart(2,'0')}`;
}

function getTC(id) { return state[id] || { done: false, fail: false, note: '' }; }
function setTC(id, patch) {
  state[id] = { ...getTC(id), ...patch };
  saveState();
  updateStats();
}

function tagHtml(tag) {
  if (!tag) return '';
  const knownCls = { pdf:'tag-pdf', docx:'tag-docx', xlsx:'tag-xlsx', all:'tag-all' };
  const cls = knownCls[tag] || 'tag-custom';
  return `<span class="tag ${cls}">${tag.toUpperCase()}</span>`;
}

function priHtml(pri) {
  if (!pri) return '';
  const cls = { P0:'pri-p0', P1:'pri-p1', P2:'pri-p2', P3:'pri-p3' }[pri] || 'pri-p3';
  return `<span class="pri ${cls}">${pri}</span>`;
}

function classHtml(tc) {
  let h = '';
  if (tc.depth === 'core') h += `<span class="cls cls-core">CORE</span>`;
  if (tc.autoClass) {
    const cls = { auto:'cls-auto', agent:'cls-agent', manual:'cls-manual' }[tc.autoClass] || 'cls-manual';
    h += `<span class="cls ${cls}">${tc.autoClass.toUpperCase()}</span>`;
  }
  return h;
}

function oracleHtml(tc) {
  let h = '';
  if (tc.oracle) h += `<div class="tc-oracle"><strong>Oracle:</strong> ${tc.oracle}</div>`;
  if (tc.agentHint) h += `<div class="tc-hint"><strong>Hint:</strong> ${tc.agentHint}</div>`;
  return h;
}

function renderSection(sec) {
  const div = document.createElement('div');
  div.className = 'section';
  div.id = `sec-${sec.id}`;

  div.innerHTML = `
    <div class="section-header" onclick="toggleSection('${sec.id}')">
      <div class="section-num">${sec.id.replace('S','')}</div>
      <div class="section-title">${sec.title}</div>
      <div class="section-prog" id="prog-${sec.id}"></div>
      <div class="section-arrow">▼</div>
    </div>
    <div class="section-body" id="body-${sec.id}"></div>
  `;

  const body = div.querySelector(`#body-${sec.id}`);
  sec.cases.forEach(tc => {
    const s = getTC(tc.id);
    const item = document.createElement('div');
    item.className = `tc${s.done ? ' done' : ''}${s.fail ? ' fail-mark' : ''}`;
    item.id = `tc-${tc.id}`;
    item.innerHTML = `
      <div class="tc-check">
        <input type="checkbox" id="chk-${tc.id}" ${s.done ? 'checked' : ''}
          onchange="onCheck('${tc.id}', this.checked)">
        <button class="tc-fail-btn ${s.fail ? 'active' : ''}"
          onclick="onFail('${tc.id}')">FAIL</button>
      </div>
      <div class="tc-body">
        <div class="tc-id">${tc.id} ${priHtml(tc.pri)}${classHtml(tc)}${tagHtml(tc.tag)}</div>
        <div class="tc-title">${tc.title}</div>
        <div class="tc-desc">${tc.desc}</div>
        <div class="tc-expect"><strong>Expect:</strong> ${tc.expect}</div>
        ${oracleHtml(tc)}
        <div class="tc-note-wrap">
          <textarea class="tc-note" placeholder="Notes / issues encountered..."
            oninput="onNote('${tc.id}', this.value)">${s.note || ''}</textarea>
        </div>
      </div>
      <div class="tc-status-badge ${s.done ? 'badge-pass' : s.fail ? 'badge-fail' : 'badge-todo'}" id="badge-${tc.id}">
        ${s.done ? 'PASS' : s.fail ? 'FAIL' : 'TODO'}
      </div>
    `;
    body.appendChild(item);
  });

  return div;
}

function onCheck(id, checked) {
  setTC(id, { done: checked, fail: checked ? false : getTC(id).fail });
  const el = document.getElementById(`tc-${id}`);
  el.classList.toggle('done', checked);
  if (checked) el.classList.remove('fail-mark');
  document.getElementById(`badge-${id}`).className =
    `tc-status-badge ${checked ? 'badge-pass' : 'badge-todo'}`;
  document.getElementById(`badge-${id}`).textContent = checked ? 'PASS' : 'TODO';
  if (checked) {
    const failBtn = el.querySelector('.tc-fail-btn');
    failBtn.classList.remove('active');
    setTC(id, { fail: false });
  }
  updateSectionProg();
}

function onFail(id) {
  const s = getTC(id);
  const nowFail = !s.fail;
  setTC(id, { fail: nowFail, done: nowFail ? false : s.done });
  const el = document.getElementById(`tc-${id}`);
  el.classList.toggle('fail-mark', nowFail);
  if (nowFail) el.classList.remove('done');
  el.querySelector('.tc-fail-btn').classList.toggle('active', nowFail);
  if (nowFail) document.getElementById(`chk-${id}`).checked = false;
  const badge = document.getElementById(`badge-${id}`);
  badge.className = `tc-status-badge ${nowFail ? 'badge-fail' : 'badge-todo'}`;
  badge.textContent = nowFail ? 'FAIL' : 'TODO';
  updateSectionProg();
}

function onNote(id, val) { setTC(id, { note: val }); }

function toggleSection(sid) {
  document.getElementById(`sec-${sid}`).classList.toggle('collapsed');
}

function updateSectionProg() {
  plan.forEach(sec => {
    const total = sec.cases.length;
    const done = sec.cases.filter(tc => getTC(tc.id).done).length;
    document.getElementById(`prog-${sec.id}`).textContent = `${done}/${total}`;
  });
}

function updateStats() {
  const all = plan.flatMap(s => s.cases);
  const total = all.length;
  const pass = all.filter(tc => getTC(tc.id).done).length;
  const fail = all.filter(tc => getTC(tc.id).fail).length;
  document.getElementById('stat-total').textContent = total;
  document.getElementById('stat-pass').textContent = pass;
  document.getElementById('stat-fail').textContent = fail;
  const pct = total ? (pass / total * 100) : 0;
  document.getElementById('progress-bar').style.width = pct + '%';
  updateSectionProg();
}

function resetAll() {
  if (!confirm('Reset all test results? This cannot be undone.')) return;
  state = {};
  saveState();
  location.reload();
}

function exportReport() {
  const all = plan.flatMap(s => s.cases);
  const total = all.length;
  const pass = all.filter(tc => getTC(tc.id).done).length;
  const fail = all.filter(tc => getTC(tc.id).fail).length;
  const todo = total - pass - fail;

  let md = `# {{SERVICE_NAME}} — {{PHASE_LABEL}} Test Report\n`;
  md += `Date: ${new Date().toLocaleString('en-US')}\n\n`;
  if (scope && (scope.inScope?.length || scope.outScope?.length)) {
    md += `## Test scope\n`;
    if (scope.depth) md += `Depth: **${scope.depth}**`;
    if (scope.locked) md += `${scope.depth ? ' · ' : ''}Scope locked: ${scope.locked}`;
    md += `\n\n`;
    if (scope.inScope?.length) md += `**In scope:**\n${scope.inScope.map(x => `- ${x}`).join('\n')}\n\n`;
    if (scope.outScope?.length) md += `**Out of scope:**\n${scope.outScope.map(x => `- ${x}`).join('\n')}\n\n`;
    if (scope.env) md += `**Environment / data:** ${scope.env}\n\n`;
  }
  md += `| | Count |\n|---|---|\n`;
  md += `| Total | ${total} |\n| Pass | ${pass} |\n| Fail | ${fail} |\n| TODO | ${todo} |\n\n`;

  plan.forEach(sec => {
    md += `## ${sec.id}: ${sec.title}\n\n`;
    md += `| TC | Pri | Title | Status | Note |\n|---|---|---|---|---|\n`;
    sec.cases.forEach(tc => {
      const s = getTC(tc.id);
      const status = s.done ? '✅ PASS' : s.fail ? '❌ FAIL' : '⬜ TODO';
      md += `| ${tc.id} | ${tc.pri || ''} | ${tc.title} | ${status} | ${(s.note||'').replace(/\n/g,' ')} |\n`;
    });
    md += '\n';
  });

  const blob = new Blob([md], { type: 'text/markdown' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = `test_report_${new Date().toISOString().slice(0,10)}.md`;
  a.click();
}

function renderScope() {
  if (!scope || (!scope.inScope?.length && !scope.outScope?.length)) return;
  const li = arr => (arr || []).map(x => `<li>${x}</li>`).join('');
  const depth = scope.depth ? `<span class="scope-depth">${scope.depth.toUpperCase()}</span>` : '';
  const locked = scope.locked ? `<span class="scope-locked">Scope locked: ${scope.locked}</span>` : '';
  const div = document.createElement('div');
  div.className = 'scope-doc';
  div.innerHTML = `
    <div class="scope-head"><h2>Test scope</h2>${depth}${locked}</div>
    <div class="scope-body">
      ${scope.inScope?.length ? `<h3>✅ In scope</h3><ul class="scope-in">${li(scope.inScope)}</ul>` : ''}
      ${scope.outScope?.length ? `<h3>🚫 Out of scope</h3><ul class="scope-out">${li(scope.outScope)}</ul>` : ''}
      ${scope.env ? `<div class="scope-meta"><strong>Environment / data:</strong> ${scope.env}</div>` : ''}
    </div>
  `;
  return div;
}

// --- init ---
loadState();
const main = document.getElementById('main');
const scopeEl = renderScope();
if (scopeEl) main.appendChild(scopeEl);
plan.forEach(sec => main.appendChild(renderSection(sec)));
updateStats();
</script>
</body>
</html>
```

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
