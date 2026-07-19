# Assessment — template (Stage 1)

Problem diagnosis, NO solution talk yet. Write to `docs/features/<feature>/assessment.md`. This is the output when the request only asks for analysis/assessment, and the input for Stage 2 when a solution is requested.

```markdown
# Assessment — <one-sentence problem statement>

- **Date**: <YYYY-MM-DD>
- **Participants**: Claude Code ⇄ Codex
- **Debate status**: Consensus | Stalemate
- **Request intent**: Diagnosis only | Diagnosis + solution (→ Stage 2)

## Problem statement (sharpened)
<One sentence, clear on what needs understanding/deciding.>

## Context & constraints
<Success criteria, constraints (stack/perf/deadline/what may not change).>

## Diagnosis
<What the problem's nature is. Root cause if found — with file:line evidence.>

## Evidence
| Claim | Source (file:line / URL + date) | Fact or opinion |
|---|---|---|
| | | |

## Debate point classification
- **True consensus**: <...>
- **True disagreement**: <...>
- **Claude-only insight**: <...>
- **Codex-only insight**: <...>
- **Same direction, different depth**: <...>

## Blindspots / open unknowns
Unknowns surfaced by the Blindspot Pass (Step 2) — what Claude has NOT verified, assumptions being relied on — + their status after the debate.
| Unknown / unverified assumption | Confidence (High/Med/Low) | How to close (read file / ask whom / test what) | Status (closed / open) |
|---|---|---|---|
| | | | |

## Remaining disagreements (if stalemate)
| Point | Claude | Codex | Recommended lean |
|---|---|---|---|

## Risks / uncertain areas
- <risk> → verify via <method>

## Confidence
<High | Medium | Low> — <why>
```
