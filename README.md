# feature-flow

A Claude Code plugin bundling **10 skills** and **3 agent lanes** that close the full delivery loop:

**discover → problem-solve → plan → implement + review → API docs → test plan → auto-test → stakeholder brief → deep-understanding handoff**

The bundle is built around two ideas:

1. **Adversarial peer review** — Claude Code and Codex debate as equals (via the `codex-think-about` / `codex-plan-review` / `codex-impl-review` skill trio) at every decision, plan, and code-review gate. No fake APPROVEs: loops run until genuine consensus or an explicit stalemate handed back to you.
2. **Cost-tiered model routing** — the strongest model steers, cheaper models do the typing:

| Lane | Model | Role |
|---|---|---|
| `ff-sonnet-explorer` | Sonnet | Search / read code / read docs / synthesize — token-heavy but cognitively simple; fans out in parallel, one angle per agent, returns tight `file:line` reports |
| `ff-opus-implementer` | Opus | Writes the code — receives a self-contained 5-part brief (objective / files / interfaces / constraints / verification), edits only the files it owns, returns a `PHASE REPORT` with real verification output |
| `ff-fable-advisor` | Fable 5 | Read-only skeptic consulted at commitment boundaries — architecture choices, migrations, API shapes, or after 2 failed attempts. Verdict + deciding risk in ~300 words. Never implements (it has no write tools) |
| Codex | — | Cross-vendor adversarial reviewer for plans and code |

## Install

```
/plugin marketplace add ghiaog123/feature-flow
/plugin install feature-flow@feature-flow
```

The debate/review gates require the `codex-think-about`, `codex-plan-review`, and `codex-impl-review` skills (Claude ⇄ Codex peer-debate trio) plus an authenticated Codex CLI. Without them, the planning/implementation skills still run but will tell you the review engine is unavailable rather than improvising a protocol.

## Skills

### Delivery pipeline

| Skill | What it does |
|---|---|
| **ff-discover** | Front door when the problem isn't stated yet — turns a vague itch ("something feels off here") into a sharp problem statement + known-unknowns map + option space, via structured interview, vocabulary ladder, and a blindspot pass over the code. Outputs a vivid `discovery_brief.html`. |
| **ff-problem-solver** | Solves a hard technical question by having Claude and Codex debate as peers: Stage 1 assessment (always), Stage 2 solution (on request). Ends in an actionable `assessment.md` / `analysis_brief.md` + decision record. |
| **ff-planning** | Turns a settled decision into a battle-tested implementation plan: independent code deep-dive (information-walled), approach debate via `codex-think-about`, plan written with a **dependency contract** (`Owns files` / `Depends on` / uncertainty flags), then adversarial plan review via `codex-plan-review` until APPROVE. |
| **ff-implement** | Executes an approved plan with the 3-model architecture: Fable analyzes the dependency DAG (what runs parallel vs sequential), Opus implementer sub-agents fan out per the DAG behind a contract-first Phase 0, Fable advises on big decisions, then `codex-impl-review` adversarially reviews the diff for quality **and plan conformance**. Reports are claims, not evidence — main re-verifies every phase. |

### Documentation & QA

| Skill | What it does |
|---|---|
| **ff-impl-status** | Live HTML implementation-status tracker (`docs/features/<feature>/implementation_status.html`): progress, off-design decisions, compromises, bugs, changed files, rollout checklist. Can generate a design `proposal.html` first. Resumable across sessions. Explicit-invoke only. |
| **ff-api-contract-writer** | Reads a service's source and writes a concise caller-facing Markdown API contract — request/response shapes, status codes, error conditions only. Lists discovered endpoints and waits for scope confirmation before generating. |
| **ff-test-case-writer** | Interactive HTML test plan (collapsible sections, pass/fail tracking, progress bar, localStorage persistence, Markdown export). Derives cases from the proposal when one exists; risk-based QA methodology (P0–P3, positive/negative/boundary). |
| **ff-service-test-runner** | Translates the manual test plan into executable pytest, runs it against the service, and reports PASS/FAIL/SKIP keyed to each TC-id, with a ship/no-ship verdict. Explicit-invoke only. |
| **ff-feature-brief** | Non-technical one-page HTML brief for PO/QC — what changed, why, how it flows, acceptance criteria, test results, gaps. Translates jargon into everyday metaphors with concrete persona examples. |
| **ff-comprehension-coach** | Patient expert teacher that ensures a human *deeply* understands a change — teaches incrementally, keeps a running understanding checklist, drills the WHYs, quizzes interactively, and won't "finish" until every item is demonstrated. |

Invoke any skill with `/feature-flow:<skill-name>`; most also trigger on natural-language requests (see each skill's description).

## Typical workflow

```
ff-discover            # only if the problem is still a vague feeling
   └─► ff-problem-solver     # Claude ⇄ Codex debate → settled decision
          └─► ff-planning          # deep-dive + debate → APPROVED implementation_plan.md
                 └─► ff-implement        # Fable DAG → Opus fan-out → Codex code review
                        ├─► ff-impl-status         # live progress tracker
                        ├─► ff-api-contract-writer # caller-facing API docs
                        ├─► ff-test-case-writer    # interactive HTML test plan
                        │      └─► ff-service-test-runner  # plan → pytest → per-TC results
                        ├─► ff-feature-brief       # PO/QC one-pager
                        └─► ff-comprehension-coach # make sure the human owns it
```

Every stage is skippable — enter the pipeline wherever your problem already is.

Artifacts land under `docs/features/<feature>/`:

```
docs/features/<feature>/
├── discovery_brief.html         # ff-discover
├── assessment.md                # ff-problem-solver
├── analysis_brief.md            # ff-problem-solver (solution stage)
├── implementation_plan.md       # ff-planning (APPROVED, with dependency contract)
├── impl_progress.md             # ff-implement (DAG + conformance report)
├── proposal.html                # ff-impl-status (optional design phase)
├── implementation_status.html   # ff-impl-status
├── test_plan_<slug>.html        # ff-test-case-writer
├── tests/                       # ff-service-test-runner (pytest, run by explicit path)
└── test_results_<slug>.md       # ff-service-test-runner
```

## Repository layout

```
.claude-plugin/marketplace.json      # marketplace manifest
plugins/feature-flow/
├── .claude-plugin/plugin.json       # plugin manifest
├── agents/
│   ├── ff-sonnet-explorer.md        # Sonnet — search/read/synthesize lane (read-only)
│   ├── ff-opus-implementer.md       # Opus — implementation lane (5-part brief contract)
│   └── ff-fable-advisor.md          # Fable 5 — read-only advisor at commitment boundaries
└── skills/
    ├── ff-discover/                 # SKILL.md + discovery brief HTML template
    ├── ff-problem-solver/           # SKILL.md + assessment/brief/decision-record references
    ├── ff-planning/                 # SKILL.md + plan template reference
    ├── ff-implement/                # SKILL.md (3-model orchestration)
    ├── ff-impl-status/              # SKILL.md + proposal/status HTML templates + triggering reference
    ├── ff-api-contract-writer/      # SKILL.md + contract-format reference
    ├── ff-test-case-writer/         # SKILL.md (embedded interactive HTML template)
    ├── ff-service-test-runner/      # SKILL.md
    ├── ff-feature-brief/            # SKILL.md + one-pager HTML template
    └── ff-comprehension-coach/      # SKILL.md (interactive teach-until-understood)
```

## Design principles

- **No silent drift** — plans stick to the settled decision; code sticks to the plan; deviations are recorded with reasons or escalated upstream.
- **Reports are claims, not evidence** — the orchestrator re-reads diffs and re-runs verification commands before accepting any sub-agent's "done".
- **Parallel only when safe** — sub-agents run concurrently only with disjoint file ownership, no dependencies, and a contract-first Phase 0 locking shared seams.
- **Main stays lean** — dependency maps and conformance reports live in files, not chat memory; sub-agents return pointers, never code dumps.
- **Judgment is expensive, so it's rationed** — Fable is consulted only at commitment boundaries or after two failed attempts, and answers in verdicts, not surveys.

## License

MIT
