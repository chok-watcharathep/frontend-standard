---
id: form
title: Form Pattern (react-hook-form + zod)
category: pattern
status: draft
applies_to: [nextjs]
requires: [nextjs]
conflicts_with: []
tags: [pattern, form, react-hook-form, zod, validation, i18n]
updated: 2026-06-16
---

# Form Pattern (react-hook-form + zod)

## Summary

The standard way to build a form: `react-hook-form` for state, `zod` for validation (via
`@hookform/resolvers`), an **interface-first** typed schema built inside a **schema hook** so error
messages can be translated, and `Controller`/`useController` to wire every field. Submission goes
through a React Query mutation hook per [[nextjs]], with API field errors mapped back via `setError`.

## When to use

- Any screen that collects and validates user input (login, register, create/edit, filters with
  validation, multi-step flows).

## When not to use

- Read-only screens or trivial single-button actions with no validated input.
- A list/table screen → [[listing]] (a filter form embedded in a listing still follows this pattern).

## Prerequisites

- [[nextjs]] — base structure, the feature module, and the React-Query + axios data layer used for
  submission.
- A UI library for the field components: [[add-mui-to-nextjs]] | [[add-antd-to-nextjs]] |
  [[add-shadcn-to-nextjs]]. Examples below use **MUI**; antd/shadcn users swap the input components
  (`TextField`, `Checkbox`, …) for their library's equivalents — the wiring is identical.

> i18n uses `next-intl`'s `useTranslations`. **TODO:** a dedicated `i18n` standard should own next-intl
> setup, the message-file layout, and the shared `error.*` namespace; this pattern only consumes it.

## Stack

| Concern        | Choice                                                  |
| -------------- | ------------------------------------------------------- |
| Form state     | `react-hook-form` ^7                                    |
| Validation     | `zod` ^3.25 (v3 message API: `required_error`)          |
| Resolver       | `@hookform/resolvers` ^3 → `zodResolver`                |
| i18n           | `next-intl` ^4 (`useTranslations`)                      |
| Field wiring   | `Controller` / `useController` (controlled)             |

## Core conventions

A form is **three colocated pieces** inside a feature, plus a submission hook:

1. **`interface {Entity}FormFields`** — the field shape, in the feature's `interfaces/`. This is the
   **source of truth** for the form's type (interface-first, per [[nextjs]]'s "interfaces/ = types only").
2. **`use{Form}FormSchema`** hook — in the feature's `hooks/`. Builds the zod schema **inside the hook**
   so it can call `useTranslations` for messages. Annotated `ZodType<{Entity}FormFields>` so the schema
   is checked against the interface. **One hook per form**, returning a single `schema`.
3. **`{Entity}Form`** component — in the feature's `components/`. Owns `useForm`, the resolver, default
   values, and field wiring via `Controller`/`useController`.
4. **`use{Op}{Entity}`** mutation hook — submission, per the [[nextjs]] data layer (service + React Query).

### 1. Interface (source of truth)

```ts
// features/auth/interfaces/register.interface.ts
export interface RegisterEmailFormFields {
  email: string
  agreedToTerms: boolean
}
```

### 2. Schema hook (i18n-aware)

The schema lives inside the hook because messages need `t`. Annotate with `ZodType<Fields>` so a
mismatch between the schema and the interface is a type error.

```ts
// features/auth/hooks/useRegisterEmailFormSchema.ts
import { useTranslations } from 'next-intl'
import { z, type ZodType } from 'zod'

import type { RegisterEmailFormFields } from '@/features/auth/interfaces'

const useRegisterEmailFormSchema = () => {
  const tError = useTranslations('error')

  const schema: ZodType<RegisterEmailFormFields> = z.object({
    email: z
      .string({ required_error: tError('form.required'), invalid_type_error: tError('form.required') })
      .trim()
      .email(tError('form.email')),
    agreedToTerms: z.boolean(),
  })

  return schema
}

export default useRegisterEmailFormSchema
```

**Message keys:** reuse a **shared** namespace for generic validations
(`error.form.required`, `error.form.email`, `error.form.maxLength`, …) across all forms; put
**form-specific** messages in that form's own namespace (e.g. `auth.register.*`). Never hardcode
literal strings in a schema.

**Cross-field validation:** use `.superRefine((data, ctx) => ctx.addIssue({ ..., path: ['field'] }))`
for rules that span fields (e.g. "note required when reason = other").

### 3. Form component (wiring)

- One `useForm<{Entity}FormFields>({ resolver: zodResolver(schema), defaultValues })`.
- **Always provide `defaultValues` for every field** (controlled from first render).
- Wire **every** field with `Controller` / `useController`. **Never** spread `register()` and
  `useController().field` onto the same input — pick `Controller`/`useController`.
- Guard controlled values: `value={field.value ?? ''}` to avoid the uncontrolled→controlled warning.
- `onSubmit(fields, setError)` — hand `setError` up so the parent can map API field errors back.

```tsx
// features/auth/components/RegisterEmailForm/RegisterEmailForm.tsx
'use client'

import { zodResolver } from '@hookform/resolvers/zod'
import { Button, Checkbox, FormControlLabel, Stack, TextField } from '@mui/material'
import { useTranslations } from 'next-intl'
import { Controller, useForm, type UseFormSetError } from 'react-hook-form'

import useRegisterEmailFormSchema from '@/features/auth/hooks/useRegisterEmailFormSchema'
import type { RegisterEmailFormFields } from '@/features/auth/interfaces'

interface RegisterEmailFormProps {
  isSubmitting?: boolean
  onSubmit: (fields: RegisterEmailFormFields, setError: UseFormSetError<RegisterEmailFormFields>) => void
}

const RegisterEmailForm = ({ isSubmitting, onSubmit }: RegisterEmailFormProps) => {
  const tCommon = useTranslations('common')
  const schema = useRegisterEmailFormSchema()

  const { control, handleSubmit, setError, watch } = useForm<RegisterEmailFormFields>({
    resolver: zodResolver(schema),
    defaultValues: { email: '', agreedToTerms: false },
  })

  const agreedToTerms = watch('agreedToTerms')

  return (
    <form noValidate onSubmit={handleSubmit((fields) => onSubmit(fields, setError))}>
      <Stack gap={2}>
        <Controller
          name="email"
          control={control}
          render={({ field, fieldState }) => (
            <TextField
              {...field}
              value={field.value ?? ''}
              label={tCommon('email')}
              error={!!fieldState.error}
              helperText={fieldState.error?.message}
            />
          )}
        />
        <Controller
          name="agreedToTerms"
          control={control}
          render={({ field }) => (
            <FormControlLabel
              control={<Checkbox checked={field.value} onChange={field.onChange} />}
              label={tCommon('acceptTerms')}
            />
          )}
        />
        <Button type="submit" loading={isSubmitting} disabled={!agreedToTerms}>
          {tCommon('submit')}
        </Button>
      </Stack>
    </form>
  )
}

export default RegisterEmailForm
```

**Validation timing:** keep react-hook-form's defaults (validate on submit, re-validate on change).
Override per-form only when the UX needs it.

**Large or reused forms — split fields:** wrap the form in `FormProvider {...form}` and extract field
subcomponents that read `control` via `useFormContext<{Entity}FormFields>()`. Use this **only when the
form is large or fields are reused** — inline `Controller`s (above) are the default.

```tsx
const EmailField = () => {
  const { control } = useFormContext<RegisterEmailFormFields>()
  return (
    <Controller name="email" control={control} render={({ field, fieldState }) => (
      <TextField {...field} value={field.value ?? ''} error={!!fieldState.error} helperText={fieldState.error?.message} />
    )} />
  )
}
```

### 4. Submission + server errors (React Query)

Submit through a mutation hook → service, per the [[nextjs]] data layer. In the page/parent, map
**field-level** API errors back into the form with `setError`, and surface **generic** errors via the
snackbar.

```tsx
const registerMutation = useRegisterEmail() // RQ mutation → service

const handleSubmit = (fields: RegisterEmailFormFields, setError: UseFormSetError<RegisterEmailFormFields>) => {
  registerMutation.mutate(fields, {
    onError: (error) => {
      if (error.fieldErrors) {
        error.fieldErrors.forEach((e) => setError(e.field, { message: e.message }))
        return
      }
      showSnackbar({ severity: 'error', message: tError('api.genericError') })
    },
  })
}

// <RegisterEmailForm isSubmitting={registerMutation.isPending} onSubmit={handleSubmit} />
```

When editing existing data, hydrate the form with `reset(data)` once the data loads (don't set
`defaultValues` from async data).

## Resulting structure

```
features/{feature}/
  interfaces/
    {entity}.interface.ts        # interface {Entity}FormFields  (source of truth)
    index.ts
  hooks/
    use{Form}FormSchema.ts       # zod schema built inside the hook (i18n messages)
    use{Form}FormSchema.spec.ts  # colocated test (renderHook + safeParse)
    use{Op}{Entity}.ts           # React Query mutation for submission
    index.ts
  components/
    {Entity}Form/
      {Entity}Form.tsx           # useForm + Controller wiring  (default export)
      {Entity}Form.style.ts      # styling — owned by the CSS-framework standard
      index.ts
    index.ts
```

## Testing

- Test the **schema hook**, not the component: colocated `use{Form}FormSchema.spec.ts` with Vitest.
- Mock next-intl so messages are identity keys: `vi.mock('next-intl', () => ({ useTranslations: () => (k) => k }))`.
- Drive it with `renderHook` + `schema.safeParse(...)`; assert `success` and, on failure,
  `error.issues[0].path`. Cover required/invalid/boundary cases and every conditional branch.

## Rules

- ✅ Do: declare `interface {Entity}FormFields` first; annotate the schema `ZodType<{Entity}FormFields>`.
- ✅ Do: build the schema inside `use{Form}FormSchema` so messages come from `useTranslations`.
- ✅ Do: one schema hook per form, returning a single `schema`.
- ✅ Do: wire every field with `Controller`/`useController`; always pass `defaultValues`.
- ✅ Do: submit via a React Query mutation hook → service; map API field errors with `setError`.
- ✅ Do: pull generic validation messages from the shared `error.form.*` namespace.
- ❌ Don't: hardcode literal validation messages in a schema (breaks i18n).
- ❌ Don't: spread both `register()` and `useController().field` onto the same input.
- ❌ Don't: derive the field type with `z.infer` from a hook-built schema — the type is trapped inside
  the hook; the interface is the source of truth here.
- ❌ Don't: build a schema outside React when it needs translated messages (no module-level schema for
  i18n forms).

## Gotchas

- **Schema-in-hook ⇒ no `z.infer`.** Because the schema is created inside the hook (to access `t`), you
  can't `z.infer<typeof schema>` at module level — that's the reason this standard is interface-first.
- **zod v3 message API.** Use `required_error` / `invalid_type_error` and per-rule message args
  (`.email(msg)`, `.min(n, msg)`). Don't use the zod v4 `error` API — the pinned version is `^3.25`.
- **Controlled-value warning.** A field whose value can be `null`/`undefined` flips controlled state —
  always `value={field.value ?? ''}`.
- **Trim once.** Prefer `z.string().trim()` in the schema; don't also add `setValueAs: trim` on the input.
- **Edit forms.** Use `reset(data)` after async load rather than async `defaultValues`.

## Related standards

- [[nextjs]] — feature module, interfaces rule, and the React-Query + axios data layer for submission.
- [[add-mui-to-nextjs]] · [[add-antd-to-nextjs]] · [[add-shadcn-to-nextjs]] — supply the field components.
- [[listing]] — filter forms in a listing follow this pattern.
- [[server-data-layer]] — server-side request validation is schema-first (zod), contrasting this
  pattern's interface-first form schema.
- [[naming]] — file/symbol naming.
</content>
