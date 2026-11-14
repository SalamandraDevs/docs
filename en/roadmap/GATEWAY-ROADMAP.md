---
title: Gateway Service Roadmap
lastUpdate: 2026-11-11
lang: en
category: planning
status: active
---

# Gateway Service Roadmap

Feature status for `services/gateway/` (`@salamandradevs/gateway`). Paths in `refs` are
relative to `services/gateway/`; plan documents under `docs/plans/` are relative to the
workspace root.

## GATEWAY features

Fastify edge service that terminates the tenant API key, meters each tenant's request pool
by plan in Redis, owns the API key registry and its administration, and proxies to upstream
services by path prefix, asserting the tenant with `x-tenant-id`. Layers: `api/` (transport
and HTTP policy plugins), `application/` (policy), `storage/` (the API key registry), and
`runtime/` (the composition root), with backend-neutral storage and signing keys consumed
from `@salamandradevs/core`.

- [implemented] GATEWAY-001 Service architecture: composition root, encapsulated HTTP-policy
  plugins, execution-context binding, and classified error mapping to RFC 9457 problems.
  (refs: `docs/plans/261106-gateway-plan.md`, `src/runtime/`, `src/api/plugins/`)
- [implemented] GATEWAY-002 Tenant API key termination at the edge: the API key plugin
  verifies the `x-api-key` header, sets `x-tenant-id` from the verified key, and the trust
  boundary strips any client-supplied `x-tenant-id` or `x-api-key` before forwarding.
  (refs: `src/api/plugins/api-key.ts`, `src/api/plugins/security.ts`,
  `src/api/plugins/proxy.ts`)
- [implemented] GATEWAY-003 API key registry and administration: issue, list, and revoke
  tenant keys through `ApiKeysService`, all in the default tenant where the registry lives.
  Routes: `POST`, `GET /gateway/v1/apikeys`, `DELETE /gateway/v1/apikeys/:recordId`.
  (refs: `src/application/api-keys-service/`, `src/api/routes/v1/apikeys/`,
  `src/storage/apikeys/`)
- [implemented] GATEWAY-004 Self-contained signed API keys: a key is
  `slm_<payload>.<signature>` carrying `{ v: 2, kid, jti, tid, plan, exp }`, signed with
  ES256 through `ISigningKeyProvider`; `verify` is a local computation over the signed key
  and an in-memory revocation set, so it makes no database read. The set is rebuilt from the
  registry each `gateway.apiKeys.revocationRefreshSecs` and whole-replaced, and a revoking
  replica rejects the key at once. (refs: `src/application/api-keys-service/`,
  `src/runtime/api-keys.ts`, `README.md`)
- [implemented] GATEWAY-005 Secure transport defaults and per-tenant rate limiting: Helmet
  security headers, closed CORS, and Redis-backed rate-limit tiers, with a `public` tier per
  client IP and one `{ max, timeWindow }` policy per plan (`platform`, `community`,
  `client`) counted under tier-prefixed keys so every replica shares one count. Every route
  must declare exactly one base tier or startup fails; an exceeded limit answers a
  classified 429 with `retry-after`, and a Redis outage fails open to each upstream's own
  tiers. (refs: `src/api/plugins/security.ts`, `src/api/routes/v1/hooks/rate-limit.ts`,
  `src/runtime/redis.ts`)
- [implemented] GATEWAY-006 Reverse proxy to upstreams: one keyed wildcard route per
  configured upstream (`ALL /<prefix>/*`) and one explicit route per public path, forwarded
  over `@fastify/reply-from`. Public routes count against the per-IP tier; the gateway
  appends the client IP to `x-forwarded-for` and forwards `x-request-id`. (refs:
  `src/api/plugins/proxy.ts`, `src/config/schema.ts`)
- [implemented] GATEWAY-007 Admin authorization via JWKS: the three key-administration
  routes require an admin bearer token, which the gateway verifies as a JWKS client against
  auth's published key set using core's `AccessTokenVerifier`, holding no token signing keys
  of its own. (refs: [CORE-010](./CORE-ROADMAP.md#core-features),
  `src/api/plugins/authentication.ts`, `src/api/plugins/authorization.ts`,
  `src/runtime/tokens.ts`)
- [implemented] GATEWAY-008 Backend-neutral storage and signing keys: `DatabaseProvider` and
  `DatabaseAdapter` connect and provision the registry, and `createKeysProvider` selects the
  signing-key provider named by `SIGNING_KEY_PROVIDER` (`openbao` or `jose`), each from
  `@salamandradevs/core`. (refs: [CORE-008](./CORE-ROADMAP.md#core-features),
  [CORE-009](./CORE-ROADMAP.md#core-features), `src/runtime/database.ts`,
  `src/runtime/keys.ts`)
- [implemented] GATEWAY-009 Platform key bootstrap seed: with `GENERATE_PLATFORM_API_KEY`
  set, the gateway issues or re-signs the default-tenant `platform` key in its own
  default-tenant context and logs it at `warn`, so an otherwise keyed edge can mint its
  first credential. The seed is idempotent at the registry, reusing one record across runs.
  (refs: `src/runtime/api-keys.ts`)
- [implemented] GATEWAY-010 Cursor pagination plugin: request decode and response
  `nextCursor` over the shared record cursor codec.
  (refs: `src/api/plugins/record-cursor-codec.ts`)
- [implemented] GATEWAY-011 Health check: `GET /health` liveness outside the version
  segment. (refs: `src/api/routes/health/`)
- [implemented] GATEWAY-012 Environment-sourced configuration: each env-backed value
  (`DB_SERVER_URI`, `REDIS_URL`, `SIGNING_KEY_PROVIDER`, `JOSE_KEYFILE_PASSPHRASE`,
  `BAO_ADDR`, `BAO_TOKEN`, the log levels) is declared on its schema with `.meta({ env })`,
  carried in `config/config.json` as a placeholder, and resolved by `loadEnvVariables` at
  the consuming boundary, so a secret never enters the decorated config object. (refs:
  [TOOLS-007](./TOOLS-ROADMAP.md#tools-features), `src/config/schema.ts`,
  `src/api/config/schema.ts`)
- [implemented] GATEWAY-013 Docker containerization and CI/CD: a multi-stage
  `node:22-alpine` image runs as a non-root user, and the `build-and-deploy` workflow runs
  `pnpm check` and `pnpm lint` as a gate before building and pushing the image by commit
  SHA. (refs: `Dockerfile`, `docker-compose.yml`, `docker-compose.dev.yml`,
  `.github/workflows/deploy.yml`)
- [implemented] GATEWAY-014 Shared API key verification in core: the verifier and the key
  format moved to `@salamandradevs/core`, so other services can verify tenant keys.
  `ApiKeysService` now signs through `ISigningKeyProvider` in a private `sign` and verifies
  through core's `ApiKeyVerifier`, and `apiKeysConfigSchema` extends core's
  `apiKeyVerifyConfigSchema`, so `prefix` and `maxLength` come from `gateway.apiKeys`.
  (refs: `docs/plans/261111-api-key-verification-plan.md`,
  [CORE-012](./CORE-ROADMAP.md#core-features), `src/application/api-keys-service/`,
  `src/application/config/schema.ts`, `src/runtime/api-keys.ts`)
