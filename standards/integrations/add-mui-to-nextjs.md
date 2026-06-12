---
id: add-mui-to-nextjs
title: Add Material UI to Next.js
category: integration
status: draft
applies_to: [nextjs]
requires: [nextjs]
conflicts_with: [add-shadcn-to-nextjs, add-antd-to-nextjs]
tags: [ui, styling, components, mui]
updated: 2026-06-12
---

# Add Material UI to Next.js

## Summary

Add Material UI (MUI) to a Next.js project as the component layer, wired up for the App Router
(Emotion SSR cache). **This standard owns MUI's own token approach** — design tokens are handled
**per CSS framework** (there is no shared design-token standard). MUI tokens live in the theme
object (`createTheme({ palette, typography, shape, ... })`), consumed via `sx`/`styled`/`useTheme` —
**not** the CSS-variable + Tailwind `@theme` approach used by [[add-shadcn-to-nextjs]].

## When to use

- The project should use MUI as its UI library.

## When not to use

- Using shadcn/ui → [[add-shadcn-to-nextjs]]. Using Ant Design → [[add-antd-to-nextjs]].
  Pick one UI library per app.

## Prerequisites

- [[nextjs]] — base project must exist.

## Steps

1. Install MUI and the Next.js integration (pnpm only, per [[nextjs]]):
   ```bash
   pnpm add @mui/material @emotion/react @emotion/styled @mui/material-nextjs
   ```
2. Set up the App Router cache provider in the root layout.
   > TODO: paste the company's standard `AppRouterCacheProvider` + `ThemeProvider` setup and theme location.
3. Define the MUI **design tokens** in the theme object.
   > TODO: specify the company `createTheme` palette/typography/shape tokens (MUI's per-framework
   > token approach — mirror the same brand palette as [[add-shadcn-to-nextjs]], expressed as MUI theme values).

## Resulting structure

```
src/
  app/
    layout.tsx          # wraps children in AppRouterCacheProvider + ThemeProvider
  theme/
    theme.ts            # MUI theme definition
```

> TODO: confirm theme file location and tokens.

## Rules

- ✅ Do: centralize the theme; consume it via the `sx` prop / `styled`.
- ❌ Don't: mix in another UI library (`conflicts_with`).

## Gotchas

- App Router needs the Emotion SSR cache provider or styles flicker / mismatch on hydration.

## Related standards

- [[nextjs]] — base.
- [[add-shadcn-to-nextjs]], [[add-antd-to-nextjs]] — alternative UI libraries (mutually exclusive).
