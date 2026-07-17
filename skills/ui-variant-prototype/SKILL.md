---
name: ui-variant-prototype
description: Scaffold temporary multi-layout UI prototypes with persisted variant switching and a floating preview toggle. Use when comparing 2–5 design options for a component, the user asks to try alternatives before picking one, or when building A/B-style layout previews in React apps with shadcn.
---

# UI Variant Prototype

Temporary workflow for comparing visual/layout options in the running app, then collapsing to a single implementation when the user picks a winner.

**Stack assumed:** React (Next.js App Router `'use client'` conventions used below, adapt for other setups), shadcn/ui `Button`, Tailwind, and whatever state management the app already uses (Jotai, Zustand, Redux, plain `useState`, React Context — see [Preview state](#phase-3--preview-infrastructure)).

## When to use

- User wants to see **2–5 layout options** side-by-side in context (not Figma-only).
- The decision is **presentational** (spacing, typography, tile vs inline, icon treatment)—not business logic.
- You will **delete** preview machinery after one variant ships.

Do **not** use for: API design, routing, or long-lived feature flags.

## File layout

Colocate under the owning feature folder (route `_components`, `app-components`, etc.):

```
feature-name/
├── feature-name.tsx                    # router: reads variant state, renders active variant
├── feature-name-data.ts                # shared data hook / pure mappers (no layout)
├── feature-name-preview.state.ts       # variant union + persisted state + labels
├── feature-name-preview-toggle.tsx     # floating switcher (dev-only)
├── feature-name-shared.tsx             # optional: icons, value formatting, a11y helpers
└── variants/
    ├── feature-name-variant-a.tsx
    ├── feature-name-variant-b.tsx
    └── ...
```

Use explicit entry files (`feature-name.tsx`), not `index.ts` barrels.

## Phase 1 — Extract shared data

1. Move fetching, context reads, and formatting into `feature-name-data.ts`.
2. Export a **stable props shape** variants consume, e.g. `FeatureItem[]` with `{ id, label, value, isEmpty }`.
3. Keep variants **dumb**: layout + markup only; no duplicate business logic.

```ts
// feature-name-data.ts
'use client';

export type FeatureItem = { key: string; label: string; value: string; isEmpty: boolean };

export function useFeatureItems(): FeatureItem[] {
  // context, hooks, formatters
}
```

## Phase 2 — Implement variants

1. Add one file per option under `variants/`.
2. Each exports a single component: `FeatureNameVariantA({ items }: { items: FeatureItem[] })`.
3. Share markup via `feature-name-shared.tsx` when variants differ only in wrapper/layout (icons, tooltips, empty states).
4. Keep variants **visually distinct**—avoid five nearly identical tweaks.

## Phase 3 — Preview infrastructure

### Preview state (`feature-name-preview.state.ts`)

The only requirement: the chosen variant must **persist** (survive a reload/navigation) so you can compare options while browsing the real app. Any state approach works as long as it reads/writes a small string union and persists it to `localStorage` or equivalent. Pick whatever the project already uses:

- **Already on Jotai?** Use `atomWithStorage` — one line, persistence included.
- **Already on Zustand?** Use the `persist` middleware.
- **Already on Redux?** A slice + a `localStorage`-syncing subscriber, or `redux-persist`.
- **No global state library / keep it local?** Plain `useState` + a `useEffect` that reads/writes `localStorage` directly.
- **React Context-based app?** A small context provider wrapping the same `useState` + `localStorage` pattern.

Whatever you pick, expose the same three things: the variant union/type, a read+write accessor, and a labels map for the toggle UI.

**Example implementation (Jotai)** — swap for your library of choice:

```ts
import { atomWithStorage } from 'jotai/utils';

export const FEATURE_PREVIEW_VARIANTS = ['a', 'b', 'c'] as const;
export type FeaturePreviewVariant = (typeof FEATURE_PREVIEW_VARIANTS)[number];

export const featurePreviewVariantAtom = atomWithStorage<FeaturePreviewVariant>(
  'my-app:feature-preview-variant-v1', // bump suffix when union changes
  'a',
);

export const FEATURE_PREVIEW_LABELS: Record<FeaturePreviewVariant, string> = {
  a: 'Option A',
  b: 'Option B',
  c: 'Option C',
};
```

**Example implementation (plain React, no library):**

```ts
'use client';
import { useEffect, useState } from 'react';

export const FEATURE_PREVIEW_VARIANTS = ['a', 'b', 'c'] as const;
export type FeaturePreviewVariant = (typeof FEATURE_PREVIEW_VARIANTS)[number];

const STORAGE_KEY = 'my-app:feature-preview-variant-v1';

export function useFeaturePreviewVariant() {
  const [variant, setVariant] = useState<FeaturePreviewVariant>(
    () => (localStorage.getItem(STORAGE_KEY) as FeaturePreviewVariant | null) ?? 'a',
  );

  useEffect(() => {
    localStorage.setItem(STORAGE_KEY, variant);
  }, [variant]);

  return [variant, setVariant] as const;
}

export const FEATURE_PREVIEW_LABELS: Record<FeaturePreviewVariant, string> = {
  a: 'Option A',
  b: 'Option B',
  c: 'Option C',
};
```

- Storage key: `{project-or-app}:{area}-{feature}-preview-variant-v{N}`.
- Bump `v{N}` if you add/remove/rename variants (avoids stale localStorage).

### Router (`feature-name.tsx`)

```tsx
'use client';

import type { ComponentType } from 'react';
import { useFeatureItems } from './feature-name-data';
// Jotai example — replace with your state hook of choice:
import { useAtomValue } from 'jotai';
import { featurePreviewVariantAtom, type FeaturePreviewVariant } from './feature-name-preview.state';
import { FeatureNameVariantA } from './variants/feature-name-variant-a';
// ...

const variantComponents: Record<
  FeaturePreviewVariant,
  ComponentType<{ items: ReturnType<typeof useFeatureItems> }>
> = {
  a: FeatureNameVariantA,
  b: FeatureNameVariantB,
  c: FeatureNameVariantC,
};

export function FeatureName() {
  const variant = useAtomValue(featurePreviewVariantAtom);
  const items = useFeatureItems();
  const Variant = variantComponents[variant];
  return <Variant items={items} />;
}
```

### Floating toggle (`feature-name-preview-toggle.tsx`)

Mount **once** on the page/layout that already wraps whatever provider your state approach needs (Jotai `Provider`, Zustand doesn't need one, Context needs its own provider, etc.)—sibling to the feature, not inside server components.

```tsx
'use client';

import { useAtom } from 'jotai'; // swap for your state hook
import { Button } from '@/components/ui/button';
import { cn } from '@/lib/utils';
import {
  FEATURE_PREVIEW_LABELS,
  FEATURE_PREVIEW_VARIANTS,
  featurePreviewVariantAtom,
  type FeaturePreviewVariant,
} from './feature-name-preview.state';

export function FeatureNamePreviewToggle() {
  const [variant, setVariant] = useAtom(featurePreviewVariantAtom);

  return (
    <div
      className="border-border bg-card/95 fixed right-4 bottom-4 z-50 flex max-w-[min(100vw-2rem,24rem)] flex-col gap-2 rounded-2xl border p-3 shadow-lg backdrop-blur-sm"
      role="region"
      aria-label="Layout preview"
    >
      <p className="text-muted-foreground text-xs font-medium">Layout preview</p>
      <div className={cn('grid gap-1', `grid-cols-${FEATURE_PREVIEW_VARIANTS.length}`)}>
        {FEATURE_PREVIEW_VARIANTS.map((option) => (
          <Button
            key={option}
            type="button"
            size="sm"
            variant={variant === option ? 'default' : 'outline'}
            className="h-8 text-xs"
            aria-pressed={variant === option}
            onClick={() => setVariant(option as FeaturePreviewVariant)}
          >
            {FEATURE_PREVIEW_LABELS[option]}
          </Button>
        ))}
      </div>
    </div>
  );
}
```

For 4–5 options, use `grid-cols-5` or `flex flex-wrap gap-1` instead of dynamic Tailwind class names (JIT may not see template literals).

### Page wiring

```tsx
import { Provider } from 'jotai'; // only needed if your state approach requires a provider

export function SomePage() {
  return (
    <Provider>
      <FeatureName />
      {/* other content */}
      <FeatureNamePreviewToggle />
    </Provider>
  );
}
```

If the page already has a provider for other state, reuse it—do not nest providers unless isolating state.

## UX and a11y notes

- Toggle is **dev/preview only**—remove before merge unless the team explicitly wants it.
- Wrap **entire interactive regions** (e.g. empty-state tiles) in tooltips, not just inner text.
- For non-button tooltip triggers, use `tabIndex={0}` and a descriptive `aria-label` on the wrapper.
- Preserve semantic structure (`role="group"`, `aria-label` on the stats region, `sr-only` labels where needed).

## Phase 4 — Ship the winner (cleanup)

When the user picks a variant:

1. **Inline** the winning variant into `feature-name.tsx` (or keep one `variants/` file only if large).
2. **Keep** `feature-name-data.ts` if it still separates data from UI.
3. **Delete:** `feature-name-preview.state.ts`, `feature-name-preview-toggle.tsx`, unused `variants/*`, `feature-name-shared.tsx` if no longer needed.
4. **Remove** toggle import from page/layout.
5. **Flatten** folder if only `feature-name.tsx` + data file remain (match repo colocation conventions).
6. Run typecheck; grep for orphaned imports (`PreviewToggle`, `preview-variant`).

Do not leave persisted preview state or floating toggles in production unless requested.

## Checklist

**Scaffold**
- [ ] Shared data hook with stable item type
- [ ] 2–5 variant components, meaningfully different
- [ ] Preview state + labels + versioned storage key
- [ ] Router component + variant map
- [ ] Floating toggle on page, wired to whatever provider the state approach needs

**Ship**
- [ ] Winner merged into main component
- [ ] Preview/toggle/dead variants removed
- [ ] Page imports cleaned up
- [ ] Typecheck passes

## Additional resources

- File templates and naming table: [reference.md](reference.md)
