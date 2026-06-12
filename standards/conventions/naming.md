---
id: naming
title: Naming Conventions
category: convention
status: stable
applies_to: []
requires: [nextjs]
conflicts_with: []
tags: [convention, naming, pointer]
updated: 2026-06-12
---

# Naming Conventions

> **Pointer page.** Naming conventions are defined inline in the [[nextjs]] standard
> (see its **Naming conventions** table). This page exists so naming is discoverable on its own;
> the source of truth is [[nextjs]].

Quick reference (full table in [[nextjs]]):

- Component file/folder → PascalCase (`ProductCard/ProductCard.tsx`)
- Hook → `use{Op}{Entity}.ts` (`useGetProductList.ts`)
- Service file → `{name}.service.ts`; service fn → `{op}{Entity}`
- Transform file → `{name}.transforms.ts`
- Interface → `{Entity}` / `{Op}{Entity}Request`/`Response`
- Enum file → `{name}.enum.ts`; query-key enum → `{Entity}QueryKey`
- Constants file → `{name}.constants.ts`; runtime const → camelCase
- Utility file → `{name}.util.ts`; lib config → `{name}.lib.ts`
- Test → `{name}.spec.ts`; mock → `{name}.mock.ts`

## Related standards

- [[nextjs]] — defines these conventions in full.
