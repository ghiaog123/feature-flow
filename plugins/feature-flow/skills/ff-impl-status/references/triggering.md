# Triggering phrases

The `ff-impl-status` skill is only active after the user explicitly opts in. Quick reference.

## Activate (skill on)

- `/ff-impl-status`
- `ff-impl-status`
- "turn on ff-impl-status"
- "turn on the status file"
- "maintain status file"
- "track implementation status"
- "use ff-impl-status skill"
- "enable status tracker"

## Deactivate / pause (keep the file, stop updating)

- "stop status file"
- "skip status"
- "no need for the status file"
- "stop updating the status"
- "pause ff-impl-status"
- "turn off the status tracker"

## Resume (after a pause within the same session, updating again)

- "resume ff-impl-status"
- "turn the status file back on"
- "go update the status"

## Reload from a previous chat (cross-session)

When the user opens a new chat and wants to continue a feature that already has files in `docs/features/`:

- "resume ff-impl-status <feature>"
- "load status <feature>"
- "resume feature X"
- "continue from the previous session"
- "re-read the status file for <feature>"
- "reload the proposal + status"
- "continue from <feature>"

Skill workflow on these phrases:
1. Locate `docs/features/<feature>/`. If the user didn't name it → list available folders for the user to choose.
2. Read `proposal.html` (if any) → `implementation_status.html` → `README.md`.
3. Summarize context in ≤10 lines for the user (progress, decisions, bugs, pending rollout).
4. Verify files/symbols mentioned in the status still exist (grep). Drift → record in the status.
5. Wait for the user's direction, don't auto-code.

## Do NOT trigger (common false positives)

These phrases must **not** trigger the skill — the user is only asking to implement, not to track:

- "implement feature X"
- "do this task for me"
- "add endpoint Y"
- "refactor module Z"
- "write the migration"
- "fix this bug"

The skill only creates files on explicit opt-in. "implement" / "start coding" / "go ahead" alone is NOT enough to trigger.
