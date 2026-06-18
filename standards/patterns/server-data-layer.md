---
id: server-data-layer
title: Server Data Layer (Payload feature server/ + utils/server.ts)
category: pattern
status: draft
applies_to: [nextjs-with-payload]
requires: [nextjs-with-payload]
conflicts_with: []
tags: [payload, server, data-layer, endpoints, error-handling, validation]
updated: 2026-06-18
---

# Server Data Layer (Payload feature `server/` + `utils/server.ts`)

## Summary

The server-side counterpart to [[nextjs]]'s client data layer. Defines how a Payload feature's
**`server/` folder** is organized (handlers, collection hooks, request schemas, server-side
services, server utils) and documents the shared **`payload/utils/server.ts`** foundations every
custom endpoint composes with (`defineEndpoint`, `pipeHandlers`, the middleware HOCs, typed
exceptions + `handleErrorResponse`).

## When to use

- Implementing any Payload **custom endpoint** (business logic, validation, permissions,
  aggregation) — anything beyond a plain collection read.
- Organizing server-only code inside a [[nextjs-with-payload]] feature.

## When not to use

- Pure client data fetching (axios → custom endpoint → React Query) — that's [[nextjs]]'s client
  data layer; the Payload variant (`adminAxiosInstance`, `useAdminGet{Entity}`) lives in
  [[nextjs-with-payload]].
- Admin authentication / login endpoints — see [[add-keycloak-auth-to-payload]], which builds on
  this pattern.

## Prerequisites

- [[nextjs-with-payload]] — the base structure, feature module, and `endpoints.config.ts` wiring.
- [[nextjs]] — naming conventions and the client-side data-layer vocabulary this mirrors.

## The `server/` folder

A feature's server-only code lives under `payload/features/{feature}/server/`. Each subfolder has
its own `index.ts` barrel; **no `server/` root barrel** (import from the subfolder), per [[nextjs]].

| Subfolder         | Holds                                                                 | File / export                                            |
| ----------------- | --------------------------------------------------------------------- | -------------------------------------------------------- |
| `auth-strategies/`| Payload `auth.strategies` objects (`{ name, authenticate }`)          | `{idp}.strategy.ts` — owned by [[add-keycloak-auth-to-payload]] |
| `handlers/`       | Endpoint handler functions                                            | `{name}.handler.ts` → `export const {op}{Entity}`         |
| `hooks/`          | **Payload collection lifecycle hooks** (not React hooks)              | `use{Behavior}.ts` → **default export** (see below)       |
| `schemas/`        | zod **request** schemas (parse `req.data` / `req.query`)              | `{name}.schema.ts` → `export const {op}{Entity}RequestSchema` |
| `services/`       | Server-side calls to **external** systems (3rd-party APIs, IAM)       | `{name}.service.ts` (+ `__mocks__/`)                      |
| `utils/`          | Feature-local server-only helpers                                     | `{name}.util.ts` (+ `.spec.ts`)                           |

### Collection hooks (`server/hooks/`)

These are **Payload collection lifecycle hooks** (`CollectionBeforeChangeHook`,
`CollectionAfterChangeHook`, …) shared by collections — *not* React Query hooks. By convention they
are **`use`-prefixed and default-exported**, deliberately distinct from [[nextjs]]'s client React
hooks (which are `useGet{Entity}`-style and **named** exports). Read the type, not the prefix.

```ts
// payload/features/article/server/hooks/useStampAudit.ts
import type { CollectionBeforeChangeHook } from 'payload'

const useStampAudit: CollectionBeforeChangeHook = async ({ data, req, operation }) => {
  if (operation === 'create') data.createdBy = req.user?.id
  data.updatedBy = req.user?.id
  return data
}

export default useStampAudit
```

### Request schemas (`server/schemas/`)

Server **request** validation is **schema-first**: a standalone `z.object` that parses
`req.data` (body) or `req.query`. The **response** shape is a hand-written interface in the
feature's `interfaces/`. This is deliberately *different* from the [[form]] pattern (which is
interface-first with `ZodType<Fields>`): a form's fields are the durable contract a typed UI binds
to, whereas a request schema is a runtime gate at the trust boundary and the response interface is
the contract — so there's no single type to make authoritative across both sides.

```ts
// payload/features/article/server/schemas/article.schema.ts
import { z } from 'zod'

export const getArticleListRequestSchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().positive().default(20),
  ids: z.array(z.string()).optional(),
})
```

### Server services vs client services

Both are `{name}.service.ts`, but they target opposite directions — keep them apart:

- **Server service** (`server/services/`) — runs on the server, calls **external** systems
  (IAM, 3rd-party REST, the Keycloak admin client). Imports server-only secrets from
  `private-environment.config`. Never imported by client code.
- **Client service** (feature `services/`, per [[nextjs]]) — runs in the browser, calls **our own**
  Payload custom endpoints via `adminAxiosInstance`.

Server services keep their mocks in a sibling `__mocks__/` folder with an `index.ts` barrel, same
as [[nextjs]].

## `payload/utils/server.ts` — shared foundations

`payload/utils/server.ts` is the **server-only** barrel every handler composes with. It is the
boundary marker: anything imported from here must never reach a client component.

```ts
// payload/utils/server.ts
export * from './endpoint.util'
export * from './error.util'
export * from './handler.util'
export * from './pipe.util'
```

### `pipe.util.ts` — HOC composition

```ts
import type { PayloadHandler } from 'payload'

export type HandlerHOC = (handler: PayloadHandler) => PayloadHandler

/**
 * Composes handler HOCs right-to-left (reduceRight).
 * pipeHandlers(withLogger, withAuth, withValidation)(baseHandler)
 *   → withLogger(withAuth(withValidation(baseHandler)))
 * Runtime order (request in): withLogger → withAuth → withValidation → baseHandler → Response
 */
export const pipeHandlers =
  (...hocs: HandlerHOC[]) =>
  (handler: PayloadHandler): PayloadHandler =>
    hocs.reduceRight((acc, hoc) => hoc(acc), handler)
```

### `endpoint.util.ts` — typed endpoint binding

```ts
import type { Endpoint } from 'payload'

/**
 * Zero-runtime-cost wrapper that binds TypeScript Request/Response types to a
 * Payload endpoint. The generics are read by `scripts/generate-openapi.ts`
 * (via ts-morph) to emit accurate OpenAPI `$ref`s.
 *
 *   defineEndpoint<GetFooRequest, GetFooResponse>({ path, method, handler })
 * Use `never` when one side has no interface: defineEndpoint<never, GetFooResponse>({ ... })
 */
// eslint-disable-next-line @typescript-eslint/no-unused-vars
export const defineEndpoint = <_TReq = never, _TRes = never>(endpoint: Endpoint): Endpoint =>
  endpoint
```

### `handler.util.ts` — middleware HOCs

The composable middleware. List them in **run order** in `pipeHandlers` (logger → client-info →
auth); the handler runs last. Note every failure path **throws a typed exception** (even the
config-level "missing `API_KEY`" case) and lets the surrounding `try/catch` → `handleErrorResponse`
render it — HOCs never hand-build an error `Response.json`.

```ts
import crypto from 'crypto'

import { trace } from '@opentelemetry/api'
import { redirect, RedirectType } from 'next/navigation'
import type { PayloadHandler, PayloadRequest } from 'payload'

import { ADMIN_ROUTE } from '@/payload/constants'
import type { AdminPermission } from '@/payload/enums'
import { ErrorCode } from '@/payload/enums'
import { clientInfoStorage, extractClientIp } from '@/payload/libs/server'
import privateEnvironmentConfig from '@/private-environment.config'

import {
  ForbiddenException,
  handleErrorResponse,
  InternalServerErrorException,
  UnauthorizedException,
} from './error.util'

// Redirect-to-login guard for server-rendered admin views (not an endpoint HOC).
export const requireAdminAuth = (req: PayloadRequest) => {
  if (!req.user) {
    redirect(
      `${ADMIN_ROUTE}/login?redirect=${encodeURIComponent(req.pathname)}`,
      RedirectType.replace,
    )
  }
}

// Machine-to-machine: constant-time API-key compare.
export const withApiKeyHandler =
  (handler: PayloadHandler): PayloadHandler =>
  async (req) => {
    try {
      const apiKey = req.headers.get('x-api-key')
      const expectedKey = privateEnvironmentConfig.API_KEY

      if (!expectedKey) {
        req.payload.logger.error('API_KEY is not configured in environment')
        throw new InternalServerErrorException({
          code: ErrorCode.INTERNAL_SERVER_ERROR,
          message: 'Server configuration error',
          subErrors: [],
        })
      }

      if (
        !apiKey ||
        apiKey.length !== expectedKey.length ||
        !crypto.timingSafeEqual(Buffer.from(apiKey), Buffer.from(expectedKey))
      ) {
        throw new UnauthorizedException({
          code: ErrorCode.UNAUTHORIZED,
          message: 'Invalid or missing API key',
          subErrors: [],
        })
      }
      return handler(req)
    } catch (error) {
      return handleErrorResponse(error, req)
    }
  }

// User session + permission gate.
export const withAuthHandler =
  (permissions: AdminPermission[]) =>
  (handler: PayloadHandler): PayloadHandler =>
  async (req) => {
    try {
      if (!req.user) {
        throw new UnauthorizedException({
          code: ErrorCode.UNAUTHORIZED,
          message: 'Unauthorized access',
          subErrors: [],
        })
      }

      if (permissions.length > 0) {
        // TODO: Handle permission checking here
        const hasPermission = true

        // eslint-disable-next-line max-depth
        if (!hasPermission) {
          throw new ForbiddenException({
            code: ErrorCode.FORBIDDEN,
            message: 'Insufficient permissions',
            subErrors: [],
          })
        }
      }

      return handler(req)
    } catch (error) {
      return handleErrorResponse(error, req)
    }
  }

// Structured request log with OTel trace context.
export const withLoggerHandler =
  (handler: PayloadHandler): PayloadHandler =>
  async (req) => {
    const span = trace.getActiveSpan()
    const spanContext = span?.spanContext()

    const context = {
      trace_flags: spanContext?.traceFlags,
      span_id: spanContext?.spanId,
      trace_id: spanContext?.traceId,
      ...spanContext,
    }

    req.payload.logger.info({ ...context, query: req.query }, `${req.method} ${req.pathname}`)

    return handler(req)
  }

// Stash ip / user-agent / correlation-id in an AsyncLocalStorage store for the request.
export const withClientInfoHandler =
  (handler: PayloadHandler): PayloadHandler =>
  (req) => {
    const clientInfo = {
      ip: extractClientIp(req),
      userAgent: req.headers.get('user-agent') ?? undefined,
      correlationId: req.headers.get('x-correlation-id') ?? undefined,
    }
    return clientInfoStorage.run(clientInfo, () => handler(req))
  }
```

### `error.util.ts` — typed exceptions + central handler

Handlers **throw** a typed exception and funnel every `catch` through `handleErrorResponse`. The
handler records the error on the active OTel span, logs it with trace context, and normalizes
`CustomError`, axios errors (passthrough of upstream `BaseErrorResponse`), and `ZodError` into the
company error-response shape; everything else becomes a 500.

```ts
import type { Exception } from '@opentelemetry/api'
import { trace } from '@opentelemetry/api'
import { isAxiosError } from 'axios'
import { NextResponse } from 'next/server'
import type { PayloadRequest } from 'payload'
import { z } from 'zod'

import { ErrorCode } from '@/payload/enums'
import type { BaseErrorResponse } from '@/shared/interfaces'
import { isBaseErrorResponse } from '@/shared/utils'

type CustomErrorResponse = Omit<BaseErrorResponse, 'correlationId' | 'timestamp'>

export class CustomError extends Error {
  public errorResponse: Omit<BaseErrorResponse, 'correlationId'>

  constructor(
    public status: number,
    error: Omit<BaseErrorResponse, 'correlationId'>,
  ) {
    super(error.message)
    this.name = 'CustomError'
    this.errorResponse = error
    this.status = status
  }
}

export class BadRequestException extends CustomError {
  constructor(error: CustomErrorResponse) {
    super(400, { ...error, timestamp: new Date().getTime() })
    this.name = 'BadRequestException'
  }
}

export class UnprocessableEntityException extends CustomError {
  constructor(error: CustomErrorResponse) {
    super(422, { ...error, timestamp: new Date().getTime() })
    this.name = 'UnprocessableEntityException'
  }
}

export class UnauthorizedException extends CustomError {
  constructor(error: CustomErrorResponse) {
    super(401, { ...error, timestamp: new Date().getTime() })
    this.name = 'UnauthorizedException'
  }
}

export class ForbiddenException extends CustomError {
  constructor(error: CustomErrorResponse) {
    super(403, { ...error, timestamp: new Date().getTime() })
    this.name = 'ForbiddenException'
  }
}

export class NotFoundException extends CustomError {
  constructor(error: CustomErrorResponse) {
    super(404, { ...error, timestamp: new Date().getTime() })
    this.name = 'NotFoundException'
  }
}

export class ConflictException extends CustomError {
  constructor(error: CustomErrorResponse) {
    super(409, { ...error, timestamp: new Date().getTime() })
    this.name = 'ConflictException'
  }
}

export class MethodNotAllowedException extends CustomError {
  constructor(error: CustomErrorResponse) {
    super(405, { ...error, timestamp: new Date().getTime() })
    this.name = 'MethodNotAllowedException'
  }
}

export class GoneException extends CustomError {
  constructor(error: CustomErrorResponse) {
    super(410, { ...error, timestamp: new Date().getTime() })
    this.name = 'GoneException'
  }
}

export class InternalServerErrorException extends CustomError {
  constructor(error: CustomErrorResponse) {
    super(500, { ...error, timestamp: new Date().getTime() })
    this.name = 'InternalServerErrorException'
  }
}

export const handleErrorResponse = (error: unknown, req: PayloadRequest) => {
  const span = trace.getActiveSpan()
  const spanContext = span?.spanContext()

  if (span) {
    span.recordException(error as Exception)
  }

  if (req) {
    const context = {
      trace_flags: spanContext?.traceFlags,
      span_id: spanContext?.spanId,
      trace_id: spanContext?.traceId,
      ...spanContext,
    }

    const errorPayload =
      // eslint-disable-next-line no-nested-ternary
      error instanceof Error
        ? { name: error.name, message: error.message, stack: error.stack }
        : typeof error === 'object' && error !== null
          ? error
          : { error }

    req.payload.logger.error({ ...context, ...errorPayload }, `${req.method} ${req.pathname}`)
  }

  if (error instanceof CustomError) {
    return NextResponse.json(
      { ...error.errorResponse, code: error.errorResponse.code },
      { status: error.status },
    )
  }

  if (isAxiosError(error) && isBaseErrorResponse(error.response?.data)) {
    return NextResponse.json(
      {
        code: error.response.data.code,
        message: error.response.data.message,
        correlationId: error.response.data.correlationId,
        subErrors: error.response.data.subErrors,
        timestamp: error.response.data.timestamp,
      },
      { status: error.response.status },
    )
  }

  if (error instanceof z.ZodError) {
    return NextResponse.json(
      {
        code: ErrorCode.BAD_REQUEST,
        message: 'Validation error',
        subErrors: error.errors,
        timestamp: new Date().getTime(),
      },
      { status: 400 },
    )
  }

  return NextResponse.json(
    {
      code: ErrorCode.INTERNAL_SERVER_ERROR,
      message: 'Internal Server Error',
      timestamp: new Date().getTime(),
    },
    { status: 500 },
  )
}
```

## Request lifecycle

```
endpoints.config.ts                          declares the endpoint:
  defineEndpoint<Req, Res>({ path, method,     binds OpenAPI types
    handler: pipeHandlers(                      composes middleware (right-to-left)
      withLoggerHandler,                        ─┐
      withClientInfoHandler,                     │ run order (left-to-right)
      withAuthHandler([Permission.X]),          ─┘
    )(getEntity),                               base handler runs last
  })
        │
        ▼
server/handlers/{name}.handler.ts            the handler:
  try {
    const req = schema.parse(req.query/data)   1. validate (server/schemas)
    const result = await req.payload.find(...)  2. Local API (req.payload) — the DEFAULT
    return Response.json(transform(result))     3. shape the typed response
  } catch (error) {
    return handleErrorResponse(error, req)      4. funnel ALL errors here
  }
```

A handler always: validates with a request schema, reads/writes via the **Local API**
(`req.payload.*`), returns with `Response.json`, and wraps the body in `try/catch` →
`handleErrorResponse`. Throw the typed exception (`BadRequestException`, `NotFoundException`, …) for
expected failures.

```ts
// payload/features/article/server/handlers/article.handler.ts
import type { PayloadRequest } from 'payload'

import { getArticleListRequestSchema } from '@/payload/features/article/server/schemas'
import type { AdminGetArticleListResponse } from '@/payload/features/article/interfaces'
import { handleErrorResponse } from '@/payload/utils/server'
import { MAX_LIMIT } from '@/shared/constants'

export const getArticleList = async (req: PayloadRequest) => {
  try {
    const request = getArticleListRequestSchema.parse(req.query ?? {})
    const limit = Math.min(request.limit, MAX_LIMIT)

    const result = await req.payload.find({
      collection: 'articles',
      overrideAccess: true,
      page: request.page,
      limit,
      ...(request.ids ? { where: { id: { in: request.ids } } } : {}),
    })

    const response: AdminGetArticleListResponse = {
      data: result.docs,
      pagination: {
        totalRecords: result.totalDocs,
        totalPages: result.totalPages,
        currentPage: result.page ?? 1,
        nextPage: result.nextPage ?? null,
        previousPage: result.prevPage ?? null,
      },
    }
    return Response.json(response)
  } catch (error) {
    return handleErrorResponse(error, req)
  }
}
```

## Resulting structure

```
payload/
  utils/
    server.ts                # barrel: endpoint + error + handler + pipe
    endpoint.util.ts         # defineEndpoint
    error.util.ts            # CustomError + typed exceptions + handleErrorResponse
    handler.util.ts          # withLogger/withClientInfo/withAuth/withApiKey + requireAdminAuth
    pipe.util.ts             # pipeHandlers
  features/{feature}/
    endpoints.config.ts      # declares endpoints (defineEndpoint + pipeHandlers)
    server/
      handlers/   {name}.handler.ts        + index.ts
      hooks/      use{Behavior}.ts          + index.ts   (collection hooks, default export)
      schemas/    {name}.schema.ts          + index.ts
      services/   {name}.service.ts (+ __mocks__/) + index.ts
      utils/      {name}.util.ts (+ .spec.ts)     + index.ts
      auth-strategies/  {idp}.strategy.ts    + index.ts  → [[add-keycloak-auth-to-payload]]
```

## Rules

- ✅ Do: import server foundations from `@/payload/utils/server` only; **never** from a client
  component (it pulls in secrets, Node `crypto`, OTel).
- ✅ Do: validate every request with a `server/schemas` zod schema; type the response with a
  `interfaces/` interface.
- ✅ Do: read/write via the **Local API** (`req.payload.*`) by default. The external-axios / BFF
  variant (`payload/libs/server`) is opt-in, per [[nextjs-with-payload]].
- ✅ Do: wrap every handler body in `try/catch` → `handleErrorResponse`; throw typed exceptions.
- ✅ Do: list HOCs in `pipeHandlers` in **run order** (logger → client-info → auth → handler).
- ✅ Do: signal **every** error path — including HOCs and config/setup failures (e.g. a missing
  `API_KEY`) — by **throwing a typed exception** (`InternalServerErrorException`,
  `UnauthorizedException`, …) so `handleErrorResponse` renders it uniformly.
- ❌ Don't: hand-build an error response with `Response.json({ code, message }, { status })` —
  that bypasses logging, OTel span recording, and the company error shape. `Response.json` is for
  **success** bodies only.
- ❌ Don't: `console.*` in handlers — use `req.payload.logger`.
- ❌ Don't: confuse `server/services/` (server → external) with the feature's client
  `services/` (browser → our endpoints).
- ❌ Don't: give client React hooks a default export, or collection hooks a named export — the
  export style is how the two are told apart.

## Gotchas

- **`pipeHandlers` is right-to-left.** `reduceRight` means the **rightmost** HOC wraps the handler
  first, so it is **outermost-last** — but at runtime the list still executes top-to-bottom. Write
  the array in the order you want them to run.
- **`use`-prefix ≠ React hook.** `server/hooks/use*.ts` are Payload collection hooks
  (default-exported). Don't import them as React hooks. (The [[nextjs-with-payload]] standard
  previously mislabeled these as generic "server logic" — they are collection lifecycle hooks.)
- **`__mocks__/` (plural) is the standard** per [[nextjs]]. Current code drifts to `__mock__/`
  (singular) — treat that as a migration debt, not the convention.
- **Schema-first request, interface response** is intentional and differs from [[form]]. Don't
  `z.infer` the response from a request schema.

## Related standards

- [[nextjs-with-payload]] — base structure; its data-layer section points here for the server side.
- [[add-keycloak-auth-to-payload]] — builds on this pattern; owns `auth-strategies/` and the auth
  endpoints.
- [[nextjs]] — the client data-layer counterpart (service → hook → transform) and naming.
- [[form]] — interface-first client form schemas; contrast with server schema-first requests.
