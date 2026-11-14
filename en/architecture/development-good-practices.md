---
title: Development Good Practices
lastUpdate: 2026-11-13
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
this guide. How Fastify services apply these practices (transport security, health
endpoints, request correlation, problem responses) lives in the house
[Build Services](../house/build-services.md) guide (`BS`).

Retired IDs, never reused: GP-06 merged into GP-05, GP-11 into GP-03, and GP-22 into
GP-25; the service rules of GP-16, GP-19, and GP-23 moved to BS-02, BS-03, and BS-01, and
the practices behind them are part of GP-15, GP-18, and GP-25.

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

## Semantic commit messages

Commit subjects use the semantic prefix convention: a `<type>` tag, an optional
`(<scope>)`, and a short imperative subject. When a commit resolves a reviewed gap, name
the involved IDs in the subject or body, for example `GP-09`.

Format: `<type>(<scope>): <subject>`; the scope is optional.

The types follow the [Semantic Commit Messages][scm] convention; `fix`, `feat`, `chore`,
and `docs` are the ones the repositories use most:

- **feat**: a new feature for the user, not a new feature for a build script.
- **fix**: a bug fix for the user, not a fix to a build script.
- **docs**: changes to the documentation.
- **style**: formatting, missing semicolons, and similar; no production code change.
- **refactor**: refactoring production code, such as renaming a variable.
- **test**: adding or refactoring tests; no production code change.
- **chore**: updating build tooling, tasks, or dependencies; no production code change.

[scm]: https://gist.github.com/joshbuchea/6f47e86d2510bce28f8e7f42ae84c716

## Platform Examples

The examples are trimmed excerpts from `core/`, `tools/`, and `services/auth/`, including
their `package.json`, Dockerfile, configuration, and workflow files. Each one starts with a
comment naming its source file. Counter-examples are marked `Avoid` and are hypothetical:
they show what would break the practice, not code that exists.

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

### GP-03 Contracts, composition-root wiring, and configured backends

A class consumed across a boundary is consumed through a contract, even when it has one
implementation, and concrete classes are created only in the composition root or in a
factory it calls. A boundary is a seam between the sections of
[GP-02](#gp-02-layered-service-structure) (transport, application, storage, external
adapters), between two use-case modules, or between a service and a shared package. The
consumer depends only on what it calls, and the composition root decides what it receives,
so the class behind the contract can be replaced by a second implementation or by a test
fake without touching the consumer. Constructing collaborators inside policy code hides
dependencies and blocks both. Backing services (databases, key stores, identity
providers) are also located by configuration, so replacing or relocating one is a
configuration change, not a code release.

- **Needs a contract**: transport consuming a use case (`ISessionsService` in
  `api/types.ts`), a use case consuming another (`ITokensService` in `SessionsService`), and
  any consumer of an adapter (`ISigningKeyProvider`, `IOAuthProvider`).
- **Does not**: classes used only inside their own module or only in `runtime/`
  (`DatabaseProvider`, `DatabaseAdapter`), and records handlers, whose substitution point
  is the `ICrud` beneath them ([DP-03](./design-patterns.md#dp-03-bridge)).

```typescript
// services/auth/src/runtime/services.ts
const tokens: ITokensService = new TokensService(config.auth, deps.signingKeys);
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
  private readonly tokens: ITokensService = new TokensService(loadConfig().auth, /* ... */);
}

// Avoid: a backend and its location fixed in code.
const client: MongoClient = new MongoClient('mongodb://mongo:27017');
```

Related: GP-09; [DP-01](./design-patterns.md#dp-01-factory),
[DP-08](./design-patterns.md#dp-08-composition-root),
[SP-05](./solid-principles.md#sp-05-dependency-inversion-principle-dip).

**Check:** `new` of an infrastructure class (`MongoCrud`, `DatabaseProvider`, a provider)
appears only under `runtime/`, `infra/*/factory.ts`, or `core/src/infra/`; no type under
`src/api/` or in a use case's constructor names a concrete application or adapter class;
switching or relocating a backing service changes configuration only.

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

### GP-05 Locked dependencies and pinned toolchain

Every dependency is declared and locked, and installs use the frozen lockfile; pnpm and
the Node.js major are pinned alike in development, CI, and images, or two builds of one
commit can differ.

```dockerfile
# services/auth/Dockerfile
FROM node:22-alpine
RUN corepack enable && corepack prepare pnpm@12.4.1 --activate
RUN pnpm install --frozen-lockfile
```

```yaml
# Avoid: an unlocked install and a runtime unlike the image.
- run: pnpm install --no-frozen-lockfile
- uses: actions/setup-node@v4
  with: { node-version: '24' }
```

Related: GP-12, GP-21.

**Check:** `pnpm-lock.yaml` is tracked and installs use `--frozen-lockfile`;
`node-version`, `FROM node:`, and `engines` agree; CI tools carry explicit versions.

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

The file is validated (GP-08) before environment values are resolved, so a placeholder
must pass its field's format: a `baoAddr` validated by `z.url()` uses
`http://READ-FROM-ENV-BAO_ADDR`, not `READ FROM ENV BAO_ADDR`.

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

Related: GP-08, GP-03.

**Check:** no committed configuration file contains a credential; every sensitive schema
field carries `.meta({ env })`; every placeholder in a committed configuration file parses
under its field's schema; every declared variable appears in `.env.example` and in every
deployment manifest.

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

### GP-12 Immutable artifacts, deployed by identity

A build produces one immutable artifact per commit, and deployments and rollbacks
reference it by an immutable identifier (commit SHA or version tag). With a moving tag,
"what is running" and "how to roll back" have no exact answer.

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

Related: GP-03, GP-15.

**Check:** no long-lived `Map`, array, or module-level variable in a service holds
request-derived state.

### GP-15 Disposability: fast start, graceful stop

A service starts without manual steps, releases every resource it opened on
`SIGTERM`/`SIGINT`, and releases already-opened resources when startup fails part way.
While it runs, it tells its orchestrator whether it is alive and whether it can serve,
so traffic reaches only an instance whose required backing services are reachable.
Orchestrators start, stop, and move processes routinely; a process that leaks
connections, hangs on stop, or takes traffic it cannot serve degrades the platform.

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

Related: GP-14, GP-03;
[BS-02](../house/build-services.md#bs-02-liveness-and-readiness).

**Check:** every resource opened in `runtime/` is closed on both the failure path and the
shutdown path, and every deployed process declares a health check its orchestrator uses.

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
with the level configurable per deployment. Every log line carries the correlation id of
the unit of work it belongs to (a request, a job, a message); the id is accepted from the
caller when present and forwarded on every call to another platform service. Files inside
containers are lost with the container and are invisible to log collectors, and log lines
without a correlation id cannot be traced to one failure across lines or services.

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

// Avoid: a call to another platform service that drops the correlation id.
await fetch(`${profilesUrl}/v1/persons/${recordId}`);
```

Related: GP-20; [DP-09](./design-patterns.md#dp-09-ambient-context);
[BS-03](../house/build-services.md#bs-03-request-correlation).

**Check:** no code writes log files; every logging variable is passed by every deployment
manifest; outbound calls to other platform services forward the correlation id.

### GP-20 Classified errors

Every error that code throws is a `CustomError` with a deliberate `ErrorName`
([CS-04](../house/code-style.md#cs-04-errors)), and a vendor or driver failure is
converted at the adapter that catches it, keeping the original as the cause. Every
boundary that exposes errors to a client maps those names to a stable, documented
response. Unclassified errors reach clients as generic failures and cannot be alerted on.

```typescript
// core/src/infra/database/mongo/mongo-crud.ts
} catch (error: unknown) {
  if (isMongoDupKeyError(error)) {
    throw new CustomError({
      name: 'DB_DUPLICATE_KEY_ERROR',
      message: errMessages.DB_DUPLICATE_KEY,
      cause: error
    });
  }

  throw new CustomError({
    name: 'DB_BACKEND_ERROR',
    message: 'Failed to create MongoDB record',
    cause: error
  });
}
```

```typescript
// Avoid: an unclassified error that the boundary can only report as a generic failure.
throw new Error('User not found');
```

Related: GP-18; [BS-04](../house/build-services.md#bs-04-problem-responses).

**Check:** no `throw new Error` or `throw new TypeError` escapes a function without being
wrapped in a `CustomError`.

### GP-21 Development and production parity

Development uses the same kinds of backing services as production, and every difference
is explicit, named, and limited to adapters selected by configuration. A backend used only
in production hides failures until release.

```typescript
// Avoid: a development-only adapter that starts silently, so a misconfiguration ships it
// to production unnoticed.
case 'dev-insecure': {
  return new InsecureDevKeyProvider(config.provider);
}
```

`docker-compose.dev.yml` builds the same Dockerfile and joins the same `services` network
with MongoDB and OpenBao. `JoseKeysProvider` is not a development-only adapter: it is a
production provider for small deployments (one host or replicas sharing one key file), so
it announces nothing.

```yaml
# Avoid: a development-only database engine that production never runs.
databaseBackend: sqlite
```

Related: GP-10, GP-05.

**Check:** every development-only adapter announces itself at startup and is rejected or
warned about in production configuration; an adapter fit for production, such as
`JoseKeysProvider`, carries no such warning.

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
strategy is planned in `docs/plans/261029-tests-suite-plan.md`.

Contracts are what make those fakes cheap. TypeScript compares a class with `private`
members nominally, so a plain object cannot stand in for `SessionsService`, but it can
stand in for `ISessionsService`. A transport test fakes the use cases and checks routing,
hooks, and error mapping; a use-case test fakes the contracts it consumes and checks
policy; neither needs a backing service.

```typescript
// Avoid: a unit test that needs a live database to check a validation rule.
const crud: ICrud<UserRecord> = new MongoCrud(realMongoDb.collection('users'));
```

Related: GP-13, GP-03.

**Check:** no fast-tier test opens a connection to a backing service; fakes enter through
the same contracts the composition root injects.

### GP-25 Secure by default and least privilege

Every new component that exposes or grants something (an HTTP route, a CLI command, a
message consumer, a token, a cookie, a container, an access policy) is created in its
most restrictive state, and while it runs it holds only the permissions its work needs, so
a compromised component cannot escalate or destroy data. Loosening either is an explicit,
named declaration in code or configuration, and when a required declaration is missing
the component refuses to start or build instead of falling back to open. A safe state that
must be remembered is lost the first time someone forgets it.

```typescript
// http/src/plugins/security.ts
fastify.addHook('onRoute', (routeOptions: RouteOptions): void => {
  const onRequest: unknown[] = [routeOptions.onRequest ?? []].flat();
  const tiers: number = onRequest.filter(
    (hook: unknown): boolean => hook === publicRateLimit || hook === exemptRateLimit
  ).length;

  if (tiers !== 1 || routeOptions.config?.rateLimit !== undefined) {
    throw new CustomError({
      name: 'CONFIG_ERROR',
      message: `Route ${String(routeOptions.method)} ${url} needs one rate-limit tier`
    });
  }
});
```

```dockerfile
# services/auth/Dockerfile
USER nodeuser
```

A new route cannot start unlimited, and the exempt tier is the only, reviewable way out.

```typescript
// Avoid: a component that is open unless someone remembers to close it.
const requireAuth: boolean = options.requireAuth ?? false;
```

```hcl
# Avoid: a service token that can also rewrite or destroy its own keys.
path "transit/keys/*" {
  capabilities = ["create", "update", "delete"]
}
```

Related: GP-08, GP-09, GP-20;
[BS-01](../house/build-services.md#bs-01-secure-transport-defaults).

**Check:** every relaxation of a security or exposure default is an explicit declaration,
and no component falls back to open when it is missing; no container runs as root, and
each token's policy lists only the operations the service performs.

## Benefits and trade-offs

Applied together, these practices make builds reproducible (GP-05, GP-12), configuration
safe (GP-08 to GP-10), infrastructure replaceable (GP-03, GP-21), processes operable and
traceable (GP-14 to GP-20), and damage contained (GP-25).

The practices have costs. Pinned toolchains need deliberate upgrades, CI gates slow every
change slightly, and readiness checks add work to every backing service. Apply them in
proportion: a practice marked "should" may be skipped with a documented reason.

Two deliberate deviations from the Twelve-Factor App are accepted:

- **Configuration file plus environment secrets**: one validated JSON file per deployment
  holds the configuration, and only secrets come from the environment (GP-09); the cost is
  that the file must stay free of secrets.
- **Startup tasks instead of one-off processes**: dataset provisioning and bootstrap-admin
  reconciliation run at every startup, which is safe because both are idempotent (GP-17).

## Relation to the Twelve-Factor App

- **I Codebase**: GP-01. **II Dependencies**: GP-05, GP-07. **III Config**: GP-08 to GP-10.
- **IV Backing services**: GP-03. **V Build, release, run**: GP-12, GP-13.
- **VI Processes** and **VIII Concurrency**: GP-14; no worker process types yet.
- **VII Port binding**: Fastify listens on `http.host` and `http.port`; no practice.
- **IX Disposability**: GP-15. **X Dev/prod parity**: GP-21. **XI Logs**: GP-18.
- **XII Admin processes**: GP-17.
- **Beyond the factors**: layers and contracts (GP-02, GP-03), schemas (GP-04), classified
  errors (GP-20), supply chain (GP-07), tests (GP-24), and secure defaults (GP-25).

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
