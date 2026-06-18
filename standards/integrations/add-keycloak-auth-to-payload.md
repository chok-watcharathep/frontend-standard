---
id: add-keycloak-auth-to-payload
title: Add Keycloak Auth to Payload
category: integration
status: draft
applies_to: [nextjs-with-payload]
requires: [nextjs-with-payload, server-data-layer]
conflicts_with: []
tags: [auth, keycloak, payload, security, oidc]
updated: 2026-06-18
---

# Add Keycloak Auth to Payload

## Summary

Replace Payload's built-in local-strategy login with an external **identity provider (IdP)** via a
custom **auth strategy**, plus the cookie-based login / refresh / logout / change-password endpoints
and the collection hooks that keep the IdP and the `users` collection in sync. **Keycloak is the
project-specific example** here — the auth-strategy slot itself is IdP-agnostic; swap the
strategy + service implementation for a different IdP.

## When to use

- Admin (and service) authentication must be delegated to an external IdP rather than Payload's
  own password store.

## When not to use

- The project is happy with Payload's built-in local strategy (email + password in Payload). Then
  none of this is needed.
- You only need a custom **endpoint** (not authentication) — see [[server-data-layer]].

## Prerequisites

- [[nextjs-with-payload]] — the base structure, collections, and `private-environment.config`.
- [[server-data-layer]] — this integration's handlers/services/schemas/hooks follow that pattern
  and compose with `pipeHandlers` + `handleErrorResponse` + the typed exceptions.

## The auth-strategy slot (IdP-agnostic)

Payload lets you replace authentication with one or more **custom strategies**. A strategy is a
plain object with a `name` and an `authenticate({ headers, payload }) => { user }` function; Payload
calls it on every request and trusts the `user` it returns. Register strategies on the collection
and disable the local strategy:

```ts
// payload/features/user/collections/user.collection.ts
import { keycloakStrategy } from '@/payload/features/user/server/auth-strategies'

const Users: CollectionConfig = {
  slug: 'users',
  auth: {
    disableLocalStrategy: true,        // no Payload password store
    strategies: [keycloakStrategy],    // our IdP strategy
  },
  // ...
}
```

**Fail-closed** is the rule for any strategy: verify the token's signature **and** confirm the
session is still live with the IdP before returning a user; on any error return `{ user: null }`,
never throw. Swapping IdPs means writing a new `{idp}.strategy.ts` with the same shape.

## Keycloak strategy (project-specific)

`payload/features/user/server/auth-strategies/keycloak.strategy.ts`. Verifies the JWT against
Keycloak's JWKS, then **introspects** so revoked sessions are rejected before token expiry, then
loads the Payload user. Accepts a `Bearer` header or the access-token cookie.

```ts
import { createRemoteJWKSet, jwtVerify, type JWTPayload } from 'jose'
import { parseCookies } from 'payload'
import type { BasePayload } from 'payload'

import { KEYCLOAK_JWKS_URL, keycloakIntrospectToken } from '@/payload/features/user/server/services'
import privateEnvironmentConfig from '@/private-environment.config'
import { AdminLocalStorageKey } from '@/shared/enums'

let JWKS: ReturnType<typeof createRemoteJWKSet> | null = null
const getJWKS = () => {
  if (!JWKS) JWKS = createRemoteJWKSet(new URL(KEYCLOAK_JWKS_URL)) // memoize across requests
  return JWKS
}

const KEYCLOAK_ISSUER = `${privateEnvironmentConfig.KEYCLOAK_SERVER_URL}/realms/${privateEnvironmentConfig.KEYCLOAK_REALM}`

const verifyAndFindUser = async (token: string, payload: BasePayload) => {
  let claims: JWTPayload
  try {
    const result = await jwtVerify(token, getJWKS(), { issuer: KEYCLOAK_ISSUER })
    // Keycloak sets aud to "account" by default; verify azp (authorized party) instead
    if (result.payload.azp !== privateEnvironmentConfig.KEYCLOAK_CLIENT_ID) {
      throw new Error(`unexpected azp claim: ${result.payload.azp}`)
    }
    claims = result.payload
  } catch (err) {
    payload.logger.error({ err }, 'keycloak strategy: jwtVerify failed')
    return null
  }

  // Fail closed: introspect so revoked sessions reject before exp
  const active = await keycloakIntrospectToken(token, payload.logger)
  if (!active) {
    payload.logger.warn('keycloak strategy: token inactive (revoked or session ended)')
    return null
  }

  // Keycloak maps the userProfileId attribute to the snake_case claim user_profile_id
  const userProfileId = (claims.user_profile_id ?? claims.userProfileId) as string | undefined
  if (!userProfileId) return null

  try {
    const user = await payload.findByID({
      collection: 'users',
      id: userProfileId,
      depth: 0,
      overrideAccess: true,
    })
    if (!user) return null
    user.collection = 'users'
    return user
  } catch (err) {
    payload.logger.error({ err }, 'keycloak strategy: findByID failed')
    return null
  }
}

export const keycloakStrategy = {
  name: 'keycloak',
  authenticate: async ({ headers, payload }: { headers: Headers; payload: BasePayload }) => {
    const cookies = parseCookies(headers)
    const bearerHeader = headers.get('authorization')
    const bearerToken = bearerHeader?.startsWith('Bearer ') ? bearerHeader.slice(7) : null

    if (bearerToken) {
      const user = await verifyAndFindUser(bearerToken, payload).catch(() => null)
      if (user) return { user }
      // Bearer not a valid Keycloak token (e.g. Payload-internal JWT) → fall through to cookie,
      // so Payload's own admin-UI calls keep working.
    }

    const cookieToken = cookies.get(AdminLocalStorageKey.ACCESS_TOKEN) ?? null
    if (!cookieToken) return { user: null }

    const cookieUser = await verifyAndFindUser(cookieToken, payload).catch(() => null)
    return { user: cookieUser }
  },
}
```

## Auth endpoints

Declared in the feature's `endpoints.config.ts` (per [[server-data-layer]]). Login / refresh /
logout run **without** `withAuthHandler` (the user isn't authenticated yet — they only get the
logger + client-info HOCs); change-password runs **behind** `withAuthHandler([])`:

```ts
// collectionEndpoints (mounted under the users collection)
{ path: '/login',         method: 'post', handler: pipeHandlers(withLoggerHandler, withClientInfoHandler)(keycloakLogin) },
{ path: '/refresh-token', method: 'post', handler: pipeHandlers(withLoggerHandler, withClientInfoHandler)(keycloakRefresh) },
{ path: '/logout',        method: 'post', handler: pipeHandlers(withLoggerHandler, withClientInfoHandler)(keycloakLogoutHandler) },

// endpoints
defineEndpoint<AdminChangePasswordRequest, void>({
  path: '/admin/change-password',
  method: 'patch',
  handler: pipeHandlers(withLoggerHandler, withClientInfoHandler, withAuthHandler([]))(changePassword),
})
```

Tokens are issued as **HttpOnly cookies**. The handlers (`server/handlers/keycloak-auth.handler.ts`)
set / clear them with shared helpers:

```ts
const COOKIE_OPTIONS = 'HttpOnly; Secure; SameSite=Lax; Path=/'

const setCookieHeader = (name: string, value: string, maxAge: number) =>
  `${name}=${value}; ${COOKIE_OPTIONS}; Max-Age=${maxAge}`

const clearCookieHeader = (name: string) =>
  `${name}=; ${COOKIE_OPTIONS}; Expires=Thu, 01 Jan 1970 00:00:00 GMT`
```

- **`keycloakLogin`** — parse `keycloakLoginRequestSchema`, confirm the user exists in Payload,
  exchange credentials with Keycloak (`keycloakLoginWithPassword`), record login history, return the
  user + access token and set the access/refresh cookies.
- **`keycloakRefresh`** — read the refresh cookie, call `keycloakRefreshAccessToken`, reset cookies.
- **`keycloakLogoutHandler`** — call `keycloakLogout` with the refresh token, record logout history,
  clear cookies.
- **`changePassword`** — verify the current password against Keycloak, enforce the password policy
  (`isValidKeycloakPassword`), `keycloakSetUserPassword`, then `keycloakClearAllSessions` and
  **re-login** to issue fresh tokens so the caller's own session survives the global session purge.

## Supporting pieces

| Path                                              | Role                                                                       |
| ------------------------------------------------- | -------------------------------------------------------------------------- |
| `server/services/keycloak.service.ts`             | Keycloak admin client (memoized, `client_credentials`) + REST helpers      |
| `server/services/admin-login-history.service.ts`  | Writes login/logout audit rows                                             |
| `server/services/__mocks__/keycloak.mock.ts`      | Test doubles for the Keycloak service                                      |
| `server/utils/keycloak-token.util.ts`             | `getKeycloakToken(req)` — Bearer-or-cookie extraction                      |
| `server/utils/password-policy.util.ts`            | `isValidKeycloakPassword` (length + upper + lower + digit)                 |
| `server/schemas/keycloak-auth.schema.ts`          | `keycloakLoginRequestSchema`                                               |
| `server/schemas/change-password.schema.ts`        | `changePasswordRequestSchema`                                              |

The Keycloak admin client is created once and re-authenticated lazily on a threshold (a module-level
singleton), so concurrent calls share one token:

```ts
let adminClient: KcAdminClient | null = null
let isAuthenticated = false
let lastAuthTime = 0
let authPromise: Promise<void> | null = null
const TOKEN_REFRESH_THRESHOLD = 50_000

export const getAdminClient = async (): Promise<KcAdminClient> => {
  if (!adminClient) {
    adminClient = new KcAdminClient({
      baseUrl: privateEnvironmentConfig.KEYCLOAK_SERVER_URL,
      realmName: privateEnvironmentConfig.KEYCLOAK_REALM,
    })
  }
  const now = Date.now()
  if (!isAuthenticated || now - lastAuthTime > TOKEN_REFRESH_THRESHOLD) {
    if (!authPromise) {
      authPromise = adminClient
        .auth({
          grantType: 'client_credentials',
          clientId: privateEnvironmentConfig.KEYCLOAK_ADMIN_API_CLIENT_ID,
          clientSecret: privateEnvironmentConfig.KEYCLOAK_ADMIN_API_CLIENT_SECRET,
        })
        .then(() => {
          isAuthenticated = true
          lastAuthTime = Date.now()
        })
        .finally(() => {
          authPromise = null
        })
    }
    await authPromise
  }
  return adminClient
}
```

## Collection sync hooks

The IdP and the `users` collection are kept in sync with **collection hooks** (per
[[server-data-layer]]'s `server/hooks/` — `use`-prefixed, default-exported). Wired on the
collection:

```ts
hooks: {
  beforeOperation: [useKeycloakUserSync],         // stage create payload / push profile+password updates
  beforeChange:    [usePreventEmailChange, useStampUserAudit],
  afterChange:     [useKeycloakUserCreate],        // create the KC user once Payload has the doc id
  afterDelete:     [useKeycloakUserDelete],
  afterMe:         [useKeycloakMe],
}
```

- **Two-phase create.** `useKeycloakUserSync` (`beforeOperation`) can't know the new doc id yet, so
  on `create` it stashes the plaintext credentials in `req.context._kcPendingCreate`;
  `useKeycloakUserCreate` (`afterChange`) creates the Keycloak user once Payload assigns the id.
- **Escape hatch.** `req.context._skipKeycloakProvisioning === true` skips sync (e.g. seed scripts).
- **`usePreventEmailChange`** pins `data.email = originalDoc.email` on update (email is the IdP key).

## Environment

Add to the zod-validated `private-environment.config.ts` (server-only), per [[nextjs-with-payload]]:

```
KEYCLOAK_SERVER_URL
KEYCLOAK_REALM
KEYCLOAK_CLIENT_ID
KEYCLOAK_ADMIN_API_CLIENT_ID
KEYCLOAK_ADMIN_API_CLIENT_SECRET
```

## Resulting structure

```
payload/features/user/
  collections/user.collection.ts     # auth.disableLocalStrategy + strategies + sync hooks
  endpoints.config.ts                 # login / refresh / logout / change-password
  server/
    auth-strategies/keycloak.strategy.ts
    handlers/keycloak-auth.handler.ts user.handler.ts
    hooks/  useKeycloakUserSync.ts useKeycloakUserCreate.ts useKeycloakUserDelete.ts
            useKeycloakMe.ts usePreventEmailChange.ts useStampUserAudit.ts
    schemas/ keycloak-auth.schema.ts change-password.schema.ts
    services/ keycloak.service.ts admin-login-history.service.ts __mocks__/
    utils/   keycloak-token.util.ts password-policy.util.ts
```

## Rules

- ✅ Do: keep strategies **fail-closed** — verify signature **and** introspect; return
  `{ user: null }` on any failure, never throw out of `authenticate`.
- ✅ Do: issue tokens as `HttpOnly; Secure; SameSite=Lax` cookies; never expose refresh tokens to
  client JS.
- ✅ Do: set `disableLocalStrategy: true` once a custom strategy owns auth.
- ✅ Do: put every IdP secret in `private-environment.config` (server-only).
- ❌ Don't: import `auth-strategies/`, the Keycloak service, or the auth handlers into client code.
- ❌ Don't: hand-roll a second token-extraction path — use `getKeycloakToken(req)`.

## Gotchas

- **Verify `azp`, not `aud`.** Keycloak defaults `aud` to `"account"`, so audience checks fail;
  assert the authorized-party claim (`azp === KEYCLOAK_CLIENT_ID`) instead.
- **Introspect to fail closed.** Signature-valid tokens stay "valid" until `exp`; without
  introspection a revoked/logged-out session keeps working until expiry.
- **Re-login after `clearAllSessions`.** Changing a password clears *all* Keycloak sessions
  including the caller's — re-login (or 204 + client redirect to login) so the current session
  isn't silently killed.
- **Memoize JWKS.** `createRemoteJWKSet` is module-level cached; recreating it per request hammers
  the JWKS endpoint.
- **Two-phase create ordering.** The KC user is created in `afterChange`, not `beforeOperation` —
  don't move it earlier expecting the doc id to exist.

## Related standards

- [[server-data-layer]] — the server pattern this integration is built on (handlers, schemas,
  services, hooks, `utils/server.ts`).
- [[nextjs-with-payload]] — base structure, collections, env config.
- [[nextjs]] — client login form + admin session handling consume these endpoints.
- [[form]] — the admin login / change-password forms that call these endpoints.
