---
title: Auth Session Issuance Implementation Plan
lastUpdate: 2026-09-05
category: architecture
lang: en
status: implemented
---

# Auth Session Issuance Implementation Plan

## Goal

Complete password and OAuth session issuance from the current `services/auth` structure.
Authentication starts in one configured default tenant, resolves an existing local user, and
issues a session only when that user is allowed to sign in. This plan deliberately excludes
tenant membership, tenant selection, and account provisioning.

The resulting initial model is single-tenant at login time. Multi-tenant selection can be
added later using the approach described in `multi-tenant-auth-design.md`, without changing
the session issuance boundary.

## Mandatory Implementation Rule
When agent needs to invent or speculate work beyond this design doc, ask question to the
developer before take a specific direction.

## Current Starting Point

The service already has these structural pieces:

- `PasswordService.authenticate()` and `OAuthService.resolveSessionCandidate()` return typed
  architecture-phase errors.
- `SessionService.create()` assembles a `SessionView` value, but its token dependency is
  still deferred. `read`, `refresh`, and `terminate` are also deferred.
- `POST /auth/v1/session` invokes password authentication and then session creation.
- The OAuth callback invokes `OAuthService.complete()`, resolves a session candidate, and then
  creates a session.
- `Users` is tenant-scoped through Core's AsyncLocalStorage execution context.
- `Users.read(entityId)` looks up an entity ID. `Users.readByEmail(email)` looks up the
  existing `email` field within the current tenant (Index `tenant_email` is in place).
- The current `User` type contains `entityId`, `tenantId`, `username`, `email`,
  `passwordHash`, `roles`.
- The API error boundary maps OAuth flow errors to `400` and provider failures to `502`.

## Scope Decisions

### Default tenant and anonymous actor

Add two required authentication configuration values:

- `defaultTenantId`: the tenant ID used by unauthenticated login and OAuth callback requests.
- `defaultActorId`: the actor ID recorded while credentials are being verified, such as
  `auth:anonymous`.

The default tenant must already exist in the database. This phase does not create, list, or
expose tenants. It also does not accept a tenant ID from the session or OAuth routes.

Every request receives an initial Core execution context in an API-wide `onRequest` hook:

```ts
{
  authenticated: {
    // TODO: read the GET 'v1/tenants' endpoint, rill actorId with tenantName
    tenantId: config.auth.defaultTenantId,
    actorId: config.auth.defaultActorId
  },
  request: { requestId: request.id }
}
```

This makes `User.readByUsername` and `Users.readByEmail()` usable during public authentication
routes. Existing protected route handling may then replace that baseline context with the 
verified session grant.

### Session candidate

For a user found in the default tenant, construct the session candidate as follows:

```ts
{
  subject: user.entityId,
  actorId: user.entityId,
  tenantId: user.tenantId
}
```

`entityId` is stable and is safer than using a mutable username in access tokens or audit
fields. The external OAuth provider subject remains useful for provider processing but is not
the local session subject in this phase.

After a later access-token verification, the request carries a `SessionGrant` rather than a
`SessionCandidate`. It contains the `sessionId`, `subject`, `actorId`, and `tenantId` needed by
protected routes. The subject remains consistently named across the authentication types.

## Error Policy

Add these shared `CustomError` classifications in `tools` and map them in the auth API error
handler:

- `AUTHENTICATION_FAILED` → `401`: unknown user, wrong password, or invalid and revoked
  refresh token.
- `AUTHORIZATION_DENIED` → `403`: an inactive or suspended local user, or an OAuth identity
  without an eligible local user.
- `OAUTH_FLOW_ERROR` → `400`: invalid state, missing callback data, or user-denied consent.
- `EXTERNAL_API_ERROR` → `502`: provider network, token, or user-info failure.

Use the same public `401` message for an unknown username and a wrong password. For `403`, use
a generic message such as `This account is not allowed to sign in`; do not disclose whether the
user was absent, inactive, suspended, or has no OAuth linkage. Log the internal reason with the
request ID, but never log a password, refresh token, access token, or password hash.

Add `401`, `403`, and `502` to the relevant route response schemas. The common problem schema
is already appropriate for these RFC 9457 responses.

## Implementation Steps

### 1. Extend configuration and establish the baseline context

1. Extend `AuthConfig`, its Zod schema, `config/local.json`, and configuration JSDoc with
   `defaultTenantId` and `defaultActorId`.
2. Change the execution-context plugin so an API-wide `onRequest` hook binds the baseline
   context for every request using `runWithExecutionCtx`.
3. Change the protected-route context hook to replace the baseline values with
   `request.sessionGrant.actorId` and `request.sessionGrant.tenantId` after token verification.
4. Verify that public session and OAuth routes can call tenant-scoped `Users` methods without a
   missing execution-context error.

### 2. Make user lookup suitable for authentication

1. Retain `Users.readByUsername()` and `Users.readByEmail` as the only authentication lookup 
   methods; it already applies the tenant from the execution context.
2. Ensure the existing unique `(tenantId, username)` index remains the lookup index.
3. Do not add a repository method that bypasses the execution context or accepts an arbitrary
   tenant ID in this phase.

### 3. Complete password authentication

1. Inject `Users` and a password-hash verifier into `PasswordService` from `bootstrap.ts`.
2. In `authenticate(credentials)` call `users.readByEmail()` in the baseline default-tenant 
   context.
3. If no user exists throw `AUTHENTICATION_FAILED`. Enumeration attacks mitigation will be 
   deferred. However, a //NOTE: comment about source of enumeration attack should be added.
4. Verify the supplied password against `user.passwordHash` using node:crypto that ships 
   crypto.argon2.
5. On a mismatch, throw `AUTHENTICATION_FAILED` with the same public detail used for an unknown
   user.
6. If the password is valid but `user.status` is not `active`, throw
   `AUTHORIZATION_DENIED`.
7. Return the local `SessionCandidate` using the user's existing `entityId` and `tenantId`.
8. Keep raw passwords inside the method only; do not return, persist, log, or place them in an
   error cause.

Password reset and administrator password setting remain separate workflows. They must not be
silently implemented as part of login because they require reset-token, delivery, and elevated
authorization policies that are outside this plan.

### 4. Resolve an OAuth identity to the same local principal

1. Keep `OAuthService.complete()` responsible only for state validation, code exchange, and
   provider identity retrieval.
2. Inject `Users` into `OAuthService` from bootstrap.ts, for use by
   `resolveSessionCandidate()`.
3. In `resolveSessionCandidate(session)`, require `identity.email` and call
   `users.readByEmail()` inside the baseline default-tenant context.
4. If the provider identity lacks an email or no active local user is found, throw
   `AUTHORIZATION_DENIED` with the generic public message. Do not provision a user.
5. If the matched user is `inactive` or `suspended`, throw the same `403` error.
6. Return the exact same local `SessionCandidate` shape used by password authentication.
7. Preserve the existing `502` behavior for Google or GitHub transport and malformed provider
   response failures. A valid provider identity that is not eligible for a local account is not
   a provider outage.

This allows password and OAuth to converge before session issuance without adding memberships,
external-identity records, or tenant discovery.

### 5. Add durable session storage

`SessionService.read`, `refresh`, and `terminate` require server-side state. Add a
tenant-scoped `SessionRecords` entity and dataset definition `src/entities/`. A session record
type (`SessionRecord`) should contain only the data needed to enforce lifecycle policy:

- `entityId` as the session ID;
- `tenantId`, `actorId`, and `subject` from the trusted session candidate;
- an expiry timestamp and termination timestamp or active status;
- a hash of the opaque refresh token, never the refresh token itself (use crypto.createHmac, 
  read env.JWT_HASH_SECRET, then use crypto.timingSafeEqual with buffer length check);
- refresh-token expiry and rotation metadata needed to invalidate old tokens.

Use the existing Core entity metadata and tenant-scoped CRUD conventions. Create a dedicated
`SessionRecords` entity policy rather than letting `SessionService` issue unscoped MongoDB
queries.

### 6. Complete session lifecycle and token dependencies

1. Inject the `SessionRecords` policy and `ITokenService` into `SessionService`.
2. In `create(candidate)`, create the durable `SessionRecord`, issue a signed access token and
   an opaque refresh token, persist only the refresh-token hash, and return `Session`.
3. Complete `ITokenService` with the already planned JOSE signing, verification, and JWKS key
   management. Access tokens must contain the session ID, subject, actor ID, tenant ID, issuer,
   audience, issued-at, and expiry claims.
4. In `read(sessionId)`, read the active session in the authenticated tenant and return safe
   `SessionView`. A missing or terminated session must not be treated as active.
5. In `refresh(refreshToken)`, hash and locate the refresh token, reject expired, revoked, or
   rotated credentials with `AUTHENTICATION_FAILED`, rotate the opaque token atomically, and
   issue a new access token from the stored session candidate.
6. In `terminate(sessionId)`, delete the session record through `SessionRecords.delete()`.
   The controller then clears the cookie as it already does.
7. Never rely on a JWT alone to make termination or refresh-token revocation effective; the
   session record is the lifecycle authority.

### 7. Compose dependencies and preserve transport boundaries

1. In `bootstrap.ts`, construct `Users` before `PasswordService` and `OAuthService` so both
   receive the same tenant-aware policy object.
2. Construct `SessionRecords` and inject it into `SessionService` together with `TokenService`.
3. Keep controllers unchanged except for any response-schema additions. They should continue to
   call authentication method → `SessionService.create()` → cookie/response mapping.
4. Do not expose the configured default tenant ID, anonymous actor ID, password hash, session
   record, or provider tokens through route responses.

## Completion Criteria

- Password login finds only an active user in the configured default tenant and issues one
  durable session and credential pair.
- OAuth login accepts a provider identity only when its normalized email matches an active
  local user's `username` in that same tenant, then issues the same kind of session.
- Unknown users and wrong passwords produce indistinguishable `401` responses.
- Inactive and suspended users, and OAuth identities without an eligible local user, produce a
  generic `403` response and no credentials.
- Provider or provider-response failures produce `502`; invalid OAuth callback state or denial
  produces `400`.
- Refresh and terminate operate against durable session state, not merely client-held tokens.
- All user and session entity access remains scoped by the execution context tenant.
- No membership collection, tenant enumeration endpoint, caller-provided tenant ID, or
  user-field expansion is introduced.

## Deferred Follow-Up

When multi-tenant login is required, extend this plan using the tenant validation and
protected tenant-discovery approach in `multi-tenant-auth-design.md`. At that point, resolve
the requested tenant before user lookup and bind it into the baseline execution context. Do not
infer a tenant from an email address or from an unscoped user lookup.
