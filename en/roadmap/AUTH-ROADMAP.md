---
title: Auth Service Roadmap
lastUpdate: 2026-10-24
lang: en
category: planning
status: active
---

# Auth Service Roadmap

Feature status for `services/auth/` (`@slmdevs/auth`). Line format, status tags, and editing
rules are defined in the platform [ROADMAP](../../ROADMAP.md#roadmap-rules). Paths in `refs`
are relative to `services/auth/`.

## AUTH features

Fastify REST API that issues and verifies sessions through password credentials and OAuth
providers. Layers: `api/` (transport), `application/` (policy), `records/` (persistence),
`infrastructure/` (OpenBao keys, OAuth providers).

- [implemented] AUTH-001 Service architecture: composition root, encapsulated route plugins,
  execution-context binding, OpenAPI. (refs: `docs/auth-plan.md`, `src/runtime/`)
- [implemented] AUTH-002 Password sign-in: Argon2 verification in the default tenant.
  Route: `POST /v1/session`. (refs: `src/application/passwords-service/`)
- [implemented] AUTH-003 OAuth authorization-code flow: Google and GitHub providers. Routes:
  `POST` and `GET /v1/oauth/:provider/authorize`, `GET /v1/oauth/:provider/callback`. (refs:
  `src/application/oauth-service/`, `src/infra/oauth/`)
- [implemented] AUTH-004 Sessions: issue, read, terminate, and refresh with rotating opaque
  refresh tokens stored as HMAC hashes. Routes: `GET`, `DELETE /v1/session`,
  `POST /v1/session/refresh`. (refs: `docs/session-plan.md`,
  `src/application/sessions-service/`)
- [implemented] AUTH-005 Access tokens: ES256 JWT issue and verification into a session
  grant. (refs: `src/application/tokens-service/`)
- [implemented] AUTH-006 Durable signing keys: OpenBao Transit signing against an
  operator-owned key lifecycle, with JWKS at `GET /.well-known/jwks.json`. (refs: AUTH-023,
  `docs/signing-plan.md`, `services/auth/src/infra/keys/`, `docs/en/howto/openbao/`)
- [implemented] AUTH-007 Users: create, register, read, update, delete, and find through
  `UsersService`. Routes under `/v1/users`. (refs: `docs/users-service-plan.md`,
  `docs/users-service-completion.md`)
- [implemented] AUTH-008 Administrator password set: `POST /v1/password/set`.
  (refs: `src/application/passwords-service/`)
- [implemented] AUTH-009 Cursor pagination plugin: request decode and response `nextCursor`.
  (refs: `docs/entity-cursor-plugin.md`, `src/api/plugins/record-cursor-codec.ts`)
- [implemented] AUTH-010 Health and OpenAPI: `GET /health`, `GET /openapi.json`, UI at
  `/docs`. (refs: `src/api/routes/health/`, `src/api/plugins/openapi.ts`)
- [implemented] AUTH-018 Resource-level authorization for user self-updates: owners can
  update username/email only, admins can update all fields. (refs:
  `src/api/routes/v1/users/hooks.ts`)
- [implemented] AUTH-019 User deletion IDOR and authorization gaps closing: admin-gated
  `DELETE /v1/users/:recordId`, protection of the configured bootstrap administrator from
  deletion, required `bootstrapAdmin` configuration, and refresh-token revocation cascade on
  user deletion and privilege change. (refs: `docs/idorGapsClosing.md`,
  `src/api/routes/v1/users/routes.ts`, `src/application/users-service/`,
  `src/application/sessions-service/`)
- [implemented] AUTH-021 Database backend selection extraction: `bootstrapService` connects
  through `DatabaseProvider` and uses `DatabaseAdapter` to provision datasets and build CRUD
  adapters, leaving no backend branch in bootstrap.
  (refs: [CORE-008](./CORE-ROADMAP.md#core-features), `services/auth/src/runtime/`)
- [implemented] AUTH-023 Operator-owned signing-key lifecycle: the OpenBao provider is
  reduced to the four `ISigningKeyProvider` operations and makes exactly two calls, one KV
  read of the lifecycle document and one Transit sign. Self-renewal, startup preflight,
  token-lifetime reporting, key creation, the lifecycle engine covering activation, rotation
  and cleanup, key destruction and the entire write path are gone, along with
  `rotationIntervalSecs` and `verificationOverlapSecs`. Provider 1238 to 642 lines. The
  runtime token is long-lived rather than periodic, and the policy is now `read` on the
  state path plus `update` on `transit/sign/*`, with `transit/keys/*` absent entirely, so a
  compromised service can neither destroy key material nor rewrite its own state.
  Provisioning, rotation and destruction are operator procedures. (refs:
  `docs/en/howto/openbao/README.md`, `docs/signing-plan.md`,
  `services/auth/src/infra/keys/openbao/`)
- [implemented] AUTH-024 Environment-sourced configuration: each env-backed value
  (`BAO_ADDR`, `BAO_TOKEN`, OAuth client secrets, `JWT_HASH_SECRET`,
  `BOOTSTRAP_ADMIN_PASSWORD`, the jose key-file passphrase) is declared on its schema with
  `.meta({ env })`, carried in `config/local.json` as a placeholder, and resolved by
  `loadEnvVariables` ([TOOLS-007](./TOOLS-ROADMAP.md#tools-features)) at the consuming
  boundary. The config object keeps the
  placeholder, so a secret never enters the decorated `request.server.config`. At `pnpm
  start` the process loads `.env` through Node's `--env-file`, so the declared variables
  reach `process.env` before the composition root reads them.
  (refs: [TOOLS-007](./TOOLS-ROADMAP.md#tools-features),
  `src/application/config/schema.ts`, `src/infra/keys/`, `src/infra/oauth/config/schema.ts`,
  `src/runtime/`, `config/devel.json`, `config/prod.json`, `package.json`)
- [implemented] AUTH-027 Docker containerization: multi-stage `node:22-alpine` image
  (corepack-pinned pnpm, `--frozen-lockfile`, `pnpm prune --prod`, non-root user, argon2
  musl prebuilds so Alpine needs no toolchain), with `docker-compose.yml` (production, pulls
  the GHCR image, mounts `config/prod.json`) and `docker-compose.dev.yml` (local build,
  mounts `config/devel.json` and the jose keyfile). The container selects config by
  `NODE_ENV` and takes secrets as environment variables rather than a `.env` file. (refs:
  `services/auth/Dockerfile`, `services/auth/.dockerignore`,
  `services/auth/docker-compose.yml`, `services/auth/docker-compose.dev.yml`, INFRA-001)
- [implemented] AUTH-028 GitHub Actions CI/CD: `build-and-push` builds and pushes
  `ghcr.io/<owner>/auth` on the production branch and version tags, and `deploy` runs on a
  self-hosted runner with a provider-aware fail-fast readiness check; it reads
  `signingKeys.provider.name` and `dbServerUri` from `config/prod.json`, always probes
  MongoDB, and probes OpenBao only when the provider is `openbao`, before
  `docker compose pull && up -d`. A commented seam triggers the infra pipeline first. (refs:
  `services/auth/.github/workflows/deploy.yml`, INFRA-001)
- [implemented] AUTH-022 Backend-neutral signing-key selection: `bootstrapSigningKeys`
  returns the concrete `OpenBaoSigningKeyProvider`, `runtime/types.ts` types the composed
  service as that class, and shutdown calls `stopTokenRenewal()` by name, so the composition
  root still knows which backend it built. Requires closing the contract gap first:
  bootstrap and shutdown use `preflight`, `describeTokenLifetime`, `startTokenRenewal` and
  `stopTokenRenewal`, none of which `ISigningKeyProvider` declares and all of which are
  token-centric. Replacing them with a readiness check that returns operator warnings, and
  with background work folded into `initialize` plus a matching shutdown operation, is
  independent of any second provider; the `type` discriminated union and the provider
  factory stay gated by the signing plan's rule against adding either before a second
  implementation exists. (refs: `docs/signing-plan.md`, `services/auth/src/runtime/keys.ts`,
  `services/auth/src/runtime/types.ts`, `services/auth/src/application/contracts.ts`)
- [planned] AUTH-011 Password reset: `POST /v1/password/reset` exists but always fails with
  `ARCHITECTURE_NOT_IMPLEMENTED`. (refs: `src/application/passwords-service/`)
- [proposed] AUTH-012 Timing-safe unknown-user path: reduce user-enumeration signal.
  (refs: NOTE in `passwords-service.ts`)
- [proposed] AUTH-013 OAuth PKCE for the authorization-code flow: per-attempt
  `code_verifier` and S256 `code_challenge`, requiring a durable pending-authorization store
  and a per-provider PKCE capability (Google supported; GitHub to verify). (refs:
  `src/application/oauth-service/`, TODO in `src/application/contracts.ts`)
- [proposed] AUTH-014 Multi-tenant login and tenant selection: login is single default
  tenant today. (refs: `docs/session-plan.md`)
- [proposed] AUTH-016 Email verification.
- [proposed] AUTH-017 User lifecycle events such as `auth.user.created`.
  (refs: `docs/es/arquitectura/convenciones-nombres.md`)
- [proposed] AUTH-020 OAuth provider-token refresh: persist and refresh provider access
  tokens for later provider-API calls on the user's behalf. Deferred while the OAuth sign-in
  flow is being tested; sign-in discards provider tokens after identity resolution today.
  (refs: TODO in `src/application/contracts.ts`)
- [proposed] AUTH-025 Test suite strategy, unit and integration tiers: Vitest with subpath
  aliases, shared in-memory fakes for `ICrud`, `ISigningKeyProvider`, and `IOAuthProvider`,
  and Fastify `app.inject` over `buildApi`, so the fast loop runs with no MongoDB, OpenBao,
  or live OAuth provider. (refs: `docs/tests-suite.md`, AUTH-001)
- [proposed] AUTH-026 End-to-end test harness: the built Docker image exercised over HTTP
  against MongoDB, JoseSigningKeyProvider, and a mock OIDC provider via docker-compose in
  CI, covering the concrete adapters and the `createRuntime` bootstrap the integration tier
  skips. Built after AUTH-025. (refs: `docs/tests-suite.md`, AUTH-025)
- [canceled] AUTH-015 Scheduled signing-key rotation: rejected. AUTH-023 moved the whole key
  lifecycle to the operator, so rotation is an OpenBao CLI procedure and the auth service
  has no rotation to schedule. (refs: `docs/en/howto/openbao/README.md`)
