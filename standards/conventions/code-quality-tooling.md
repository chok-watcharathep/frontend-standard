---
id: code-quality-tooling
title: Code Quality Tooling (ESLint, Prettier, Husky, lint-staged, VS Code)
category: convention
status: draft
applies_to: [nextjs]
requires: [nextjs]
conflicts_with: []
tags: [tooling, eslint, prettier, husky, git-hooks, formatting]
updated: 2026-06-18
---

# Code Quality Tooling (ESLint, Prettier, Husky, lint-staged, VS Code)

## Summary

The canonical lint / format / git-hook / editor setup for a [[nextjs]] project: Prettier
formatting, a hand-rolled ESLint v9 flat config, lint-staged + Husky hooks (pre-commit,
commit-msg, pre-push), the commit-message and branch-name conventions those hooks enforce, and
the shared `.vscode/` editor config. Apply it once per repo after the base [[nextjs]] scaffold.

## When to use

- Setting up code quality tooling on a fresh [[nextjs]] (or [[nextjs-with-payload]]) repo.
- Reconciling an existing repo's lint/format/hooks against the company baseline.

## When not to use

- Non-Next projects — the ESLint config and the `.vscode` defaults assume Next. The Prettier /
  Husky / lint-staged parts transfer, but this page does not target them.

## Prerequisites

- [[nextjs]] — the base scaffold. This standard installs on top of it and reuses its
  `type-check` script in the `pre-push` hook.

## Steps

1. Install dev dependencies (major-version pins; take the latest minor of each):

   ```bash
   pnpm add -D \
     eslint@9 prettier@3 husky@9 lint-staged@16 cross-env@7 \
     @eslint/eslintrc@3 \
     @typescript-eslint/eslint-plugin@8 @typescript-eslint/parser@8 \
     eslint-config-next@16 @next/eslint-plugin-next@16 \
     eslint-plugin-import@2 eslint-import-resolver-alias@1 \
     eslint-plugin-jsx-a11y@6 eslint-plugin-react@7 eslint-plugin-react-hooks@7
   ```

2. Add the quality scripts to `package.json`. Keep the `cross-env NODE_OPTIONS` wrappers — they
   silence Node deprecation noise and raise the ESLint heap. (`dev`/`build`/`start`/`type-check`
   are owned by [[nextjs]]; do not redefine them here.)

   ```jsonc
   // package.json → "scripts"
   {
     "lint": "cross-env NODE_OPTIONS=\"--no-deprecation --max-old-space-size=2560\" eslint . --cache --cache-location .eslintcache",
     "lint:fix": "cross-env NODE_OPTIONS=\"--no-deprecation --max-old-space-size=2560\" eslint . --fix --cache --cache-location .eslintcache",
     "format": "cross-env NODE_OPTIONS=--no-deprecation prettier --write .",
     "format:check": "cross-env NODE_OPTIONS=--no-deprecation prettier --check .",
     "prepare": "cross-env NODE_OPTIONS=--no-deprecation husky"
   }
   ```

3. Write `.prettierrc.json`:

   ```json
   {
     "singleQuote": true,
     "trailingComma": "all",
     "printWidth": 100,
     "semi": false
   }
   ```

4. Write `.prettierignore` (generic baseline — for Payload add the extra paths from the
   [When using Payload](#when-using-payload) callout):

   ```
   # Ignore files managed by package manager
   pnpm-lock.yaml

   # Ignore build and output directories
   /build
   /dist
   /.next
   /out

   # Ignore specific file types
   *.png
   *.jpg
   *.jpeg
   ```

5. Write `eslint.config.mjs` (flat config v9). This is the canonical rule set — copy it verbatim:

   ```js
   // eslint.config.mjs
   import eslintPluginNext from '@next/eslint-plugin-next'
   import eslintPluginTypeScript from '@typescript-eslint/eslint-plugin'
   import typescriptParser from '@typescript-eslint/parser'
   import eslintPluginImport from 'eslint-plugin-import'
   import eslintPluginJSXA11y from 'eslint-plugin-jsx-a11y'
   import eslintPluginReactHooks from 'eslint-plugin-react-hooks'
   import eslintPluginReact from 'eslint-plugin-react'

   const eslintConfig = [
     {
       files: ['**/*.{js,mjs,cjs,ts,jsx,tsx}'],
       ignores: [
         '**/*.config.{js,mjs,cjs,ts,jsx,tsx}',
         '**/node_modules/**',
         '**/.next/**',
         '**/coverage/**',
         '**/next-env.d.ts',
         '**/scripts/**',
         // --- Project-specific ignores: see "When using Payload" callout ---
       ],
       languageOptions: {
         ecmaVersion: 2022,
         sourceType: 'module',
         parser: typescriptParser,
         parserOptions: {
           project: './tsconfig.json',
           ecmaFeatures: {
             jsx: true,
           },
         },
       },
       plugins: {
         '@typescript-eslint': eslintPluginTypeScript,
         react: eslintPluginReact,
         'jsx-a11y': eslintPluginJSXA11y,
         import: eslintPluginImport,
         '@next/next': eslintPluginNext,
         'react-hooks': eslintPluginReactHooks,
       },
       rules: {
         ...eslintPluginTypeScript.configs.recommended.rules,
         ...eslintPluginReact.configs.recommended.rules,
         ...eslintPluginJSXA11y.configs.recommended.rules,
         ...eslintPluginNext.configs.recommended.rules,
         ...eslintPluginImport.configs.recommended.rules,

         'react-hooks/exhaustive-deps': 'warn',
         'no-console': 'warn',
         '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
         '@typescript-eslint/no-empty-object-type': 'off',
         '@typescript-eslint/consistent-type-imports': 'error',
         '@typescript-eslint/consistent-type-exports': 'error',
         'react/react-in-jsx-scope': 'off',
         'react/prop-types': 'off',
         'jsx-a11y/anchor-is-valid': 'off',
         'import/order': [
           'error',
           {
             groups: ['builtin', 'external', 'internal'],
             pathGroups: [
               {
                 pattern: 'react',
                 group: 'external',
                 position: 'before',
               },
             ],
             pathGroupsExcludedImportTypes: ['builtin'],
             'newlines-between': 'always',
             alphabetize: { order: 'asc', caseInsensitive: true },
           },
         ],
         '@next/next/no-img-element': 'error',
         '@next/next/no-duplicate-head': 'off',
         'import/no-named-as-default': 'off',
         'import/no-named-as-default-member': 'off',
         'import/namespace': 'off',
         'react/no-unknown-property': 'off',
         'max-depth': ['error', { max: 2 }],
         'no-nested-ternary': 'error',
         'import/default': 'off',
       },
       settings: {
         react: {
           version: 'detect',
         },
         'import/resolver': {
           typescript: {
             project: './tsconfig.json',
           },
           node: {
             paths: ['src'],
             extensions: ['.js', '.jsx', '.ts', '.tsx'],
           },
           alias: {
             map: [
               ['@', './src'],
               ['@assets', './public'],
             ],
             extensions: ['.js', '.jsx', '.ts', '.tsx'],
           },
         },
       },
     },
     {
       ignores: ['.next/'],
     },
     {
       files: ['**/scripts/**'],
       rules: { 'no-console': 'off', 'max-depth': 'off' },
     },
   ]

   export default eslintConfig
   ```

6. Write `.lintstagedrc.json`:

   ```json
   {
     "*.{ts,tsx,js,jsx}": ["eslint --fix", "prettier --write"],
     "*.{json,css,md}": ["prettier --write"]
   }
   ```

7. Initialize Husky and create the hooks:

   ```bash
   pnpm exec husky init   # creates .husky/ and the prepare-managed _/ dir
   ```

   `.husky/pre-commit`:

   ```bash
   pnpm lint-staged
   ```

   `.husky/commit-msg`:

   ```bash
   commit_msg_file=$1
   commit_msg=$(cat "$commit_msg_file")

   # Enforce "<Type>: <message>" or "Merge ..."
   if ! echo "$commit_msg" | grep -qE "^(Fix|Feat|Docs|Style|Refactor|Perf|Test|Chore): .*|^Merge .*"; then
       echo "Invalid commit message format. Please use: <Type>: <message> or Merge branch ..."
       echo "Type: Fix, Feat, Docs, Style, Refactor, Perf, Test, Chore"
       exit 1
   fi

   exit 0
   ```

   `.husky/pre-push` (runs `type-check` — defined in [[nextjs]] — then enforces the branch name):

   ```bash
   BRANCH_REGEX="^(feature|bugfix|hotfix|release|dev|setup|base|refactor|chore|story)\/[a-zA-Z0-9._-]+$"
   BRANCH_NAME=$(git symbolic-ref --short HEAD)

   pnpm run type-check

   if [[ ! $BRANCH_NAME =~ $BRANCH_REGEX ]]; then
       echo "Error: Branch name '$BRANCH_NAME' does not follow the naming convention."
       echo "Use one of: feature/, bugfix/, hotfix/, release/, dev/, setup/, base/, refactor/, chore/, story/"
       exit 1
   fi
   ```

8. Write the shared editor config under `.vscode/`.

   `.vscode/extensions.json` — this standard owns this file (ESLint + Prettier only). A future
   testing/debugging standard that needs another extension amends this list:

   ```json
   {
     "recommendations": ["dbaeumer.vscode-eslint", "esbenp.prettier-vscode"]
   }
   ```

   `.vscode/settings.json`:

   ```jsonc
   {
     "npm.packageManager": "pnpm",
     "editor.formatOnSave": true,
     "editor.formatOnPaste": true,
     "editor.codeActionsOnSave": {
       "source.fixAll.eslint": "explicit"
     },
     "editor.defaultFormatter": "esbenp.prettier-vscode",
     "[javascript]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
     "[javascriptreact]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
     "[typescript]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
     "[typescriptreact]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
     "[json]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
     "[jsonc]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
     "[html]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
     "[css]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
     "[scss]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
     "[markdown]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
     "typescript.tsdk": "node_modules/typescript/lib"
   }
   ```

## Commit & branch conventions

These are enforced by `commit-msg` and `pre-push`. Documented here for now; they will migrate to
a dedicated `git-workflow` convention later.

**Commit message** — `<Type>: <message>`, type is **PascalCase**, case-sensitive. Merge commits
(`Merge ...`) are exempt.

| Type       | Use for                          |
| ---------- | -------------------------------- |
| `Fix`      | bug fix                          |
| `Feat`     | new feature                      |
| `Docs`     | documentation only               |
| `Style`    | formatting, no code-meaning change |
| `Refactor` | restructure, no behavior change  |
| `Perf`     | performance                      |
| `Test`     | tests                            |
| `Chore`    | tooling, deps, housekeeping      |

Example: `Feat: add product listing`.

**Branch name** — `<prefix>/<name>`, prefix is **lowercase**:
`feature` · `bugfix` · `hotfix` · `release` · `dev` · `setup` · `base` · `refactor` · `chore` · `story`.
Example: `feature/product-listing`.

## When using Payload

For [[nextjs-with-payload]], generated and vendored files must be excluded from ESLint and
Prettier. Add these to the `eslint.config.mjs` `ignores` array (replacing the
`Project-specific ignores` comment in step 5):

```js
'**/src/payload/migrations/**',
'**/src/app/(payload)/admin/importMap.js',
'**/src/payload-types.ts',
'**/src/shared/types/next-auth.d.ts',   // present when next-auth is used
```

Add the matching `import/resolver.alias` entry so `@payload-config` resolves:

```js
['@payload-config', './src/payload.config.ts'],
```

`next-intl` trips ESLint's import resolver (e.g. `useLocale`); silence it via `settings`:

```js
'import/ignore': ['next-intl'],
```

And add the Payload-generated paths to `.prettierignore`:

```
/src/migrations
/src/payload/migrations/
/src/app/(payload)/admin/importMap.js
/src/payload-types.ts
```

## Resulting structure

```
repo/
  .prettierrc.json
  .prettierignore
  eslint.config.mjs
  .lintstagedrc.json
  .husky/
    pre-commit          # pnpm lint-staged
    commit-msg          # <Type>: <message> enforcement
    pre-push            # pnpm type-check + branch-name enforcement
    _/                  # husky internals (git-ignored, managed by `prepare`)
  .vscode/
    extensions.json     # eslint + prettier
    settings.json       # formatOnSave + eslint fixAll
  package.json          # lint, lint:fix, format, format:check, prepare scripts
```

## Rules

- ✅ Do: copy `eslint.config.mjs` verbatim — `max-depth: 2` and `no-nested-ternary` are
  deliberate company choices, not accidents.
- ✅ Do: keep the `cross-env NODE_OPTIONS` wrappers on the lint/format/prepare scripts.
- ✅ Do: use PascalCase commit types and lowercase branch prefixes.
- ✅ Do: let `prepare` (Husky) run on `pnpm install` so hooks are always wired.
- ❌ Don't: redefine `dev`/`build`/`start`/`type-check` here — they belong to [[nextjs]].
- ❌ Don't: add unrelated VS Code extensions to `extensions.json` from this standard; amend it
  from the standard that introduces the tool (e.g. a testing standard adds the Vitest extension).
- ❌ Don't: lint Payload-generated files — keep them in the `ignores` list.

## Gotchas

- **Casing mismatch is intentional.** Commit *types* are PascalCase (`Feat`) while branch
  *prefixes* are lowercase (`feature`). They are separate regexes; the future `git-workflow`
  standard will reconcile the vocabulary.
- **`scripts/**` is both ignored and overridden.** It is listed in the main block's `ignores`
  (so the strict rules don't apply) and re-included by the trailing `files: ['**/scripts/**']`
  block with `no-console`/`max-depth` off. This is a valid flat-config exclude-then-relax
  pattern, not a contradiction — keep both.
- **`prepare` must exist before hooks fire.** If `.husky/_` is missing, run `pnpm install` (or
  `pnpm exec husky`) to regenerate it; hooks silently no-op otherwise.
- **`pre-push` depends on `type-check`.** That script lives in [[nextjs]] — if it's absent the
  hook fails. Ensure the base scaffold is applied first.

## Related standards

- [[nextjs]] — base scaffold; owns `type-check` and the non-quality scripts. Its **Tooling &
  config** section points here.
- [[nextjs-with-payload]] — needs the extra ESLint/Prettier ignores in the Payload callout.
- [[naming]] — file/symbol naming the ESLint `import/order` and project conventions assume.
