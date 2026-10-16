---
title: Salamandra Devs Platform Roadmap
lastUpdate: 2026-10-16
lang: en
category: planning
status: active
---

# Salamandra Devs Platform Roadmap

Brief map of what each repository does and the status of its features. Verified against the
last update date source code; where a plan document disagrees with the code, the code wins and
the disagreement is listed under [Known discrepancies](#known-discrepancies).

## How to read this document

Every feature is one line with a stable ID, so people and agents can grep and cite it:

```text
- [status] ID Name; what it does. (refs: source files or plan documents)
```

- Status values: `implemented`, `planned`, `proposed`, `canceled`.
- `implemented`; present in the code on the main working tree.
- `planned`; approved in a plan document, not yet built.
- `proposed`; inferred or under discussion; no approved design.
- `canceled`; dropped; kept only for context.
- Scope: only `core/`, `tools/`, `services/auth`, `services/profiles`, `infra/` and `docs/`.
- Paths are relative to each repository root. Route paths are relative to the auth service
  root, before any gateway prefix.

Rules for agents editing this file:

- Change a status only after checking the code; cite the file or plan in `refs`.
- Never reuse or renumber an ID. Add new IDs at the end of a repository section.

## Repositories

| ID | Path | Package | Role | Depends on |
| --- | --- | --- | --- | --- |
| CORE | `core/` | `@slmdevs/core` | Backend-neutral records layer | TOOLS |
| TOOLS | `tools/` | `@slmdevs/tools` | Shared errors, guards, crypto | none |
| AUTH | `services/auth/` | `@slmdevs/auth` | Authentication service | CORE, TOOLS |
| PROFILES | `services/profiles/` | `@slmdevs/profiles` | Identity and entitlements | CORE, TOOLS |
| INFRA | `infra/` | none | Shared deployment stack; network, MongoDB, OpenBao, operator tooling | none |
| DOCS | `docs/` | none | Documentation and plans | none |

Dependency direction: services depend on `core` and `tools`; `core` depends on `tools`.
Architecture rules (ports and adapters, SOLID, naming) live in `docs/en/architecture/`.

## CORE: `@slmdevs/core`

Shared record and database foundation: backend-neutral contracts, adapters, tenant-aware
record handling, and request execution context.

- [implemented] CORE-001 Database contracts; connection lifecycle, CRUD, and dataset
  contracts. (refs: `src/database/contracts.ts`)
- [implemented] CORE-002 MongoDB adapter; connection, CRUD, dataset provisioning.
  (refs: `src/database/mongo/`)
- [implemented] CORE-003 MySQL adapter; connection, CRUD, dataset provisioning.
  (refs: `src/database/mysql/`)
- [implemented] CORE-004 Dataset provisioner; creates datasets and indexes from Zod-based
  definitions. (refs: `src/database/dataset-provisioner.ts`)
- [implemented] CORE-005 `RecordsHandler`; tenant scope, `recordId`, audit metadata, and
  deterministic cursor pagination ordered by `createdAt` then `recordId`, both descending.
  (refs: `src/records/records-handler.ts`)
- [implemented] CORE-006 Execution context; AsyncLocalStorage carrying actor, tenant, and
  request id. (refs: `src/utils/context.ts`)
- [proposed] CORE-007 MySQL schema conversion; depends on TOOLS-005. (refs: TOOLS-005)
- [implemented] CORE-008 Database provider and backend-bound adapters; `DatabaseProvider`
  (`IDatabaseProvider`) owns backend selection and connection lifecycle, exposing a
  discriminated `DatabaseHandle`, and `DatabaseAdapter` selects the backend once to
  `provision` dataset definitions (via `IDatasetProvisioner`) and `createCrud` `ICrud`
  adapters by name.
  (refs: `src/infra/database/database-provider.ts`,
  `src/infra/database/database-adapter.ts`)

## TOOLS: `@slmdevs/tools`

Small shared helpers with no application policy. Exports `@slmdevs/tools/shared` (browser
safe) and `@slmdevs/tools/node` (Node only).

- [implemented] TOOLS-001 `CustomError` and error names. (refs: `src/shared/errors.ts`)
- [implemented] TOOLS-002 `makeApiRequest` fetch helper. (refs: `src/shared/requests.ts`)
- [implemented] TOOLS-003 Type guards. (refs: `src/shared/typeguards.ts`)
- [implemented] TOOLS-004 `zodToMongoSchema`. (refs: `src/shared/transform.ts`)
- [proposed] TOOLS-005 `zodToMySqlSchema` real conversion; the current function is a
  placeholder. (refs: `docs/zod-to-mysql-schema.md`)
- [implemented] TOOLS-006 JWT signing input and timing-safe hash comparison.
  (refs: `src/node/crypto.ts`)
- [implemented] TOOLS-007 `loadEnvVariables`; reads a config schema's `.meta({ env })`
  fields from `process.env`, validates each with its own field schema, and returns the
  resolved values without touching the config object, throwing `ENV_VAR_ERROR` on a missing
  or malformed variable. (refs: `src/node/environment.ts`)

## AUTH: `@slmdevs/auth`

Fastify REST API that issues and verifies sessions through password credentials and OAuth
providers. Layers: `api/` (transport), `application/` (policy), `records/` (persistence),
`infrastructure/` (OpenBao keys, OAuth providers).

- [implemented] AUTH-001 Service architecture; composition root, encapsulated route plugins,
  execution-context binding, OpenAPI. (refs: `docs/auth-plan.md`, `src/runtime/`)
- [implemented] AUTH-002 Password sign-in; Argon2 verification in the default tenant.
  Route: `POST /v1/session`. (refs: `src/application/passwords-service/`)
- [implemented] AUTH-003 OAuth authorization-code flow; Google and GitHub providers.
  Routes: `POST` and `GET /v1/oauth/:provider/authorize`, `GET /v1/oauth/:provider/callback`.
  (refs: `src/application/oauth-service/`, `src/infra/oauth/`)
- [implemented] AUTH-004 Sessions; issue, read, terminate, and refresh with rotating opaque
  refresh tokens stored as HMAC hashes. Routes: `GET`, `DELETE /v1/session`,
  `POST /v1/session/refresh`. (refs: `docs/session-plan.md`,
  `src/application/sessions-service/`)
- [implemented] AUTH-005 Access tokens; ES256 JWT issue and verification into a session
  grant. (refs: `src/application/tokens-service/`)
- [implemented] AUTH-006 Durable signing keys; OpenBao Transit signing against an
  operator-owned key lifecycle, with JWKS at `GET /.well-known/jwks.json`. (refs: AUTH-023,
  `docs/signing-plan.md`, `services/auth/src/infra/keys/`, `docs/en/howto/openbao/`)
- [implemented] AUTH-007 Users; create, register, read, update, delete, and find through
  `UsersService`. Routes under `/v1/users`. (refs: `docs/users-service-plan.md`,
  `docs/users-service-completion.md`)
- [implemented] AUTH-008 Administrator password set; `POST /v1/password/set`.
  (refs: `src/application/passwords-service/`)
- [implemented] AUTH-009 Cursor pagination plugin; request decode and response `nextCursor`.
  (refs: `docs/entity-cursor-plugin.md`, `src/api/plugins/record-cursor-codec.ts`)
- [implemented] AUTH-010 Health and OpenAPI; `GET /health`, `GET /openapi.json`, UI at `/docs`.
  (refs: `src/api/routes/health/`, `src/api/plugins/openapi.ts`)
- [planned] AUTH-011 Password reset; `POST /v1/password/reset` exists but always fails with
  `ARCHITECTURE_NOT_IMPLEMENTED`. (refs: `src/application/passwords-service/`)
- [proposed] AUTH-012 Timing-safe unknown-user path; reduce user-enumeration signal.
  (refs: NOTE in `passwords-service.ts`)
- [proposed] AUTH-013 OAuth PKCE for the authorization-code flow; per-attempt `code_verifier`
  and S256 `code_challenge`, requiring a durable pending-authorization store and a per-provider
  PKCE capability (Google supported; GitHub to verify). (refs:
  `src/application/oauth-service/`, TODO in `src/application/contracts.ts`)
- [proposed] AUTH-014 Multi-tenant login and tenant selection; login is single default tenant
  today. (refs: `docs/session-plan.md`)
- [canceled] AUTH-015 Scheduled signing-key rotation; rejected. AUTH-023 moved the whole key
  lifecycle to the operator, so rotation is an OpenBao CLI procedure and the auth service has
  no rotation to schedule. (refs: `docs/en/howto/openbao/README.md`)
- [proposed] AUTH-016 Email verification.
- [proposed] AUTH-017 User lifecycle events such as `auth.user.created`.
  (refs: `docs/es/arquitectura/convenciones-nombres.md`)
- [implemented] AUTH-018 Resource-level authorization for user self-updates; owners can update
  username/email only, admins can update all fields. (refs:
  `src/api/routes/v1/users/hooks.ts`)
- [implemented] AUTH-019 User deletion IDOR and authorization gaps closing; admin-gated
  `DELETE /v1/users/:recordId`, protection of the configured bootstrap administrator from
  deletion, required `bootstrapAdmin` configuration, and refresh-token revocation cascade on
  user deletion and privilege change. (refs: `docs/idorGapsClosing.md`,
  `src/api/routes/v1/users/routes.ts`, `src/application/users-service/`,
  `src/application/sessions-service/`)
- [proposed] AUTH-020 OAuth provider-token refresh; persist and refresh provider access tokens
  for later provider-API calls on the user's behalf. Deferred while the OAuth sign-in flow is
  being tested; sign-in discards provider tokens after identity resolution today. (refs:
  TODO in `src/application/contracts.ts`)
- [implemented] AUTH-021 Database backend selection extraction; `bootstrapService` connects
  through `DatabaseProvider` and uses `DatabaseAdapter` to provision datasets and build CRUD
  adapters, leaving no backend branch in bootstrap. (refs: CORE-008,
  `services/auth/src/runtime/`)
- [planned] AUTH-022 Backend-neutral signing-key selection; `bootstrapSigningKeys` returns the
  concrete `OpenBaoSigningKeyProvider`, `runtime/types.ts` types the composed service as that
  class, and shutdown calls `stopTokenRenewal()` by name, so the composition root still knows
  which backend it built. Requires closing the contract gap first: bootstrap and shutdown use
  `preflight`, `describeTokenLifetime`, `startTokenRenewal` and `stopTokenRenewal`, none of
  which `ISigningKeyProvider` declares and all of which are token-centric. Replacing them with
  a readiness check that returns operator warnings, and with background work folded into
  `initialize` plus a matching shutdown operation, is independent of any second provider; the
  `type` discriminated union and the provider factory stay gated by the signing plan's rule
  against adding either before a second implementation exists. (refs: `docs/signing-plan.md`,
  `services/auth/src/runtime/keys.ts`, `services/auth/src/runtime/types.ts`,
  `services/auth/src/application/contracts.ts`)
- [implemented] AUTH-023 Operator-owned signing-key lifecycle; the OpenBao provider is reduced
  to the four `ISigningKeyProvider` operations and makes exactly two calls, one KV read of the
  lifecycle document and one Transit sign. Self-renewal, startup preflight, token-lifetime
  reporting, key creation, the lifecycle engine covering activation, rotation and cleanup, key
  destruction and the entire write path are gone, along with `rotationIntervalSecs` and
  `verificationOverlapSecs`. Provider 1238 to 642 lines. The runtime token is long-lived rather
  than periodic, and the policy is now `read` on the state path plus `update` on
  `transit/sign/*`, with `transit/keys/*` absent entirely, so a compromised service can neither
  destroy key material nor rewrite its own state. Provisioning, rotation and destruction are
  operator procedures. (refs: `docs/en/howto/openbao/README.md`, `docs/signing-plan.md`,
  `services/auth/src/infra/keys/openbao/`)
- [implemented] AUTH-024 Environment-sourced configuration; each env-backed value
  (`BAO_ADDR`, `BAO_TOKEN`, OAuth client secrets, `JWT_HASH_SECRET`,
  `BOOTSTRAP_ADMIN_PASSWORD`, the jose key-file passphrase) is declared on its schema with
  `.meta({ env })`, carried in `config/local.json` as a placeholder, and resolved by
  `loadEnvVariables` (TOOLS-007) at the consuming boundary. The config object keeps the
  placeholder, so a secret never enters the decorated `request.server.config`. At `pnpm start`
  the process loads `.env` through Node's `--env-file`, so the declared variables reach
  `process.env` before the composition root reads them. (refs: TOOLS-007,
  `src/application/config/schema.ts`, `src/infra/keys/`, `src/infra/oauth/config/schema.ts`,
  `src/runtime/`, `config/devel.json`, `config/prod.json`, `package.json`)
- [proposed] AUTH-025 Test suite strategy, unit and integration tiers; Vitest with subpath
  aliases, shared in-memory fakes for `ICrud`, `ISigningKeyProvider`, and `IOAuthProvider`,
  and Fastify `app.inject` over `buildApi`, so the fast loop runs with no MongoDB, OpenBao,
  or live OAuth provider. (refs: `docs/tests-suite.md`, AUTH-001)
- [proposed] AUTH-026 End-to-end test harness; the built Docker image exercised over HTTP
  against ephemeral MongoDB, OpenBao, and a mock OIDC provider via docker-compose in CI,
  covering the concrete adapters and the `createRuntime` bootstrap the integration tier
  skips. Built after AUTH-025. (refs: `docs/tests-suite.md`, AUTH-025)
- [implemented] AUTH-027 Docker containerization; multi-stage `node:22-alpine` image
  (corepack-pinned pnpm, `--frozen-lockfile`, `pnpm prune --prod`, non-root user, argon2 musl
  prebuilds so Alpine needs no toolchain), with `docker-compose.yml` (production, pulls the
  GHCR image, mounts `config/prod.json`) and `docker-compose.dev.yml` (local build, mounts
  `config/devel.json` and the jose keyfile). The container selects config by `NODE_ENV` and
  takes secrets as environment variables rather than a `.env` file. (refs:
  `services/auth/Dockerfile`, `services/auth/.dockerignore`,
  `services/auth/docker-compose.yml`, `services/auth/docker-compose.dev.yml`, INFRA-001)
- [implemented] AUTH-028 GitHub Actions CI/CD; `build-and-push` builds and pushes
  `ghcr.io/<owner>/auth` on the production branch and version tags, and `deploy` runs on a
  self-hosted runner with a provider-aware fail-fast readiness check; it reads
  `signingKeys.provider.name` and `dbServerUri` from `config/prod.json`, always probes
  MongoDB, and probes OpenBao only when the provider is `openbao`, before
  `docker compose pull && up -d`. A commented seam triggers the infra pipeline first. (refs:
  `services/auth/.github/workflows/deploy.yml`, INFRA-001)

## PROFILES: `@slmdevs/profiles`

Planned identity and entitlements service (person and organization profiles, capabilities).
It starts after the auth service works standalone. Only the persistence scaffold exists.

- [implemented] PROFILES-001 Entity records and datasets; `persons`, `organizations`, and
  `outbox` with schemas, and dataset provisioning at startup. Uncommitted at the time of
  writing. (refs: `src/entities/`, `src/index.ts`)
- [proposed] PROFILES-002 Service runtime; configuration loading and an HTTP API following the
  auth layering. Nothing exists yet.
- [proposed] PROFILES-003 Person-to-user linking; `Person.userId` refers to an auth user.
  (refs: `src/entities/persons/schema.ts`)
- [proposed] PROFILES-004 Capabilities and entitlements; what an identity may do, evaluated per
  tenant and organization. Inferred; no design yet.
- [proposed] PROFILES-005 Outbox publisher; deliver `outbox` events, for example
  `profile.person.updated`. (refs: `docs/es/arquitectura/convenciones-nombres.md`)
- [proposed] PROFILES-006 First internal OIDC provider for auth. (refs: `docs/auth-plan.md`)

## INFRA: `infra/`

Shared deployment stack that platform services depend on. It owns the Docker `services`
network, the backing MongoDB and OpenBao instances, and the signing-key operator tooling.
Services join the external `services` network and never provision this infrastructure
themselves. Its own git repository, initialized without remotes.

- [implemented] INFRA-001 Shared stack compose; `docker-compose.yml` (production) and
  `docker-compose.dev.yml` (development) create the `services` network and run MongoDB and
  OpenBao (file storage, Shamir seal, config in `openbao/openbao.hcl`). Production enables
  MongoDB authentication and expects a real OpenBao auto-unseal; development runs MongoDB
  unauthenticated. (refs: `infra/docker-compose.yml`, `infra/docker-compose.dev.yml`,
  `infra/openbao/openbao.hcl`, `infra/README.md`)
- [implemented] INFRA-002 Signing-key operator tooling; the OpenBao lifecycle scripts
  (`unseal.sh`, `rotate.sh` and its steps) and the jose dev-keyfile provisioner
  (`provision.sh`, which copies the encrypted keyfile into the auth service so local dev reads
  it by default) are owned here, not by the auth service, which only consumes signing
  material. (refs: `infra/operator/openbao/`, `infra/operator/jose/provision.sh`, AUTH-023)

## DOCS: `docs/`

- [implemented] DOCS-001 Architecture guides in English and Spanish. (refs: `en/architecture/`,
  `es/arquitectura/`)
- [implemented] DOCS-002 Plan documents for auth. (refs: `auth-plan.md`, `session-plan.md`,
  `signing-plan.md`, `users-service-plan.md`, `entity-cursor-plugin.md`, `tests-suite.md`)
- [implemented] DOCS-003 Front matter and status conventions. (refs: `CLAUDE.md`)
- [proposed] DOCS-004 Reconcile stale plan documents with the code. (refs: see below)
- [implemented] DOCS-005 TypeDoc API documentation; TSDoc `/** */` source comments rendered to
  a per-package HTML site via `pnpm docs`, wired for `core`, `tools`, and `services/auth`
  (auth uses an expand entry-point strategy over `src`), with the source-comment convention
  codified in CLAUDE.md. Output is local and git-ignored; publishing/hosting is not yet set up.
  (refs: `CLAUDE.md`, `core/typedoc.json`, `tools/typedoc.json`, `services/auth/typedoc.json`)

## Known discrepancies

Read these before trusting a plan document.

- `docs/entity-cursor-plugin.md` and parts of `docs/session-plan.md` say `entityId` and
  `Entities`; the code uses `recordId` and `RecordsHandler`.
- Early plan documents and comments mention `userId`; as of 2026-09-24 the code uses
  `recordId` consistently throughout transport, route, and application layers
  (`/v1/users/:recordId`).
- `docs/es/arquitectura/convenciones-nombres.md` names the service `profile`, the package
  `@slmdevs/profile`, and the entity `Org` with singular route segments such as
  `/auth/v1/user`. The repository uses `services/profiles`, `@slmdevs/profiles`,
  `organizations`, and plural routes such as `/v1/users`. The naming document is the stated
  authority, so the code is the side that would change.
- `docs/session-plan.md` refers to `multi-tenant-auth-design.md`, which is not in `docs/`.











