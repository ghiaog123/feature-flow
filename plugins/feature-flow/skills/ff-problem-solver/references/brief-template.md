# Analysis Brief — template (Stage 2)

From the diagnosis (`assessment.md`) → an actionable solution. Write to `docs/features/<feature>/analysis_brief.md`. This is the input for `ff-planning`.

```markdown
# Analysis Brief — <one-sentence problem statement>

- **Date**: <YYYY-MM-DD>
- **Participants**: Claude Code ⇄ Codex
- **Debate status**: Consensus | Stalemate (user must decide)
- **Source assessment**: docs/features/<feature>/assessment.md

## Decision
<Which option (or combination) is chosen. Decisive, one paragraph.>

## Rationale
- <reason 1 — grounded in consensus / the strongest evidence from the debate>
- <reason 2>

## Options considered
| Option | Pros | Cons | Why chosen/rejected |
|---|---|---|---|
| A (chosen) | | | |
| B | | | |

## Accepted trade-offs
- <what is sacrificed, in exchange for what>

## Next steps (for ff-planning to follow)
- [ ] <concrete action>
- [ ] <...>

## Remaining disagreements (if stalemate)
| Point | Claude | Codex | Recommendation |
|---|---|---|---|

## Risks / needs further verification
- <risk> → verify via <method>

## Consolidated sources
- <link/file:line + access date>

## Confidence
<High | Medium | Low> — <why>
```
