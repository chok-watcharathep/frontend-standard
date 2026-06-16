---
id: add-antd-to-nextjs
title: Add Ant Design to Next.js
category: integration
status: stable
applies_to: [nextjs]
requires: [nextjs]
conflicts_with: [add-shadcn-to-nextjs, add-mui-to-nextjs]
tags: [ui, styling, components, antd, antd-style, design-tokens, theme, typescript]
updated: 2026-06-16
---

# Add Ant Design to Next.js

## Summary

Adds Ant Design (antd v5) + **`antd-style`** as the UI layer of a [[nextjs]] app, wired for the App
Router (cssinjs SSR), **and defines the complete design-token system** for antd projects. Tokens are
handled **per CSS framework** — this standard owns the antd approach end-to-end. It mirrors the
**3-tier token concept** of [[add-shadcn-to-nextjs]] / [[add-mui-to-nextjs]] (primitive → semantic →
component): primitives in `colors.ts`, the authored Figma-named layer on `antd-style`'s **`customToken`**,
consumed via `createStyles` / `useTheme`. antd's **native Seed → Map → Alias** token system is used as
the "sync" layer that keeps **default antd components on-brand**. There is no shared, framework-agnostic
token standard.

This is the standard that fills the **CSS-framework boundary** [[nextjs]] defers for antd: `src/theme/`
(antd `ThemeConfig` + the token object + per-component `theme.components`), `src/types/theme.d.ts`
augmentation, provider + SSR wiring, `components/ui/` composite components, and the `design-system/`
showcase routes.

## When to use

- The app's UI library is **Ant Design (antd)**.

## When not to use

- Using shadcn/ui → [[add-shadcn-to-nextjs]]. Using MUI → [[add-mui-to-nextjs]].
  Exactly **one** UI-library integration per app (they `conflicts_with` each other).

## Prerequisites

- [[nextjs]] — the base structure must exist first (flat `src/`, `@/*` → `src/*`, pnpm).

## Stack added

| Concern         | Choice                                                                       |
| --------------- | ---------------------------------------------------------------------------- |
| Component model | antd components used **directly** from `antd`, themed globally               |
| Styling engine  | `antd-style` (`createStyles`, `customToken`) on antd's `@ant-design/cssinjs` |
| App Router SSR  | `antd-style` `StyleRegistry` (`extractStaticStyle` + `useServerInsertedHTML`) |
| Theme           | antd `ThemeConfig` split across `src/theme/*.theme.ts`, assembled in `index.ts` |
| Tokens          | `customToken` (typed, Figma-named), 3-tier concept (see below)               |
| Default sync    | antd **Seed → Map → Alias** (`theme.token` + `algorithm`) fed from `colors.ts` |
| Composite UI    | `src/components/ui/<Name>/` folder + `.style.ts` (per [[nextjs]])            |

> **Not** `@ant-design/nextjs-registry`. Because `antd-style` shares `@ant-design/cssinjs` with antd
> core, a single `antd-style` `StyleRegistry` extracts **both** antd component styles and our
> `createStyles`. Adding the nextjs-registry too creates a second cache / double insertion.

## Steps

1. **Install dependencies** (pnpm only, per [[nextjs]]):
   ```bash
   pnpm add antd antd-style
   ```

2. **Author Tier 1 primitives** — `src/theme/colors.ts` (the single source of hex; see
   [Design token system](#design-token-system)).

3. **Author the token layer** — `src/theme/tokens.theme.ts` exports the `tokens` object
   (Tier 2 semantic + Tier 3 component, Figma-named), every value referencing `colors.ts`. This object
   is passed to `antd-style` as `customToken`.

4. **Build the antd `ThemeConfig`** — `src/theme/base.theme.ts`: **seed** `theme.token` from
   `tokens`/`colors`, keep antd's default **algorithm**, override Map/Alias tokens **only where Figma is
   explicit** (the hybrid; see [Default-component sync](#default-component-sync--srcthemebasethemets)).

5. **Add the TypeScript augmentation** — `src/types/theme.d.ts`: declare the `ThemeTokens` interface
   and `declare module 'antd-style' { interface CustomToken extends ThemeTokens }` so `useTheme()` and
   `createStyles`' `token` are fully typed.

6. **Write per-component overrides** — one `src/theme/<component>.theme.ts` per antd component you
   restyle (`button.theme.ts`, …), mapping `tokens.*` onto antd's `theme.components.<Component>` slots.

7. **Assemble the theme** — `src/theme/index.ts` builds `themeConfig` (base + `components`) and
   re-exports `{ themeConfig, tokens }`.

8. **Wire SSR + provider** — `StyleRegistry` (`src/components/providers/StyleRegistry/`) wrapping an
   `AppThemeProvider` (antd-style `ThemeProvider`), mounted in `src/app/layout.tsx`.

9. **Add a showcase route** per themed component under `src/app/(public)/design-system/<component>/`.

## Resulting structure

```
src/
  app/
    layout.tsx                     # <StyleRegistry> → <AppThemeProvider>
    (public)/design-system/        # live component showcase routes (one per themed component)
  components/
    providers/
      StyleRegistry/               # 'use client'; StyleProvider(cache) + useServerInsertedHTML
      AppThemeProvider/            # 'use client'; antd-style ThemeProvider (theme + customToken)
      index.ts
    ui/
      <Name>/                      # composite components: <Name>.tsx + <Name>.style.ts + index.ts
  theme/
    colors.ts                      # Tier 1 — primitive color scales (source of hex)
    tokens.theme.ts                # Tier 2 semantic + Tier 3 component (Figma-named) — the customToken
    base.theme.ts                  # antd ThemeConfig: token seeds + algorithm (default-component sync)
    button.theme.ts                # per-component theme.components override (one file per component)
    …                              # input, select, modal, table, …
    index.ts                       # assembles themeConfig.components; exports { themeConfig, tokens }
  types/
    theme.d.ts                     # module augmentation (antd-style CustomToken = ThemeTokens)
```

> Per [[nextjs]] the `src/` tree is **flat** — `src/theme/`, `src/types/`, `src/components/` — not
> `src/shared/*`.

## Design token system

**Two layers, two jobs** (the same split as [[add-mui-to-nextjs]]):

1. **antd Seed → Map → Alias** (`theme.token` + `algorithm`) — antd's *native* token system, fed from
   `colors.ts`. Its job is to keep **default antd components on-brand**. antd's algorithm derives the
   full Map/Alias set (hover/active/disabled states, contrast-safe text, borders) from a few seeds.
2. **`customToken`** (`antd-style`) — our **Figma-named Tier 2/3 layer**, the exact `colors.ts` values
   under design-system names. Its job is **exact brand fidelity in authored styling**. This is the
   analog of MUI's `theme.tokens.*` and shadcn's `--color-*` tokens.

antd's alias tokens (`colorPrimary`, `colorBgContainer`, …) are **not Figma-named**, so authoring brand
styling against them is guesswork — that's why authored styling goes through `customToken`.

**Source-of-truth chain (no drift):** `colors.ts` → (`base.theme.ts` seeds **and** `tokens.theme.ts`).
Both derive from the same `colors.ts`.

### Three tiers

| Tier          | Lives in          | Example                          | Use it…                                                  |
| ------------- | ----------------- | -------------------------------- | -------------------------------------------------------- |
| 1 — Primitive | `colors.ts`       | `colors.primary[500]`            | **Only inside `src/theme/`** to build seeds + tokens     |
| 2 — Semantic  | `tokens.theme.ts` | `token.text.base.primary`        | **Default** in `createStyles` / `.style.ts`              |
| 3 — Component | `tokens.theme.ts` | `token.button.solid.bg`          | **Only when Figma defines an explicit component token**  |

Token names mirror [[add-shadcn-to-nextjs]] / [[add-mui-to-nextjs]] (`text.base.primary`,
`button.solid.bg`) so all three standards point at the same Figma tokens.

### Tier 1 — `src/theme/colors.ts` (primitive scales)

Identical to [[add-mui-to-nextjs]] — the canonical "HI Design System" palette, the **only** place raw
hex lives. Copy verbatim; do not redefine elsewhere.

```ts
// src/theme/colors.ts — Tier 1 primitives. The ONLY place raw hex lives.
export const colors = {
  primary: {
    50: '#f5f5ff', 100: '#ede8ff', /* …200–400… */ 500: '#945dff',
    600: '#8030f7', 700: '#721ee3', 800: '#5f18bf', 900: '#4f169c',
  },
  grey:    { 50: '#fafafa', /* …100–800… */ 500: '#767279', 900: '#19181b' },
  error:   { /* 50–900 */ 500: '#ef4444', 700: '#b91c1c' },
  warning: { /* 50–900 */ 500: '#f59e0b' },
  success: { /* 50–900 */ 500: '#10b981' },
  info:    { /* 50–900 */ 500: '#0ea5e9' },
  // accent families (each 50–900): red, orange, yellow, lime, green, teal, sky, blue,
  // indigo, violet, purple, pink
  accent:  { red: { /* 50–900 */ }, /* … */ },
  common:  { white: '#ffffff', black: '#000000' },
} as const
```

### Tier 2 + Tier 3 — `src/theme/tokens.theme.ts` (the `customToken`)

The authored layer. **Every value references `colors.*`** — never raw hex (except intentional
absolutes). Shape matches the `ThemeTokens` interface; this object is passed to `ThemeProvider` as
`customToken`.

```ts
// src/theme/tokens.theme.ts — Tier 2 semantic + Tier 3 component. Built from colors.ts.
import { colors } from './colors'
import type { ThemeTokens } from 'antd-style'

export const tokens: ThemeTokens = {
  // ── Tier 2: Semantic ──
  text: {
    base: { primary: colors.grey[900], secondary: colors.grey[500] },
    link: { primary: colors.primary[800] },
    system: { error: colors.error[700] },
  },
  background: { base: { white: colors.common.white }, brand: { primaryMedium: colors.primary[500] } },
  fill:   { base: { grayDark2: colors.grey[800] } },
  border: { base: { grayMedium: colors.grey[300] } },

  // ── Tier 3: Component (only because Figma defines explicit tokens for these) ──
  button: {
    solid:   { bg: colors.primary[500], bgHover: colors.primary[800], text: colors.common.white },
    outline: { border: colors.primary[700] },
    ghost:   { text: colors.primary[800] },
  },
  input: { borderFocus: colors.primary[700] },
}
```

### Default-component sync — `src/theme/base.theme.ts`

antd's `ThemeConfig`. **Seed** from `tokens`/`colors`, keep the default algorithm, override Map/Alias
**only** where Figma defines an explicit value antd's derivation misses. This keeps un-styled default
antd components on-brand without fighting the algorithm.

```ts
// src/theme/base.theme.ts
import { theme as antdTheme } from 'antd'
import type { ThemeConfig } from 'antd'
import { colors } from './colors'
import { tokens } from './tokens.theme'

const baseTheme: ThemeConfig = {
  algorithm: antdTheme.defaultAlgorithm, // light. darkAlgorithm = future.
  token: {
    // Seeds — antd derives Map/Alias (states, contrast, borders) from these.
    colorPrimary: colors.primary[500],
    colorError:   colors.error[700],
    colorWarning: colors.warning[500],
    colorSuccess: colors.success[600],
    colorInfo:    colors.info[600],
    colorTextBase: tokens.text.base.primary,
    colorBgBase:   tokens.background.base.white,
    borderRadius: 8,
    fontFamily: 'var(--font-noto-sans-thai)',
    // Pin a Map/Alias token ONLY where Figma is explicit and antd's derivation diverges:
    colorPrimaryHover: colors.primary[800],
  },
}

export default baseTheme
```

> antd's algorithm generates its own 10-step palette from a seed; it will **not** equal our hand-tuned
> `colors.ts` scale. That is expected — exact brand fidelity comes from `customToken`, not from pinning
> every shade here.

### Token usage rules

1. **Priority in code:** component token (Tier 3, if Figma defines it) → semantic token (Tier 2).
2. **NEVER read a primitive (Tier 1) outside `src/theme/`.** `colors.primary[500]` is for building
   `base.theme.ts` / `tokens.theme.ts` only. Authored styling uses `token.<figma.path>`.
3. **NEVER author brand styling against bare antd alias tokens.** `token.colorPrimary` ❌ for a brand
   value → `token.button.solid.bg` / `token.text.base.primary` ✅. (antd alias tokens are fine for
   *structural/layout* values — `token.colorBgLayout`, `token.paddingLG`, `token.borderRadiusLG`.)
4. **Never hardcode hex** in styles (`color: '#945dff'` ❌).
5. **Don't invent component tokens.** Add a Tier 3 token only when Figma defines an explicit
   component-level token; otherwise compose from Tier 2.

```ts
// ❌ never
color: colors.primary[600]          // primitive outside src/theme/
background: token.colorPrimary      // bare antd alias for a brand value
color: '#945dff'                    // hardcoded hex
// ✅ always
background: token.button.solid.bg   // component token (Figma-defined)
color: token.text.base.primary      // semantic token
padding: token.paddingLG            // antd alias — fine for structural values
```

## TypeScript (`src/types/theme.d.ts`)

`src/**` is in `tsconfig.json`'s `include`, so this is picked up automatically. Declare the
`ThemeTokens` interface (same shape as [[add-mui-to-nextjs]]) and merge it into `antd-style`'s
`CustomToken`, so `useTheme().button.solid.bg` and `createStyles`' `token.button.solid.bg` are typed.

```ts
// src/types/theme.d.ts
import 'antd-style'

declare module 'antd-style' {
  interface ThemeTokens {
    text: {
      base: { primary: string; secondary: string }
      link: { primary: string }
      system: { error: string }
    }
    background: { base: { white: string }; brand: { primaryMedium: string } }
    fill: { base: { grayDark2: string } }
    border: { base: { grayMedium: string } }
    button: {
      solid: { bg: string; bgHover: string; text: string }
      outline: { border: string }
      ghost: { text: string }
    }
    input: { borderFocus: string }
  }

  // antd-style merges customToken into the theme; this makes useTheme()/createStyles token typed.
  // eslint-disable-next-line @typescript-eslint/no-empty-object-type
  interface CustomToken extends ThemeTokens {}
}
```

> Keep the `ThemeTokens` interface and the `tokens` object in `tokens.theme.ts` in lock-step — the
> interface is the contract Figma maps onto; the object is its single implementation.

## Theme assembly — `src/theme/index.ts`

Builds the final `ThemeConfig` (base + `components`) and re-exports the token object. Plain module — no
`'use client'` needed (it's config, consumed by the client `AppThemeProvider`).

```ts
// src/theme/index.ts
import type { ThemeConfig } from 'antd'
import baseTheme from './base.theme'
import buttonTheme from './button.theme'
// …other *.theme.ts imports

export const themeConfig: ThemeConfig = {
  ...baseTheme,
  components: {
    Button: buttonTheme,
    // …register every component override here
  },
}

export { tokens } from './tokens.theme'
```

```ts
// src/theme/button.theme.ts — antd component tokens, mapped from our Figma tokens
import type { ThemeConfig } from 'antd'
import { tokens } from './tokens.theme'

const buttonTheme: NonNullable<ThemeConfig['components']>['Button'] = {
  colorPrimary: tokens.button.solid.bg,
  colorPrimaryHover: tokens.button.solid.bgHover,
  primaryColor: tokens.button.solid.text,
  borderRadius: 100, // pill, per design system
}

export default buttonTheme
```

> antd's component-token vocabulary is fixed (`colorPrimary`, `controlHeight`, `paddingInline`, …). When
> Figma needs something the vocabulary can't express, style that component with `createStyles` instead
> of inventing tokens.

## Provider + SSR wiring

`antd-style`'s SSR registry collects cssinjs styles (antd core **and** `createStyles`) and inserts them
via `useServerInsertedHTML`. The `AppThemeProvider` (antd-style `ThemeProvider`) supplies the antd
`themeConfig` and our `customToken`.

```tsx
// src/components/providers/StyleRegistry/StyleRegistry.tsx
'use client'
import { StyleProvider, extractStaticStyle } from 'antd-style'
import { useServerInsertedHTML } from 'next/navigation'
import { useRef, type PropsWithChildren } from 'react'

const StyleRegistry = ({ children }: PropsWithChildren) => {
  const isInsert = useRef(false)
  useServerInsertedHTML(() => {
    if (isInsert.current) return
    isInsert.current = true
    return <>{extractStaticStyle().map((item) => item.style)}</>
  })
  return <StyleProvider cache={extractStaticStyle.cache}>{children}</StyleProvider>
}

export default StyleRegistry
```

```tsx
// src/components/providers/AppThemeProvider/AppThemeProvider.tsx
'use client'
import { ThemeProvider } from 'antd-style'
import type { PropsWithChildren } from 'react'
import { themeConfig, tokens } from '@/theme'

const AppThemeProvider = ({ children }: PropsWithChildren) => (
  <ThemeProvider themeMode="light" theme={themeConfig} customToken={tokens}>
    {children}
  </ThemeProvider>
)

export default AppThemeProvider
```

```tsx
// src/app/layout.tsx (server component)
import { Noto_Sans_Thai } from 'next/font/google'
import { AppThemeProvider, StyleRegistry } from '@/components/providers'

const notoSansThai = Noto_Sans_Thai({
  subsets: ['thai', 'latin'], weight: ['300', '400', '500', '600', '700'],
  display: 'swap', variable: '--font-noto-sans-thai',
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="th" className={notoSansThai.variable}>
      <body>
        <StyleRegistry>
          <AppThemeProvider>{children}</AppThemeProvider>
        </StyleRegistry>
      </body>
    </html>
  )
}
```

## Component conventions

antd primitives (`Button`, `Input`, `Select`, …) are imported **directly from `antd`** and themed
globally — there is **no thin `components/ui` wrapper** over them. The shadcn "route all interactive UI
through primitives" rule maps to: **use the themed antd component; never style raw HTML.**

### Two places styling lives

1. **Global per-component overrides — `src/theme/<component>.theme.ts`** (antd `theme.components`),
   registered in `index.ts`. Fed from `tokens`. This restyles every instance app-wide.

2. **Component-local styles — `<Name>.style.ts`** for composite `components/ui/<Name>/` components
   (folder pattern per [[nextjs]]). A `createStyles` hook reads our tokens (and antd structural tokens)
   off the merged `token`:

   ```ts
   // src/components/ui/ReportModal/ReportModal.style.ts
   import { createStyles } from 'antd-style'

   export const useStyles = createStyles(({ token, css }) => ({
     title: css`
       font-weight: 600;
       font-size: 20px;
       color: ${token.text.base.primary};      /* our semantic token (customToken) */
     `,
     field: css`
       padding: ${token.paddingLG}px;           /* antd structural alias — allowed */
       &:focus-within { border-color: ${token.input.borderFocus}; }  /* our component token */
     `,
   }))
   ```

### Rules

- App code **must not style raw interactive HTML** (`<button>`, `<input>`, …). Use the themed antd
  component.
- Recurring styling for a component → its `*.theme.ts` (`theme.components`). One-off / composite →
  `createStyles` (still consuming `token.<figma.path>`).
- Composite/repeated UI → a `components/ui/<Name>/` component with a `.style.ts`.

## Design system showcase

Each themed component gets a live reference route under
`src/app/(public)/design-system/<component>/page.tsx` showing all variants × sizes × states. When you
theme a new component, add its showcase page. **Exclude design-system routes from the app's home route
index.**

## Rules

- ✅ Do: keep `colors.ts` as the single source of hex; build seeds **and** tokens from it.
- ✅ Do: seed antd + keep its algorithm; pin Map/Alias only where Figma is explicit.
- ✅ Do: author brand styling against `customToken` (`token.<figma.path>`); antd alias tokens only for
  structural/layout values.
- ✅ Do: import antd components directly; theme them in `*.theme.ts`; put composite UI in `components/ui/`.
- ❌ Don't: read primitives (`colors.*`) outside `src/theme/`; don't hardcode hex.
- ❌ Don't: add `@ant-design/nextjs-registry` alongside antd-style (double cache).
- ❌ Don't: add a `tailwind.config.ts` or mix in [[add-shadcn-to-nextjs]] / [[add-mui-to-nextjs]].

## Gotchas

- **SSR FOUC:** without `StyleRegistry` (`extractStaticStyle` + `useServerInsertedHTML`), cssinjs styles
  flash / mismatch on hydration.
- **`'use client'`:** `StyleRegistry` and `AppThemeProvider` are client components; the cache lives in
  the client boundary. `layout.tsx` stays a server component.
- **One registry only:** `antd-style` shares `@ant-design/cssinjs` with antd core, so its registry
  covers both. Do **not** also add `@ant-design/nextjs-registry`.
- **React 19 (Next 15):** if antd static methods (`message`/`notification`/`Modal`) or the wave effect
  warn, add `@ant-design/v5-patch-for-react-19` and import it once at the app entry.
- **Algorithm ≠ brand scale:** antd derives its own shades from seeds; default components won't match
  `colors.ts` exactly. That's by design — use `customToken` for exact fidelity.
- **`customToken` is merged into `token`:** inside `createStyles`/`useTheme`, our nested groups
  (`token.button…`) and antd's flat keys (`token.colorPrimary`) coexist — no collision.

## Related standards

- [[nextjs]] — base structure; this standard fills its deferred CSS-framework boundary for antd.
- [[add-shadcn-to-nextjs]] — owns the shadcn token system; this standard reuses its token names.
- [[add-mui-to-nextjs]] — closest analog: same 3-tier model, `customToken` here playing the role of
  MUI's `theme.tokens.*` (mutually exclusive with this).
- [[listing]] — listing/table pattern; uses these themed components.
