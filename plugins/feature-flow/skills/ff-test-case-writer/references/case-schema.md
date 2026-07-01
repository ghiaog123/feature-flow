# Test Case Schema — shared contract

**Single source of truth** for the `plan` array shared by two skills:

- `ff-test-case-writer` **produces** it (embeds it in the test plan HTML).
- `ff-service-test-runner` **consumes** it (parses the array verbatim to generate tests).

If you change a field name or its values, change it **here** and both skills follow. Do not redefine the schema inline in either `SKILL.md` — point to this file.

## Shape

```js
const plan = [
  {
    id: 'S1',
    title: 'Section Title',
    cases: [
      {
        // --- core fields (always present) ---
        id: 'TC-01',
        title: 'Short descriptive title',
        pri: 'P0',           // P0 | P1 | P2 | P3 — risk-based priority
        tag: 'pdf',          // optional UI tag: pdf | docx | xlsx | all | any custom string
        desc: 'Steps / context. What to do.',
        expect: 'Exact expected outcome.',

        // --- contract fields (optional; added by ff-test-case-writer's deep pass) ---
        depth: 'core',       // 'surface' | 'core' — does this case exercise core business logic?
        autoClass: 'agent',  // 'auto' | 'agent' | 'manual' — how it should be tested
        oracle: '...',       // where truth lives BEYOND the HTTP response (see below)
        agentHint: '...'     // guidance for the runner's agent-assisted tier (see below)
      }
    ]
  }
];
```

## Field semantics

### `depth` — `'surface' | 'core'`
- `surface` — CRUD shape, status code, field presence, basic validation. Truth is fully in the response.
- `core` — exercises the service's core business logic: state machine transitions, computed values/formulas, invariants (balances, stock never negative), idempotency, concurrency, multi-step flows. **These are the cases that surface-only plans miss.**

### `autoClass` — `'auto' | 'agent' | 'manual'`
The producer's recommendation for how the runner should test this case. **The discriminator is one question: *can a deterministic assertion capture the truth, given the right fixture?***

| value | meaning | runner behaviour |
|-------|---------|------------------|
| `auto` | A deterministic assertion captures it — whether the oracle is the **response** (status/body/error) OR a **side-channel** (DB row, emitted event, log, recomputed formula). Side-channel oracles just need a richer fixture (DB/broker/log access); they are still fully reproducible. | static pytest `assert` (response and/or `oracle` checks) |
| `agent` | **No deterministic oracle exists** — correctness needs judgement: free-text summary accuracy, "is this ranking reasonable," recommendation sensibility, fuzzy NL output. Nothing to `assert ==` against. | **agent-assisted tier**: agent calls the service, reasons about the output, emits `🤖 agent-judged` verdict **with cited evidence** |
| `manual` | visual / UX / exploratory / human judgement | `pytest.skip("manual: <reason>")`, listed in report |

A DB/event/log oracle does **not** make a case `agent` — it stays `auto` with a white-box fixture. Reserve `agent` for cases where no assertion, even with full DB access, can decide correctness. Most `core` cases are `auto` (often white-box `auto`); `agent` is the rare residue.

### `oracle` — string (the truth source beyond the HTTP response)
Where the truth lives, when it is **not** fully in the response body. Used by **`auto` white-box cases** (deterministic side-channel assertion) and as grounding for `agent` cases. Examples:
- `"query bookings.status == 'CANCELLED' and refund_log has exactly 1 row"` → `auto` white-box
- `"recompute expected fee = price * 0.9 for cancel <24h; compare to response.refund_amount"` → `auto`
- `"consume Kafka topic listing.created; assert one event with listing_id"` → `auto` white-box
- `"summary should faithfully reflect the 3 input bullet points (no hallucinated facts)"` → `agent` (no deterministic oracle)

A `core` case whose truth is purely in the response (a computed field returned directly) is `auto` with the formula in `expect`/`oracle` — no fixture needed.

### `agentHint` — string (optional)
Free-text guidance for the runner's agent-assisted tier: preconditions/state to set up, the business rule under test, edge reasoning. Example:
> "Setup: create booking + pay first. Rule: cancel <24h ⇒ 10% penalty. Verify refund == price*0.9 and booking moves PAID→CANCELLED."

**`agentHint` is GUIDANCE, not a spec.** It does not carry exact endpoint paths or request payloads. The runner must still resolve those from `proposal.html` §3 API surface + the implemented handler code (its Step 4). A vague hint is never grounds to ship a test with a guessed path — if path/payload can't be resolved, the case is `SKIP (blocked)`, not a fabricated test.

## Backward compatibility

The four contract fields are **optional**. Plans written before this schema (or by a shallow pass) have only the core fields. Both skills must degrade gracefully:
- `ff-test-case-writer` HTML rendering: a missing field renders nothing (follow the existing `if (!tag) return ''` pattern).
- `ff-service-test-runner`: if `autoClass` is absent, fall back to self-classifying the case (its Step 3 heuristics).

## Ownership (who writes / reads what)

- **ff-test-case-writer owns semantic derivation** → reads proposal + implemented code, sets `depth`, `autoClass`, `oracle`, `agentHint`.
- **ff-service-test-runner owns execution mechanics** → resolves exact path/payload from the contract + handler code, runs tests, emits the `auto` PASS/FAIL and `🤖 agent-judged` verdicts. It *consumes* `autoClass` rather than re-deriving it (self-classify only as a fallback for plans that lack it).
