---
name: full-project-thermo-review
description: Run a thermo-nuclear-grade code quality audit across an entire codebase (not just a diff) by sharding it into domain-scoped chunks and fanning out one parallel sub-agent per shard, each applying the thermo-nuclear-code-quality-review standard. Output is one ticket-ready file per finding (not one file per shard), with real functional bugs split into their own `bugs/` folder, plus an overview that ends by offering to file the findings as tickets in Linear/Jira/GitHub Issues/etc. Use when the user asks for a "full project thermo review", "thermonuclear review of the whole codebase/repo", "audit the whole codebase for code quality", or wants a repo-wide maintainability audit broken into parallel chunks rather than a single-pass review of one diff. Portable — copy this skill folder into any repo; it adapts to whatever codebase it's run in.
---

# Full-Project Thermo Review

This is an **orchestration wrapper**, not a review standard of its own. It exists because the actual review standard — [`thermo-nuclear-code-quality-review`](https://github.com/cursor/plugins/blob/main/thermos/skills/thermo-nuclear-code-quality-review/SKILL.md) — is written to review one PR/branch diff at a time, and reading an entire codebase in a single pass either blows the context budget or forces skimming, which defeats the point of a strict maintainability audit. This skill's job is purely the **sharding + fan-out + synthesis** process: split the codebase into pieces small enough to read in full, review each piece in parallel with the real standard, then roll the results up into one coherent report.

Do not use this for reviewing a single PR or branch diff — hand that straight to the underlying `thermo-nuclear-code-quality-review` skill (or the equivalent `/code-review` flow) instead. Use this skill specifically when the ask is "the whole codebase" or "the whole repo" or a large subtree of it.

## Step 1 — Locate the actual review standard

The rules you apply in every shard come from `thermo-nuclear-code-quality-review`, not from this file. Find it in this order:

1. **Local first.** Check this repo's skill locations for an already-installed copy (naming and directory may vary by tool: `.agents/skills/thermo-nuclear-code-quality-review/SKILL.md`, `.claude/skills/...`, `.cursor/skills/...`, `.codex/skills/...`). If present, read it directly.
2. **Remote fallback.** If no local copy exists, fetch the canonical source: `https://github.com/cursor/plugins/blob/main/thermos/skills/thermo-nuclear-code-quality-review/SKILL.md` (raw content — via `gh api repos/cursor/plugins/contents/thermos/skills/thermo-nuclear-code-quality-review/SKILL.md --jq '.content' | base64 -d`, or a web fetch of the raw file).
3. **Cache it once.** Save the resolved standard to a single scratch file for this run. Every sub-agent should `Read` that same file rather than each re-fetching it — cheaper, and guarantees every shard is judged against identical rules.

**The adaptation every sub-agent needs to know:** the standard's baseline prompt talks about reviewing "the current branch's changes" — a diff. In full-project mode there usually is no diff. Tell every sub-agent explicitly: apply the same rules and the same severity bar to the code **as it currently stands**, as a standalone audit, not a diff review. Everything else in the standard (the 1000-line file-size rule, the anti-spaghetti rules, boundary/type cleanliness, canonical-layer discipline, the priority ordering, the output format) applies unmodified.

## Step 2 — Shard the codebase

Goal: every shard is small enough that a sub-agent can **read every file in it**, not sample it, while the union of all shards covers the whole target with no gaps and minimal overlap.

1. Enumerate the natural top-level units: workspaces/apps/packages in a monorepo, or top-level `src` subdirectories in a single app.
2. Get a quick size read per unit — file count and line count (a one-line `find | wc -l` loop is enough) — to see where the mass actually is before guessing at boundaries.
3. Group along **domain** boundaries, not arbitrary line-count buckets: a backend feature/domain folder, a frontend route or feature folder, a shared/canonical package, an infra or cross-cutting layer. Don't split one cohesive feature across two shards; don't lump two unrelated large domains into one shard just to hit a target size.
4. If a single directory alone exceeds the target shard size, split it further along its own subdirectories (feature areas, route segments, sub-domains) instead of reviewing it as one oversized shard.
5. Target roughly **3,000–8,000 lines / well under ~150 files** per shard as a starting point — small enough to read in full, large enough that you're not paying fan-out overhead for trivial pieces. Scale the number of shards to the codebase: a small repo might need only 1–3 shards, a large monorepo can reasonably need 15–20. Keep shards roughly balanced so no single sub-agent becomes the long pole the others wait on.
6. Give every shard a short slug and an explicit, unambiguous path list. No shard's scope should overlap another's.

## Step 3 — Brief each sub-agent

**One file per actionable item, not one file per shard.** A single combined report per shard is hard to turn into real work — findings get lost inside a wall of markdown, nobody can hand one item to someone else, and a genuine bug ends up filed next to a cosmetic nit as if they were the same kind of task. Every sub-agent writes its findings as individual, ticket-ready files from the start, laid out like this:

```
docs/thermo-review/
  README.md                      — overview, written by the orchestrator in Step 5
  bugs/                          — real functional bugs, flat across all shards
    <shard-slug>-<bug-slug>.md
  <shard-slug>/
    00-summary.md                — scope, verdict, size watch list, "explicitly fine" notes,
                                    index of every finding this shard produced
    01-<finding-slug>.md         — one file per maintainability/quality finding
    02-<finding-slug>.md
    ...
```

Every sub-agent needs the same shape of prompt, scoped to its shard:

- Where to read the cached standard from Step 1, with an instruction to read it in full and apply all of it.
- The full-project adaptation note (auditing current state, not a diff) from Step 1.
- Its exact scope: explicit paths, "read every file in scope, do not skim or sample."
- Any architecture/convention context you already know about that shard (layering rules, styling conventions, established patterns for this codebase) — this lets the reviewer tell a genuine smell apart from an intentional, established pattern, and lets it catch convention violations as first-class canonical-layer findings rather than missing them.
- Its shard slug and the exact output layout below, including the **bug-vs-finding rule**.
- An explicit "identification only — do not modify any code" instruction.
- A request for a short final text summary (finding counts, bug count, 2–3 headline issues) — the orchestrator uses these to track progress and build the overview without re-reading every file in full.

### Bug-vs-finding rule (each sub-agent applies this itself)

The standard is a maintainability lens, not a correctness lens, but a strict-enough audit will sometimes surface a genuine functional bug alongside the structural findings. Route each finding to the right place:

- **Quality/maintainability finding** — code that behaves correctly today but is poorly structured, duplicated, oversized, or hard to reason about. Goes in the shard's own folder.
- **Bug** — code that is currently producing wrong behavior, wrong output, or silently corrupting/dropping data for real users, discovered incidentally while auditing structure. Goes in the shared top-level `bugs/` folder instead, since it needs to be triaged as a defect, not filed alongside general cleanup work.

Every sub-agent writes directly into the shared `bugs/` folder when it finds one — **prefix the filename with the shard slug** (`<shard-slug>-<bug-slug>.md`) so parallel agents writing into the same folder can never collide; no cross-agent coordination or shared counter is needed.

### Required file structure

Every finding file (quality or bug) uses this structure:

```markdown
# <Finding title — specific enough to stand alone as a ticket title>

**Source shard:** [<Shard Name>](00-summary.md)   <!-- or ../<shard-slug>/00-summary.md from bugs/ -->
**Priority:** P1 (category)   <!-- category: structural-regression | code-judo-opportunity |
                                    spaghetti | boundary-type | file-size | modularity | legibility -->
**Effort:** S | M | L
**Type:** Bug — functional correctness issue found incidentally during a code-quality audit
<!-- ^ only present on files in bugs/ -->

## File(s)
`path/to/file.ts:120-180`

## Problem
<what's wrong, concretely>

## Why it matters
<concrete consequence, not vibes>

## Suggested fix
<specific and actionable>
```

Every shard also gets exactly one `00-summary.md`:

```markdown
# <Shard Name>

**Scope:** <paths>
**Files reviewed:** <count>  **Lines (approx):** <count>

## Summary
<2-4 sentences: overall verdict>

## Findings from this shard
- [<title>](01-<slug>.md) — P<n>
- [<title>](../bugs/<shard-slug>-<slug>.md) — **moved to `bugs/`** (functional bug) — P<n>
(one line per finding this shard produced, in priority order, whichever folder it landed in)

## Files flagged for size
- `path` — N lines (crosses 1000, or approaching it)

## Not flagged / explicitly fine
<call out genuinely clean areas explicitly instead of inventing nits to fill space>
```

Prefer a small number of high-conviction findings with real, verified file:line references over a long list of cosmetic nits — this mirrors the underlying standard's own output-priority guidance. A shard that comes back clean should say so in its summary, not manufacture filler files.

## Step 4 — Fan out

Launch one sub-agent per shard, **all in parallel, in a single batch**, running in the background. Track them (a lightweight task list works well) so completions don't get lost across a large fan-out. Let each shard's agent work independently — they should not need to coordinate with each other; scopes don't overlap, and the shard-prefixed filename convention means `bugs/` never collides either.

## Step 5 — Synthesize the overview

Once every shard has reported back, write one overview file (`docs/thermo-review/README.md`). Its job is to add value the individual files can't: cross-shard pattern-spotting. Read each shard's `00-summary.md` (not every individual finding file) to build it.

- **What this is + method.** State plainly that this was a sharded, parallel audit adapting a diff-review standard to a whole-codebase pass, how many shards were used, and the file layout (link straight to individual finding/bug files, not just shard summaries).
- **Cross-cutting patterns.** Findings that independent, non-communicating shards surfaced separately are the highest-signal material in the whole review — link directly to every finding file that hit the pattern. This is usually the most useful section in the overview.
- **Real bugs found incidentally.** Surface the whole `bugs/` folder as its own table with direct links — don't bury these inside a per-shard list.
- **File-size watch list.** Even if nothing crosses the hard threshold, note files that are close and growing, linking to the relevant finding.
- **Per-shard index.** A table: scope, files/lines reviewed, finding count, bug count, link to the shard's `00-summary.md`.
- **Suggested next step.** Point at the highest-leverage items to act on first (usually: `bugs/` before general cleanup) — a recommendation, not a mandate to fix everything found.
- **Close by offering to file tickets** — see Step 6.

## Step 6 — Offer to turn findings into tickets

End the run with something like: *"If you'd like, I can create tracking tickets (Linear, Jira, GitHub Issues, or wherever this team tracks work) for any or all of these findings — each finding file is already structured as a ready-to-file ticket. Just say the word and which findings/bugs to file."*

Do not create tickets unprompted — this pass is identification-only by default (see the output contract below). But because every finding is already a self-contained file with a title, problem statement, impact, and suggested fix, turning a batch of them into real tickets afterward is cheap **if the requester wants that** and if you have (or can get) access to their tracker. Don't assume which tool they use; ask, or check this repo's own memory/reference notes for a tracker if one is already documented.

## Output contract — what the requester should get

- One overview file (`docs/thermo-review/README.md`) as described in Step 5.
- One `bugs/` folder of shard-prefixed bug files, flat, no subfolders.
- One folder per shard, containing `00-summary.md` plus one file per quality finding.
- **This is a read-only, identification-only pass.** No source files are modified by this skill, and no tickets are filed in any external tracker unless the requester explicitly says yes to the offer in Step 6. The deliverable is a prioritized, actionable backlog of findings — each one specific enough that turning it into an actual fix (or an external ticket) is a separate, well-scoped follow-up, not a research project.

## Notes

- If the codebase is small enough to read in one pass, say so and skip the sharding — this skill exists to make otherwise-impossible full-codebase audits tractable, not to force fan-out where it isn't needed.
- If you're unsure how finely to shard a very large or very small target, ask the requester rather than guessing wildly in either direction.
- Everything in this file is generic. The only repo-specific work happens in Step 2 (shard boundaries) and the convention context you add to Step 3's briefs — the process itself is identical across codebases.
