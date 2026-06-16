---
id: add-mui-to-nextjs
title: Add Material UI to Next.js
category: integration
status: stable
applies_to: [nextjs]
requires: [nextjs]
conflicts_with: [add-shadcn-to-nextjs, add-antd-to-nextjs]
tags: [ui, styling, components, mui, emotion, design-tokens, theme, typescript]
updated: 2026-06-16
---

# Add Material UI to Next.js

## Summary

Adds Material UI (MUI v6 + Emotion) as the UI layer of a [[nextjs]] app, wired for the App Router
(Emotion SSR cache), **and defines the complete design-token system** for MUI projects. Tokens are
handled **per CSS framework** — this standard owns the MUI approach end-to-end. It mirrors the
**3-tier token concept** of [[add-shadcn-to-nextjs]] (primitive → semantic → component), but expressed
in the MUI theme: primitives in `colors.ts`, the authored token layer on a typed **`theme.tokens.*`**
object, consumed via `useTheme()` / `sx` / component `*.theme.ts` overrides. There is no shared,
framework-agnostic token standard.

This is the standard that fills the **CSS-framework boundary** [[nextjs]] intentionally defers for
MUI: `src/theme/` (palette, tokens, per-component overrides), the TypeScript module augmentation in
`src/types/theme.d.ts`, the provider wiring, `components/ui/` composite components, and the
`design-system/` showcase routes.

## When to use

- The app's UI library is **Material UI (MUI)**.

## When not to use

- Using shadcn/ui → [[add-shadcn-to-nextjs]]. Using Ant Design → [[add-antd-to-nextjs]].
  Exactly **one** UI-library integration per app (they `conflicts_with` each other).

## Prerequisites

- [[nextjs]] — the base structure must exist first (flat `src/`, `@/*` → `src/*`, pnpm).

## Stack added

| Concern         | Choice                                                                      |
| --------------- | --------------------------------------------------------------------------- |
| Component model | MUI components used **directly** from `@mui/material`, themed globally       |
| Styling engine  | Emotion (`@emotion/react`, `@emotion/styled`)                               |
| App Router SSR  | `@mui/material-nextjs` → `AppRouterCacheProvider` (Emotion cache)            |
| Theme           | `createTheme` split across `src/theme/*.theme.ts`, assembled in `index.ts`  |
| Tokens          | `theme.tokens.*` (typed object), 3-tier concept (see below)                 |
| Typography      | `next/font/google` Noto Sans Thai → `typography.fontFamily`                 |
| Composite UI    | `src/components/ui/<Name>/` folder + `.style.ts` (per [[nextjs]])           |

## Steps

1. **Install dependencies** (pnpm only, per [[nextjs]]):
   ```bash
   pnpm add @mui/material @emotion/react @emotion/styled @mui/material-nextjs
   ```
   Add `@mui/x-data-grid`, `@mui/x-date-pickers`, etc. only as features require them.

2. **Author Tier 1 primitives** — `src/theme/colors.ts` (the single source of hex; see
   [Design token system](#design-token-system)).

3. **Build `base.theme.ts`** — `palette` / `typography` / `breakpoints` / `shape` / `spacing`,
   all derived from `colors.ts`. The palette is populated **only so default MUI components render
   on-brand** (the "sync"); authored styling never reads from it.

4. **Author the token layer** — `src/theme/tokens.theme.ts` exports the `theme.tokens.*` object
   (Tier 2 semantic + Tier 3 component), every value referencing `colors.ts`.

5. **Add the TypeScript augmentation** — `src/types/theme.d.ts` (see [TypeScript](#typescript-srctypesthemedts)).
   Declares the **global `ThemeTokens` interface** and merges `tokens` onto MUI's `Theme`/`ThemeOptions`
   so `useTheme().tokens` and `sx={(theme) => …}` are fully typed, plus palette/typography/breakpoint/
   variant overrides.

6. **Write per-component overrides** — one `src/theme/<component>.theme.ts` per MUI component you
   restyle (`button.theme.ts`, `alert.theme.ts`, …). Each consumes `theme.tokens.*` — **never**
   palette scales or raw hex.

7. **Assemble the theme** — `src/theme/index.ts` (`'use client'`) calls `createTheme`, attaches
   `tokens`, registers every override under `theme.components`, and loads the font.

8. **Wire the providers** — `AppRouterCacheProvider` in `src/app/layout.tsx` wrapping a client
   `ThemeProvider` (`src/components/providers/ThemeProvider/`).

9. **Add a showcase route** per themed component under `src/app/(public)/design-system/<component>/`.

## Resulting structure

```
src/
  app/
    layout.tsx                     # AppRouterCacheProvider (server) → <ThemeProvider> (client)
    (public)/design-system/        # live component showcase routes (one per themed component)
  components/
    providers/
      ThemeProvider/               # 'use client'; MuiThemeProvider + CssBaseline
      index.ts                     # export { ThemeProvider } ...
    ui/
      <Name>/                      # composite components: <Name>.tsx + <Name>.style.ts + index.ts
  theme/
    colors.ts                      # Tier 1 — primitive color scales (source of hex)
    base.theme.ts                  # palette/typography/breakpoints/shape/spacing (from colors.ts)
    tokens.theme.ts                # theme.tokens.* — Tier 2 semantic + Tier 3 component
    button.theme.ts                # per-component override (one file per MUI component)
    alert.theme.ts
    …                              # chip, dialog, text-field, data-grid, …
    index.ts                       # 'use client'; createTheme + tokens + theme.components + font
  types/
    theme.d.ts                     # module augmentation (global ThemeTokens + palette/variants)
```

> Per [[nextjs]] the `src/` tree is **flat** — `src/theme/`, `src/types/`, `src/components/` — not
> `src/shared/*`. (The internal reference app namespaces under `src/shared/`; that is a property of
> that app, not of this standard.)

## Design token system

**Architecture: typed JS object on the theme.** MUI has no CSS-variable/utility-class layer, so the
tier boundary is enforced by **where values live** plus lint/review — not by syntax. The token layer
is a plain object exposed as `theme.tokens.*` and typed globally via [`ThemeTokens`](#typescript-srctypesthemedts).

**Source-of-truth chain (no drift):** `colors.ts` → (`base.theme.ts` palette **and** `tokens.theme.ts`).
Both the palette and the tokens are built from the **same** `colors.ts`, so the brand colors MUI's
defaults use and the tokens our code uses can never diverge.

### Three tiers

| Tier          | Lives in          | Example                              | Use it…                                                  |
| ------------- | ----------------- | ------------------------------------ | -------------------------------------------------------- |
| 1 — Primitive | `colors.ts`       | `colors.primary[500]`                | **Only inside `src/theme/`** to build palette + tokens   |
| 2 — Semantic  | `tokens.theme.ts` | `theme.tokens.text.base.primary`     | **Default** in overrides, `sx`, `.style.ts`              |
| 3 — Component | `tokens.theme.ts` | `theme.tokens.button.solid.bg`       | **Only when Figma defines an explicit component token**  |

**Token names mirror [[add-shadcn-to-nextjs]]** (and the Figma token paths) so both standards point at
the same design tokens — shadcn `btn/solid/bg` → `--color-btn-solid-bg` and here →
`theme.tokens.button.solid.bg`. Nested groups follow the Figma path.

> **Why not reuse MUI's native semantic slots (`palette.text.primary`, `palette.action.hover`)?**
> Those don't map to Figma token names, so design→code translation is guesswork. We keep the palette
> populated for MUI's *internal* defaults, but **author all styling against `theme.tokens.*`** so the
> token name in code equals the token name in Figma.

### Tier 1 — `src/theme/colors.ts` (primitive scales)

Raw brand/system palette, each family a `50`–`900` scale. **Company-specific values** — the canonical
"HI Design System" palette; copy verbatim. Single source of hex; do not redefine these anywhere else.

```ts
// src/theme/colors.ts — Tier 1 primitives. The ONLY place raw hex lives.
export const colors = {
  primary: {
    50: '#f5f5ff', 100: '#ede8ff', 200: '#dcd4ff', 300: '#c5b1ff', 400: '#a885ff',
    500: '#945dff', 600: '#8030f7', 700: '#721ee3', 800: '#5f18bf', 900: '#4f169c',
  },
  secondary: { /* 50–900 (cream) */ 500: '#f3f1ed' },
  tertiary:  { /* 50–900 (teal)  */ 500: '#3c919e' },
  grey: {
    50: '#fafafa', 100: '#f5f4f5', 200: '#eaeaea', 300: '#deddde', 400: '#a5a2a9',
    500: '#767279', 600: '#56535a', 700: '#424045', 800: '#242325', 900: '#19181b',
  },
  error:   { /* 50–900 */ 500: '#ef4444', 700: '#b91c1c' },
  warning: { /* 50–900 */ 500: '#f59e0b' },
  success: { /* 50–900 */ 500: '#10b981' },
  info:    { /* 50–900 */ 500: '#0ea5e9' },
  // accent families (each 50–900): red, orange, yellow, lime, green, teal, sky, blue,
  // indigo, violet, purple, pink
  accent: { red: { /* 50–900 */ }, /* … */ },
  common: { white: '#ffffff', black: '#000000' },
} as const
```

### Tier 1 → palette (the "sync") — `src/theme/base.theme.ts`

Build the MUI palette **from `colors.ts`**. Each color carries its numbered scale **and** the
`main`/`light`/`dark` slots MUI internals need. This is what keeps un-overridden default components
on-brand.

```ts
// src/theme/base.theme.ts
import { createTheme } from '@mui/material'
import { colors } from './colors'

const baseTheme = createTheme({
  breakpoints: {
    // Custom-only. Defaults (xs..xl) are disabled in theme.d.ts. Aligns with [[add-shadcn-to-nextjs]]
    // containers: compact 912 / desktop 1180 / wide 1552.
    values: { mobile: 0, tablet: 768, compact: 912, desktop: 1180, wide: 1552 },
  },
  palette: {
    mode: 'light',
    common: colors.common,
    primary:   { ...colors.primary,   main: colors.primary[500],  light: colors.primary[100],  dark: colors.primary[700] },
    secondary: { ...colors.secondary, main: colors.secondary[500] },
    tertiary:  { ...colors.tertiary,  main: colors.tertiary[600] },
    error:     { ...colors.error,     main: colors.error[700],    light: colors.error[100] },
    success:   { ...colors.success,   main: colors.success[600] },
    warning:   { ...colors.warning,   light: colors.warning[100] },
    info:      { ...colors.info,      main: colors.info[600] },
    grey: colors.grey,
    accent: colors.accent,
    text:       { primary: colors.grey[900], secondary: colors.grey[500] },
    background: { default: colors.primary[50], paper: colors.common.white },
  },
  shape: { borderRadius: 8 },
  spacing: 8,
})

export default baseTheme
```

### Tier 2 + Tier 3 — `src/theme/tokens.theme.ts`

The authored layer. **Every value references `colors.*`** — never raw hex (except intentional
absolutes like white/black/overlays). Shape matches the `ThemeTokens` interface.

```ts
// src/theme/tokens.theme.ts — Tier 2 semantic + Tier 3 component. Built from colors.ts.
import { colors } from './colors'
import type { ThemeTokens } from '@mui/material/styles'

export const tokens: ThemeTokens = {
  // ── Tier 2: Semantic ──
  text: {
    base: { primary: colors.grey[900], secondary: colors.grey[500] },
    link: { primary: colors.primary[800] },
    system: { error: colors.error[700] },
  },
  background: {
    base: { white: colors.common.white },
    brand: { primaryMedium: colors.primary[500] },
  },
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

### Token usage rules

1. **Priority in code:** component token (Tier 3, if Figma defines it) → semantic token (Tier 2).
2. **NEVER read a primitive (Tier 1) in app/theme code.** `colors.primary[500]` and `palette.primary[500]`
   are for building `base.theme.ts` / `tokens.theme.ts` only. Everywhere else use `theme.tokens.*`.
3. **NEVER read MUI palette slots for authored styling** — `theme.palette.text.primary` ❌ →
   `theme.tokens.text.base.primary` ✅. (Palette exists only for MUI's own default components.)
4. **Never hardcode hex** in overrides/styles (`color: '#945dff'` ❌, `'rgba(255,241,178,1)'` ❌).
5. **Don't invent component tokens.** Add a Tier 3 token only when Figma defines an explicit
   component-level token; otherwise compose from Tier 2. Avoids token bloat / Figma drift.

```ts
// ❌ never
color: colors.primary[600]               // primitive
color: theme.palette.primary.main        // MUI slot for authored styling
backgroundColor: '#945dff'               // hardcoded hex
// ✅ always
color: theme.tokens.button.ghost.text    // component token (Figma-defined)
color: theme.tokens.text.base.primary    // semantic token
```

## TypeScript (`src/types/theme.d.ts`)

All MUI module augmentation lives here. `src/**` is in `tsconfig.json`'s `include`, so the file is
picked up automatically. The **`ThemeTokens` interface is declared globally** and merged onto MUI's
`Theme`/`ThemeOptions`, so `useTheme().tokens`, `sx={(theme) => …}`, and `styled` are all typed.

```ts
// src/types/theme.d.ts
import '@mui/material/styles'
import '@mui/material/Button'

declare module '@mui/material/styles' {
  // ── The authored token layer (global) ──
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
  interface Theme { tokens: ThemeTokens }
  interface ThemeOptions { tokens?: ThemeTokens }

  // ── Palette extensions (numbered scales + custom families) ──
  interface PaletteColor { 50?: string; 100?: string; /* …200–800… */ 900?: string }
  interface PaletteColorOptions { 50?: string; 100?: string; /* … */ 900?: string }
  interface Palette {
    tertiary: Palette['primary']
    accent: Record<'red' | 'orange' | 'yellow' | 'lime' | 'green' | 'teal' | 'sky' | 'blue'
      | 'indigo' | 'violet' | 'purple' | 'pink', Record<50 | 100 | 200 | 300 | 400 | 500 | 600 | 700 | 800 | 900, string>>
    border: Palette['primary'] & { primary: { light?: string }; grey: { light?: string; medium?: string } }
    shadow: { main: string; midnight: string }
  }
  interface PaletteOptions { tertiary?: PaletteOptions['primary']; accent?: Palette['accent']; /* border?, shadow? */ }
  interface TypeBackground { primary: string; light: string }
  interface TypeText { placeholder: string; dark: string }

  // ── Custom typography variants ──
  interface TypographyVariantsOptions { body?: React.CSSProperties; displayS?: React.CSSProperties; displayM?: React.CSSProperties }
}

// ── Custom breakpoints: disable defaults, add the design-system set ──
declare module '@mui/material/styles' {
  interface BreakpointOverrides {
    xs: false; sm: false; md: false; lg: false; xl: false
    mobile: true; tablet: true; compact: true; desktop: true; wide: true
  }
}

declare module '@mui/material/Button' {
  interface ButtonPropsVariantOverrides { ghost: true; textDark: true; ghostDark: true; outlinedDark: true; containedDark: true }
  interface ButtonPropsSizeOverrides { extraSmall: true }
}
declare module '@mui/material/Chip' {
  interface ChipPropsSizeOverrides { extraSmall: true }
  interface ChipPropsColorOverrides { errorLight: true; warningLight: true; infoLight: true }
}
declare module '@mui/material/Typography' {
  interface TypographyPropsVariantOverrides { body: true; displayS: true; displayM: true }
}
```

> Keep the `ThemeTokens` interface and the `tokens` object in `tokens.theme.ts` **in lock-step** —
> the interface is the contract Figma maps onto; the object is its single implementation.

## Theme assembly — `src/theme/index.ts`

`'use client'` (Emotion runs on the client). Calls `createTheme`, attaches `tokens`, and registers
every per-component override under `theme.components`.

```ts
// src/theme/index.ts
'use client'
import { createTheme } from '@mui/material'
import { Noto_Sans_Thai } from 'next/font/google'
import baseTheme from './base.theme'
import { tokens } from './tokens.theme'
import buttonTheme from './button.theme'
import alertTheme from './alert.theme'
// …other *.theme.ts imports

const notoSansThai = Noto_Sans_Thai({
  subsets: ['thai', 'latin'], weight: ['300', '400', '500', '600', '700'],
  display: 'swap', variable: '--font-noto-sans-thai',
})

const theme = createTheme({
  ...baseTheme,
  tokens,
  typography: {
    fontFamily: notoSansThai.style.fontFamily,
    // h1..h6, subtitle, body/body1/body2, displayS, displayM … (responsive via breakpoints)
  },
})

theme.components = {
  MuiCssBaseline: { styleOverrides: { body: { WebkitFontSmoothing: 'antialiased' } } },
  MuiButton: buttonTheme,
  MuiAlert: alertTheme,
  // …register every override here
}

export default theme
```

## Provider wiring

`AppRouterCacheProvider` (server, from `@mui/material-nextjs`) wraps a **client** `ThemeProvider`.
Without the cache provider, Emotion styles flicker / mismatch on hydration.

```tsx
// src/app/layout.tsx (server component)
import { AppRouterCacheProvider } from '@mui/material-nextjs/v15-appRouter'
import { ThemeProvider } from '@/components/providers'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="th">
      <body>
        <AppRouterCacheProvider>
          <ThemeProvider>{children}</ThemeProvider>
        </AppRouterCacheProvider>
      </body>
    </html>
  )
}
```

```tsx
// src/components/providers/ThemeProvider/ThemeProvider.tsx
'use client'
import { CssBaseline, ThemeProvider as MuiThemeProvider } from '@mui/material'
import theme from '@/theme'

const ThemeProvider = ({ children }: { children: React.ReactNode }) => (
  <MuiThemeProvider theme={theme}>
    <CssBaseline />
    {children}
  </MuiThemeProvider>
)

export default ThemeProvider
```

## Component conventions

MUI primitives (`Button`, `TextField`, `Chip`, …) are imported **directly from `@mui/material`** and
themed globally — there is **no thin `components/ui` wrapper** over them (the opposite of shadcn,
where primitives are copied in). The shadcn "route all interactive UI through primitives" rule maps to:
**use the themed MUI component; never style raw HTML.**

### Two places styling lives

1. **Global per-component overrides — `src/theme/<component>.theme.ts`.** One file per MUI component,
   registered in `index.ts`. This is where variants/sizes/states are defined for the whole app.
   Consume `theme.tokens.*`:

   ```ts
   // src/theme/alert.theme.ts  — improved: tokens, not palette scales
   import type { Components, Theme } from '@mui/material'

   const alertTheme: Components<Theme>['MuiAlert'] = {
     styleOverrides: {
       standardError: ({ theme }) => ({
         backgroundColor: theme.tokens.background.base.white,
         '& .MuiAlert-icon':    { color: theme.tokens.text.system.error },
         '& .MuiAlert-message': { color: theme.tokens.text.system.error },
       }),
     },
   }
   export default alertTheme
   ```

   > **Before/after:** the reference app wrote `color: palette.error[700]` (a Tier 1 scale) inline.
   > The standard requires the callback form `({ theme }) => ({ color: theme.tokens.text.system.error })`
   > so the value is a Figma-mapped token, not a primitive.

   Custom **Button** variants (`ghost`, `textDark`, `ghostDark`, `outlinedDark`, `containedDark`) and
   sizes (`extraSmall`), and custom **Typography** variants (`body`, `displayS`, `displayM`) are
   declared in `theme.d.ts` and implemented in their `*.theme.ts`.

2. **Component-local styles — `<Name>.style.ts`.** For composite `components/ui/<Name>/` components
   (folder pattern per [[nextjs]]). A `useStyles` hook reads tokens via `useTheme()` and returns
   `SxProps`:

   ```ts
   // src/components/ui/ReportModal/ReportModal.style.ts
   import { useTheme, type SxProps } from '@mui/material'

   export const useStyles = () => {
     const theme = useTheme()
     const titleStyle: SxProps = { fontWeight: 600, fontSize: 20, color: theme.tokens.text.base.primary }
     const fieldStyle: SxProps = { '&:focus-within': { borderColor: theme.tokens.input.borderFocus } }
     return { titleStyle, fieldStyle }
   }
   ```

### Rules

- App code (`src/features/**`, `src/app/**`, `src/components/layouts/**`) **must not style raw
  interactive HTML** (`<button>`, `<input>`, …). Use the themed `@mui/material` component.
- Recurring styling for a primitive → add a **variant** in its `*.theme.ts` and use it everywhere.
  One-off → `sx` on the MUI component (still consuming `theme.tokens.*`).
- Composite/repeated UI → a `components/ui/<Name>/` component with a `.style.ts`.

## Design system showcase

Each themed component gets a live reference route under
`src/app/(public)/design-system/<component>/page.tsx` showing all variants × sizes × states. When you
theme a new component, add its showcase page. **Exclude design-system routes from the app's home route
index.**

## Rules

- ✅ Do: keep `colors.ts` as the single source of hex; build palette **and** tokens from it.
- ✅ Do: author all styling against `theme.tokens.*`; use semantic (Tier 2) by default, component
  (Tier 3) only when Figma defines it.
- ✅ Do: declare `ThemeTokens` globally so `useTheme().tokens` / `sx` are typed.
- ✅ Do: import MUI primitives directly; theme them in `*.theme.ts`; put composite UI in `components/ui/`.
- ❌ Don't: read primitives (`colors.*`) or MUI palette slots in authored styling; don't hardcode hex.
- ❌ Don't: wrap MUI primitives in a `components/ui` re-export layer (that's the shadcn model).
- ❌ Don't: add a `tailwind.config.ts` or mix in [[add-shadcn-to-nextjs]] / [[add-antd-to-nextjs]].

## Gotchas

- **Emotion SSR:** App Router needs `AppRouterCacheProvider` or styles flicker / mismatch on hydration.
- **`'use client'`:** `theme/index.ts` and `ThemeProvider` must be client — `next/font` + Emotion run
  client-side. The cache provider stays in the server `layout.tsx`.
- **Override callback form:** to reach tokens inside `*.theme.ts`, use the
  `({ theme }) => ({ … })` style override (typed via `Components<Theme>`), not a bare object —
  a bare object has no `theme` in scope, which is what pushed the reference app to inline primitives.
- **Breakpoints:** defaults (`xs..xl`) are turned off in `theme.d.ts`; only `mobile/tablet/compact/
  desktop/wide` exist. Using `sx={{ p: { sm: 2 } }}` will not type-check — use the custom keys.
- **Palette is not the token layer:** it's the MUI-internal mirror. Editing palette won't update
  `theme.tokens.*` and vice-versa — both derive from `colors.ts`, so change colors there.

## Related standards

- [[nextjs]] — base structure; this standard fills its deferred CSS-framework boundary for MUI.
- [[add-shadcn-to-nextjs]] — the shadcn UI + token layer; this standard mirrors its 3-tier token
  concept and reuses its token names (mutually exclusive with this).
- [[add-antd-to-nextjs]] — alternative UI library; handles tokens its own way (mutually exclusive).
- [[listing]] — listing/table pattern; uses these themed components.
