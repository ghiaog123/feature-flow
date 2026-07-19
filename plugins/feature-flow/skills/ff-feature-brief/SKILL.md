---
name: ff-feature-brief
description: Create a "Feature Brief" — a 1-page non-technical HTML doc that helps PO/QC understand a new feature (what changed, why, how it flows, acceptance criteria, test results, gaps). Synthesizes from docs/features/<feature>/ (proposal.html, test_plan_*.html, implementation_status.html) when available, reads code or asks the user when sources are missing. Translates technical concepts into everyday metaphors (JWT → "a signed access pass") with concrete character-based examples. Use this skill when the user says "feature brief", "one-pager", "a doc for PO/QC to read", "summarize the feature for the PO", "QC handover doc", "explain the feature to non-technical people", "present the feature in plain language", "feature demo doc for stakeholders" — even without explicitly saying the word "brief". Do NOT use for: detailed test plans (ff-test-case-writer), code progress tracking (ff-impl-status), API docs (ff-api-contract-writer).
---

# Feature Brief

Generate **1 HTML page** summarizing a feature for the **PO** (understand the business, decide) and **QC** (know what to test). Core principle: the reader is **not a dev** — every technical concept must be translated into everyday language, and every abstract rule must come with a concrete simulated example.

```
docs/features/<feature>/          (read whichever exist)
├── proposal.html             → What changed, Why, Flow, AC
├── test_plan_*.html          → Test status, SKIP reasons
├── implementation_status.html → code status, remaining work
└── feature_brief.html        ← OUTPUT of this skill
```

## Workflow

### Step 1 — Identify the feature + gather sources

1. User names the feature → look for `docs/features/<feature>/`. Multiple candidate folders → ask.
2. Glob the sources: `proposal.html`, `test_plan_*.html`, `implementation_status.html`, `implementation_plan.md`.
3. **Flexible, non-blocking:** for any missing source, compensate by reading the relevant code or asking the user 2-3 questions. Missing test plan → the Test status section explicitly says "not tested yet" (don't fabricate numbers).

Extract text from the source HTML (files are usually very long, don't Read the whole file):

```bash
python3 -c "
import re, html
src = open('<file>').read()
text = re.sub(r'<style.*?</style>|<script.*?</script>', '', src, flags=re.S)
text = re.sub(r'<[^>]+>', '\n', text)
print(chr(10).join(l.strip() for l in html.unescape(text).split('\n') if l.strip()))
"
```

Test plans from ff-test-case-writer/ff-service-test-runner have a built-in summary box (PASS/SKIP/FAIL headline + SKIP reasons) — prefer pulling from that.

### Step 2 — Distill the content

From the sources, extract (these are **real facts**, not inferences):

| Needed | Source |
|---|---|
| Before/after (3-5 biggest differences) | proposal architecture/goals section |
| Business rationale (2-3 bullets) | proposal goals + ask the user if unclear |
| Main flow + rejection/error branches | proposal detailed flows / sequence diagrams |
| Acceptable behaviors (→ AC) | proposal expects + test plan P0 cases |
| Test numbers + reasons not yet tested | test plan summary box |
| Remaining work before release | implementation_status / test plan |

### Step 3 — Translate into non-technical language

**Output in English by default; follow the user's language if they ask.** Tone: like explaining to a colleague from the sales department.

Translation rules — this is what determines the brief's quality:

1. **Each technical concept → 1 everyday metaphor, used consistently across the whole page.** Pick 1 framing metaphor (building/access card, post office, supermarket…) and map everything onto it. Examples that have worked well:
   - JWT/token → "a signed access pass"
   - internal service → "the tool warehouse", "the checkpoint"
   - fail-closed → "a fake card opens no floor at all"
   - permission intersection → "both in the assigned set AND hired by the app"
   - cache invalidation → "takes effect immediately, no 5-minute wait"
2. **Cast a concrete character** (e.g. "An uses the Meeting Note app") and tell every example through that character — readers remember stories, not terminology.
3. **Each AC comes with 1 `💡 Example:` line** simulating a real situation with concrete times/numbers ("10:00 admin revokes the permission → 10:00 An asks again → the AI refuses immediately").
4. **Abstract rules → arithmetic-style diagram.** Multi-clause permission/condition formulas: draw as a row of chips (① this ∩ ② that − ③ minus this = ✅ result) instead of writing the formula.
5. Real technical names (endpoints, DB tables, class names) only appear in the QC-facing parts (sequence diagram, code refs) — never in the PO parts.

### Step 4 — Generate the HTML

Copy `assets/template.html` (in this skill's folder) → `docs/features/<feature>/feature_brief.html`, and fill it in. The template ships with full CSS + Mermaid CDN; see the `<!-- FILL: ... -->` comments in the template for each area.

**Standard 7-block structure — flexibly drop blocks that don't apply** (feature not yet tested → drop block 5 or write "not tested yet"; no gaps → drop block 6):

| # | Block | Audience | Content |
|---|---|---|---|
| 0 | 📌 30-second summary | Both | 2-3 sentences + a "quick mental picture" box using the framing metaphor |
| 1 | What changed | Both | 3-5 row before/after table + arithmetic diagram if there's a formula |
| 2 | Why it's needed | PO | 2-3 business bullets |
| 3 | How it flows | Both | 2 Mermaid diagrams: **3a character-story swimlane** (for PO, no jargon) + **3b sequence diagram** (for QC, readable labels but keeping technical detail: token, fail-closed, log) |
| 4 | Acceptance criteria | QC | 5-8 If/When/Then ACs, each with a 1-line 💡 example. Cover: happy path, rejection on invalid input, permission boundaries, revocation, mid-flow failures |
| 5 | Test results | Both | 3 number cards (PASS/NOT TESTED/FAIL) + verdict + bugs found through testing (that's the value!) + link to the test plan |
| 6 | Out of scope & remaining work | Both | Deliberately deferred items / reasons cases aren't tested yet (1 line/group) / pre-launch checklist |

Each block carries an audience badge (`PO` / `QC` / `PO + QC`) — readers skim straight to their part.

**Diagrams:** Mermaid via CDN (`mermaid@10` esm) — needs internet when the file is opened; acceptable on a corporate network. Node labels in the user's language; the PO swimlane tells exactly 1 story of the character (real question input → the checkpoints → ✅/⛔).

### Step 5 — Verify + hand over

1. Every number (PASS/FAIL, case counts) must be traceable to a source — **no fabrication, no pretty rounding**. Source missing → write "no data yet".
2. Relative links to the source docs in the Test status block (`test_plan_*.html`, `proposal.html`, `implementation_status.html`).
3. `open <file>` for the user to review, ask what needs adjusting.
4. Remind the user if `docs/` is gitignored: the file is local-only — send it directly or put it on Confluence/wiki.

## Anti-patterns

- ❌ Pasting a raw technical proposal paragraph into the PO section ("backend signs an RS256 JWT with aud=mcp-server…")
- ❌ ACs with no example, or generic examples ("user does something wrong → system rejects")
- ❌ Fabricating test numbers / a verdict when there's no test plan
- ❌ Mixing multiple metaphors ("access card" plus "safe key" plus "movie ticket") — pick 1 frame and stick with it
- ❌ Brief longer than ~2 scroll-screens/block — this is a one-pager; details link out to source docs
