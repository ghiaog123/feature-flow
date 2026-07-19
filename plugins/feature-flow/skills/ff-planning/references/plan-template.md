# Implementation Plan — template

Write to `docs/features/<feature>/implementation_plan.md`. Keep `#`/`##` headings so `/codex-plan-review` can anchor on the structure.

**Dependency contract** = the `Owns files` + `Depends on` + `Parallel?` columns in the "Implementation steps" table. This is the contract that lets `ff-implement` parallelize safely. Declare **honestly**: only mark parallel when files are disjoint AND there are no dependencies; in doubt, keep sequential.

```markdown
# Implementation Plan — <feature/problem name>

## Goals / Outcomes
<What counts as done. This is the source to derive acceptance criteria — write it measurable.>
- [ ] <outcome 1>
- [ ] <outcome 2>

## Context & decision
<Summary of the settled decision + rationale. Link brief/decision record if any:
docs/features/<feature>/analysis_brief.md · docs/decisions/...>

## Impact area
| File | Role in this change |
|---|---|
| `path/to/file.py:120` | <what changes> |
| `path/to/other.py` | <what's added> |

## References & gotchas
<Only when the implementation ports/adapts from a reference (another language/library/repo/prior art). Drop this section if there is no reference.>
| Aspect | Reference | Target stack | Gotcha / difference |
|---|---|---|---|
| <behavior/API> | <matching fragment / how the reference does it> | <how this stack does it> | <what does NOT transfer 1:1: concurrency, memory, timezone/locale, implicit errors, init order...> |

## Implementation steps
> `Owns files` = files this step owns (gets to modify). `Depends on` = steps that must finish first.
> Safe to parallelize ⟺ Owns files are disjoint AND no mutual dependencies.
> `Uncertainty` = high/medium/low — steps most likely to need rework (architecture, schema, unverified assumptions) get **high**. This flag only sets **attention/review priority** (tweakable-first), it does NOT change execution order — execution order still follows `Depends on`. Put high-uncertainty steps first / make them prominent.

| # | Step | Owns files | Depends on | Parallel? | Uncertainty |
|---|---|---|---|---|---|
| 0 | <contract-first: lock shared types/signatures/stubs if there is a seam> | `path/shared.py` (types, stubs) | — | — | high |
| 1 | <what to do, concretely> | `path/a.py` | 0 | with 2 | medium |
| 2 | <...> | `path/b.py` | 0 | with 1 | low |
| 3 | <integration/assembly step> | `path/wire.py` | 1, 2 | — | low |

## Test & verify
| Part | How to verify | Pass criteria |
|---|---|---|
| <step/feature> | <unit/integration/manual> | <condition> |

## Risks & rollback
- **Risk**: <what can break> → **Mitigation**: <how>
- **Rollback**: <how to back out if it breaks>

## Out of scope
- <deliberately not done this time>
```

## Note on Phase 0 contract-first (for large plans)

If multiple steps share a seam (data types, function signatures, interfaces, stubs), add a **Step 0** that locks that seam first. Later steps then fill in a fixed contract → `ff-implement` can parallelize more safely (each step owns its own files, nobody steps on the shared seam).
