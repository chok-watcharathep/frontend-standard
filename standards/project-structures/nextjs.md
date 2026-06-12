---
id: nextjs
title: Next.js Project Structure
category: project-structure
status: stable
applies_to: [nextjs]
requires: []
conflicts_with: []
tags: [nextjs, react, app-router, react-query, axios, scaffold, conventions]
updated: 2026-06-12
---

# Next.js Project Structure

## Summary

The base structure and conventions for every Next.js (App Router) project at the company. This
standard is **CSS-framework-agnostic** — everything styling-related is added by a separate
integration standard ([[add-shadcn-to-nextjs]] / [[add-mui-to-nextjs]] / [[add-antd-to-nextjs]]).
Follow this exactly when scaffolding or extending a Next.js app.

## When to use

- Starting any new Next.js web app.
- Adding a feature, route, service, or component to an existing one — match these conventions.

## When not to use

- The app embeds Payload CMS in the same codebase → start from [[nextjs-with-payload]] (it
  builds on this standard).

## Stack baseline

| Concern        | Choice                                                              |
| -------------- | ------------------------------------------------------------------- |
| Framework      | Next.js 16, App Router                                               |
| UI runtime     | React 19                                                            |
| Language       | TypeScript, `strict: true`, **no `any`**                            |
| Package manager| **pnpm only**                                                       |
| Data fetching  | `@tanstack/react-query` (hooks) + `axios` (services)                |
| Forms          | `react-hook-form` + `zod`                                           |
| Testing        | Vitest, colocated `*.spec.ts`                                       |
| Path alias     | `@/*` → `src/*`                                                     |
| CSS framework  | **Not decided here** — see the integration standards.               |

## Top-level structure

`src/` is flat (no domain namespacing).

```
src/
  app/                 # App Router. (public)/(protected) groups → nested layout groups. page.tsx is THIN.
  components/
    ui/                # primitive components — folder declared here, contents owned by the CSS-framework standard
    layouts/           # shared layout components (TheMainLayout, Navbar, Footer, ...)
    icons/             # icon components, exported via index.ts
    providers/         # app-wide React providers (incl. QueryClientProvider, wired into app/layout.tsx)
  features/            # feature modules (see "Feature module pattern")
  hooks/               # global React hooks
  utils/               # global utilities (generate-path.util.ts, match-route.util.ts, ...)
  libs/                # third-party client configs — axios.lib.ts (axiosInstance)
  types/               # global TypeScript types
  enums/               # global enums — route.enum.ts (the Route enum)
```

> Every component group, feature subfolder, `hooks/`, and `utils/` **must** have an `index.ts`
> barrel. Import from the barrel, never from deep paths. **Features have no root barrel** — import
> directly from the subfolder (the folder type), not the feature:
> ```ts
> import { Button } from '@/components/ui'               // ✅
> import { Button } from '@/components/ui/button'        // ❌
> import { ProductCard } from '@/features/product/components' // ✅ (subfolder barrel)
> import { ProductCard } from '@/features/product'            // ❌ (no feature-root barrel)
> ```

## Routing (`app/`)

- **Group by access control, then by shared layout**, using route groups:
  ```
  app/
    (public)/(main)/...        (public)/(slides)/...
    (protected)/(main)/...
    layout.tsx                 # root layout: fonts, providers, <html>/<body>
  ```
- **`page.tsx` is thin** — it renders a feature page and (optionally) sets metadata:
  ```tsx
  // app/(protected)/(main)/products/page.tsx
  import { ProductListPage } from '@/features/product/pages'

  export default function Page() {
    return <ProductListPage />
  }
  ```
- **Route registry is required.** Define all routes in `src/enums/route.enum.ts` and build URLs
  with the typed `generatePath` util (never hand-concatenate paths):
  ```ts
  // src/enums/route.enum.ts
  export enum Route {
    HOME = '/',
    PRODUCT = '/products',
    PRODUCT_DETAIL = '/products/:id',
  }

  // usage
  import { generatePath } from '@/utils'
  import { Route } from '@/enums'
  generatePath(Route.PRODUCT_DETAIL, { id: '42' }) // "/products/42"
  ```
  `generatePath` and `matchRoute` live in `src/utils/`.

## Feature module pattern

```
features/{feature}/
  components/   # feature UI components            (REQUIRED)
  pages/        # page-composition components rendered by app/.../page.tsx
  hooks/        # React Query hooks + other feature hooks
  services/     # axios API calls (+ *.spec.ts, __mocks__/)
  transforms/   # pure API-response → UI/form mappers (+ *.spec.ts)
  interfaces/   # TS interfaces/types ONLY — no runtime values
  enums/        # enum declarations ONLY (incl. {Entity}QueryKey)
  constants/    # runtime `const` values ONLY
  utils/        # feature-local pure utilities
  providers/    # feature-scoped React providers
  (no root index.ts — import from the subfolder barrel directly)
```

**Rules:**
- Only `components/` is mandatory. **Create the other subfolders on demand.** Each subfolder that
  exists has its own `index.ts` barrel.
- **Strict separation** — never mix: `interfaces/` = types only · `enums/` = enums only ·
  `constants/` = runtime values only.
- **No feature-root barrel.** Import directly from the subfolder you need (reference the folder
  type, not the feature):
  ```ts
  import { ProductCard } from '@/features/product/components'
  import { useGetProductList } from '@/features/product/hooks'
  import type { Product } from '@/features/product/interfaces'
  ```

### Data layer: service → hook → transform

**Service** (`services/product.service.ts`) — named async functions, typed with
`Request`/`Response` interfaces, errors normalized via `isAxiosError`:
```ts
import { isAxiosError } from 'axios'
import { axiosInstance } from '@/libs'
import type { GetProductListResponse } from '@/features/product/interfaces'

const PRODUCT_URL = '/products'

export const getProductList = async (): Promise<GetProductListResponse> => {
  try {
    const { data } = await axiosInstance.get<GetProductListResponse>(PRODUCT_URL)
    return data
  } catch (error) {
    if (isAxiosError(error)) throw error.response?.data
    throw error
  }
}
```

**Hook** (`hooks/useGetProductList.ts`) — React Query, **default export**, `queryKey` from the
feature's `QueryKey` enum, `queryFn` calls the service:
```ts
import { useQuery, type UseQueryOptions } from '@tanstack/react-query'
import { ProductQueryKey } from '@/features/product/enums'
import { getProductList } from '@/features/product/services'
import type { GetProductListResponse } from '@/features/product/interfaces'

const useGetProductList = (
  options?: Omit<UseQueryOptions<GetProductListResponse, Error>, 'queryKey' | 'queryFn'>,
) =>
  useQuery<GetProductListResponse, Error>({
    queryKey: [ProductQueryKey.GET_PRODUCT_LIST],
    queryFn: () => getProductList(),
    ...options,
  })

export default useGetProductList
```

**Transform** (`transforms/product.transforms.ts`) — pure functions mapping API responses to
UI/form shapes, named `transform{X}To{Y}`. No side effects.

`axiosInstance` is configured once in `src/libs/axios.lib.ts`.

## Component convention

```
ComponentName/
  ComponentName.tsx     # the component — DEFAULT export
  index.ts              # export { default } from './ComponentName'
  ...                   # styling file (.style/.variants) is owned by the CSS-framework standard
```

- Components **default-export**. The component's own `index.ts` re-exports the default as default:
  ```ts
  export { default } from './ComponentName'
  ```
- Aggregate barrels name the defaults:
  ```ts
  // features/product/components/index.ts
  export { default as ProductCard } from './ProductCard'
  export { default as ProductFilters } from './ProductFilters'
  ```
- **Exception — shadcn `components/ui/`:** flat files (`button.tsx`) re-exported with `export *`,
  to follow shadcn's generator. This applies only when using [[add-shadcn-to-nextjs]]; other CSS
  frameworks use the folder + default-export convention above for `ui/` too.
- **Props order:** `key`/`ref` → `id`/`className` → data props → config props (`variant`, `size`,
  `disabled`) → content props (`children`, `label`) → event handlers (`onClick`, `onChange`) last.
- Prefer Server Components; add `'use client'` only for interactivity.

## Naming conventions

| Thing                 | Pattern / suffix              | Example                       |
| --------------------- | ----------------------------- | ----------------------------- |
| Component file/folder | PascalCase                    | `ProductCard/ProductCard.tsx` |
| Hook                  | `use{Op}{Entity}.ts`          | `useGetProductList.ts`        |
| Service file          | `{name}.service.ts`           | `product.service.ts`          |
| Service function      | `{op}{Entity}`                | `getProductList`              |
| Transform file        | `{name}.transforms.ts`        | `product.transforms.ts`       |
| Interface             | `{Entity}` / `{Op}{Entity}Request`/`Response` | `Product`, `GetProductListResponse` |
| Enum file             | `{name}.enum.ts`              | `route.enum.ts`               |
| Query-key enum        | `{Entity}QueryKey`            | `ProductQueryKey`             |
| Constants file        | `{name}.constants.ts`         | `products.constants.ts`       |
| Runtime const         | camelCase                     | `navItems`                    |
| Utility file          | `{name}.util.ts`              | `generate-path.util.ts`       |
| Lib config file       | `{name}.lib.ts`               | `axios.lib.ts`                |
| Test file             | `{name}.spec.ts`              | `product.service.spec.ts`     |
| Mock file             | `{name}.mock.ts`              | `product.mock.ts`             |

## Testing

- **Vitest.** Test files are `*.spec.ts`, **colocated** next to the source (especially services
  and transforms).
- Mocks live in a sibling `__mocks__/` folder with its own `index.ts` barrel:
  ```
  services/
    product.service.ts
    product.service.spec.ts
    __mocks__/
      product.mock.ts
      index.ts
  ```

## Tooling & config

- **pnpm only.**
- **Prettier:** single quotes, no semicolons, trailing comma `all`, `printWidth: 100`.
  Format-on-save enabled (Prettier default formatter + ESLint `fixAll`).
- **ESLint:** `eslint-config-next`. No `any`.
- **Husky:**
  | Hook         | Runs                                                       |
  | ------------ | ---------------------------------------------------------- |
  | `pre-commit` | `lint-staged` → ESLint `--fix` + Prettier on staged files  |
  | `commit-msg` | Enforces `<Type>: <message>` (PascalCase type)             |
  | `pre-push`   | `pnpm type-check`                                          |
- **Commit types** (case-sensitive): `Fix` · `Feat` · `Docs` · `Style` · `Refactor` · `Perf` ·
  `Test` · `Chore`. Example: `Feat: add product listing`. Merge commits allowed.
- **Scripts:** `dev` · `build` · `start` · `lint` · `lint:fix` · `format` · `format:check` ·
  `type-check`.

## CSS-framework boundary (deferred)

This standard intentionally does **not** define any of the following — they are owned by the
chosen CSS-framework integration standard:

- Contents of `components/ui/` (primitive components).
- `src/theme/` design tokens.
- `src/libs/utils.lib.ts` (`cn()` helper).
- Per-component styling files (`.variants.ts` / cva, `.style.ts`, etc.).
- `components.json`, PostCSS/Tailwind config, `globals.css`.
- `design-system/` showcase routes.

## Rules

- ✅ Do: keep `page.tsx` thin; put page composition in `features/{feature}/pages/`.
- ✅ Do: route through the `Route` enum + `generatePath`; never hardcode URL strings.
- ✅ Do: barrel-export everything; import from barrels.
- ✅ Do: default-export components; name them at the aggregate barrel.
- ❌ Don't: put business logic in route files or mix types/enums/constants folders.
- ❌ Don't: introduce any styling decisions here — defer to the CSS-framework standard.
- ❌ Don't: use a package manager other than pnpm, or use `any`.

## Gotchas

- `components/ui/` exists in this standard but is **empty** until a CSS-framework standard is
  applied. Don't author primitives here ad hoc.
- shadcn's `ui/` (`export *`, flat files) is the **only** exception to the default-export +
  folder component convention.
- Keep `Route` enum values and search-param names stable — bookmarked URLs depend on them.

## Related standards

- [[nextjs-with-payload]] — same base, plus embedded Payload CMS.
- [[add-shadcn-to-nextjs]] · [[add-mui-to-nextjs]] · [[add-antd-to-nextjs]] — choose exactly one;
  each fills the CSS-framework boundary above.
- [[listing]] — listing/table page pattern built on this base.
- [[naming]] — pointer; naming is defined in this standard.
