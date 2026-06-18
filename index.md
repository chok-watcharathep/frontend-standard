# Standards Index

Catalog of every standard in this wiki. Grouped by category. `status`: 🟢 stable · 🟡 draft · ⚪ deprecated.
Agents: read this first to locate a standard, then read the page and its `requires:` chain.

## Project structures
Base scaffolds for a whole project type.

| Standard | Status | Summary |
| -------- | ------ | ------- |
| [[nextjs]] | 🟢 stable | Base Next.js (App Router) structure, conventions, data layer (RQ + axios), tooling. CSS-framework-agnostic. |
| [[nextjs-with-payload]] | 🟡 draft | Next.js + embedded Payload CMS. Domain roots (frontend/payload/shared), custom-endpoint data layer, collections, migrations/seeds, logger, env. Overrides [[nextjs]]'s flat src. |

## Integrations
Adding a library/tool onto an existing base.

| Standard | Status | Summary |
| -------- | ------ | ------- |
| [[add-mui-to-nextjs]] | 🟢 stable | Add MUI (+ Emotion) to a Next.js project; owns the MUI token system (`theme.tokens.*`, 3-tier, typed). |
| [[add-shadcn-to-nextjs]] | 🟢 stable | Add shadcn/ui + Tailwind v4 to a Next.js project; owns the shadcn 3-tier design-token system. |
| [[add-antd-to-nextjs]] | 🟢 stable | Add antd v5 (+ antd-style) to a Next.js project; owns the antd token system (`customToken`, 3-tier, typed). |
| [[add-keycloak-auth-to-payload]] | 🟡 draft | Admin auth for Payload via a custom auth strategy (fail-closed verify→introspect→findByID) + HttpOnly-cookie login/refresh/logout/change-password + IdP sync hooks. Keycloak example. |

## Patterns
Reusable feature/implementation recipes.

| Standard | Status | Summary |
| -------- | ------ | ------- |
| [[form]] | 🟡 draft | Form pattern: react-hook-form + zod, interface-first typed schema in an i18n schema hook, Controller wiring, RQ-mutation submit with setError mapping. |
| [[listing]] | 🟡 draft | Standard list/table page with pagination, filtering, and sorting. |
| [[server-data-layer]] | 🟡 draft | Payload feature `server/` taxonomy (handlers/hooks/schemas/services/utils) + the `payload/utils/server.ts` foundations (`defineEndpoint`, `pipeHandlers`, middleware HOCs, typed exceptions + `handleErrorResponse`). Server counterpart to [[nextjs]]'s client data layer. |

## Conventions
Cross-cutting rules applied across many standards.

| Standard | Status | Summary |
| -------- | ------ | ------- |
| [[naming]] | 🟢 stable | Pointer → naming conventions defined in [[nextjs]]. |
| [[code-quality-tooling]] | 🟡 draft | ESLint v9 flat config, Prettier, Husky (pre-commit/commit-msg/pre-push), lint-staged, commit-message & branch-name conventions, and shared `.vscode/`. Applies on top of [[nextjs]]. |
