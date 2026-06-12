---
id: nextjs-with-payload
title: Next.js with Payload CMS Project Structure
category: project-structure
status: draft
applies_to: [nextjs, payload]
requires: [nextjs]
conflicts_with: []
tags: [nextjs, payload, cms, scaffold]
updated: 2026-06-12
---

# Next.js with Payload CMS Project Structure

## Summary

A Next.js project with Payload CMS embedded in the same app (shared deployment). Use when the
app needs structured content management alongside the frontend.

## When to use

- The app needs an admin/CMS for content editors in the same codebase.

## When not to use

- No CMS needed → use [[nextjs]].

## Prerequisites

- [[nextjs]] — the base Next.js structure this builds on.

## Steps

1. Start from [[nextjs]], then add Payload:
   ```bash
   npx create-payload-app@latest
   ```
   > TODO: confirm whether we scaffold fresh with the Payload template or layer onto an existing [[nextjs]] app, and pin versions.

2. Configure the database adapter and storage.
   > TODO: company default DB (Postgres/Mongo) and file storage (S3/local).

3. Mount the Payload admin route and API.

## Resulting structure

```
<app-name>/
  src/
    app/
      (frontend)/         # public site routes
      (payload)/          # payload admin + api routes
    collections/          # Payload collections
    payload.config.ts
    ...                   # plus everything from [[nextjs]]
```

> TODO: confirm collection layout and config conventions.

## Rules

- ✅ Do: keep frontend and CMS concerns in separate route groups.
- ❌ Don't: import Payload server code into client components.

## Gotchas

- Payload config and generated types must stay in sync; regenerate types after schema changes.

## Related standards

- [[nextjs]] — base structure.
- [[naming]] — naming conventions.
