---
name: ff-problem-solver
description: Solve a hard technical problem by having Claude Code and Codex debate as two peer experts, then lock in an actionable conclusion. Runs 2 stages depending on the original request — Stage 1 ASSESSMENT (diagnosis, always runs), Stage 2 SOLUTION (proposed solution, only when the user asks). The skill takes a problem (architecture decision, technology choice, hard bug, design trade-off, murky root cause), clarifies scope + intent, lets Claude analyze independently, then invokes /codex-think-about for Claude and Codex to challenge each other over multiple rounds, and finally merges into assessment.md and (if requested) analysis_brief.md + a decision record. Use when the user says "solve this problem", "have the two sides debate", "have claude and codex discuss this", "challenge it then lock in a conclusion", "how should we approach this problem", "analyze/assess this", "give the final conclusion for...", "debate then decide". Do NOT use for reviewing written code/PRs (use codex-review), producing docs/tests/status (other skills in the bundle), or simple questions answerable immediately without debate.
---

# Problem Solver (Claude ⇄ Codex)

Solve hard technical problems through a **peer debate** between Claude and Codex, then **lock in an actionable conclusion**. This skill is a wrapper around `/codex-think-about`: it adds **problem framing** up front, **two debate stages** in the middle, and **merging artifacts to disk** at the end — `/codex-think-about` handles the debate engine within each stage.

The skill produces **one of two deliverables** depending on the original request:
- **Diagnosis only** ("analyze/assess this") → stop at `assessment.md`.
- **Diagnosis + solution** ("what should we do / how to fix") → continue to Stage 2 → `analysis_brief.md`.

```
Problem (user)
   │
   ▼
[1] Frame + detect intent (diagnosis? / diagnosis+solution?)   ← this skill
   │
   ▼
[2] Code deep-dive (adaptive: inline or fan out sub-agents) — Claude independent, information barrier
   │
   ▼
╔═══ STAGE 1 — ASSESSMENT (always runs) ═════════════════════╗
║  each side analyzes independently → own md                 ║
║  → peer debate  ← delegate /codex-think-about (framing: diagnosis) ║
║  → merge assessment.md                                      ║
╚═════════════════════════════════════════════════════════════╝
   │
   ├─ request does NOT ask for a solution → deliver assessment, STOP
   │
   └─ request DOES ask for a solution
        │
        ▼
╔═══ STAGE 2 — SOLUTION (only when asked) ═══════════════════╗
║  from the assessment → each side proposes independently → own md║
║  → peer debate  ← delegate /codex-think-about (framing: solution)║
║  → merge analysis_brief.md                                  ║
╚═════════════════════════════════════════════════════════════╝
   │
   ▼
[Controlled compact] → chain to ff-planning
```

## Core principles

- **Claude and Codex are peers** — neither is the reviewer, neither is the executor. Two experts challenging each other.
- **Information barrier**: Claude MUST complete its independent analysis BEFORE reading any Codex output. No peeking, to avoid anchoring bias.
- **Debate to converge, not to win**: the goal is the correct conclusion, not defending the initial position. Changing your mind when the other side has better evidence is a good outcome.
- **Separate facts from opinions**: verifiable claims must have sources; inferences/judgments are explicitly labeled as opinion.
- **Codex does NOT modify project files** — thinking + (if needed) web research only. If Codex is caught modifying files, stop immediately per `/codex-think-about`'s File Modification Guard.
- **Reuse the debate engine, compose it multiple times**: each stage calls `/codex-think-about` once with different framing (diagnosis ≠ solution). Don't invent your own codex protocol.
- **Artifacts to disk, main stays lean**: each stage's final result is written to a `.md` file; debate transcripts + each side's md + deep-dive reports are intermediates — discard after merging (see "Context hygiene").

## Workflow

### Step 1 — Frame the problem + detect intent

Before debating, clarify so both sides solve the same problem. Ask the user (batch at most 2-3 questions, don't interrogate):

| Need | Why |
|---|---|
| **Problem statement** in one sentence, clear on "what to decide/understand" | Prevents the two sides from debating different problems |
| **Success criteria** / constraints (perf, deadline, stack, what may not change) | Makes the conclusion measurable, not generic |
| **Project context** + relevant files/code | Grounds the debate in reality, not pure theory |
| **Options already under consideration** (if the user has any) | Extend/challenge instead of starting from zero |
| **Effort level** for Codex (default `high`) | Hard problem → high effort |

**Intent detection (decides the deliverable):**
- **Diagnosis-only** signals: "analyze", "assess", "understand why", "what is happening", "what's the root cause" → stop at Stage 1.
- **Diagnosis + solution** signals: "what should we do", "how to fix", "solution", "how to handle", "decide which option" → continue to Stage 2.
- **Ambiguous** → use `AskUserQuestion` to ask whether to stop at diagnosis or continue to a solution. Don't guess.

If the problem statement is vague → propose a sharper rewrite, confirm with the user, then run. (See codex-think-about's `references/question-sharpening.md` if you need a clarification framework.)

**Vocabulary ladder (when the statement uses fuzzy words — "make it better", "fix it so it's fast", "improve this"):** before framing, translate fuzzy words → precise technical language:
- **List the fuzzy words**: exactly the unmeasured words ("better", "fast", "clean", "tidy").
- **Anchor each word to a concrete technical meaning**, grounded in the real codebase/domain ("fast" = p95 latency of endpoint X < 200ms; "better" = reduce coupling between modules A↔B).
- **Confirm the anchored version with the user** before running.
Purpose: both sides must debate **the same sharpened problem**, not two different fuzzy ones.

**Do not offer solution opinions at this step** — stay neutral until the debate.

### Step 2 — Code deep-dive (INFORMATION BARRIER, adaptive)

Before calling Codex / reading any Codex output, Claude maps the problem terrain itself. **Pick the approach by size (adaptive-by-size — the anti-over-engineering gate):**

- **Small problem / narrow scope** (a few files, location known) → deep-dive **inline** with Glob/Grep/Read right on main. No sub-agents.
- **Large problem / wide sweep needed** (many modules, touchpoints unknown, or web research also needed) → **fan out in parallel**:
  1. **Cheap indexing first**: if a code index tool exists (e.g. `graphify`), run it for a quick map; otherwise → one Glob/Grep pass to locate the area.
  2. **Fan-out in 1 message**: multiple `subagent_type: 'ff-sonnet-explorer'` sub-agents in parallel, each on one angle (module, data flow, existing tests, docs/web) — Sonnet handles search/read/synthesize work at near-parity for a fraction of the cost. The agent's own definition enforces the tight `EXPLORE REPORT` format (`file:line` findings, no code dumps, read-only).
  3. **Main keeps only the merged report** — don't swallow entire files into main context.

From that understanding, Claude writes its **independent analysis**: possibilities/hypotheses, trade-offs, risks, preliminary recommendation + confidence + assumptions being relied on. MAY use MCP tools (web_search, context7) — cite sources.

This analysis must be **complete and final** before entering the debate. It is Claude's "independent ballot".

**Blindspot Pass (unknowns check — mandatory, right after the independent analysis, BEFORE the debate):** Claude audits what it has NOT verified — assumptions being relied on, code/domain areas not read, edge cases not considered, "where could I be wrong here?". Output a short structured list, each unknown with a **confidence tag** (High/Medium/Low) and **how to close it** (which file to read / whom to ask / what to test). These are NOT weaknesses to hide — each blindspot becomes an **explicit debate item** for Codex to attack; the debate must probe the unknowns, not just the stated hypotheses. This list goes straight into the debate framing and into `assessment.md`.

### Stage 1 — ASSESSMENT (always runs)

**Goal**: diagnose the problem correctly (nature, causes, constraints, risks) — NO solution talk yet.

1. **Each side independently → own md**: Claude writes its diagnosis; Codex (via the debate engine) gives its own. Maintain the information barrier.
2. **Peer debate**: invoke Skill `codex-think-about`, framed on **diagnosis**:
   - **QUESTION** = "Diagnose the problem: `<statement>` — what are its nature/causes/constraints/risks?"
   - **PROJECT_CONTEXT** = context + constraints + success criteria + the merged deep-dive report from Step 2 + **the blindspot list** (unverified unknowns) so Codex probes them, not just the stated hypotheses.
   - **RELEVANT_FILES** = relevant files/code (paths, don't paste whole blocks).
   - **CONSTRAINTS** = what must not be violated.
   - **EFFORT** = the agreed level (default `high`).
   Bring the **independent analysis from Step 2** as Claude's opening position — don't discard it and rethink from scratch.
3. **Merge `assessment.md`**: write `docs/features/<feature>/assessment.md` (or where the user specifies) per `references/assessment-template.md`. During the debate, classify each point: **True consensus / True disagreement / Claude-only insight / Codex-only insight / Same direction, different depth**. Also record the **blindspots** closed / still open after the debate + confidence in the "Blindspots / open unknowns" section.

Respect all codex-think-about rules (information barrier, exit only on consensus or stalemate, File Modification Guard, always finalize+stop) — **but cap each debate at 5 rounds per stage** (this skill can run the engine twice: Stage 1 + Stage 2): past round 5 without consensus, force-exit as a stalemate and record the remaining disagreements in the `Point | Claude | Codex` table — an honest capped stalemate beats an unbounded loop re-feeding the whole transcript every round. If `/codex-think-about` is unavailable → tell the user to install it; do NOT invent your own codex protocol.

### Branch — stop or continue?

- **Request does NOT ask for a solution** → present `assessment.md`, summarize the diagnosis + remaining disagreements (if stalemate), **STOP**. Suggest: if a solution is wanted later, rerun this skill with a "propose a solution" request (reusing assessment.md).
- **Request DOES ask for a solution** → go to Stage 2.

### Stage 2 — SOLUTION (only when asked)

**Goal**: from the locked diagnosis → propose an actionable solution.

1. **Each side proposes independently → own md**: based on `assessment.md`, Claude proposes its option; Codex proposes its own. Information barrier again.
2. **Peer debate**: invoke Skill `codex-think-about` again, framed on **solution**:
   - **QUESTION** = "Given the locked diagnosis, what is the best solution for `<statement>`?"
   - **PROJECT_CONTEXT** = summary of `assessment.md` + constraints + success criteria.
   - **RELEVANT_FILES**, **CONSTRAINTS**, **EFFORT** as above.
3. **Merge `analysis_brief.md`**: consolidate into an actionable brief, written to `docs/features/<feature>/analysis_brief.md` per `references/brief-template.md`:
   1. **Decision**: which option (or combination) is chosen — decisively.
   2. **Rationale**: 2-4 main reasons, based on consensus + the strongest evidence from the debate.
   3. **Accepted trade-offs**: be blunt about what is sacrificed.
   4. **Remaining disagreements** (if stalemate): a `Point | Claude | Codex` table + a recommendation on which side to lean toward; if the user truly must decide → ask.
   5. **Next steps**: a concrete action checklist to implement the decision.
   6. **Risks / needs further verification**: what's uncertain, how to check.
   7. **Consolidated sources** + **confidence** of the conclusion.

### Final step — Decision record (optional) + chain

- **Decision record (optional, ask the user)**: if it's an architecture decision needing traceability → write `docs/decisions/<slug>-<YYYY-MM-DD>.md` per `references/decision-record.md`.
- **Chain**: `analysis_brief.md` is the natural input for `ff-planning` (adversarially-reviewed planning). Remind the user of the next step; state the brief's path.

## Context hygiene (controlled compact)

Never run a bare `/compact`. After merging artifacts to disk, propose a compact block **with real paths filled in, explicit about what to KEEP / DROP**:

```
/compact KEEP: contents of docs/features/<feature>/assessment.md (+ analysis_brief.md if Stage 2 ran),
the sharpened problem statement, the intent (diagnosis/solution), the decision + remaining disagreements.
DROP: the 2 /codex-think-about debate transcripts, each side's analysis md, sub-agent deep-dive reports
(already synthesized into the artifacts; diffs/details remain in the files on disk).
```

Anything **not yet** written into an artifact → flag as must-keep; don't let compact swallow it.

## Anti-patterns

- ❌ Skipping Step 2, calling Codex straight away and going along with Codex's view (breaks the information barrier → loses the value of the challenge).
- ❌ Skipping the Blindspot Pass, debating only the obvious hypotheses (ignoring the unknowns → the debate loses half its value).
- ❌ Debating a vague statement without anchoring the vocabulary first (the two sides solve two different problems).
- ❌ Jumping straight to solutions when the request only asks for a diagnosis (skipping the Stage 1 stop branch).
- ❌ Guessing intent when ambiguous instead of using `AskUserQuestion`.
- ❌ Stopping at "here's what both sides said" without merging into an actionable artifact.
- ❌ Swallowing whole files into main context when sub-agents returning tight reports should have been fanned out (large problem).
- ❌ Fanning out sub-agents for a small problem with a known location (over-engineering) — inline is enough.
- ❌ Rewriting the codex-calling protocol yourself instead of delegating to `/codex-think-about`.
- ❌ Forcing fake consensus when real disagreement remains — an honest stalemate beats forced agreement.
- ❌ Letting Codex modify project files without stopping per the File Modification Guard.
- ❌ A bare `/compact` losing an assessment/brief not yet written to disk.
