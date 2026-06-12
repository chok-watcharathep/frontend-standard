# Change Log

Append-only. Newest at the bottom. Each entry: `## [YYYY-MM-DD] <op> | <id> — <note>`
where `<op>` ∈ {add, update, deprecate, lint}. `grep "^## \[" log.md | tail` shows recent activity.

## [2026-06-12] add | wiki — Initialized standards wiki: schema (CLAUDE.md), index, log, template.
## [2026-06-12] add | nextjs — Seed page (skeleton, status draft).
## [2026-06-12] add | nextjs-with-payload — Seed page (skeleton, status draft).
## [2026-06-12] add | add-mui-to-nextjs — Seed page (skeleton, status draft).
## [2026-06-12] add | add-shadcn-to-nextjs — Seed page (skeleton, status draft).
## [2026-06-12] add | add-antd-to-nextjs — Seed page (skeleton, status draft).
## [2026-06-12] add | listing — Seed page (skeleton, status draft).
## [2026-06-12] add | naming — Seed page (skeleton, status draft).
## [2026-06-12] update | nextjs — Authored full standard from ai-driven-experience (structure) + example-payload (data layer). Decided via grill session: flat src/, (public)/(protected) route groups, thin page.tsx → feature pages/, full feature subfolder vocab (on-demand), default-export components + barrel rules, RQ+axios data layer (service→hook→transform), vitest .spec.ts + __mocks__, tooling baked in. CSS concerns deferred. status → stable.
## [2026-06-12] update | naming — Converted to thin pointer to [[nextjs]]. status → stable.
## [2026-06-12] update | nextjs — Removed feature-root barrel rule. Features have NO root index.ts; import directly from the subfolder barrel (@/features/x/components, /hooks, /interfaces) — reference the folder type, not the feature.
## [2026-06-12] update | add-shadcn-to-nextjs — Authored full standard from ai-driven-experience. Owns the entire shadcn UI + token layer (the CSS-framework boundary nextjs defers): Tailwind v4 CSS-first (no tailwind.config), components.json (new-york, baseColor stone, utils → @/libs/utils.lib), cn() in libs/utils.lib.ts, postcss, hand-written 3-tier design tokens in src/theme (colors.css primitives + tokens.css semantic/component via @theme), globals.css shadcn var mapping via @theme inline, ui/ flat-files + export * + cva (.variants.ts for JS-side hex maps), shared-components rule, design-system showcase routes, framer-motion, Tailwind IntelliSense. Token rules: never primitives/hex in components, Tier3 only when Figma defines it. status → stable.
## [2026-06-12] update | wiki — Decision (grill): design tokens are handled PER CSS FRAMEWORK; NO shared/agnostic design-tokens standard. Each add-{shadcn,mui,antd}-to-nextjs owns its token mechanism (shadcn = CSS vars + Tailwind @theme; MUI/antd to define their own). Earlier-explored options (neutral DTCG source + Terrazzo generation) were rejected in favor of matching each framework's native approach.
