# feature-flow

Claude Code plugin bundling 3 skills that close the loop **proposal → implement → test plan → auto-test**:

| Skill | What it does | Invoke |
|-------|--------------|--------|
| **impl-status** | Maintains a live HTML implementation-status tracker (`docs/features/<feature>/implementation_status.html`) recording progress, decisions, compromises, bugs, files changed, rollout checklist. Optionally generates a design `proposal.html` first. Resumable across sessions. | `/feature-flow:impl-status` |
| **test-case-writer** | Generates a self-contained interactive HTML test plan (collapsible sections, pass/fail tracking, progress bar, localStorage persistence, Markdown export). Derives cases from an impl-status proposal when one exists; applies risk-based QA methodology (P0–P3, positive/negative/boundary). | `/feature-flow:test-case-writer` |
| **service-test-runner** | Translates the manual test plan into executable pytest tests, runs them against the implemented service, and reports PASS/FAIL/SKIP keyed back to each TC-id. | `/feature-flow:service-test-runner` |

All three skills are **explicit-invoke** for status tracking and test running — they do not auto-trigger on ordinary "implement X" requests. `test-case-writer` triggers on any request to write test cases / test plans.

## Install

```
/plugin marketplace add <owner>/feature-flow
/plugin install feature-flow@feature-flow
```

## Typical workflow

1. `/feature-flow:impl-status` → design `proposal.html`, then track implementation in `implementation_status.html`.
2. `/feature-flow:test-case-writer` → derive an interactive HTML test plan from the proposal.
3. `/feature-flow:service-test-runner` → turn the plan into pytest, run it, get per-TC results + ship/no-ship verdict.

Artifacts land under `docs/features/<feature>/` in your repo:

```
docs/features/<feature>/
├── proposal.html
├── implementation_status.html
├── test_plan_<slug>.html
├── tests/                       # generated pytest (run by explicit path, not collected by CI)
└── test_results_<slug>.md
```

## License

MIT
