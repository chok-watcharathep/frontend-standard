---
id: add-antd-to-nextjs
title: Add Ant Design to Next.js
category: integration
status: draft
applies_to: [nextjs]
requires: [nextjs]
conflicts_with: [add-mui-to-nextjs, add-shadcn-to-nextjs]
tags: [ui, styling, components, antd]
updated: 2026-06-12
---

# Add Ant Design to Next.js

## Summary

Add Ant Design (antd) to a Next.js project as the component layer, wired up for the App Router
(SSR style registry). **This standard owns antd's own token approach** — design tokens are handled
**per CSS framework** (there is no shared design-token standard). antd tokens are configured through
`ConfigProvider`'s `theme.token` / `theme.components` (and design-token algorithms) — **not** the
CSS-variable + Tailwind `@theme` approach used by [[add-shadcn-to-nextjs]].

## When to use

- The project should use Ant Design as its UI library.

## When not to use

- Using shadcn/ui → [[add-shadcn-to-nextjs]]. Using MUI → [[add-mui-to-nextjs]].
  Pick one UI library per app.

## Prerequisites

- [[nextjs]] — base project must exist.

## Steps

1. Install antd and the Next.js registry helper (pnpm only, per [[nextjs]]):
   ```bash
   pnpm add antd @ant-design/nextjs-registry
   ```
2. Wrap the app with `AntdRegistry` in the root layout and configure the theme via `ConfigProvider`.
   > TODO: paste the company's standard `AntdRegistry` + `ConfigProvider` theme setup.
3. Define the antd **design tokens** via `ConfigProvider` `theme.token` / `theme.components`.
   > TODO: specify the company token set (antd's per-framework token approach — mirror the same brand
   > palette as [[add-shadcn-to-nextjs]], expressed as antd theme tokens).

## Resulting structure

```
src/
  app/
    layout.tsx          # wraps children in AntdRegistry + ConfigProvider
  theme/
    antd-theme.ts       # antd theme tokens
```

> TODO: confirm theme file location and tokens.

## Rules

- ✅ Do: configure theme tokens via `ConfigProvider`, not per-component overrides.
- ❌ Don't: mix in another UI library (`conflicts_with`).

## Gotchas

- App Router needs `@ant-design/nextjs-registry` or styles are missing/flicker on SSR.

## Related standards

- [[nextjs]] — base.
- [[add-mui-to-nextjs]], [[add-shadcn-to-nextjs]] — alternative UI libraries (mutually exclusive).
