# UI Variant Prototype — Reference

## Naming conventions

| Artifact | Pattern | Example |
|----------|---------|---------|
| Data | `{feature}-data.ts` | `recipe-header-time-stats-data.ts` |
| Preview state | `{feature}-preview.state.ts` | `card-actions-preview.state.ts` |
| Toggle | `{feature}-preview-toggle.tsx` | `card-actions-preview-toggle.tsx` |
| Variant | `variants/{feature}-{slug}.tsx` | `variants/recipe-header-time-stats-tiles.tsx` |
| Shared UI | `{feature}-shared.tsx` | optional icons, empty tooltips |
| Storage key | `{app}:{area}-{feature}-preview-variant-v{N}` | `acme:checkout-summary-preview-variant-v1` |

Variant slugs: short, descriptive (`tiles`, `inline`, `minimal`)—not `option1`.

## Variant count

| Count | Toggle layout |
|-------|----------------|
| 2–3 | `grid grid-cols-3` (extra empty cells OK) or `flex gap-1` |
| 4–5 | `grid grid-cols-5` or `flex flex-wrap` with `text-[10px]` on small screens |

Cap at **5** options; more creates decision fatigue and noisy toggles.

## What belongs in shared vs variant

**Shared (`*-shared.tsx` or data file):**
- Icon map keyed by item type
- Formatters (dates, durations, currency)
- Empty/missing value tooltip trigger pattern
- `role` / `aria-label` on the outer group

**Per variant:**
- Grid vs flex vs inline flow
- Borders, backgrounds, pill shapes
- Label position (above vs beside value)
- Whether icons appear at all

## Empty / missing states

Prefer tooltip on the **whole tile/row** when the hit target is larger than the value:

```tsx
if (!item.isEmpty) {
  return <div className={tileClassName}>{content}</div>;
}

return (
  <Tooltip>
    <TooltipTrigger asChild>
      <div className={tileClassName} tabIndex={0} aria-label={`${item.label}: not added`}>
        {content}
      </div>
    </TooltipTrigger>
    <TooltipContent>Not added</TooltipContent>
  </Tooltip>
);
```

## Import paths after flattening

When removing the subfolder, update consumers to the explicit entry file:

```ts
// before
import { Feature } from './feature-name/feature-name';

// after
import { Feature } from './feature-name';
```

## Anti-patterns

- Feature flags or env vars for layout preview—use persisted local state (e.g. `localStorage`) + toggle instead; delete before ship.
- Duplicating API/query logic in each variant file.
- Leaving preview toggle in production "just in case."
- `index.ts` barrels in component folders (many repos forbid this).
- Six+ variants—split into a second prototyping round instead.
- Pulling in a new state management library just for the preview toggle—use whatever the app already has.

## Example flow (generic)

**Task:** Compare header stat layouts for a dashboard card.

1. `useDashboardStatItems()` → `{ key, label, value, isEmpty }[]`
2. Variants: `columns`, `inline`, `tiles`, `minimal`, `icon-stack`
3. User picks `tiles` → merge tiles markup into `dashboard-card-stats.tsx`, delete preview folder, remove toggle from `dashboard/page.tsx`.
