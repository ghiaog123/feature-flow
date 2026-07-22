---
name: ff-planning
description: Turn a settled problem/decision into a thoroughly challenged implementation plan. Claude Code and Codex deep-dive into the real code together (Claude fans out parallel sub-agents when scope is large), debate as peers via /codex-think-about to agree on the implementation approach, write the plan to a .md file with a dependency contract (Owns files/Depends on), then run /codex-plan-review so both sides review the plan (adversarial, fixing in place until APPROVE or stalemate). Use when the user says "draw up an implementation plan", "deep dive into the code then make a plan", "have claude and codex discuss the approach then review the plan", "implementation plan for this problem", "plan then review the plan", "create a plan with adversarial review". Usually runs AFTER ff-problem-solver (receiving analysis_brief.md) but also accepts a clearly stated problem directly. NOT for: debating an unsettled decision (ff-problem-solver), reviewing written code (codex-review), progress tracking (ff-impl-status), writing tests (ff-test-case-writer).
---

# Implementation Planner (Claude ⇄ Codex)

From a **settled decision/problem** → an **implementation plan that has been challenged**. Two peer-debate phases in sequence:

```
Decision/problem (usually analysis_brief.md from ff-problem-solver)
   │
   ▼
[1] Code deep-dive (Claude independent, information-walled; adaptive: inline or fan-out sub-agents)
   │
   ▼
[2] Debate HOW to implement   ← delegate /codex-think-about
   │
   ▼
[3] Write plan to a .md file (with dependency contract: Owns files / Depends on)   ← this skill
   │
   ▼
[4] Review plan (adversarial, fix in place)   ← delegate /codex-plan-review
   │
   ▼
[5] Final APPROVED plan + action steps
```

This skill **does not invent its own codex protocol** — it chains two existing engines (`/codex-think-about` for approach debate, `/codex-plan-review` for plan review) and adds the deep-dive + plan-writing in between.

## Core principles

- **Claude and Codex are peers** — no reviewer/implementer split. Two engineers challenging each other.
- **Information wall** in the deep-dive phase: Claude MUST read the code and form its own approach BEFORE reading Codex output.
- **Stick to the settled solution (MANDATORY)**: the plan only covers *how to implement* the settled decision — do NOT drift/change/add/remove scope. If the deep-dive reveals the settled solution **doesn't work** → **STOP, escalate back to `ff-problem-solver`** to re-decide; do NOT re-decide the solution inside this skill. (Fidelity flows back upstream.)
- **Plan grounded in real code**, not theory — every step must point to real files/functions/modules it will touch.
- **Dependency-honest**: each step declares `Owns files` (files it owns) + `Depends on` (prerequisite steps). Only mark "parallelizable" when files are disjoint AND there are no mutual dependencies. In doubt → default to **sequential**. Falsely declaring "parallel" is dangerous for the executor.
- **Codex does NOT modify project files** in the debate phase (`/codex-think-about`). In the review phase (`/codex-plan-review`), the only file modified **is the plan file**, not code — and Claude edits it, not Codex.
- **Plan as `.md`** so `/codex-plan-review` can easily fix it in place.
- **Artifacts to disk, main stays lean**: sub-agent deep-dive reports + debate transcripts are intermediates; the result lives in the plan `.md`.

## Workflow

### Step 0 — Take input

Requires a **decision/problem clear enough to plan for**:

- If the user just ran `ff-problem-solver` → **use `analysis_brief.md` as the primary input** (decision + rationale + trade-offs + next steps + constraints). This is the source of truth; the plan must stick to it.
- If there is no brief yet → ask the user to state: what needs doing, success criteria, constraints (stack, what must not change, deadline). If the problem still has multiple undecided options → suggest running `ff-problem-solver` first; don't plan on an unsettled foundation.
- Determine the **feature/scope** → choose the output directory `docs/features/<feature>/implementation_plan.md` (per bundle convention). Ask if the feature is unclear.
- Lock the **effort** for Codex (default `high`).

### Step 1 — Code deep-dive (INFORMATION WALL, adaptive, Claude independent)

Before calling Codex, Claude reads the code itself to understand the terrain. **Pick the approach by scale (adaptive-by-size — an anti-over-engineering gate):**

- **Small scope / known location** → deep-dive **inline** with Glob/Grep/Read right on main.
- **Large scope / broad sweep required** (many modules, unknown touchpoints) → **fan out in parallel, keep main lean**:
  1. **Cheap index first**: if a code-indexing tool exists (e.g. `graphify`) → run it for a quick map; otherwise → one Glob/Grep pass to locate the area.
  2. **Fan-out in 1 message**: multiple `subagent_type: 'ff-sonnet-explorer'` sub-agents in parallel, each covering one angle (entry points, modules to change, data model, integration points, existing tests, conventions) — Sonnet handles search/read/synthesize work at near-parity for a fraction of the cost (reserve Opus for implementation, Fable for judgment). The agent's own definition enforces the tight `EXPLORE REPORT` format (`file:line` findings, no code dumps, read-only).
  3. **Synthesis on main**: merge reports into an **impact-area map** (file:line touchpoints, dependencies, fragile spots). Main does not ingest whole files.

From that map, Claude drafts its own **preliminary implementation approach**: the steps, ordering, files each step touches, risks, parts needing experimentation. This must be done and be Claude's independent position BEFORE Step 2.

**If this implementation ports/adapts from a reference** (another language, library, repo, prior art) → add a **reference analysis** pass before drafting steps: read the actual reference code, extract **matching fragments** (what maps 1:1), **semantic differences** (API/type/behavior mismatches), and **platform-/stack-specific gotchas** that don't carry over intact (concurrency model, memory management, timezone/locale, implicit errors, initialization order...). Record them so the plan does NOT assume reference behavior transfers automatically to the target stack. Results feed the plan's **"References & gotchas"** section (Step 3). No reference to port → skip this pass.

**Fidelity check right here**: if the deep-dive shows the solution settled in the brief **contradicts the real code / is infeasible** → STOP, state clearly why, escalate back to `ff-problem-solver`. Don't silently swap the solution and plan for something else.

### Step 2 — Debate HOW to implement: invoke /codex-think-about

Delegate the approach debate to `/codex-think-about` (the engine handles init→poll→cross-analysis→resume→consensus/stalemate→cleanup).

Invoke Skill `codex-think-about`, passing:
- **QUESTION** = "What is the best implementation approach for `<the settled decision>`?" (with success criteria). Debate *how to do it*, don't reopen *whether to do it*.
- **PROJECT_CONTEXT** = impact-area map + conventions + constraints from Steps 0-1 + summary of `analysis_brief.md`.
- **RELEVANT_FILES** = touchpoint files found in Step 1 (paths).
- **CONSTRAINTS** = what must not be violated (including: no changing the scope of the settled solution).
- **EFFORT** = the locked level.

Bring the **preliminary approach from Step 1** as Claude's opening position. Debate to converge on: approach architecture, step ordering, integration points, test/rollback strategy, trade-offs, and **file-ownership boundaries between steps** (feeding the dependency contract in Step 3). Respect all codex-think-about rules (information wall, File Modification Guard, always finalize+stop) — **but cap the debate at 5 rounds**: past round 5 without consensus, force-exit as a stalemate and report the deadlocked points (an uncapped debate re-feeds the accumulated transcript every round — cost grows quadratically in rounds). If the skill is unavailable → tell the user to install codex; do NOT improvise a protocol.

### Step 3 — Write the implementation plan to a .md file (with dependency contract)

Synthesize the debate outcome into an **actionable** plan, written to `docs/features/<feature>/implementation_plan.md`. Minimum structure (see `references/plan-template.md`):

| Section | Content |
|---|---|
| **Goals / Outcomes** | What counts as done (source to derive acceptance criteria) |
| **Context & decision** | Decision summary + rationale (link `analysis_brief.md` / decision record if any) |
| **Impact area** | Table of file:line to be touched + role |
| **References & gotchas** *(if porting)* | Matching code fragments, semantic differences, platform-/stack-specific gotchas — only when the implementation adapts a reference (from the Step 1 reference analysis) |
| **Implementation steps** | Numbered; each step declares: what to do, **Owns files** (files it owns), **Depends on** (prerequisite steps), parallelizable or not, **Uncertainty** (high/medium/low — the more likely a decision needs rework, the higher) |
| **Test & verify** | How to verify each part + pass criteria |
| **Risks & rollback** | What can break + how to back out |
| **Out of scope** | Deliberately not done this time |

The **dependency contract** (`Owns files` / `Depends on` per step) is the contract that lets `ff-implement` parallelize safely: steps with disjoint files + no dependencies → run in parallel; everything else sequential. Declare **honestly** — in doubt, keep sequential.

**Tweakable-first (Uncertainty)** layers **ON TOP OF** the dependency contract, it does not replace it: each step also gets an uncertainty flag (high/medium/low). The decisions **most likely to need rework** — architecture choices, schema shapes, unverified assumptions — must be stated **first and loudest** so review targets them **while changing is still cheap**; mechanical/certain work is batched or pushed later. Note: the uncertainty flag only sets **attention order / review priority**, while `Depends on` is what determines the real **execution order / parallelization** — don't conflate the two axes.

If the project needs a JSON plan per `.claude/schemas/plan-schema.json` (see `.claude/docs/plan-execution-guide.md`) → generate it alongside; but the `.md` version is the one `/codex-plan-review` reviews. Keep clear headings (`#`/`##`) — `/codex-plan-review` needs the heading structure to anchor on.

### Step 4 — Review the plan: invoke /codex-plan-review

Hand the freshly written plan to `/codex-plan-review` so Claude and Codex review it adversarially (the engine handles the poll→verdict→apply/rebut→resume loop until APPROVE or stalemate, fixing the plan in place).

Invoke Skill `codex-plan-review`, passing:
- **PLAN_PATH** = absolute path to the `implementation_plan.md` just written.
- **USER_REQUEST** = the original decision/problem (link `analysis_brief.md` if any).
- **SESSION_CONTEXT** = this plan came from a Claude⇄Codex debate; constraints; feature scope; **the plan must stick to the settled solution** (don't let review widen scope).
- **ACCEPTANCE_CRITERIA** = taken from the Goals/Outcomes section.
- **EFFORT** = the locked level.

Every valid issue Codex raises → Claude **fixes directly in the plan file** then resumes; invalid issues → rebut with reasons. Loop until `verdict === APPROVE` or stalemate, **hard cap 5 rounds** — past the cap, treat as stalemate: list the open points and hand the decision to the user instead of burning more rounds. Always finalize+stop. If review proposes **changing the core solution** (not the implementation approach) → that's a signal to go back to `ff-problem-solver`, not something to decide inside plan review.

### Step 5 — Wrap up + handoff

- **APPROVE** → report the plan is ready to implement, the file path, and a summary: review rounds, issues found/fixed/rebutted, remaining risks, next steps.
- **Stalemate** → list the deadlocked points (`Point | Claude | Codex`), recommend which side to lean toward, ask the user to decide.
- **Chain**: the APPROVED `implementation_plan.md` is the input for `ff-implement` (implement + code review). Mention the bundle's next steps: implement → `ff-impl-status` (tracking), `ff-test-case-writer`/`ff-service-test-runner` (tests), `ff-feature-brief` (PO/QC handoff).

## Context hygiene (controlled compact)

Don't run a bare `/compact`. After the plan is APPROVED, propose a compact block with the real path filled in, clear KEEP/DROP:

```
/compact KEEP: contents of docs/features/<feature>/implementation_plan.md (APPROVED version, incl. dependency contract),
summary: review round count + issues fixed, remaining risks.
DROP: sub-agent deep-dive reports, /codex-think-about and /codex-plan-review transcripts, intermediate plan drafts
(already synthesized into the plan file; diffs remain in git).
```

Anything not yet in the plan file → flag as must-keep.

## Anti-patterns

- ❌ Skipping the Step 1 deep-dive and debating the approach in a vacuum → plan not grounded in real code.
- ❌ **Drifting from the settled solution** — changing/adding/removing scope yourself instead of escalating back to `ff-problem-solver`.
- ❌ Planning when the decision isn't settled (should have run `ff-problem-solver` first).
- ❌ Ingesting whole files into main when scope is large (should fan out sub-agents returning compact reports); or fanning out sub-agents for a small known-location task (over-engineering).
- ❌ Carelessly declaring "parallelizable" when files overlap/depend on each other → the executor breaks. In doubt, sequential.
- ❌ Ordering the plan purely by dependency and burying the riskiest architectural decision in the middle → discovering it's wrong when fixing is already expensive. Surface the high-uncertainty parts first and loudest.
- ❌ Porting from a reference while assuming behavior transfers to the target stack, without recording gotchas / semantic differences.
- ❌ Writing a prose plan with no numbered steps / no file touchpoints / no dependency contract → unreviewable, unparallelizable.
- ❌ Letting Codex modify project code (only the plan file may be modified, and by Claude).
- ❌ Hand-rolling a codex-calling protocol instead of delegating to the two existing skills.
- ❌ Accepting a fake APPROVE when `/codex-plan-review` hasn't actually produced `verdict: APPROVE`.
