# Standards Index

Catalog of every standard in this wiki. Grouped by category. `status`: 🟢 stable · 🟡 draft · ⚪ deprecated.
Agents: read this first to locate a standard, then read the page and its `requires:` chain.

## Project structures
Base scaffolds for a whole project type.

| Standard | Status | Summary |
| -------- | ------ | ------- |
| [[nextjs]] | 🟢 stable | Base Next.js (App Router) structure, conventions, data layer (RQ + axios), tooling. CSS-framework-agnostic. |
| [[nextjs-with-payload]] | 🟡 draft | Next.js project with Payload CMS integrated in the same app. |

## Integrations
Adding a library/tool onto an existing base.

| Standard | Status | Summary |
| -------- | ------ | ------- |
| [[add-mui-to-nextjs]] | 🟢 stable | Add MUI (+ Emotion) to a Next.js project; owns the MUI token system (`theme.tokens.*`, 3-tier, typed). |
| [[add-shadcn-to-nextjs]] | 🟢 stable | Add shadcn/ui + Tailwind v4 to a Next.js project; owns the shadcn 3-tier design-token system. |
| [[add-antd-to-nextjs]] | 🟡 draft | Add Ant Design to a Next.js project (owns its own antd token approach). |

## Patterns
Reusable feature/implementation recipes.

| Standard | Status | Summary |
| -------- | ------ | ------- |
| [[listing]] | 🟡 draft | Standard list/table page with pagination, filtering, and sorting. |

## Conventions
Cross-cutting rules applied across many standards.

| Standard | Status | Summary |
| -------- | ------ | ------- |
| [[naming]] | 🟢 stable | Pointer → naming conventions defined in [[nextjs]]. |
