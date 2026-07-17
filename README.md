# Dylan Clemann's Agent Skills

[![skills.sh](https://skills.sh/b/clemannd/skills)](https://skills.sh/clemannd/skills)

Agent skills I use in my day-to-day software engineering work. Everything here has been battle-tested through daily use on real projects — I only publish a skill once it has proven itself in practice, so this collection grows slowly and deliberately.

Skills are self-contained, markdown-based workflows that a coding agent can load and follow instead of improvising. They work with any agent that supports the `SKILL.md` format (Claude Code, Cursor, Codex, and others). Each skill lives in `skills/<name>/` with a `SKILL.md` entry point — frontmatter (`name`, `description`) that the agent uses to decide when the skill applies, and a body with the actual workflow — plus optional reference files for details that don't need to load up front.

## Install

```bash
npx skills@latest add ClemannD/skills
```

Or copy a skill folder directly into your agent's skills directory (e.g. `.claude/skills/` for Claude Code).

## Skills

### [ui-variant-prototype](./skills/ui-variant-prototype/SKILL.md)

Compare 2–5 layout options for a UI component live in the running app instead of in mockups. The skill scaffolds one component per variant behind a small router, a persisted variant switcher, and a floating preview toggle, so you can flip between options while browsing the real app. Once a winner is picked, it collapses everything back to a single implementation and deletes the preview machinery.

Works with any React setup (Vite, Next.js, Remix, …) and any state library; examples use shadcn/ui and Tailwind.

### [full-project-thermo-review](./skills/full-project-thermo-review/SKILL.md)

Audit an entire codebase for code quality — not just a diff. The skill shards the repo into domain-scoped chunks small enough to read in full, fans out one parallel sub-agent per shard applying the [thermo-nuclear-code-quality-review](https://github.com/cursor/plugins/blob/main/thermos/skills/thermo-nuclear-code-quality-review/SKILL.md) standard, and rolls everything up into ticket-ready files: one file per finding, genuine functional bugs split into their own `bugs/` folder, and a cross-shard overview that spots patterns no single reviewer would see. Identification-only by default, with an offer to file the findings as tickets (Linear, Jira, GitHub Issues, …) at the end.

Language- and framework-agnostic — the sharding, fan-out, and synthesis process adapts to whatever codebase it runs in.
