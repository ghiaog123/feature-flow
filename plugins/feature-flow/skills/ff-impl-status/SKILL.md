---
name: ff-impl-status
description: "Generate and maintain a live HTML status file for implementation tasks at `docs/features/<feature>/implementation_status.html`. ONLY triggers when the user explicitly invokes it: `/ff-impl-status`, `ff-impl-status`, 'use the ff-impl-status skill', 'turn on the status file', 'maintain status file', 'track implementation status', or 'resume ff-impl-status <feature>' / 'load status <feature>' / 'resume feature X' to reload from an existing proposal + implementation_status.html in a new chat. Does NOT auto-trigger even when the user asks to implement a large feature. Once activated, the skill stays active until the session ends or the user says 'stop status file' / 'skip status'. The file records progress, off-design decisions, compromises, bugs found while coding, changed files, and a rollout checklist. Supports generating proposal.html first if the user wants a design phase; implementation_status.html is only created once the user confirms implementation starts. Also supports resuming context from a previous session by re-reading those 2 HTML files."
---

# ff-impl-status — Live Implementation Status HTML

Generate + maintain an HTML tracker file while the user implements a multi-step task. The file is a **communication artifact** — it records decisions, compromises, and bugs so the user can skim quickly and challenge anything they disagree with.

## When this skill applies

**Only triggers on explicit user activation**:
- `/ff-impl-status`
- "use the ff-impl-status skill", "turn on ff-impl-status"
- "turn on the status file", "maintain status file"
- "track implementation status"
- Or the user references this skill in an equivalent way

**Never auto-trigger**, even when the user asks to implement a large feature / refactor / migration. This is an explicit requirement from the skill owner: this skill must be opt-in every session.

Once activated, the skill stays **active until the session ends** (or until the user pauses it). Every subsequent implementation task in the session follows this workflow — no reminder needed.

**Pause** (keep the existing file, stop updating): "stop status file", "skip status", "no need for the status file", "stop updating the status". **Resume** when the user asks again.

## Principles

**The HTML file is a live document, not a log.** Concise, organized. Update because the user needs to know — not because "step 4 is done".

**Output in English by default.** Write the HTML in English unless the user asks for another language — then follow the user's language.

**The HTML file is the source of truth.** Read it before updating to preserve continuity. Don't rewrite unless the structure changes significantly.

## Workflow

### Step -1 — Resume from a previous chat (when the user asks)

When the user says "resume ff-impl-status <feature>", "load status <feature>", "resume feature X", "re-read the status file", "continue from the previous session":

1. **Locate folder.** Find `docs/features/<feature>/` in the working directory. If the user didn't name the feature:
   - List `docs/features/*/` (Bash or Glob).
   - Exactly 1 folder → use it.
   - Multiple folders → ask the user to choose (bullet list of feature names).
   - No folder → report the error: "Could not find `docs/features/<feature>/`. The skill has never run for this feature — start fresh with `/ff-impl-status`."

2. **Read files in order**:
   - `proposal.html` (if it exists) — understand the design intent + initial decisions.
   - `implementation_status.html` (if it exists) — pick up progress + off-design decisions + bugs caught + files changed + rollout checklist.
   - `README.md` (if it exists) — index/summary.

3. **Reconstruct context.** After reading, present a short summary to the user (≤10 lines):
   - Current progress (steps done / in_progress / todo).
   - Key decisions made.
   - Bugs fixed.
   - Which rollout steps are still pending.

4. **Verify still-valid.** Memory check: files/paths/symbols mentioned in the status may have changed between sessions. Before suggesting which step to continue, grep / read the real files to confirm. If skew is detected (file renamed/deleted) → update the status file immediately (new "Drift detected" section or update the relevant entry).

5. **Wait for the user's direction** on what's next (resume which step, edit which decision, finalize, etc.). Don't auto-code.

After this step, the skill switches to normal active mode (Step 3 — update on events).

### Step 0 — Determine the feature name

Infer `snake_case` / `kebab-case` from:
1. The user prompt (e.g. "add project_id" → `project_id`).
2. The git branch (e.g. `develop-feature-project_id` → `project_id`).
3. The proposal content if one already exists.

Don't ask the user unless all 3 sources are empty/ambiguous.

### Step 1 — Proposal (when the user asks for a design phase first)

When the user says "evaluate the solution options", "write a proposal", "design first", "write the proposal HTML":

#### 1a — Ask clarifying questions (MANDATORY before generating the proposal)

The proposal locks the design intent — a wrong assumption here propagates into the status file + real code. Asking first is far cheaper. So **always** ask **3–5 clarifying questions** before writing `proposal.html`.

**Pick the number of questions based on how detailed the user's description is:**
- Thorough description with clear scope + data model + API + edge cases → **3** questions (only what's genuinely ambiguous).
- Medium description → **4** questions.
- Sparse / one-liner / many gaps → **5** questions.

Don't ask to fill a quota — ask where it's actually ambiguous. If everything is so clear you can't think of 3 worthwhile questions, still ask ≥3: switch to confirming assumptions ("I understand X — correct?", "Does the scope include Y?", "Any constraint Z?").

**Order questions by architectural impact — heavy & hard-to-reverse FIRST.** A wrong answer to a high-impact question poisons everything downstream, so it must surface earliest. Order:
1. **High impact, hard to fix later** — data model, storage choice, backward-compatibility / migration constraints, who consumes the API.
2. **Medium impact** — scope (what's in / out), data sources, edge-case behavior.
3. **Low impact / easy to change** — cosmetic details, last.

**Previous-session context is LOW TRUST.** It may be reused to *suggest* answers, but is never final. If the current request **contradicts** something said earlier in the session (e.g. Postgres before, Mongo implied now) → **you must ask to resolve the conflict**, naming both sides; don't pick on your own.

Use a multiple-choice tool (AskUserQuestion) when a question has discrete options; ask open-ended when free-form description is needed. Only generate the proposal after the user answers. If the answers open significant new ambiguity, ask one short follow-up round — don't drag it out indefinitely.

**After the user answers → lock into a decision table.** Synthesize the answers into a compact **decision table** (`Question/Decision | Settled | Rationale`). This table is the **canonical input** for building the proposal — the proposal must NOT be built on unrecorded assumptions. Every non-trivial assumption must trace back to a row in the table. The table feeds directly into the top of the proposal (sections 1–2): insert it into the "Decisions from interview" box near the TL;DR / Scope (available in `assets/proposal_template.html`).

#### 1b — Generate the proposal

Generate `docs/features/<feature>/proposal.html` with 8 sections, ordered WHY → WHAT → HOW.

**Before section 1 — a TL;DR box "Core idea".** 1–3 sentences in plain language, NO jargon: what the feature does + what pain it solves. The user should grasp the gist in 10 seconds before diving into technical detail. This is the most important part for less-technical readers — write clearly, concretely, avoid terminology. (The `.tldr` box is in the template.)

1. **Context & Problem** — why do this, where it hurts, what triggered the proposal (1–2 paragraphs).
2. **Goals & Scope** — goals (done = what) + In-scope / Non-goals.
3. **Data model changes** (table).
4. **API surface** (table: method, path, description).
5. **Detailed flows** for new APIs (numbered steps).
6. **Acceptance criteria** — verifiable acceptance criteria; `ff-test-case-writer` + `ff-feature-brief` map directly from here.
7. **Assessment** — Pros / Risks / Rejected alternatives.
8. **High-level implementation checklist**.

**Technical logic → draw SIMPLE + COLORFUL diagrams for readability, don't present as long text.** The template embeds Mermaid.js (CDN, `base` theme + matching color palette). Use `<pre class="mermaid">` blocks inside `<div class="diagram">`:
- **Detailed flows (5)** → default to `flowchart` (boxes + arrows + `{condition}` branches) — simple, readable. Use `sequenceDiagram` only when multi-party interaction matters, `stateDiagram-v2` when a state lifecycle matters. Don't over-draw when a flowchart suffices.
- **Data model (3)** → keep the column table + `erDiagram` for entity relationships.
- Keep text only for what diagrams can't carry (specific validation rules, edge-case rationale). API surface (4) stays a table — tabular, no drawing needed.

**Make diagrams VIVID + easy to grasp** — 3 layers of visual signal (full example in the template), so the user understands at a glance without reading:

1. **Color by role** (`classDef` + `class`, `stroke-width:2px`): 🔵 blue = input/start · 🟡 yellow = decision/condition · 🟣 purple = processing/logic · ⚪ gray = data store · 🟢 green = success · 🔴 red = error/rejection. Use exactly this palette, don't invent colors.
2. **Shape by node type**: `([...])` stadium = entry/exit point (request, response) · `{...}` diamond = decision · `[/.../]` parallelogram = processing step · `[(...)]` cylinder = DB/store. Different shape = different role, instantly recognizable.
3. **Emoji icon at label start** for quick meaning: 👤 client · ⚙️ processing · 🗄️ DB · ✅ success · ❌ error · 🔑 auth · 📤 send · 📥 receive. 1 icon per node, don't spam.

Also: short marked edge labels (`-->|✓ Yes|`, `-->|✗ No|`); `subgraph` to group nodes by actor (Client / Service / DB) when the flow crosses multiple parties.

Rules to keep diagrams readable:
- **≤7 nodes per diagram.** More → split into 2 diagrams or merge minor steps.
- If a passage describes >4–5 sequential steps / many branches / many relationships → convert to a diagram. Pick the simplest diagram type that expresses the logic.
- Short node labels (≤4–5 words), plain language, don't cram detail into boxes.

Template: `assets/proposal_template.html`. Do **not** generate `implementation_status.html` at this stage — the proposal is a design doc; the status file is a live tracker created only once coding starts.

### Step 2 — When the user says "implement"/"go ahead"/"start coding" → init the status file

Create `docs/features/<feature>/implementation_status.html` from `assets/status_template.html`. The "Progress" section lists the high-level steps (copied from the proposal checklist if one exists), default status `todo`. Flip `→ in_progress → done` as each step actually starts / finishes.

Notify the user in 1 line: "Status file initialized at `<path>`."

### Step 3 — Update on significant events

| Event | Section | Content |
|-------|---------|----------|
| A step starts / finishes | Progress | Flip status tag |
| Decision outside the original design | Decisions | Title + **Why** |
| Compromise / deviation | Compromises | Bullet + reason + impact |
| Bug caught (by you or an advisor) | Bugs caught & fixed | Description + fix |
| File created / modified | Files changed | Path (grouped "New" / "Modified") |
| Task completed | Top badge + Rollout checklist | `IN PROGRESS` → `DONE`, populate checklist |

**Do not update** for: internal debugging, tool-call retries, syntax fixes, reading files to understand code, intermediate steps that don't affect the outcome.

### Step 4 — Final pass

Before reporting "done":
1. Re-read the HTML. Confirm: all progress rows = `done`, top badge = `DONE`, files changed complete, rollout checklist has clear items.
2. Optional "Smoke check" section: test/import-check commands run + outcome.
3. Notify the user in 1 line: "Status file updated."

## Structure of `implementation_status.html`

4 mandatory sections, in order:

1. **Progress** — 3-column table (`#`, `Step`, `Status`). Tags: `t-done`/`t-prog`/`t-todo`.
2. **Decisions + Compromises + Bugs** — merged / split into `.decision`-style cards. Each entry: short title + **Why** + **How to apply** / **Fix**.
3. **Files changed** — split "New" / "Modified".
4. **Rollout checklist** — ordered list of steps the user takes after merge.

Optional when relevant: **Smoke check**, **Things you should know**.

Use the `assets/status_template.html` template. Do **not** write HTML from scratch — always start from the template.

## Folder structure the user ends up with

```
docs/features/<feature>/
├── README.md                       (optional index, on completion)
├── proposal.html                   (if the user asked for a design phase)
└── implementation_status.html      (live tracker)
```

If the folder already exists → **append** to existing files, don't overwrite. Read files before editing.

## Why this skill exists

The user runs many implementation tasks in parallel. They need an artifact that, read a week later, still explains "what Claude did, what it decided differently from the design, what needs review". A long end-of-session answer isn't enough. HTML allows structure (tables, sections), is easy to share, and diffs in git.

This skill does not replace PR descriptions / commit messages. It's a **scratchpad for the user**.

## Further reading

- `assets/status_template.html` — HTML template for `implementation_status.html`.
- `assets/proposal_template.html` — HTML template for `proposal.html`.
- `references/triggering.md` — list of trigger / pause / resume phrases.
