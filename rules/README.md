# Rules

Cursor-style **workspace rules** — short, always-on (or glob-scoped) policies that shape how an agent writes code. Unlike [skills](../skills), which are workflows loaded when a matching task appears, rules are conventions the agent should follow whenever it is editing the relevant files.

Each rule is a `.mdc` file (YAML frontmatter + markdown). Frontmatter controls when the rule applies:

| Field | Meaning |
| ----- | ------- |
| `description` | Shown in the agent UI; also helps the agent decide relevance |
| `alwaysApply: true` | Include in every chat |
| `globs` | Include when matching files are in context |

Adjust `alwaysApply` / `globs` when installing into a project that shouldn't load every rule on every chat.

## Install

Copy the `.mdc` files you want into the project's rules directory:

```bash
# Cursor
cp rules/react/*.mdc path/to/project/.cursor/rules/

# Canonical copy used by several agents (Cursor, Codex, …)
cp rules/react/*.mdc path/to/project/.agents/rules/
```

Or copy individual files. Rule filenames are the identifiers — keep them.

These files are **not** installed by `npx skills add`. Skills and rules are separate: skills go in `skills/<name>/SKILL.md`; rules go in `.cursor/rules/` or `.agents/rules/`.

## Layout

```text
rules/
  react/          # React / Next.js UI conventions
```

Future stacks (NestJS, Expo, …) can get their own folder next to `react/`.

## React

Rules for React and Next.js App Router UI. Copy from [`react/`](./react/).

| File | Summary |
| ---- | ------- |
| [`component-ordering.mdc`](./react/component-ordering.mdc) | In a `.tsx` file, place helper/subcomponents **below** the main exported component. |
| [`lift-shared-route-utils.mdc`](./react/lift-shared-route-utils.mdc) | Lift shared helpers to the lowest common `_lib/`; never import across feature folders. |
| [`main-container-layout.mdc`](./react/main-container-layout.mdc) | Use `@*/main:` container queries for main-column density; viewport breakpoints for shell/device. |
| [`modular-components.mdc`](./react/modular-components.mdc) | Feature folders for complex UI; leaf components own API/actions; no god hooks or handler props. |
| [`nextjs-app-file-organization.mdc`](./react/nextjs-app-file-organization.mdc) | App Router colocation: `_components`, `_hooks`, `_state`, `_lib` at the lowest owning segment. |
| [`no-callback-props.mdc`](./react/no-callback-props.mdc) | Colocate event handlers in the lowest component that needs them; don't pass callbacks as props. |
| [`no-component-index-files.mdc`](./react/no-component-index-files.mdc) | Never use `index.ts` / `index.tsx` as a component folder barrel; name the entry file explicitly. |
| [`react-inline-prop-types.mdc`](./react/react-inline-prop-types.mdc) | Inline prop types in the function signature; default-export the primary component. |
| [`use-cn-for-conditional-classes.mdc`](./react/use-cn-for-conditional-classes.mdc) | Use the `cn()` helper for conditional `className` values, not template-literal class strings. |
