---
id: listing
title: Listing Page Pattern
category: pattern
status: draft
applies_to: [nextjs]
requires: [nextjs]
conflicts_with: []
tags: [pattern, list, table, pagination, filtering]
updated: 2026-06-12
---

# Listing Page Pattern

## Summary

The standard way to build a list/table page: server-driven pagination, filtering, sorting, and
empty/loading/error states. Used for any "list of records" screen.

## When to use

- Any page that displays a paginated/filterable collection of records.

## When not to use

- Single-record detail or form screens.

## Prerequisites

- [[nextjs]] — base structure.
- A UI library: [[add-shadcn-to-nextjs]] | [[add-mui-to-nextjs]] | [[add-antd-to-nextjs]].

## Steps

1. Drive list state (page, pageSize, sort, filters) from the URL search params.
2. Fetch data on the server using those params.
3. Render the table + pagination + filter controls using the project's UI library.
4. Handle loading, empty, and error states explicitly.

> TODO: specify the company standard for state source (URL vs client store), data-fetching
> approach (Server Component fetch / React Query / etc.), and the canonical query param names.

## Resulting structure

```
src/features/<entity>/
  list/
    page.tsx            # route entry, reads search params
    <Entity>Table.tsx   # presentational table
    <Entity>Filters.tsx # filter controls
    use<Entity>List.ts  # data fetching (if client-side)
```

## Rules

- ✅ Do: keep filter/sort/pagination state shareable via the URL.
- ✅ Do: always render explicit empty and error states.
- ❌ Don't: fetch the full collection and paginate on the client for large datasets.

## Gotchas

- Keep search-param names stable; they're effectively a public contract (bookmarkable URLs).

## Related standards

- [[nextjs]] — base.
- [[add-shadcn-to-nextjs]], [[add-mui-to-nextjs]], [[add-antd-to-nextjs]] — supplies the table/UI.
- [[naming]] — naming conventions for files and components.
