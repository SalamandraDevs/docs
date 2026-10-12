---
title: User Deletion IDOR and Authorization Gaps Closing Plan
lastUpdate: 2026-09-25
lang: en
category: architecture
status: implemented
owner: Odair Trujillo
---

# User Deletion IDOR and Authorization Gaps Closing Plan

## Overview

The `DELETE /v1/users/:recordId` route in `services/auth` is guarded only by
`protectedRouteHooks`; that is, authentication plus execution-context binding, with no
authorization check. Any authenticated user can therefore delete any other user within their
tenant by supplying a different `recordId`; a broken-access-control / IDOR defect. The route is
the outlier among the user mutations: `POST /v1/users` is admin-gated and `PATCH /v1/users/:id`
is owner-or-admin gated, while `DELETE` inherited neither.

Investigating the fix surfaced three coupled gaps that must be closed together, plus one large
effort that is deliberately deferred:

1. Admin gate on user deletion, with protection against orphaning a tenant of its admins.
2. Refresh-token revocation on deletion and privilege loss, so a deleted user (or a demoted
   admin) loses standing instead of retaining it through refresh or an unexpired access token.
3. `bootstrapAdmin` treated as a real startup requirement, because it is the only mechanism
   that can originate an admin user.
4. Tenant provisioning and multi-tenant operation; deferred, tracked as AUTH-014.

This plan ships items 3, 1, and 2 in that order and records what matters about item 4 so the
groundwork does not have to be unwound later.

## Guiding architectural decision

Adopt one commitment that de-risks the whole sequence:

- `defaultTenantId` is the future platform / system tenant. Every record today is created
  under it because there is no tenant-provisioning flow and no tenant selection at login; the
  service is single-tenant in practice while multi-tenancy exists only as scaffolding
  (tenant-scoped record handlers and the reserved `tenant` role). Declaring `defaultTenantId`
  the platform tenant means items 1 to 3 are groundwork for item 4 rather than assumptions that
  item 4 must reverse.

This decision is a prerequisite for the plan and should be ratified before implementation.

## Why the admin surface is currently dormant

`admin` can only originate from the bootstrap seed. `UsersService.register()` hard-codes
`['user']`; `UsersService.create()` accepts `roles` but sits behind `adminProtectedRouteHooks`,
so it needs a pre-existing admin. That is a chicken-and-egg: the API cannot mint the first
admin. The bootstrap seed in `bootstrap.ts`, driven by `config.auth.bootstrapAdmin`, is the
only path that creates an admin from nothing. The `bootstrapAdmin` entry is currently absent
from `config/local.json` and is marked `.optional()` in the config schema, so no admin exists
today and every admin-gated route is unreachable. Item 3 must land first for items 1 and 2 to
be exercisable at all.

## Item 3; bootstrapAdmin as a startup requirement

Goal: guarantee the root admin of the platform tenant exists on every boot.

- Make `bootstrapAdmin` required in the Zod config schema by dropping its `.optional()`. A
  missing entry then fails at parse time and surfaces as `CONFIG_ERROR` through the existing
  loader (`config/local.ts` already wraps `serviceConfigSchema.parse` in a try/catch that maps
  any failure, including a `ZodError`, to `CustomError({ name: 'CONFIG_ERROR', ... })`). No
  hand-written presence check in `bootstrap.ts` is added; the schema is the single source of
  truth for the requirement.
- Keep the runtime resolution of the password in `bootstrap.ts`: the schema validates the
  `passwordEnv` variable name, while the secret value is read from that environment variable at
  boot and is never stored in `config/local.json`. The existing `CONFIG_ERROR` raised when the
  named variable is unset or empty stays, because that is an environment concern the schema
  cannot express.
- Add the `bootstrapAdmin` entry (`username`, `email`, `passwordEnv`) to the committed config.
  There is no auth deployment runbook today, so document the required environment variable
  named by `passwordEnv` where the deploy actually reads it: minimally an `.env.example` entry
  and a note in the auth service `README`. Creating a proper deployment runbook under
  `docs/en/infrastructure/` is a recommended follow-up but is out of scope for this plan.
- Preserve the current idempotency: when a user with the configured email already exists,
  reuse it and reject startup only when that user lacks the `admin` role.
- Keep the seed running under the baseline execution context (`defaultActorId`,
  `defaultTenantId`), consistent with the platform-tenant decision above.

Deliverables:

- Config schema change dropping `.optional()` on `bootstrapAdmin`; the loader is unchanged and
  already maps the parse failure to `CONFIG_ERROR`.
- `config/local.json` entry plus deployment-secret documentation.

## Item 1; admin gate on user deletion

Goal: restrict deletion to administrators and prevent locking a tenant out of user management.

- Replace `protectedRouteHooks(fastify)` with `adminProtectedRouteHooks(fastify)` on the
  `DELETE /v1/users/:recordId` route in `api/routes/v1/users/routes.ts`. This reuses the
  existing `requireAdminRole` decoration and needs no route-local hook, mirroring
  `POST /v1/users`.
- Protect the designated root administrator in `UsersService.delete`: read the target record
  and refuse the deletion when its email matches the configured bootstrap administrator,
  throwing `CustomError({ name: 'AUTHORIZATION_DENIED', ... })`. Because item 3 guarantees the
  bootstrap administrator exists and holds the `admin` role, protecting that one account
  guarantees the tenant is never orphaned of an admin. The comparison is case-insensitive and
  trimmed, matching the normalization the bootstrap seed already uses for `readByEmail`; the
  normalized email is injected into `UsersService`.
- A true per-tenant "last active admin" count was rejected during implementation because it is
  not portable: matching users whose `roles` array contains `admin` cannot be expressed through
  the backend-neutral CRUD contract, since both adapters filter by field equality only (Mongo
  `Filter<T>` and MySQL `field = ?`) and `roles` is an array. Protecting the designated root
  administrator is cheap (one read plus a string compare), backend-neutral, and
  forward-compatible: under item 4 the tenant-provisioning flow designates each tenant's own
  protected root admin.
- The controller keeps trusting the hook chain and passes `request.params.recordId` through;
  no ownership check is added, because self-deletion is out of scope for this plan and is left
  to a future high-level account-deletion flow.

Deliverables:

- Route hook swap for `DELETE /v1/users/:recordId`.
- Root-administrator guard in `UsersService.delete`, matching the target against the configured
  bootstrap administrator email injected into the service.

Notes:

- Access tokens keep a role snapshot until expiry, so an admin deleted or demoted elsewhere
  retains privilege until item 2 revokes their sessions and their access token expires. Item 2
  closes that window.

## Item 2; refresh-token revocation on user deletion and privilege loss

Goal: make user deletion and privilege loss take effect instead of leaving live credentials.

Current behavior confirmed in code:

- Authentication is stateless JWT verification (`TokensService.verify`); it never reads the
  session record and never checks that the user still exists.
- `SessionsService.refresh` looks up the session by refresh-token hash and rebuilds the
  candidate from the session record; it also never checks user existence. Deleting a user
  therefore revokes nothing today, because user deletion does not touch session records and
  refresh keeps minting tokens from the surviving session.
- Deleting a session record does invalidate its refresh token: `refresh` throws
  `AUTHENTICATION_FAILED` when the record is missing or inactive. So revocation must delete the
  deleted subject's session records.

Design: cascade from the aggregate that already knows the subject. `UsersService.delete` holds
the deleted user's `recordId`, which is the session `subject`. A user can hold several sessions
at once (multiple devices), so revocation is one-to-many; it cannot be a single record read.
Rather than enumerate with the paginated `find` (whose `FindResult` carries no cursor), delete
by subject directly, because `ICrud.delete` is match-and-remove-one and each deletion is
exactly what invalidates the associated refresh token.

Work:

- Add `deleteBySubject(subject)` to `SessionRecordsHandler`. Bind `tenantId` from
  `getExecutionCtx().authenticated` (the pattern used by `readByRefreshToken`) and loop
  `crud.delete({ subject, tenantId })` until it returns null, returning the count removed. This
  needs no paginated `find` and no cursor handling; add a safety cap on the loop if desired.
- Add `deleteBySubject(subject): Promise<number>` to the existing `ISessionsService`
  contract and implement it on `SessionsService`, delegating to
  `SessionRecordsHandler.deleteBySubject`. The same name flows across the layers, the way
  `recordId` passes transparently from transport to storage, rather than introducing a `revoke`
  synonym at the application boundary. Reuse the established session boundary rather than
  minting a one-method interface; the codebase already treats `ISessionsService` as the single
  session boundary and does not create a narrow interface per consumer.
- Inject `sessions: ISessionsService` into `UsersService`, typed as the interface exactly as
  `passwords: IPasswordsService` is today, and call `deleteBySubject` from `delete` after
  the user record is removed, so deletion cascades to the subject's sessions. Apply the same
  cascade to the admin-driven role/status change path in `UsersService.update`, which resolves
  the revocation behavior that `docs/users-service-plan.md` explicitly asked to be chosen and
  documented before role updates shipped.
- Compose the dependency in `bootstrap.ts` in `tokens` then `sessions` then `users` order;
  there is no cycle because `SessionsService` never depends on `UsersService`.
- Keep the access-token TTL short so the residual stateless window between revocation and token
  expiry stays small; document that window as the accepted residual risk.

Deliverables:

- `SessionRecordsHandler.deleteBySubject`, tenant-scoped delete-by-subject loop.
- `deleteBySubject` added to `ISessionsService` and implemented on `SessionsService`.
- `ISessionsService` injection into `UsersService` and cascade calls from `delete` and the
  privilege-change path in `update`.

Notes:

- Ordering: delete the user first, so `delete` confirms the record existed (it returns null
  otherwise), then revoke. If revocation throws, the residual gap is bounded by the
  access-token `exp`; log or alert on that failure. Revoking first is the alternative but can
  log a user out and then fail to delete them.
- Scope of revocation: for a deleted user, remove all their session records by subject
  regardless of status, as cleanup. For a demoted admin, the same call forces re-authentication
  so new tokens carry the updated roles.
- Scope by tenant from the start. `deleteBySubject` binds the current tenant, so a session in
  another tenant is never touched; a global cross-tenant by-subject delete would become
  incorrect the moment a subject can belong to more than one tenant under item 4.
- Cascade placement and cycle risk. The cascade adds a `UsersService` to `SessionsService`
  dependency; there is no cycle today because `SessionsService` (`refresh` and authentication)
  deliberately does not check user existence, which is precisely why this cascade is needed.
  The latent risk is a future change that adds the reverse edge, for example making `refresh`
  or authentication validate that the user still exists as an extra safeguard; that would close
  a `UsersService` and `SessionsService` circular dependency. If user-existence validation is
  ever wanted inside sessions, route it through a narrow user-read port or move revocation to
  lifecycle events (AUTH-017) rather than importing `UsersService` into `SessionsService`. When
  the future high-level account-deletion flow lands, it can become the caller of
  `deleteBySubject`, and
  the cascade moves up to that orchestrator with no change to `SessionsService`.

## Sequencing and acceptance

Order: 3, then 1, then 2.

- 3 first, because no admin exists until the seed is guaranteed, so 1 and 2 cannot be
  exercised without it.
- 1 next, because it closes the live IDOR with a small, self-contained change.
- 2 last, because it is the largest change and depends on having a real admin and a gated
  delete to protect.

Acceptance conditions (manual, per the repository testing policy):

- A non-admin authenticated caller receives `AUTHORIZATION_DENIED` from
  `DELETE /v1/users/:recordId`.
- An admin cannot delete the configured bootstrap administrator.
- After an admin deletes a user, that user's refresh token is rejected and no new access token
  can be minted; the user's access token stops working once it expires.
- The service refuses to start when `bootstrapAdmin` is absent, through the schema parse
  failure, and when the `passwordEnv` variable is unset or empty, through the runtime check.

## Forward compatibility with tenant provisioning (item 4)

Item 4 (AUTH-014 in `docs/ROADMAP.md`) is large and currently unspecified; its referenced
design document is missing. Gating the security fixes behind it would leave the IDOR and the
revocation gap open for the entire duration of that effort, so item 4 is deferred. Building 1
to 3 first carries low rework cost given the constraints in this plan. The relevant impacts:

- Admin gate (item 1); forward-compatible. `requireAdminRole` combined with tenant-scoped
  record handlers naturally becomes "a tenant admin acts within their own tenant" once
  sessions carry a real `tenantId`. No change to the gate is required for item 4.
- Root-administrator protection (item 1); forward-compatible. Item 4's tenant-provisioning flow
  designates each tenant's own protected root admin, generalizing the platform tenant's
  bootstrap
  administrator. A portable per-tenant "last active admin" count is not available today because
  role-array containment is not expressible through the backend-neutral CRUD contract.
- Refresh-token revocation (item 2); forward-compatible because `deleteBySubject` binds the
  current tenant. Item 4 may later extend revocation to a cross-tenant identity, but that is an
  extension, not a rewrite.
- `bootstrapAdmin` required (item 3); this is the item that hardens a single-tenant assumption.
  It is safe under the guiding decision that `defaultTenantId` becomes the platform tenant and
  `bootstrapAdmin` becomes that tenant's root admin. If the organization later chooses a pure
  tenant-per-customer model with no default tenant, `bootstrapAdmin` and `defaultTenantId`
  become legacy configuration and the existing default-tenant records require migration.

Work that item 4 must own regardless of this plan:

- Move role assignment from a global array on the user record to a per-tenant membership; a
  user is an admin in a specific tenant, not globally. `requireAdminRole` and the roles claim
  in the access token become tenant-aware at that point.
- Provide the tenant-provisioning flow that seeds each new tenant's first admin, analogous to
  what `bootstrapAdmin` does for the platform tenant.
- Migrate or reinterpret the records currently created under `defaultTenantId`.

## References

- `docs/users-service-plan.md`; admin role authorization, the bootstrap seed decision, and the
  request to choose and document session-revocation behavior for privilege changes.
- `docs/session-plan.md`; session lifecycle and refresh rotation.
- `docs/ROADMAP.md`; AUTH-014 multi-tenant login, AUTH-018 self-update authorization.
- `docs/en/architecture/current-patterns.md`; feature-module and hook composition conventions.
