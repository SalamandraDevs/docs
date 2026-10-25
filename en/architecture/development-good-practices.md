---
title: Development Good Practices
lastUpdate: 2026-10-25
lang: en
category: architecture
status: published
---

## Introduction

A **good practice** is a rule of engineering that prevents a known class of failure:
builds that cannot be reproduced, secrets that leak, processes that lose state, or
incidents nobody can trace. This guide covers the practices that keep Salamandra Devs
packages and services modular, reproducible, deployable, observable, and secure. They apply
to `core/`, `tools/`, and every service under `services/`.

Each practice carries a stable ID (`GP-NN`) for citation; IDs follow
[Document IDs](../house/naming-conventions.md#document-ids). Where the code does not meet a
practice yet is tracked in the [Architecture Review](../../architecture-review.md), not in
this guide.

## Terminology used in this guide

- **Artifact**: the immutable output of one build, such as a container image or a
  published npm version.
- **Release**: an artifact combined with the configuration of one deployment.
- **Deployment manifest**: the file that runs a release (`docker-compose.yml`, a CI deploy
  job).
- **Backing service**: a resource the process reaches over the network, such as MongoDB,
  OpenBao, or an OAuth provider.
- **Composition root**: the startup code that builds concrete objects and injects them
  (`services/auth/src/runtime/`); see
  [DP-08 Composition Root](./design-patterns.md#dp-08-composition-root).
- **Gate**: an automated check that must pass before a change can be built, published, or
  deployed.
- **Must / should**: "must" is mandatory; "should" allows a documented exception.

## Platform Examples

The examples are trimmed excerpts from `core/`, `tools/`, and `services/auth/`, including
their `package.json`, Dockerfile, configuration, and workflow files. Each one starts with a
comment naming its source file. Counter-examples are marked `Avoid` and are hypothetical:
they show what would break the practice, not code that exists. Where the real code does
break a practice, the case is listed in the
[Architecture Review](../../architecture-review.md) instead.

Every section ends with a **Check:** line, a condition a reviewer or an agent can verify in
the repository.

### GP-01 One repository per package or service

Each library and each deployable service lives in its own repository with its own build,
version, and release, and shared code is consumed as a versioned package. Shared relative
paths couple release cycles and break any build outside the original folder layout.

```json
// services/auth/package.json
"dependencies": {
  "@salamandradevs/core": "^0.1.0",
  "@salamandradevs/tools": "^0.1.0"
}
```

```json
// Avoid: resolves only inside one folder layout and breaks CI elsewhere.
"dependencies": {
  "@salamandradevs/core": "file:../../core"
}
```

Related: GP-05, GP-12; [Package Publishing](../infra/packaging.md).

**Check:** no `package.json` dependency uses `file:` or `link:`.

### GP-02 Layered service structure

A service splits transport, use cases, persistence, external adapters, configuration, and
wiring into separate sections ([Service structure](../house/naming-conventions.md#service-structure)),
and dependencies point inward: transport depends on application contracts and types, never
on storage or vendor adapters. Mixed layers make a change in one concern ripple through
all of them.

```typescript
// services/auth/src/api/routes/v1/users/controller.ts
import type { RecordParams } from '#api/schemas/types.js';
import type { UserView } from '#application/users-service/types.js';
```

```typescript
// Avoid: a controller that talks to storage skips the use case and its policy.
import { UserRecordsHandler } from '#storage/users/index.js';
```

Related: GP-03; [SP-05](./solid-principles.md#sp-05-dependency-inversion-principle-dip).

**Check:** files under `src/api/` import no module from `#storage/` or `#infra/`.

### GP-03 Contracts at boundaries, wiring at the composition root

Code that crosses a boundary depends on a contract, and concrete classes are created only
in the composition root or in a factory it calls. Constructing collaborators inside policy
code hides dependencies and blocks substitution and testing.

```typescript
// services/auth/src/runtime/services.ts
const tokens: ITokenService = new TokensService(config.auth, deps.signingKeys);
const sessions: ISessionsService = new SessionsService(
  deps.sessionRecordsHandler,
  tokens,
  config.auth,
  hashSecret
);
```

```typescript
// Avoid: the service builds its own dependency and its configuration.
export class SessionsService {
  private readonly tokens: ITokenService = new TokensService(loadConfig().auth, /* ... */);
}
```

Related: [DP-08](./design-patterns.md#dp-08-composition-root),
[SP-05](./solid-principles.md#sp-05-dependency-inversion-principle-dip).

**Check:** `new` of an infrastructure class (`MongoCrud`, `DatabaseProvider`, a provider)
appears only under `runtime/`, `infra/*/factory.ts`, or `core/src/infra/`.

### GP-04 Schemas as the single source of truth

A runtime shape (record, request, response, configuration) is defined once as a Zod
schema; its TypeScript type is inferred, and the same schema drives validation,
serialization, and dataset provisioning. Parallel hand-written types drift from what is
validated and stored.

```typescript
// services/auth/src/storage/users/types.ts
export type UserRecord = z.infer<typeof userRecordSchema>;

// services/auth/src/storage/users/datasetdef.ts
export const userDatasetDef: DatasetDef = {
  name: 'users',
  schema: userRecordSchema,
  indexes: [recordIndexDef /* , ... */]
};
```

```typescript
// Avoid: a second definition of the same shape that nobody validates against.
export type UserRecord = { username: string; email: string; roles: string[] };
```

Related: [CS-02](../house/code-style.md#cs-02-zod-schemas-and-derived-types).

**Check:** a type that mirrors a schema's fields is declared with `z.infer`.

### GP-05 Declared and locked dependencies

Every dependency is declared in `package.json`, the lockfile is committed, and CI and image
builds install with a frozen lockfile. Without a locked tree, two builds of the same commit
can resolve different dependencies.

```dockerfile
# services/auth/Dockerfile
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
RUN pnpm install --frozen-lockfile
```

```yaml
# Avoid: resolves a new tree on every release.
- run: pnpm install --no-frozen-lockfile
```

Related: GP-06, GP-12.

**Check:** `pnpm-lock.yaml` is tracked in git and every install step in CI and in
Dockerfiles uses `--frozen-lockfile`.

### GP-06 Pinned toolchain

The package manager and the Node.js major version are pinned and match across local
development, CI, and container images. A build that passes on one runtime and fails on
another is not reproducible.

```dockerfile
# services/auth/Dockerfile
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@12.4.1 --activate
```

```yaml
# Avoid: an unpinned tool and a runtime that differs from the image.
- run: npm install -g npm@latest
- uses: actions/setup-node@v4
  with:
    node-version: '24'
```

Related: GP-05, GP-21.

**Check:** the Node.js major in every workflow `node-version`, every `FROM node:` line, and
`engines` agree; tools installed in CI carry an explicit version.

### GP-07 Supply-chain safeguards

Published packages carry build provenance, dependency install scripts are allow-listed, and
new dependency versions respect a minimum release age. A compromised or freshly hijacked
dependency must not run code or ship silently.

```yaml
# services/auth/pnpm-workspace.yaml
allowBuilds:
  argon2: true
  simple-git-hooks: true
minimumReleaseAgeExclude:
  - '@salamandradevs/*'
```

```json
// core/package.json
"publishConfig": { "access": "public", "provenance": true }
```

```yaml
# Avoid: lets every dependency run its install script.
dangerouslyAllowAllBuilds: true
```

Related: GP-05; [Package Publishing](../infra/packaging.md).

**Check:** every publishable `package.json` sets `provenance: true`; every package with an
install script is listed in `allowBuilds`.

### GP-08 Validated configuration, fail fast

Configuration is validated against a schema before the service starts any work, and an
invalid or missing value stops startup with a classified error. A service that starts with
a bad value fails later, on a live request, far from the cause.

```typescript
// services/auth/src/config/config.ts
try {
  const serializedConfig: string = readFileSync(configPath, 'utf8');
  const parsedConfig: unknown = JSON.parse(serializedConfig);

  validatedConfig = serviceConfigSchema.parse(parsedConfig);
} catch (error: unknown) {
  throw new CustomError({
    name: 'CONFIG_ERROR',
    message: `Configuration could not be loaded or validated at: ${configPath}`,
    cause: error
  });
}
```

```typescript
// Avoid: an unchecked read that turns a missing value into a runtime surprise.
const port: number = Number(process.env.PORT);
```

Related: GP-09, GP-04.

**Check:** every value read from configuration or the environment passes through a Zod
schema before use.

### GP-09 Secrets and credentials only from the environment

Secrets and credentials (tokens, passwords, client secrets, connection strings with
credentials) are resolved from the environment at the code that uses them. The
configuration file carries only a `READ FROM ENV ...` placeholder, and the resolved value
is never stored back in the configuration object. Configuration files are committed, copied
into images, and decorated onto requests; a secret there leaks through all three.

```typescript
// services/auth/src/application/config/schema.ts
hashSecret: z
  .string({ error: 'is required' })
  .min(1, 'must not be empty')
  .meta({ env: 'JWT_HASH_SECRET' })

// services/auth/src/runtime/services.ts
const { hashSecret } = loadEnvVariables(refreshTokenConfigSchema);
```

```json
// Avoid: a credential committed in the deployment file.
"dbServerUri": "mongodb://auth:s3cret@mongo:27017"
```

Related: GP-08, GP-11.

**Check:** no committed configuration file contains a credential; every sensitive schema
field carries `.meta({ env })`; every declared variable appears in `.env.example` and in
every deployment manifest.

### GP-10 Configuration varies per deployment, not per build

The same image runs in every environment. Deployment-specific values are supplied at
deploy time, and adding an environment (for example `staging`) needs no code change. An
image that embeds one environment's values cannot be promoted unchanged.

```yaml
# services/auth/docker-compose.yml
volumes:
  - ./config/prod.json:/app/config/prod.json:ro
environment:
  - JWT_HASH_SECRET=${JWT_HASH_SECRET}
```

```typescript
// Avoid: a fixed two-way switch that maps every non-production name to development.
const configFileName: string =
  process.env.NODE_ENV === 'production' ? 'prod.json' : 'devel.json';
```

Related: GP-12, GP-21.

**Check:** the config file is chosen by a deploy-time value that can name any environment,
and the image does not need to contain environment-specific files.

### GP-11 Backing services are attached resources behind contracts

Databases, key stores, and identity providers are reached through a contract and located
by configuration, so one can be replaced or relocated without a code change. Hard-wired
backends turn an infrastructure move into a code release.

```typescript
// services/auth/src/runtime/database.ts
const dbProvider: DatabaseProvider = new DatabaseProvider(
  config.databaseBackend,
  config.dbServerUri,
  config.databaseName
);
```

```typescript
// Avoid: a backend and its location fixed in code.
const client: MongoClient = new MongoClient('mongodb://mongo:27017');
```

Related: GP-09; [DP-01](./design-patterns.md#dp-01-factory),
[DP-02](./design-patterns.md#dp-02-adapter).

**Check:** switching a backend changes configuration only; its location and credentials
follow GP-09.

### GP-12 Immutable artifacts, deployed by identity

A build produces one immutable artifact per commit, a release combines that artifact with
deployment configuration, and deployments and rollbacks reference the artifact by an
immutable identifier (commit SHA or version tag). With a moving tag, "what is running" and
"how to roll back" have no exact answer.

```yaml
# services/auth/.github/workflows/deploy.yml
tags: |
  ${{ env.IMAGE }}:${{ github.sha }}
```

```yaml
# Avoid: a moving tag in the deployment manifest.
image: ghcr.io/salamandradevs/auth:latest
```

Related: GP-05, GP-10.

**Check:** the deployment manifest pins an image by SHA or version tag.

### GP-13 Quality gates run in CI

Format, lint, type check, and tests run in CI on every change and block a release when they
fail. Local hooks are a convenience, not the gate: they can be skipped, and they do not run
on the code that is actually built.

```json
// services/auth/package.json
"scripts": {
  "lint": "eslint ./src --ext .ts --fix",
  "check": "tsc --noEmit"
},
"simple-git-hooks": {
  "pre-push": "pnpm run format && pnpm run lint"
}
```

```yaml
# Avoid: a release job with no gate before it.
jobs:
  build-and-push:
    steps:
      - run: pnpm run build
```

Related: GP-24.

**Check:** each repository has a workflow that runs `pnpm check`, `pnpm lint`, and the test
suite before build or publish jobs.

### GP-14 Stateless processes

A service process keeps no state that another instance or a restart would need; durable or
shared state lives in a backing service. In-memory state breaks horizontal scaling, rolling
deploys, and restarts.

```typescript
// services/auth/src/application/sessions-service/sessions-service.ts
return createHmac('sha256', this.refreshHashSecret)
  .update(refreshToken, 'utf8')
  .digest('base64url');
```

The refresh token's HMAC is stored in the session record, so any instance can verify a
refresh request.

```typescript
// Avoid: request-derived state held only in this process.
private readonly pendingStates = new Map<string, OAuthPendingState>();
```

Related: GP-11, GP-15.

**Check:** no long-lived `Map`, array, or module-level variable in a service holds
request-derived state.

### GP-15 Disposability: fast start, graceful stop

A service starts without manual steps, releases every resource it opened on
`SIGTERM`/`SIGINT`, and releases already-opened resources when startup fails part way.
Orchestrators stop and move processes routinely; a process that leaks connections or hangs
on stop degrades the platform.

```typescript
// services/auth/src/runtime/runtime.ts
shutdown: async (): Promise<void> => {
  try {
    await api.close();
  } finally {
    await dbProvider.close();
  }
}

// services/auth/src/index.ts
process.once('SIGINT', shutdown);
process.once('SIGTERM', shutdown);
```

```typescript
// Avoid: exits without draining requests or closing the connection.
process.on('SIGTERM', () => process.exit(0));
```

Related: GP-14, GP-16.

**Check:** every resource opened in `runtime/` is closed on both the failure path and the
shutdown path.

### GP-16 Liveness and readiness

A service exposes a liveness check (the process responds) and a readiness check (its
required backing services are reachable), and the container definition uses them. Without
readiness, traffic reaches an instance that cannot serve it.

```typescript
// services/auth/src/api/routes/health/controller.ts
const response: HealthResponse = {
  service: 'auth',
  status: 'ok',
  timestamp: new Date().toISOString()
};
```

```yaml
# Avoid: a container definition with no health check, so a broken instance stays in
# rotation until someone notices.
services:
  auth:
    image: ghcr.io/salamandradevs/auth:<sha>
```

Related: GP-15, GP-11.

**Check:** a readiness endpoint exists and the Dockerfile or compose file declares a
`HEALTHCHECK` or `healthcheck`.

### GP-17 Administrative tasks are idempotent and operator-owned

One-off or administrative tasks run against the same code and configuration as the
service, are idempotent, and are owned by an operator procedure rather than ad-hoc scripts.
Admin work with different code or configuration produces state the service does not
expect.

```typescript
// services/auth/src/runtime/admin.ts
const existingUser: UserRecord | null = await deps.userRecordsHandler.readByEmail(
  bootstrapAdmin.email.trim().toLowerCase()
);

if (!existingUser) {
  await deps.users.create({ /* configured administrator */ });

  return;
}
```

The second run finds the administrator and changes nothing. Signing-key provisioning,
rotation, and destruction are operator procedures outside the service (ROADMAP
INFRA-002).

```typescript
// Avoid: a startup task that creates a new administrator on every boot.
await deps.users.create({ /* configured administrator */ });
```

Related: GP-15.

**Check:** every startup task can run twice with no change on the second run.

### GP-18 Logs as an event stream

Services write structured logs to stdout and leave routing and storage to the environment,
with the level configurable per deployment. Files inside containers are lost with the
container and are invisible to log collectors.

```typescript
// services/auth/src/api/api.ts
const logs: Record<string, string> = loadEnvVariables(logsConfigSchema);

const base: FastifyInstance = Fastify({
  logger: isDevelopment
    ? { level: logs.consoleLevel, transport: { target: 'pino-pretty' /* , ... */ } }
    : { level: logs.consoleLevel },
  requestIdHeader: 'x-request-id'
});
```

```typescript
// Avoid: logs written to a file inside the container.
logger: { level: 'info', file: '/app/logs/auth.log' }
```

Related: GP-19, GP-20.

**Check:** no code writes log files; every logging variable is passed by every deployment
manifest.

### GP-19 Request correlation

Every request carries a correlation id that is accepted from the caller when present,
attached to every log line, and available to downstream code. Without it, one failing
request cannot be traced across log lines or services.

```typescript
// services/auth/src/api/plugins/execution-context.ts
const executionContext: ExecutionContext = {
  authenticated: { actorId: request.sessionGrant.actorId, /* ... */ },
  request: { requestId: request.id }
};
```

Fastify fills `request.id` from the `x-request-id` header configured in GP-18.

```typescript
// Avoid: a call to another platform service that drops the correlation id.
await fetch(`${profilesUrl}/v1/persons/${recordId}`);
```

Related: GP-18; [DP-09](./design-patterns.md#dp-09-ambient-context).

**Check:** outbound calls to other platform services forward `x-request-id`.

### GP-20 Classified errors

Every error that code throws is a `CustomError` with a deliberate `ErrorName`
([CS-04](../house/code-style.md#cs-04-errors)), and transport maps names to RFC 9457
problem responses with stable problem `type` URIs. Unclassified errors reach clients as
generic 500s and cannot be alerted on.

```typescript
// services/auth/src/api/plugins/errors.ts
} else if (error.name === 'RESOURCE_NOT_FOUND') {
  status = 404;
  type = 'https://api.slmdevs.com/problems/resource-not-found';
```

```typescript
// Avoid: an unclassified error that the handler can only report as 500.
throw new Error('User not found');
```

Related: GP-18.

**Check:** no `throw new Error` or `throw new TypeError` escapes a function without being
wrapped in a `CustomError`; every problem `type` URI uses one base URL.

### GP-21 Development and production parity

Development uses the same kinds of backing services and the same container image as
production, and every difference is explicit, named, and limited to adapters selected by
configuration. A backend used only in production hides failures until release.

```typescript
// services/auth/src/infra/keys/factory.ts
case 'jose': {
  process.emitWarning(
    'JoseSigningKeyProvider is a development provider; private keys live in a local ' +
      'encrypted file on a single host. Use a vault provider in production.',
    { code: 'DEV_SIGNING_KEY_PROVIDER' }
  );

  return new JoseSigningKeyProvider(config.provider, config.algorithm);
}
```

`docker-compose.dev.yml` builds the same Dockerfile and joins the same `services` network
with MongoDB and OpenBao.

```yaml
# Avoid: a development-only database engine that production never runs.
databaseBackend: sqlite
```

Related: GP-10, GP-06.

**Check:** every development-only adapter announces itself at startup and is rejected or
warned about in production configuration.

### GP-22 Least privilege

Processes, tokens, and policies hold only the permissions their work needs, so a
compromised component cannot escalate or destroy data.

```dockerfile
# services/auth/Dockerfile
RUN addgroup -S nodegroup && adduser -S nodeuser -G nodegroup
USER nodeuser
```

```hcl
# docs/en/howto/openbao/auth-signing-policy.hcl
path "transit/sign/auth-signing-*" {
  capabilities = ["update"]
}

path "secret/data/auth/signing-lifecycle" {
  capabilities = ["read"]
}
```

```hcl
# Avoid: a service token that can also rewrite or destroy its own keys.
path "transit/keys/*" {
  capabilities = ["create", "update", "delete"]
}
```

Related: GP-09, GP-23.

**Check:** no container runs as root, and each token's policy lists only the operations the
service performs.

### GP-23 Secure transport defaults

HTTP services enable security headers, explicit CORS, rate limits on every unauthenticated
endpoint that creates or changes state, and secure cookie attributes. Defaults are what
ship when nobody remembers to configure them.

```typescript
// services/auth/src/api/plugins/security.ts
await fastify.register(helmet);
await fastify.register(cors, { origin: false, credentials: false });
await fastify.register(rateLimit, { global: false });

// services/auth/src/api/routes/v1/session/routes.ts
config: { rateLimit: { max: 5, timeWindow: '1 minute' } },
```

The rate limiter is registered with `global: false`, so each route opts in; sign-in and
public registration do. The refresh cookie is `__Host-` prefixed with `secure`,
`httpOnly`, and `sameSite`.

```typescript
// Avoid: an unauthenticated route that stores state per call, with no limit.
fastify.get('/:provider/authorize', { schema: { /* ... */ } }, startOAuthBrowserFlow);
```

Related: GP-22, GP-14.

**Check:** every unauthenticated route that creates or changes state declares
`config.rateLimit`.

### GP-24 Tests at contract seams

Tests exercise behavior through the same contracts the composition root injects, using
in-memory fakes for backing services in the fast tier and real backing services in an
integration tier. Tests coupled to drivers are slow and break on infrastructure changes
that do not change behavior.

```typescript
// services/auth/src/api/api.ts
/**
 * Constructs the auth Fastify application without binding a network listener.
 */
export async function buildApi(options: BuildApiOptions): Promise<ZodFastifyInstance> {
```

Because `buildApi` takes its services as options and binds no port, a test can inject
fakes for `ICrud`, `ISigningKeyProvider`, and `IOAuthProvider` and call `app.inject`. The
strategy is planned in `docs/plans/tests-suite.md`.

```typescript
// Avoid: a unit test that needs a live database to check a validation rule.
const crud: ICrud<UserRecord> = new MongoCrud(realMongoDb.collection('users'));
```

Related: GP-13, GP-03.

**Check:** each package has a test command that runs in CI.

## Benefits and trade-offs

Applied together, these practices give the platform:

- **Reproducibility**: the same commit builds the same artifact (GP-05, GP-06, GP-12).
- **Safe configuration**: invalid settings stop startup and secrets stay out of files and
  images (GP-08, GP-09, GP-10).
- **Replaceable infrastructure**: backing services swap by configuration (GP-11, GP-21).
- **Operability**: processes start, stop, and report health predictably, and failures are
  classified and traceable (GP-14 to GP-20).
- **Contained damage**: least privilege and secure defaults limit what a compromise can do
  (GP-22, GP-23).

The practices have costs. Pinned toolchains need deliberate upgrades, CI gates slow every
change slightly, and readiness checks add work to every backing service. Apply them in
proportion: a practice marked "should" may be skipped with a documented reason.

Two deliberate deviations from the Twelve-Factor App are accepted:

- **Configuration file plus environment secrets**: configuration lives in a validated JSON
  file, not entirely in environment variables. Only secrets and credentials come from the
  environment (GP-09). The file gives one reviewed, schema-checked document per
  deployment; the cost is that it must stay free of secrets.
- **Startup tasks instead of one-off processes**: dataset provisioning and bootstrap-admin
  reconciliation run at every startup. Both are idempotent (GP-17), so running them on each
  start is safe and removes a manual step.

## Relation to the Twelve-Factor App

How each factor maps to the practices above:

- **I Codebase**: [GP-01](#gp-01-one-repository-per-package-or-service).
- **II Dependencies**: [GP-05](#gp-05-declared-and-locked-dependencies),
  [GP-06](#gp-06-pinned-toolchain), [GP-07](#gp-07-supply-chain-safeguards).
- **III Config**: [GP-08](#gp-08-validated-configuration-fail-fast),
  [GP-09](#gp-09-secrets-and-credentials-only-from-the-environment),
  [GP-10](#gp-10-configuration-varies-per-deployment-not-per-build).
- **IV Backing services**:
  [GP-11](#gp-11-backing-services-are-attached-resources-behind-contracts).
- **V Build, release, run**: [GP-12](#gp-12-immutable-artifacts-deployed-by-identity),
  [GP-13](#gp-13-quality-gates-run-in-ci).
- **VI Processes**: [GP-14](#gp-14-stateless-processes).
- **VII Port binding**: met by Fastify listening on `http.host` and `http.port`; no
  separate practice.
- **VIII Concurrency**: met by stateless processes (GP-14); no separate practice until a
  service needs worker process types.
- **IX Disposability**: [GP-15](#gp-15-disposability-fast-start-graceful-stop),
  [GP-16](#gp-16-liveness-and-readiness).
- **X Dev/prod parity**: [GP-21](#gp-21-development-and-production-parity).
- **XI Logs**: [GP-18](#gp-18-logs-as-an-event-stream).
- **XII Admin processes**:
  [GP-17](#gp-17-administrative-tasks-are-idempotent-and-operator-owned).

Where the platform goes beyond the factors:

- **Architecture inside the process**: layers, contracts, and a composition root (GP-02,
  GP-03), which the factors leave open.
- **Shared definitions**: schemas as the single source of truth (GP-04) and classified
  errors (GP-20).
- **Operational safety**: supply-chain safeguards (GP-07), readiness (GP-16), correlation
  ids (GP-19), security defaults (GP-22, GP-23), and testing (GP-24).

## References

### Foundational sources

- Wiggins, Adam. [The Twelve-Factor App](https://12factor.net/). Heroku, 2011.
- Humble, Jez, and David Farley. *Continuous Delivery: Reliable Software Releases through
  Build, Test, and Deployment Automation*. Addison-Wesley, 2010.
- Nottingham, Mark, Erik Wilde, and Sanjay Dalal.
  [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457). IETF,
  2023.

### Supplementary sources

- npm Docs.
  [Generating provenance statements](https://docs.npmjs.com/generating-provenance-statements).
- SLSA. [Supply-chain Levels for Software Artifacts](https://slsa.dev/).
