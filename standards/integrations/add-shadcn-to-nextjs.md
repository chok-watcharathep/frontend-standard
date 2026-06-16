---
id: add-shadcn-to-nextjs
title: Add shadcn/ui to Next.js
category: integration
status: stable
applies_to: [nextjs]
requires: [nextjs]
conflicts_with: [add-mui-to-nextjs, add-antd-to-nextjs]
tags: [ui, styling, components, shadcn, tailwind, design-tokens, theme]
updated: 2026-06-12
---

# Add shadcn/ui to Next.js

## Summary

Adds shadcn/ui (Radix-based copy-in components) + **Tailwind CSS v4** as the UI layer of a
[[nextjs]] app, **and defines the complete design-token system** for shadcn projects. Tokens are
handled **per CSS framework** — this standard owns the shadcn approach end-to-end (CSS custom
properties authored with Tailwind v4 `@theme`); [[add-mui-to-nextjs]] and [[add-antd-to-nextjs]]
handle tokens their own way. There is no shared, framework-agnostic token standard.

This is the standard that fills the **CSS-framework boundary** [[nextjs]] intentionally defers:
`components/ui/` contents, `src/theme/` tokens, `src/libs/utils.lib.ts` (`cn()`), per-component
styling, `components.json`, PostCSS/Tailwind, `globals.css`, and the `design-system/` routes.

## When to use

- The app's UI library is **shadcn/ui** (the company default for greenfield Next.js apps).

## When not to use

- Using Material UI → [[add-mui-to-nextjs]]. Using Ant Design → [[add-antd-to-nextjs]].
  Exactly **one** UI-library integration per app (they `conflicts_with` each other).

## Prerequisites

- [[nextjs]] — the base structure must exist first (flat `src/`, `@/*` → `src/*`, pnpm).

## Stack added

| Concern         | Choice                                                                   |
| --------------- | ------------------------------------------------------------------------ |
| Component model | shadcn/ui (`style: new-york`), components copied into `src/components/ui` |
| Primitives      | Radix UI (`radix-ui`, `@radix-ui/react-*`)                               |
| CSS engine      | **Tailwind CSS v4** (CSS-first, **no `tailwind.config.ts`**)             |
| PostCSS         | `@tailwindcss/postcss`                                                    |
| Variants        | `class-variance-authority` (cva)                                         |
| Class merging   | `clsx` + `tailwind-merge` → `cn()`                                       |
| Icons           | `lucide-react`                                                           |
| Animation       | `tw-animate-css` (Tailwind utilities) + `framer-motion` (JS animation)   |
| Tokens          | CSS custom properties via Tailwind v4 `@theme`, 3-tier (see below)       |

## Steps

1. **Install dependencies** (pnpm only):
   ```bash
   pnpm add radix-ui class-variance-authority clsx tailwind-merge lucide-react
   pnpm add -D shadcn tailwindcss @tailwindcss/postcss tw-animate-css
   ```
   Individual `@radix-ui/react-*` packages are added automatically as components are generated.

2. **PostCSS** — `postcss.config.mjs`:
   ```js
   const config = {
     plugins: {
       '@tailwindcss/postcss': {},
     },
   }

   export default config
   ```

3. **`cn()` helper** — `src/libs/utils.lib.ts` (note: `libs/`, suffix `.lib.ts`, per [[nextjs]]):
   ```ts
   import { clsx, type ClassValue } from 'clsx'
   import { twMerge } from 'tailwind-merge'

   export function cn(...inputs: ClassValue[]) {
     return twMerge(clsx(inputs))
   }
   ```

4. **`components.json`** — point shadcn at the [[nextjs]] paths (note `utils` → `@/libs/utils.lib`):
   ```json
   {
     "$schema": "https://ui.shadcn.com/schema.json",
     "style": "new-york",
     "rsc": true,
     "tsx": true,
     "tailwind": {
       "config": "",
       "css": "src/app/globals.css",
       "baseColor": "stone",
       "cssVariables": true,
       "prefix": ""
     },
     "iconLibrary": "lucide",
     "rtl": false,
     "aliases": {
       "components": "@/components",
       "utils": "@/libs/utils.lib",
       "ui": "@/components/ui",
       "lib": "@/libs",
       "hooks": "@/hooks"
     },
     "registries": {}
   }
   ```

5. **Author the design tokens** in `src/theme/` (see [Design token system](#design-token-system)).
   Hand-written CSS — `colors.css` (Tier 1) then `tokens.css` (Tier 2 + 3).

6. **Wire `globals.css`** — import Tailwind, the token files, then map shadcn's semantic vars to the
   tokens (see [globals.css](#globalscss-the-shadcn-binding-layer)).

7. **Add components** as needed:
   ```bash
   pnpm dlx shadcn@latest add button input dialog
   ```
   Then **re-skin** each generated component to consume the design tokens (`bg-btn-solid-bg`, not
   the default shadcn classes) — see [Component conventions](#component-conventions).

8. **Load the font** in `src/app/layout.tsx` via `next/font/google`, exposing it as `--font-sans`
   (consumed by the `@theme inline` block in `globals.css`).

## Resulting structure

```
src/
  app/
    globals.css            # Tailwind import + token imports + shadcn var mapping + base layer
    layout.tsx             # loads font via next/font, applies --font-sans
    (public)/design-system/ # live component showcase routes (one per component)
  components/
    ui/                    # shadcn primitives — FLAT files, export * (button.tsx, input.tsx, ...)
      button.tsx
      button.variants.ts   # (optional) cva block extracted for complex components
      index.ts             # export * from './button' ...
  libs/
    utils.lib.ts           # cn()
  theme/
    colors.css             # Tier 1 — primitive color scales (@theme)
    tokens.css             # Tier 2 semantic + Tier 3 component tokens (@theme)
components.json
postcss.config.mjs
```

> No `tailwind.config.ts` — Tailwind v4 is CSS-first; configuration lives in `globals.css` and the
> `@theme` blocks under `src/theme/`.

## Design token system

**Architecture: CSS-first (Tailwind v4).** All tokens are CSS custom properties declared in
`@theme` blocks. `@theme` (without `inline`) generates **both** `:root` CSS variables **and**
Tailwind utility classes from the same declaration — so `--color-btn-solid-bg` automatically yields
the `bg-btn-solid-bg` utility.

**Import chain (no circular deps):** `colors.css` → `tokens.css` → `globals.css`.

### Three tiers

| Tier             | Source       | CSS var example             | Tailwind utility         | Use it…                                              |
| ---------------- | ------------ | --------------------------- | ------------------------ | ---------------------------------------------------- |
| 1 — Primitive    | `colors.css` | `--color-brand-primary-500` | `bg-brand-primary-500`   | **Only inside `src/theme/`** to define higher tiers  |
| 2 — Semantic     | `tokens.css` | `--color-text-base-primary` | `text-text-base-primary` | **Default in components** + general layout/typography |
| 3 — Component    | `tokens.css` | `--color-btn-solid-bg`      | `bg-btn-solid-bg`        | **Only when Figma defines an explicit component token** |

All color tokens live under the `--color-*` namespace so Tailwind generates `bg-`/`text-`/`border-`
utilities. **Naming = Figma kebab path → CSS var:** Figma `btn/solid/bg` → `--color-btn-solid-bg` →
`bg-btn-solid-bg`.

### Tier 1 — `src/theme/colors.css` (primitive scales)

Raw brand/system palette. Each family is a `50`–`900` scale. **Company-specific values** — the set
below is the canonical "HI Design System" palette; copy it verbatim.

```css
/* Tier 1: Primitive color scales — Tailwind v4 CSS-first theme definition. */
@theme {
  /* Brand Primary (Purple) */
  --color-brand-primary-50: #f2f2ff;
  /* ...100–400... */
  --color-brand-primary-500: #945dff;
  /* ...600–800... */
  --color-brand-primary-900: #4f169c;

  /* Brand Secondary (Cream) · Brand Tertiary (Teal) — same 50–900 shape */

  /* Neutral Gray */
  --color-gray-50: #fafafa;
  --color-gray-500: #767279;
  --color-gray-900: #19181b;

  /* System: error (red) · warning (yellow) · success (green) · info (blue) — each 50–900 */
  --color-error-500: #ef4444;
  --color-warning-500: #f59e0b;
  --color-success-500: #10b981;
  --color-info-500: #0ea5e9;

  /* Accent families (each 50–900): red, orange, yellow, lime, green, teal, sky, blue,
     indigo, violet, purple, pink */
}
```

> The full hex tables (all 50–900 steps for every family) are the company brand source of truth.
> Keep `src/theme/colors.css` as that source; do not redefine palette values anywhere else.

### Tier 2 + Tier 3 — `src/theme/tokens.css`

Semantic and component tokens. **Every value references a `var(--color-*)` from `colors.css`** —
never a raw hex (except a few intentional absolutes like `#ffffff`/`#000000` and overlays).

```css
/* Tier 2: Semantic + Tier 3: Component tokens. All reference var(--color-*) from colors.css. */
@theme {
  /* ── Tier 2: Text ── */
  --color-text-base-primary: var(--color-gray-900);
  --color-text-base-secondary: var(--color-gray-500);
  --color-text-link-primary: var(--color-brand-primary-800);
  --color-text-system-error: var(--color-error-700);

  /* ── Tier 2: Background / Fill / Border (same pattern) ── */
  --color-bg-base-white: #ffffff;
  --color-bg-brand-primary-medium: var(--color-brand-primary-500);
  --color-fill-base-gray-dark2: var(--color-gray-800);
  --color-border-base-gray-medium: var(--color-gray-300);

  /* ── Tier 3: Button (only because Figma defines explicit button tokens) ── */
  --color-btn-solid-bg: var(--color-brand-primary-500);
  --color-btn-solid-bg-hover: var(--color-brand-primary-800);
  --color-btn-solid-text: #ffffff;
  --color-btn-outline-border: var(--color-brand-primary-700);
  --color-btn-ghost-text: var(--color-brand-primary-800);

  /* ── Tier 3: Input / Tag / Tooltip / Chip … (only where Figma defines them) ── */
  --color-input-border-focus: var(--color-brand-primary-700);
}
```

### `globals.css` — the shadcn binding layer

shadcn expects its own semantic vars (`--primary`, `--background`, `--radius`, …). Map them to the
tokens here; **shadcn's vars hold no raw hex** — they reference `var(--color-*)`.

```css
@import 'tailwindcss';
@import '../theme/colors.css';
@import '../theme/tokens.css';
@import 'tw-animate-css';

@custom-variant dark (&:is(.dark *));

@theme inline {
  /* Typography */
  --font-sans: var(--font-noto-sans-thai), 'Noto Sans', sans-serif;

  /* Radius scale derived from --radius */
  --radius-sm: calc(var(--radius) - 4px);
  --radius-md: calc(var(--radius) - 2px);
  --radius-lg: var(--radius);
  --radius-xl: calc(var(--radius) + 4px);

  /* shadcn semantic color mappings → expose --primary etc. as Tailwind color utilities */
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-border: var(--border);
  --color-ring: var(--ring);
  /* …card, popover, secondary, muted, accent, destructive, input, sidebar… */
}

:root {
  --radius: 0.5rem;

  /* shadcn vars → reference the design tokens (never raw brand hex) */
  --background: #ffffff;
  --foreground: var(--color-gray-900);
  --primary: var(--color-brand-primary-500);
  --primary-foreground: #ffffff;
  --secondary: var(--color-brand-secondary-500);
  --muted: var(--color-gray-100);
  --accent: var(--color-brand-primary-50);
  --destructive: var(--color-error-700);
  --border: var(--color-gray-300);
  --input: var(--color-gray-300);
  --ring: var(--color-brand-primary-500);
  /* …sidebar-* … */
}

@layer base {
  * {
    @apply border-border outline-ring/50;
  }
  body {
    @apply bg-background text-foreground font-sans;
  }
}

@layer components {
  /* Responsive containers: padding 16px → 32px → 56px across breakpoints. */
  .container-compact { @apply mx-auto w-full max-w-[912px] px-4 md:px-6 lg:px-14; }
  .container-desktop { @apply mx-auto w-full max-w-[1180px] px-4 md:px-6 lg:px-14; }
  .container-wide    { @apply mx-auto w-full max-w-[1552px] px-4 md:px-6 lg:px-14; }
}
```

### Token usage rules

1. **Priority in components:** component token (Tier 3, if Figma defines it) → semantic token
   (Tier 2). 
2. **NEVER use a primitive (Tier 1) token in components.** `bg-brand-primary-500` / `text-gray-900`
   are for `src/theme/` only. In components use `bg-btn-solid-bg` (Tier 3) or
   `bg-bg-brand-primary-medium` / `text-text-base-primary` (Tier 2).
3. **Never hardcode hex** in components (`bg-[#945dff]` ❌).
4. **Don't invent component tokens.** Add a Tier 3 token only when Figma defines an explicit
   component-level token; otherwise compose from Tier 2 semantics. Avoids token bloat / Figma drift.
5. **No inline `style={{}}`** except for values computed at runtime.

```tsx
// ❌ never
'bg-[#945dff]'            // hardcoded hex
'bg-brand-primary-500'    // primitive token in a component
// ✅ always
'bg-btn-solid-bg'         // component token (Figma-defined)
'bg-bg-brand-primary-medium' // semantic token
```

### Token values needed in JavaScript

When a value must exist in JS (e.g. canvas/SVG arc colors, chart palettes) it cannot read a Tailwind
class. Keep those hex values in the component's `*.variants.ts` as a typed `const` map — colocated,
not scattered:

```ts
// components/ui/progress-ring.variants.ts
export const PROGRESS_RING_COLOR_MAP = {
  amber: { arcStart: '#fbd34d', arcEnd: '#f59e0b', fill: '#f59e0b' },
  brandSolid: { arcStart: '#8030f7', arcEnd: '#8030f7', fill: '#8030f7' },
} as const
export type ProgressRingColor = keyof typeof PROGRESS_RING_COLOR_MAP
```

## Component conventions

shadcn's generated primitives are the **only** exception to [[nextjs]]'s default-export + folder
component convention.

- **`src/components/ui/` is FLAT:** one file per primitive (`button.tsx`, `input.tsx`), **named
  exports**, re-exported with `export *` so the shadcn generator can keep appending:
  ```ts
  // components/ui/index.ts
  export * from './button'
  export * from './input'
  export * from './dialog'
  ```
- **Variants use `cva()`.** Inline the `cva` block in the component file; extract it to a sibling
  `*.variants.ts` only when it's large or shared (the `*.variants.ts` also holds any JS-side token
  maps). Reference design tokens inside variants:
  ```tsx
  // components/ui/button.tsx
  import { cva, type VariantProps } from 'class-variance-authority'
  import { cn } from '@/libs/utils.lib'

  const buttonVariants = cva('inline-flex items-center justify-center rounded-full font-semibold …', {
    variants: {
      variant: {
        solid: ['bg-btn-solid-bg text-btn-solid-text', 'hover:bg-btn-solid-bg-hover'],
        outline: ['border border-btn-outline-border text-btn-outline-text'],
        ghost: ['bg-transparent text-btn-ghost-text', 'hover:bg-btn-ghost-bg-hover'],
      },
      size: { xs: 'h-7 text-sm', sm: 'h-9', md: 'h-12 text-lg', lg: 'h-14 text-lg' },
    },
    defaultVariants: { variant: 'solid', size: 'md' },
  })
  ```
- **Props order** (per [[nextjs]]): `key`/`ref` → `id`/`className` → data → config (`variant`,
  `size`, `disabled`) → content (`children`) → event handlers last.

### Shared-components rule (critical)

App code (`src/features/**`, `src/app/**`, `src/components/layouts/**`) **must not style raw
interactive HTML** (`<button>`, links, inputs, cards…) with Tailwind. Always go through a primitive
in `components/ui/`.

- Recurring need (used in ≥ 2 places / part of the design system) → **add a `cva` variant** to the
  primitive and use it everywhere.
- One-off tweak → still use the primitive; pass overrides via `className`. Never drop to raw HTML.
- **Exception:** primitives *inside* `components/ui/` may use raw HTML — they *are* the building
  blocks. The rule binds consumers, not primitive authors.

```tsx
// ❌ never — raw button with Tailwind in app code
<button className="rounded-full bg-fill-base-gray-dark2 px-5 text-white">Set target</button>
// ✅ always
<Button variant="solidNeutral">Set target</Button>
```

## Design system showcase

Each primitive gets a live reference route under `src/app/(public)/design-system/<component>/page.tsx`
showing all variants × sizes × states. When you add a primitive, add its showcase page. **Exclude
design-system routes from the app's home route index.**

## Animation

`framer-motion` is the standard for JS-driven animation; Tailwind `transition-*` (+ `tw-animate-css`)
for simple hover/focus. Animated components must be client (`'use client'`); animate `transform`/
`opacity` only; don't double up Tailwind transitions with framer-motion on the same property; no
other animation libraries.

## Rules

- ✅ Do: keep `src/theme/` (`colors.css`, `tokens.css`) as the single source of palette + tokens.
- ✅ Do: use semantic (Tier 2) tokens by default; component (Tier 3) tokens only when Figma defines them.
- ✅ Do: keep `components/ui/` flat with `export *`; use `cva` for variants.
- ✅ Do: route all interactive UI through `components/ui/` primitives.
- ❌ Don't: use primitive tokens or hardcoded hex in components.
- ❌ Don't: add a `tailwind.config.ts` (Tailwind v4 is CSS-first).
- ❌ Don't: combine with [[add-mui-to-nextjs]] / [[add-antd-to-nextjs]].

## Gotchas

- shadcn components are **copied into the repo** (not a dependency) — you own and re-skin them to
  consume tokens; expect to edit generated files after `shadcn add`.
- `@theme` (no `inline`) declares vars **and** generates utilities; `@theme inline` only maps
  existing vars to utilities. Tier 1/2/3 use `@theme`; the shadcn `--primary`-style mapping uses
  `@theme inline`. Don't mix them up.
- The reference repo's `.dark` variant is declared but unpopulated (light-only today). Add dark
  values to a dark token set when the design defines one.
- VS Code: install **Tailwind CSS IntelliSense** and point it at the v4 entry —
  `tailwindCSS.experimental.configFile: "./src/app/globals.css"` and
  `tailwindCSS.classFunctions: ["cva", "cx", "cn"]` so token utilities autocomplete inside `cva()`.

## Related standards

- [[nextjs]] — base structure; this standard fills its deferred CSS-framework boundary.
- [[add-mui-to-nextjs]] — alternative UI library; **mirrors this 3-tier token concept and reuses these
  token names** (as a typed `theme.tokens.*` object instead of CSS vars). Mutually exclusive with this.
- [[add-antd-to-nextjs]] — alternative UI library; also mirrors this 3-tier token concept and reuses
  these token names (via antd-style `customToken`). Mutually exclusive with this.
- [[listing]] — listing/table pattern; uses these primitives.
