---
name: ff-implement
description: Implement an implementation plan using a 3-model architecture — Fable 5 (architect) analyzes dependencies to decide parallel/sequential, Opus fans out sub-agents to implement, Fable serves as a read-only advisor for big technical decisions — then have Claude Code and Codex jointly review the freshly written code via /codex-impl-review, ensuring code quality AND plan conformance. After implementation, invoke /codex-impl-review for an adversarial two-party review on uncommitted changes (working-tree) or branch diff, fixing until APPROVE or stalemate, also checking plan conformance. Use when the user says "read the plan then implement", "implement then review the code", "code per the plan then have codex review", "implement this plan", "implement and review the code just written", "make sure the code follows the plan". Usually runs AFTER ff-planning (plan already APPROVED). NOT for: making a plan (ff-planning), debating decisions (ff-problem-solver), reviewing a plan before coding (codex-plan-review), reviewing code only without implementing (codex-impl-review directly).
---

# Plan Implementer (Fable orchestrates → Opus implements → Claude ⇄ Codex review)

From a **finalized implementation plan** → **code written + reviewed to quality + plan-conformant**. 3-model architecture — the strongest model steers, cheaper models do the typing:

- **Fable 5 (architect/advisor)**: analyzes the dependency contract → decides which phases run in parallel vs sequentially; and acts as a **read-only advisor** when Opus hits a big technical decision.
- **Opus (implementer)**: `ff-opus-implementer` sub-agents fan out to write code per the DAG, each agent owns its own files, receives a 5-part brief (objective/files/interfaces/constraints/verification), self-verifies and returns a report with evidence.
- **Codex (reviewer)**: adversarial review via `/codex-impl-review` — unchanged from before.

```
implementation_plan.md (usually from ff-planning)
   │
   ▼
[1] Claude reads the plan, confirms scope → Fable analyzes the DAG (parallel/sequential)
   │
   ▼
[2] Phase 0 contract-first on main → fan out OPUS sub-agents per the DAG
   │      (Opus hits a big decision → consult FABLE ADVISOR, fresh agent, verdict ~300 tokens)
   ▼
[3] Integration join + self-check vs plan (build/test on main)
   │
   ▼
[4] Code review: Claude ⇄ Codex   ← delegate /codex-impl-review (1 join on main, no fan out)
   │
   ▼
[5] Wrap up + handoff + controlled compact
```

This skill **does not invent its own codex protocol** — the review part is delegated to `/codex-impl-review`. Added value: reading & following the plan while implementing, assigning model roles by strength/cost, orchestrating safely, and making **plan conformance** an explicit review criterion.

## Core principles

- **Follow the plan**: every implementation step points back to a plan item. Deviating from the plan → either update the plan with a reason, or don't deviate. No silent drift from the plan.
- **Assign roles by cost**: Fable only does the "expensive reasoning" work (DAG analysis, verdicts on big decisions) — each call short and targeted. Opus does all the code typing (~90% of the session's tokens). Don't use Fable to implement; don't use Opus to judge hard architectural decisions.
- **Orchestrator stays lean**: main = conductor — keep the **dependency map + conformance report in a FILE**, not in chat memory. Each phase's guts live in sub-agents; main doesn't swallow code.
- **Parallel only when disjoint + no-dep + contract-first**: two phases run in parallel ⟺ `Owns files` are disjoint AND no dependency (per the DAG Fable approved), AND the shared seam is locked in Phase 0.
- **5-part brief + concise report**: each `ff-opus-implementer` sub-agent receives a 5-part brief (objective/files/interfaces/constraints/verification — the implementer doesn't see the conversation, the brief must stand on its own), edits **ONLY the files it owns**, returns a concise `PHASE REPORT` with verification evidence, **NO code dumps**.
- **Advisor behind a gate (consult gate)**: Opus ONLY asks the Fable advisor when a decision (a) cannot be derived from the plan/existing conventions AND (b) affects many files or is hard to reverse — OR (c) **the same problem has been tried twice and failed** (hard threshold: 2nd failure → mandatory consult, no blind grinding onward). Small decisions → Opus decides itself + records in the deviation report. The advisor is a **read-only skeptic** — use the `ff-fable-advisor` agent (only has Read/Grep/Glob tools; even if it wanted to edit files it has no tool for it): advises only, never edits files.
- **Reports are claims, not evidence**: main does not trust a sub-agent's "done" report — it must read the phase's git diff itself + run the verification command itself before marking the phase done in the conformance file. A report missing real command output = not done; a phase reporting a gap → resend a fixed brief, don't interpret on its behalf.
- **Advisor answers concisely**: verdict + short reasoning, soft limit **~300 tokens** — "emit judgment not volume". Don't truncate if the problem genuinely needs more, but concise is the default.
- **Review = 1 join on main, no fan out**: `/codex-impl-review` reads the **git diff**, not the chat. Claude writes code; Codex only reviews; Claude applies fixes after evaluating each issue.
- **Need to edit code → must exit plan mode** (this skill modifies files). `/codex-impl-review` requires exiting plan mode first.
- **Adversarial review to the end**: only stop at `verdict === APPROVE` or stalemate. No fake APPROVEs.
- **Preserve functional intent**: fixes from review don't change behavior unless the issue demands a behavior change.

## Workflow

### Step 1 — Read the plan + confirm scope + Fable analyzes the DAG

- Find the plan: user gives a path → use it; otherwise → `docs/features/<feature>/implementation_plan.md` (output of `ff-planning`). Multiple/none → ask.
- Read the plan: grasp the Goals/Outcomes, steps, **dependency contract** (`Owns files`/`Depends on`/parallel?), affected areas (file:line), test & verify, out of scope.
- If the plan hasn't been reviewed (`ff-planning`/`codex-plan-review`) → tell the user, suggest reviewing the plan first; still allow proceeding if the user wants.
- **Fable analyzes the DAG**: spawn an agent with `model: 'fable'`, input = dependency contract + phase list + relevant plan excerpts. Task: build the DAG (node = phase, edge = `Depends on`), decide **which batches run in parallel, which sequentially**, flag phases touching shared files that must be merged/serialized, and identify the **shared seam** to lock in Phase 0. Concise output (DAG + reasoning), implements NOTHING.
- Write Fable's DAG + batch plan into the working file `docs/features/<feature>/impl_progress.md` — this file is also the **conformance report** (per phase: status / file:line touched / deviations / advisor consults) updated incrementally.
- Lock the later review scope: **working-tree** (uncommitted changes, default) or **branch** (diff vs base). Repo is clean → changes will be created in Step 2.

### Step 2 — Phase 0 contract-first + fan out Opus sub-agents per the DAG

Main = **conductor**, phase guts live in sub-agents. Exit plan mode first (this skill modifies files). Sequence:

1. **Phase 0 — contract-first (on main)**: lock the **shared seam** (data types, function signatures, interfaces, stubs) that Fable identified, FIRST on main. Subsequent parallel phases only fill in this fixed contract → no stepping on the shared seam.
2. **Fan out Opus sub-agents per Fable's DAG**: parallel batch → multiple `subagent_type: 'ff-opus-implementer'` sub-agents in 1 message; sequential batch → run along dependency edges. Each sub-agent receives a **self-sufficient 5-part brief** (the implementer doesn't see the conversation — the brief must stand on its own):
   1. **Objective** — phase goal + related Outcomes + that phase's plan excerpt.
   2. **Files** — `Owns files` (may only edit these files).
   3. **Interfaces** — the contract from Phase 0 (types, signatures, API shape to match).
   4. **Constraints** — conventions to follow, no-go zones.
   5. **Verification** — command proving the phase works; require running it and putting real output in the report.

   The agent knows the advisor consult protocol + the `PHASE REPORT` format itself (defined in `agents/ff-opus-implementer.md`): each plan item → status → file:line, with `VERIFIED`/`CONSULTS`/`DEVIATIONS`/`GAPS`, **NO code dumps** (the diff lives in git). If two phases must touch a shared file → **not** parallel; serialize or merge into one sub-agent (Fable should have caught this in Step 1; a sub-agent discovering more → stop, report to main).
3. **Fable advisor consult protocol** (applies to each Opus sub-agent):
   - **When (consult gate)**: ONLY when a technical decision (a) cannot be derived from the plan/existing conventions AND (b) affects many files or is hard to reverse — e.g. choosing between two data-structure approaches affecting the API, plan ambiguity at an architectural fork; OR (c) **the same problem has been tried twice and failed** → mandatory consult before attempt 3. Small decisions → decide + record a `deviation`.
   - **How to ask**: spawn a NEW `subagent_type: 'ff-fable-advisor'` agent for EACH question (the advisor reads code fresh, carrying no accumulated assumptions; this agent only has Read/Grep/Glob tools — read-only by permission, not by instruction). The brief sent to the advisor must be **self-sufficient but distilled**: the specific question + the options under consideration with trade-offs + the relevant plan excerpt + **file:line pointers** so the advisor reads the real code itself (don't paste whole files).
   - **Answer constraints**: the advisor is a read-only skeptic — returns a **verdict + the deciding risk + short reasoning, soft limit ~300 tokens**; no file edits, no writing code on someone's behalf, advice only. Opus makes the final call and owns it, recording the verdict in its report.
4. **Main collects reports + self-verifies** into the conformance file (`impl_progress.md`) after each batch — including advisor consults; no swallowing sub-agent code into context. **Reports are claims, not evidence**: before marking a phase = done, main reads the `git diff` of the phase's `Owns files` + runs the phase's verification command (if the plan has one). A "done" report without real command output, or a diff that doesn't match the report → phase not done: resend a fixed brief to the sub-agent, don't interpret on its behalf.

→ proceed to Step 3.

### Step 3 — Integration join + self-check vs plan (on main)

- **Integration join**: after collecting all phases → run the **full build/lint/test suite on main** (not in a sub-agent). Clear integration errors → fix on main first, don't push garbage into review.
- **Self-check vs plan**: re-check every Outcome in the plan — achieved yet? Steps not done → finish them or record the deferral reason in the conformance file.
- Confirm there are changes to review: working-tree must have staged/unstaged (`git status --short`), or the branch must differ from base. No changes → don't call review.

### Step 4 — Code review: invoke /codex-impl-review

Delegate the adversarial review to `/codex-impl-review` — **a single join on main, no fan out**; the engine reads the **git diff** (not the chat, not sub-agent reports), handles init→start→poll→verdict→apply fix/rebut→resume until APPROVE or stalemate, auto-detects scope + effort by file count.

Invoke Skill `codex-impl-review`, passing:
- **USER_REQUEST** = the feature goal + "this code implements the plan at `<plan path>`".
- **SESSION_CONTEXT** = **plan conformance is an explicit review criterion** — list the Outcomes + plan steps so Codex checks the code follows them, plus conventions/constraints. If there are intentional deviations from the plan (recorded in the conformance file, including deviations per the Fable advisor's verdict) → state the reason clearly so Codex doesn't misflag them. Don't paste sub-agent reports — only point at the plan + let the engine read the diff.
- Scope (working-tree/branch) + base branch if branch.
- Effort: let the engine infer from file count (`<10`=medium, `10-50`=high, `>50`=xhigh) unless the user overrides.

For each issue Codex raises: valid → Claude **fixes the code**, verifies, then resumes; invalid → rebut with concrete evidence. Loop until `verdict === APPROVE` or stalemate. Codex does NOT edit files. Always finalize+stop.

Especially scrutinize: **places where the code doesn't match the plan** (missing steps, going a different direction than agreed) — that's the main reason to run this skill, beyond ordinary bugs/quality.

### Step 5 — Wrap up + handoff

- **APPROVE** → report: files touched, review round count, issues found/fixed/rebutted, defects fixed by severity, remaining risks, final plan conformance (cross-checked against the conformance file), and a summary of Fable advisor consults (what decision, what verdict).
- **Stalemate** → list the deadlocked points (`Point | Claude | Codex`), recommendation, ask the user to decide.
- Final cross-check: every Outcome in the plan ✅ or what's still owed.
- Suggest the bundle's next step: `ff-impl-status` (update progress), `ff-api-contract-writer` (API docs), `ff-test-case-writer`/`ff-service-test-runner` (test plan + run), `ff-feature-brief` (PO/QC handoff). Commit/PR only when the user asks.

## Context hygiene (controlled compact)

Don't run a bare `/compact` — context balloons easily from the many collection turns after fanout. After review is done, propose a compact block with real paths filled in, explicit KEEP/DROP:

```
/compact KEEP: conformance report docs/features/<feature>/impl_progress.md (Fable's DAG, per-Outcome status,
deviations, advisor consults), plan docs/features/<feature>/implementation_plan.md,
review results (verdict, issues fixed, remaining risks).
DROP: step-by-step edit history, detailed sub-agent reports, /codex-impl-review debate transcript
(already synthesized into the conformance report; full diff is in git).
```

Anything not yet in the conformance file/plan → flag as must-keep before compacting.

## Anti-patterns

- ❌ Implementing without reading the plan, or drifting from the plan without recording a reason/updating the plan.
- ❌ Using Fable to implement (type code) — Fable only analyzes the DAG and advises; Opus is the implementer.
- ❌ Opus asking the advisor about small decisions derivable from the plan/conventions → burns input tokens for nothing; the consult gate exists to block exactly this.
- ❌ Sending the advisor a pile of pasted files instead of a distilled brief + file:line pointers — the cost is in the INPUT, not the 300-token output.
- ❌ Treating 300 tokens as an absolute hard cap — truncating an architectural verdict mid-way is worse than spending a few hundred more tokens. Soft limit: concise by default, allowed to exceed when genuinely needed.
- ❌ Letting the Fable advisor edit files or write code on someone's behalf — the advisor is read-only (the `ff-fable-advisor` agent only has Read/Grep/Glob); Opus decides and owns it.
- ❌ Blindly grinding a 3rd time on the same problem that failed twice instead of consulting the advisor — the 2-fail threshold is hard.
- ❌ Marking a phase = done based only on the sub-agent's report, without reading the diff/running verify — reports are claims, not evidence.
- ❌ Running sub-agents in parallel when `Owns files` overlap or Phase 0 isn't locked → stepping on each other, file conflicts.
- ❌ Keeping the dependency map/conformance in chat memory instead of a FILE → main balloons, loses track when context gets cut.
- ❌ Sub-agents dumping code back to main instead of a concise report → defeats the goal of keeping main lean.
- ❌ Fanning out for the review phase, or letting `/codex-impl-review` read chat/reports instead of the git diff.
- ❌ Calling `/codex-impl-review` when there are no changes yet (clean repo) → engine pre-flight fails.
- ❌ Pushing code that fails build/test into review instead of fixing sanity at the integration join first.
- ❌ Dropping the "plan conformance" criterion from SESSION_CONTEXT → the review becomes a generic code review, losing its purpose.
- ❌ Letting Codex edit files (Codex only reviews; Claude applies fixes).
- ❌ Accepting a fake APPROVE when the engine hasn't emitted `verdict: APPROVE`.
- ❌ Committing/PR-ing on your own when the user hasn't asked.
- ❌ A bare `/compact` losing a conformance report not yet written to disk.
