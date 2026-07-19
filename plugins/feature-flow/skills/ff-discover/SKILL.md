---
name: ff-discover
description: "Front door of the feature-flow bundle — use WHEN the user cannot yet articulate the problem and only has a vague feeling. Turns an unformed intent (\"something feels off here\", \"I want to improve X but I'm not sure what exactly\", \"we should add Y but I don't know exactly what\", \"I don't know where to start\", \"help me find out what the real problem is\", \"explore this area before we decide\") into a SHARP PROBLEM STATEMENT + a map of known-unknowns + intervention directions to lock scope. Combines structured interviewing, a vocabulary ladder, a blindspot pass (auditing the code area to surface unknown-unknowns), and timescale-based direction brainstorming. Runs Claude-solo + user interview (does NOT call Codex — there is nothing to debate yet). Outputs a vivid `discovery_brief.html` (plain-language TL;DR, Mermaid area + coupling map, unknown cards colored by confidence, timescale option-space) that feeds straight into ff-problem-solver. Use this skill whenever the request is still too vague to form a clear problem — even when the user does not say the word \"discover\". Do NOT use for: a clearly stated problem (ff-problem-solver), planning (ff-planning), code review (codex-review), progress tracking (ff-impl-status)."
---

# ff-discover — Front door: surface the unknowns before framing the problem

The rest of the bundle (`ff-problem-solver` → `ff-planning` → `ff-implement` → …) all **assume a clearly stated problem already exists**. But reality usually starts earlier than that: the user only has a *feeling* — "something feels off here", "I want it to be better", "seems like something should change" — not sharp enough for two experts to debate or to build a plan on. Debating/planning on a vague foundation = building a house on sand.

`ff-discover` fills exactly that gap: **turn unformed intent → sharp problem statement + unknowns map + option space**, so downstream has solid ground. The method's tagline: *"The map is not the territory — the gap between them is your unknowns."* Surfacing unknowns **while they're still cheap** (before code) beats discovering them after building the wrong thing.

```
Vague intent (user: "something feels off / want it better / don't know where to start")
   │
   ▼
[1] Structured interview (ordered by architectural impact)   ← surface latent intent
   │
   ▼
[2] Vocabulary ladder — fuzzy words → measurable technical meaning
   │
   ▼
[3] Blindspot Pass — audit the code area, surface unknown-unknowns (adaptive: inline / fan out sub-agents)
   │
   ▼
[4] Brainstorm intervention directions by timescale — TO LOCK SCOPE, not to decide
   │
   ▼
[5] Assemble discovery_brief.html — vivid (sharp statement + area diagram + unknown cards + directions + next skill)
   │
   ▼
[Controlled compact] → chain to ff-problem-solver (or straight to ff-planning if the direction is already clear)
```

## Skill boundary (read before doing anything)

**Discover, do NOT decide, do NOT solve.** This skill's output is a problem *made sharp* + an option space — **not** a diagnosis and **not** a chosen solution:
- Diagnosing the nature/cause → `ff-problem-solver`'s job.
- Choosing a solution / implementation approach → `ff-problem-solver` (Stage 2) and `ff-planning`'s job.

If during discovery you catch yourself *settling on* a cause or *choosing* a solution → stop; that's a boundary violation. Your job is to make the problem **askable correctly**, not to answer it.

## Core principles

- **Claude-solo + user interview.** At this stage there is no hypothesis/decision for two AIs to debate — the value lies in **extracting intent from the user** and **auditing real code**, not debating. So do NOT call Codex here; debate starts downstream in `ff-problem-solver`.
- **Unknowns are the main product, not a byproduct.** The goal isn't a "quick answer" but *exposing what the user (and you) don't know you don't know*. A good discovery brief usually leaves more named unknowns than answers.
- **Interactive, but resilient without a human.** This skill lives on conversation with the user (AskUserQuestion). But if it runs in a context **with no one to answer** (batch/eval/CI), don't hang: **state the assumptions you're using, flag them as unknowns, and continue** — the brief still ships, just with a thicker "unconfirmed" section.
- **Grounded in real code, not theory.** The blindspot pass must point to real `file:line`. Unknowns inferred from reading code > imagined unknowns.
- **No endless dragging.** Discover until *sharp enough to move on*, not to understand everything. When the problem statement is measurable and the big unknowns are named → close the brief, hand off. Don't turn discovery into an investigation without end.
- **Vivid, easy-to-grasp presentation.** The brief is for the user to grasp at a glance, not a wall of text: plain-language TL;DR, diagrams that make hidden coupling visible, unknown cards colored by confidence. Use the bundle's house style (`assets/discovery_brief_template.html`) — don't invent your own HTML.
- **Artifacts to disk, main stays lean.** Sub-agent audit reports are intermediates; the result lives in `discovery_brief.html`.

## Workflow

### Step 0 — Take the intent + identify the area + output

1. **Capture the raw intent** exactly as the user said it, without interpreting yet.
2. **Identify the area**: the subsystem / module / feature / flow the feeling points at. If the user doesn't specify → ask 1 question to narrow it (don't guess the area and audit the wrong place).
3. **Derive the `feature`/slug** (snake_case/kebab-case) from the intent or git branch → choose the output directory `docs/features/<feature>/discovery_brief.html`. Name unclear → ask, or use a temporary slug and confirm later.

### Step 1 — Structured interview (ordered by architectural impact)

Bring latent intent into the light. **Ask in impact order — heavy & hard-to-reverse questions FIRST**, because a wrong answer to a high-impact question poisons the whole brief:

1. **High impact** — Where does it hurt, specifically (observable symptoms, not inference)? What *triggered* this feeling now (a new bug, a complaint, a deadline, an upstream change)? Who/what is affected?
2. **Medium impact** — What does "done / better" mean to the user? Hard constraints (stack, nothing may change, deadline)? Anything already tried?
3. **Low impact** — Cosmetic preferences, last.

Batch at most 3–5 questions per round; don't interrogate. Use `AskUserQuestion` when options are discrete; open-ended when a description is needed. **Do not offer solution opinions** at this step — stay neutral; you're extracting the problem, not answering it.

If **no one is available to answer** (see the "resilient without a human" principle): derive plausible answers from the prompt + the code itself, write them down as **explicit assumptions**, and mark them as unknowns for the user to confirm.

### Step 2 — Vocabulary ladder (fuzzy → measurable meaning)

Vague intent is always full of unmeasured words: "fast", "clean", "better", "tidy", "stable". Anchor each word to a concrete technical meaning grounded in the real code/domain:

- **List the fuzzy words** — exactly the unmeasured words in the intent.
- **Anchor each word** to a measurable statement: "fast" → *p95 latency of endpoint X < 200ms*; "clean" → *remove the A↔B import cycle*; "breaks whenever we touch it" → *editing module M tends to break N because …*.
- **Confirm with the user** (if present) or record as assumptions (if absent).

This step turns "I want it better" into something `ff-problem-solver` can diagnose. Can't anchor a word for lack of information → that itself is an unknown; record it in the unknowns map (Step 3).

### Step 3 — Blindspot Pass (audit the area, surface unknown-unknowns)

This is the heart of the skill: Claude reads the code area the user points at and **hunts for what neither side knows they don't know** — not just confirming what's already assumed. **Pick the approach by size (adaptive-by-size — the anti-over-engineering gate):**

- **Small area / location known** → audit **inline** with Glob/Grep/Read right on main.
- **Large area / touchpoints unknown** → **fan out in parallel, keep main lean**:
  1. **Cheap indexing first**: if an index tool exists (e.g. `graphify`) → run it for a quick map; otherwise one Glob/Grep pass to locate.
  2. **Fan-out in 1 message**: multiple `subagent_type: 'ff-sonnet-explorer'` sub-agents in parallel, each on one angle (entry points, data model, hidden dependencies, coupling, architectural assumptions, existing tests, prior art/docs) — Sonnet handles search/read/synthesize work at near-parity for a fraction of the cost. The agent's own definition enforces the tight `EXPLORE REPORT` format (`file:line` findings + `SURPRISES` for the not-as-expected, no code dumps, read-only).
  3. **Synthesis on main** — main doesn't swallow whole files.

Produce a **structured list of unknowns** (cards), each with:
- Description of the unknown / unverified assumption.
- **Confidence** (High / Medium / Low).
- **Impact if wrong** (why it matters).
- **How to close it** (which file to read / whom to ask / what to try running).

Types of unknowns to actively hunt: **hidden dependencies** (places that seem unrelated but are), implicit **architectural assumptions**, **edge cases** no one has mentioned, **domain knowledge gaps**, **the gap between what the user believes and what the code actually does**.

**Point at a Reference (if the user cites prior art)** — if the intent is "do it like X / repo Y / library Z does" → read the actual reference, record **what maps 1:1**, **semantic differences**, and **platform-/stack-specific gotchas** that don't carry over intact. Don't let the brief assume the reference's behavior transfers to the target stack as-is.

### Step 4 — Brainstorm intervention directions (TO LOCK SCOPE, not to decide)

With the pain clarified, sketch the **option space** of intervention directions by **timescale** — so the user sees the range of options and **locks scope/ambition**, NOT to pick a direction here (picking is `ff-problem-solver`/`ff-planning`'s job):

| Timescale | Intervention type | Examples |
|---|---|---|
| Now / same day | Patch the symptom | workaround, guard, extra logging |
| Short (days–weeks) | Fix the local cause | logic fix, add tests, small refactor |
| Medium (weeks–months) | Change local structure | split a module, change the data model |
| Long (quarters) | Change the architecture | swap approach/technology |

Mark which directions touch the **symptom** vs the **root**. Then ask the user (or record an assumption): *which layer do you want to solve at this time?* The answer locks the **scope** of the problem statement — not the solution.

If the intent is really a new feature (not "fix a pain"), replace the timescale table with scope questions: *what's the minimum version, what's deferred, what's definitively out.*

### Step 5 — Assemble discovery_brief.html (vivid)

Synthesize into a brief that is **easy to grasp at a glance**, written to `docs/features/<feature>/discovery_brief.html` from the template `assets/discovery_brief_template.html`. **Never write new HTML from scratch — always base it on the template.** Language follows the user: VI prompt → VI HTML, EN prompt → EN HTML; **unclear → default to Vietnamese**. **Write REAL UTF-8 characters** (real — dashes, real " quotes, real newlines) — do NOT stuff escape sequences (`—`, `\"`, literal `\n`) into the HTML.

Layout has **3 reading tiers** (matching the template) — grouped so the eye knows where to look first, not 8 equally-weighted blocks:

**Header** — `<h1>` is the feature name only (drop jargon); below it the `.sub` line (Discovery Brief · area · date). Fill the `.glance` **quick counts** to match the actual content: number of unknowns + number of high-priority ones (add class `.hot` if any), number of hidden coupling points, estimated reading time. Fix the `<a href>`s in `.toc` to match the section ids that actually exist.

**Tier 1 — Quick understanding** (grasped in a 30-second read):
1. **TL;DR — the REAL problem** (box `.tldr`, `#tldr`): 1–2 sentences in **plain language, NO jargon** — what the user's vague feeling actually is. The most important part for skimmers; be concrete (e.g. "auth is fragile" → "refresh shares in-memory state with auth; a restart wipes everything").
2. **Problem & scope** (`#scope`) — one sharp sentence of "what needs understanding/deciding" + pain/trigger/who's affected, then measurable success signals, then ✅ In scope / 🚫 Not this time cards. Include a `.lane-note`: not yet diagnosed / no solution chosen.

**Tier 2 — The picture** (read if you want to dig deeper):
3. **Area map & hidden coupling** (`#map`) — a **Mermaid diagram** (see "Drawing the diagram" below): draw the code area + dependencies, **hidden/risky coupling = red dashed arrows** (`linkStyle ... stroke:#DC2626`). Add 1–2 sentences of "how to read this map".
4. **Anchored vocabulary** — inside a `<details class="fold">` (collapsed by default): table of fuzzy word → measurable meaning → code source. It's reference material, not read-through content → fold it to keep the brief tight.

**Tier 3 — Unknowns & next steps** (the most important part):
5. **Unknowns (known-unknowns)** — `<section class="hero">` `#unknowns`, the **MAIN PRODUCT**; this block gets an accent border that clearly stands out. Card colors = **REVIEW PRIORITY** (matching the red=urgent intuition), NOT "confidence": `.card.pri-hi` 🔴 look at first · `.card.pri-mid` 🟡 next · `.card.pri-lo` 🟢 later/speculative. Keep the `.legend` block at the top so no one has to guess the colors. Each card: unknown name + **what was observed** + **impact if ignored** + **how to close**. Include `t-del`/`t-warn`/`t-add` tags.
6. **Intervention directions (option space)** (`#directions`) — 4 timescale columns (`.grid.cols-4`) + `ts-symptom`/`ts-root` markers + the scope tier the user locked. `.lane-note`: *no direction decided yet*. If it's a new feature → replace with In scope / Later / Not doing.
7. **Next step — next skill** (`#next`) — `ff-problem-solver` (diagnosis/decision still needed) or `ff-planning` (direction already clear). State the brief's path for chaining.

**Appendix (optional):** References & gotchas — `<details class="fold">` at the end of the page; **DELETE the whole `<details>` if nothing was ported from prior art** (don't leave an empty table).

**Drawing the area map diagram (`#map`) — 3 layers of visual signal, understandable at a glance:**
1. **Color by role** (`classDef`, `stroke-width:2px`): 🟣 purple = module/processing · ⚪ gray = data store · 🔴 red = risk point/unknown. Hidden/risky coupling = **red dashed edge** (`-.->` + `linkStyle`).
2. **Shape by node type**: `[/.../]` = module · `[(...)]` = store · `{{...}}` = warning/risk.
3. **Emoji icon at the start of the label** hints meaning: 🔑 auth · ⚙️ processing · 🗄️ DB/store · ♻️ refresh/cycle · ⚠️ risk.

Rules to keep the diagram readable: **≤7 nodes** (more → split into 2 diagrams); short labels (≤4–5 words); pick the simplest type that expresses it (default `flowchart`). The diagram's purpose = making hidden coupling **visible**, not decoration.

### Final step — Handoff + chain

- Present a summary ≤10 lines: the sharp problem statement, the 2–3 biggest open unknowns, the scope tier the user locked, the next skill.
- **Chain**: `discovery_brief.html` is the natural input for `ff-problem-solver` (use it as the "problem statement" + carry the unknowns map for the debate to probe; downstream can read HTML, as the bundle already does with `proposal.html`). If the direction is clear and it's only "how" → can go straight to `ff-planning`.

## Context hygiene (controlled compact)

Don't run a bare `/compact`. After assembling the brief to disk, propose a compact block with real paths filled in, explicit KEEP/DROP:

```
/compact KEEP: contents of docs/features/<feature>/discovery_brief.html (sharp statement, unknowns map,
directions + locked scope tier, next skill). DROP: sub-agent audit reports, per-round interview transcripts
(already synthesized into the brief).
```

Unknowns/scope decisions not yet in the brief → flag as must-keep.

## Anti-patterns

- ❌ Boundary violation: settling on a cause or choosing a solution during discovery (that's `ff-problem-solver`/`ff-planning`'s job).
- ❌ Skipping the Blindspot Pass, just interviewing the user then writing the brief → loses exactly the "unknown-unknowns" part the skill exists to surface.
- ❌ Auditing only to confirm what's already assumed instead of hunting surprises → the blindspot pass becomes an echo chamber.
- ❌ Debating/calling Codex at this stage when there's no hypothesis to challenge (over-engineering; save it for downstream).
- ❌ Hanging because the user doesn't answer, instead of stating explicit assumptions + flagging unknowns and continuing.
- ❌ Sloppy vocabulary anchoring ("fast = faster") — it must yield a measurable statement or be admitted as an unknown.
- ❌ Endless investigation: trying to understand everything instead of "sharp enough to move on".
- ❌ Guessing the code area and auditing the wrong place instead of asking 1 narrowing question.
- ❌ Swallowing whole files into main when the area is large (should fan out sub-agents returning tight reports); or fanning out sub-agents for a small, known area.
- ❌ A jargon-filled TL;DR — kills the "understand in 10 seconds" purpose. Write plain language.
- ❌ Diagrams with >7 nodes / drawn for the sake of it — they confuse instead of making hidden coupling visible. Split them, or drop them if they add no insight.
- ❌ Writing new HTML from scratch instead of basing on `assets/discovery_brief_template.html` (breaks the bundle's house style).

## Further reading

- `assets/discovery_brief_template.html` — HTML template for `discovery_brief.html` (TL;DR box, color-coded unknown cards, Mermaid, timescale columns).
