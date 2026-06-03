# feature-flow

Claude Code plugin bundling 5 skills that close the loop **proposal → implement → API docs → test plan → auto-test → stakeholder brief**:

| Skill | What it does | Invoke |
|-------|--------------|--------|
| **impl-status** | Maintains a live HTML implementation-status tracker (`docs/features/<feature>/implementation_status.html`) recording progress, decisions, compromises, bugs, files changed, rollout checklist. Optionally generates a design `proposal.html` first. Resumable across sessions. | `/feature-flow:impl-status` |
| **api-contract-writer** | Reads a service's source code and writes a concise caller-facing Markdown API contract — request/response shapes, status codes, error conditions only. Semi-automatic: lists discovered endpoints, waits for scope confirmation, then generates the contract. | `/feature-flow:api-contract-writer` |
| **test-case-writer** | Generates a self-contained interactive HTML test plan (collapsible sections, pass/fail tracking, progress bar, localStorage persistence, Markdown export). Derives cases from an impl-status proposal when one exists; applies risk-based QA methodology (P0–P3, positive/negative/boundary). | `/feature-flow:test-case-writer` |
| **service-test-runner** | Translates the manual test plan into executable pytest tests, runs them against the implemented service, and reports PASS/FAIL/SKIP keyed back to each TC-id. | `/feature-flow:service-test-runner` |
| **feature-brief** | Generates a non-technical one-page HTML "Feature Brief" for PO/QC — what changed, why, how it flows, acceptance criteria, test results, gaps. Synthesizes from `docs/features/<feature>/` artifacts (proposal, test plan, status) when present; translates technical concepts into everyday metaphors with concrete persona examples. | `/feature-flow:feature-brief` |

Status tracking and test running are **explicit-invoke** — they do not auto-trigger on ordinary "implement X" requests. `test-case-writer` triggers on any request to write test cases / test plans; `api-contract-writer` triggers on requests to document APIs / write API contracts.

## Install

```
/plugin marketplace add ghiaog123/feature-flow
/plugin install feature-flow@feature-flow
```

## Typical workflow

1. `/feature-flow:impl-status` → design `proposal.html`, then track implementation in `implementation_status.html`.
2. `/feature-flow:api-contract-writer` → document the implemented endpoints as a caller-facing Markdown contract.
3. `/feature-flow:test-case-writer` → derive an interactive HTML test plan from the proposal.
4. `/feature-flow:service-test-runner` → turn the plan into pytest, run it, get per-TC results + ship/no-ship verdict.
5. `/feature-flow:feature-brief` → one-page non-technical brief for PO/QC handoff, synthesized from the artifacts above.

Artifacts land under `docs/features/<feature>/` in your repo:

```
docs/features/<feature>/
├── proposal.html
├── implementation_status.html
├── test_plan_<slug>.html
├── tests/                       # generated pytest (run by explicit path, not collected by CI)
└── test_results_<slug>.md
```

## Repository layout

```
.claude-plugin/marketplace.json      # marketplace manifest
plugins/feature-flow/
├── .claude-plugin/plugin.json       # plugin manifest
└── skills/
    ├── impl-status/                 # SKILL.md + proposal/status HTML templates + triggering reference
    ├── api-contract-writer/         # SKILL.md + contract-format reference
    ├── test-case-writer/            # SKILL.md
    ├── service-test-runner/         # SKILL.md
    └── feature-brief/               # SKILL.md + one-pager HTML template
```

## License

MIT
