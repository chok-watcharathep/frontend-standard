# Technical Standards Wiki — Agent Schema

This repository is a **company technical-standards wiki**. It is a persistent, interlinked
collection of markdown files that defines how we build software (project structures,
library integrations, reusable patterns, and conventions).

**Primary audience: AI coding agents.** When an agent is asked to scaffold a project, add a
library, or implement a feature, it reads the relevant standard(s) here and follows them
exactly. Write every page so an agent can execute it without guessing. Humans read it too,
but the pages are optimized for unambiguous machine consumption.

You (the LLM) own and maintain this wiki. The human curates topics, makes the engineering
decisions, and reviews. You do the writing, cross-referencing, filing, and bookkeeping.

---

## Layout

```
standard/
  CLAUDE.md                  # this file — the schema & workflows
  index.md                   # catalog of every standard (you keep current)
  log.md                     # append-only chronological change log
  templates/
    standard-template.md     # copy this to author a new standard
  standards/
    project-structures/      # base scaffolds for a whole project type
    integrations/            # adding a library/tool onto an existing base
    patterns/                # reusable feature/implementation patterns
    conventions/             # cross-cutting rules (naming, git, ts-config, ...)
```

### Categories — what goes where

| Category             | Folder                 | Purpose                                                        | Example ids                                            |
| -------------------- | ---------------------- | ------------------------------------------------------------- | ------------------------------------------------------ |
| Project structure    | `project-structures/`  | A complete base scaffold for a project type                   | `nextjs`, `nextjs-with-payload`                        |
| Integration          | `integrations/`        | Adding a library/tool onto an existing base                   | `add-mui-to-nextjs`, `add-shadcn-to-nextjs`, `add-antd-to-nextjs` |
| Pattern              | `patterns/`            | A reusable feature/implementation recipe, framework-agnostic-ish | `listing`, `form`, `auth`, `data-fetching`          |
| Convention           | `conventions/`         | A cross-cutting rule that applies across many standards       | `naming`, `git-workflow`, `typescript-config`          |

If a new standard doesn't fit a category, propose a new category to the human before creating
a folder for it.

---

## Page conventions

- **One standard per file.** Filename is the `id` in kebab-case + `.md` (e.g. `add-shadcn-to-nextjs.md`).
- The `id` in frontmatter **must equal** the filename without extension.
- Cross-link other standards with `[[id]]` (e.g. `[[nextjs]]`, `[[add-shadcn-to-nextjs]]`).
- Prefer copy-pasteable commands, code blocks, and file trees over prose. Be explicit about
  versions, file paths, and exact commands.
- Keep company-specific decisions marked clearly. Use `> TODO:` callouts for anything not yet decided.

### Required frontmatter

```yaml
---
id: add-shadcn-to-nextjs        # == filename without .md, kebab-case
title: Add shadcn/ui to Next.js
category: integration            # project-structure | integration | pattern | convention
status: draft                    # draft | stable | deprecated
applies_to: [nextjs]             # ids/tech this standard targets ([] if general)
requires: [nextjs]               # standards that must be applied first ([] if none)
conflicts_with: [add-mui-to-nextjs, add-antd-to-nextjs]   # mutually exclusive standards
tags: [ui, styling, components]
updated: 2026-06-12              # ISO date, set to today on every edit
---
```

### Recommended section skeleton

See `templates/standard-template.md`. In order:
1. `# Title` (H1)
2. **Summary** — 1–2 sentences: what it is and when to apply it.
3. **When to use / When not to use**
4. **Prerequisites** — link required standards via `[[id]]`.
5. **Steps** — numbered, each with the exact command and/or code block.
6. **Resulting structure** — a file tree the agent should expect afterward.
7. **Rules** — do / don't, enforced conventions.
8. **Gotchas** — known pitfalls.
9. **Related standards** — `[[id]]` cross-links.

---

## Workflows

### Add a standard
1. Confirm the category and `id` with the human (or infer if obvious).
2. Copy `templates/standard-template.md` to `standards/<category-folder>/<id>.md`.
3. Fill it in. Mark undecided company specifics as `> TODO:`.
4. Add `[[id]]` cross-links to/from related standards (update their **Related standards** too).
5. Add a row to `index.md` under the right category.
6. Append an entry to `log.md`.

### Update a standard
1. Edit the page; bump `updated:` to today.
2. If the change affects other standards, update their cross-links and frontmatter
   (`requires`, `conflicts_with`).
3. Update the `index.md` one-liner if the summary changed.
4. Append a `log.md` entry.

### Use a standard (when scaffolding/implementing for the human)
1. Read `index.md` to find the relevant standard(s).
2. Read the standard and everything in its `requires:` chain, in dependency order.
3. Check `conflicts_with:` to avoid combining incompatible standards.
4. Follow the steps exactly. If the standard is ambiguous or missing a decision, ask the
   human and then fix the standard so it's unambiguous next time.

### Lint (health-check)
Periodically, on request, scan for:
- Contradictions between standards.
- Broken `[[id]]` links and `requires` / `conflicts_with` pointing at non-existent ids.
- `status: draft` pages that are actually in use and should be promoted to `stable`.
- Orphan standards (no inbound links and not in `index.md`).
- Concepts referenced repeatedly that deserve their own page.
- `index.md` rows that don't match an existing file, and files missing from `index.md`.
Report findings; fix the safe ones; ask before deleting or deprecating.

---

## index.md and log.md

- **`index.md`** is content-oriented: the catalog. Every standard appears as one row
  (id link · status · one-line summary), grouped by category. Update on every add/rename.
- **`log.md`** is chronological and append-only. Each entry starts with a parseable prefix:
  `## [YYYY-MM-DD] <op> | <id> — <note>` where `<op>` is `add` | `update` | `deprecate` | `lint`.
  This lets `grep "^## \[" log.md | tail` show recent activity.
