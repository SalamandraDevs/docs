---
title: Auth Service Architecture Build Plan
lastUpdate: 2026-09-01
category: architecture
lang: en
status: implemented
---

# Auth Service Architecture Build Plan

## Purpose

Build the structural foundation of `services/auth` as a Fastify REST API. The result must
contain enough working code to make the intended architecture, dependency direction, route
composition, and OpenAPI contract visible to a human reviewer.

This phase builds the bones and cartilage of the service. It does not implement complete
authentication protocols, cryptographic key management, or production
authorization policy. Those details are deliberately deferred until the architecture, patterns,
and principles used here have been reviewed.

## Required outcome

At the end of this phase:

- The source tree and contracts express how the auth service will create, compose, and close a
  Fastify instance; compilation success is not a phase requirement.
- Common API infrastructure is separated from versioned routes.
- Version 1 route families are registered as encapsulated Fastify plugins.
- Route schemas are the source for request validation, response serialization, TypeScript
  inference, and an OpenAPI 3.1 document.
- The integration point between Fastify requests and the Core execution context is explicit and
  usable by existing entity classes.
- Password and OAuth authentication methods have visible route boundaries.
- `services/profiles` is represented as the first internal OIDC provider, without implementing
  its provider-side protocol in this service.
- Unimplemented security-sensitive behavior fails explicitly. It must not appear to work
  through development credentials or permissive fallbacks.
- A reviewer can evaluate naming, encapsulation, dependency inversion, separation of concerns,
  and route design before deeper implementation begins.

## Mandatory constraints

1. Do not implement a production JWT signer, key rotation system, refresh-token rotation,
   password hashing, or external OIDC token exchange.
3. Do not copy the legacy Express middleware behavior. In particular, do not branch on URL
   paths or HTTP methods inside shared hooks.
4. Treat `config/local.json` as the authoritative configuration source. Only
   `src/config/local.ts` may read `process.env`, and only `DB_SERVER_URI` may override the file
   because deployment connection strings can contain credentials.
5. Do not use `dummyExecutionCtx` in API runtime code.
6. Do not place business workflows in controllers or route plugins.
7. Do not introduce filesystem discovery for authentication plugins. Register a fixed, typed
   set of known plugins from validated configuration.
8. Do not add test files, run test commands, run `tsc`, run the TypeScript build, or attempt to
   resolve TypeScript errors solely to make the build pass during this phase.
9. Every function, type alias, interface, class, Fastify plugin, hook, controller, and schema
   constant must have a meaningful JSDoc comment.
10. Use NodeNext `.js` extensions in relative and package import specifiers.
11. Give variable declarations explicit type annotations. The only lint rule permitted for this
    phase is the existing `@typescript-eslint/typedef` rule with
    `{ variableDeclaration: true }`.

## Implementer role and phase discipline

The implementer’s role in this phase is to connect architecture and policies, not to finish a
production-ready implementation. Prioritize these links:

- configuration → composition root → API construction;
- common plugins → encapsulated version plugins → feature routes;
- route schemas → validation, serialization, and OpenAPI metadata;
- authentication boundary → trusted principal → Core execution context;
- controllers → application-service ports → existing entities;
- authentication configuration → enabled or disabled method plugins;
- OIDC provider identifiers → typed provider registry → future provider integration.

Use minimal explicit placeholders where machinery is deferred. A placeholder may document a
contract or throw a typed not-implemented error, but it must not invent fake credentials,
tokens, keys, challenges, or authorization results.

Do not spend phase time on compiler repairs, other lint-rule compliance,
formatting-driven refactors, test scaffolding, or unrelated cleanup. A syntax or type issue
may be recorded for the review handoff when fixing it would distract from the architecture
being expressed.

Do not run `pnpm lint`, because it loads the complete ESLint configuration and applies fixes.
Run ESLint in isolated mode only for `@typescript-eslint/typedef`; no other ESLint rule is part
of the implementation or review criteria for this phase.

## Architecture decisions expressed by the skeleton

### Composition root

`src/bootstrap.ts` remains the composition root. It owns construction and lifecycle, including
MongoDB connection, dataset provisioning, application services, and the Fastify application.
Lower layers receive dependencies rather than constructing global services.

`src/index.ts` remains the process entry point. It loads configuration, starts the runtime,
installs signal handlers, and delegates shutdown. It must not contain route or plugin
definitions.

### Transport and application separation

Files under `src/api/` are HTTP transport code. Controllers translate an HTTP request into an
application-service call and translate its result into an HTTP response. Business policy must
live under `src/application/` or the existing entity modules so a future API version can reuse
it.

### Fastify encapsulation

Each route family is a Fastify plugin. Common plugins are ancestors of all API versions.
Version hooks apply only to versioned routes. Authentication-method hooks apply only to the
corresponding method subtree.

Use `fastify-plugin` only for infrastructure that intentionally decorates the registration
context. Do not wrap route plugins with it because that would break route prefixing and weaken
encapsulation.

### Honest scaffolding

Working health and OpenAPI endpoints are appropriate in this phase. Incomplete authentication
operations must return or throw a typed not-implemented problem. They must not issue
placeholder tokens, accept arbitrary credentials, or use hard-coded authenticated identities.

## Proposed source layout

```text
services/auth/config/
└── local.json

services/auth/src/
├── index.ts
├── bootstrap.ts
│
├── config/
│   ├── local.ts
│   ├── schema.ts
│   └── types.ts
│
├── api/
│   ├── index.ts
│   ├── app.ts
│   ├── types.ts
│   │
│   ├── plugins/
│   │   ├── index.ts
│   │   ├── openapi.ts
│   │   ├── security.ts
│   │   ├── cookies.ts
│   │   ├── authentication.ts
│   │   ├── execution-context.ts
│   │   └── errors.ts
│   │
│   ├── schemas/
│   │   ├── common.ts
│   │   ├── pagination.ts
│   │   └── problem.ts
│   │
│   ├── routes/
│   │   ├── health/
│   │   │   ├── routes.ts
│   │   │   ├── controller.ts
│   │   │   └── schemas.ts
│   │   └── discovery/
│   │       ├── routes.ts
│   │       ├── controller.ts
│   │       └── schemas.ts
│   │
│   └── v1/
│       ├── index.ts
│       ├── hooks/
│       │   ├── authenticate.ts
│       │   └── authorize.ts
│       └── routes/
│           ├── users/
│           │   ├── routes.ts
│           │   ├── controller.ts
│           │   ├── hooks.ts
│           │   ├── schemas.ts
│           │   └── types.ts
│           ├── sessions/
│           │   ├── routes.ts
│           │   ├── controller.ts
│           │   ├── hooks.ts
│           │   ├── schemas.ts
│           │   └── types.ts
│           └── auth/
│               ├── index.ts
│               ├── types.ts
│               ├── password/
│               │   ├── plugin.ts
│               │   ├── routes.ts
│               │   ├── controller.ts
│               │   ├── hooks.ts
│               │   └── schemas.ts
│               ├── oidc/
│               │   ├── plugin.ts
│               │   ├── routes.ts
│               │   ├── controller.ts
│               │   ├── hooks.ts
│               │   ├── schemas.ts
│               │   ├── providers.ts
│               │   └── types.ts
│
├── application/
│   ├── auth/
│   │   ├── ports.ts
│   │   ├── session-service.ts
│   │   ├── password-service.ts
│   │   ├── oidc-service.ts
│   │   └── token-service.ts
│   └── users/
│       └── users-service.ts
│
├── infrastructure/
│   ├── keys/
│   │   ├── signing-key-provider.ts
│   │   └── jwks.ts
│   └── oidc/
│       └── provider-registry.ts
│
└── entities/
    └── ... existing entity modules
```


## File and symbol responsibilities

### Configuration

#### `config/schema.ts`

Define JSDoc-commented Zod schemas for the complete service configuration. The schema must at
least express:

- MongoDB URI and database name.
- Host, port, and API prefix.
- OpenAPI title, service version, and exposure settings.
- Token issuer and audience identifiers.
- Cookie name and secure defaults.
- Enabled authentication methods.
- OIDC provider configuration, including the `profiles` provider.

The schema validates and normalizes parsed values. It must not read files, environment
variables, discovery documents, or secrets. Use named transforms for normalization so their
intent can be reviewed independently from the schema declaration.

#### `config/local.ts`

Resolve and parse `config/local.json` as unknown input, apply the optional `DB_SERVER_URI`
environment override, then validate and transform the result through the Zod service schema.
Export the resulting validated configuration value. Invalid JSON or schema input must fail
before database or HTTP startup. No other environment variable belongs in this phase.

Resolve the file before startup, then keep file reading, JSON parsing, override application, and
Zod validation or transformation inside one fail-fast `try`/`catch` boundary. Any failure must
throw a `CustomError` during module evaluation so `bootstrapService()` and the HTTP listener are
never reached.

#### `config/local.json`

Provide the human-authored configuration for the local service, including HTTP, OpenAPI,
authentication-method, cookie, MongoDB, and OIDC-provider settings. Keep deployable secrets out
of this file; the database server URI may be supplied through `DB_SERVER_URI` when credentials are
required.

#### `config/types.ts`

Export documented configuration types inferred from the schemas where practical. Use specific
nested types such as `HttpConfig`, `AuthenticationConfig`, and `OidcProviderConfig` when they
make ownership clear.

### API construction

#### `api/index.ts`

Expose only the API construction contract needed by the composition root. Avoid a broad barrel
that makes internal controllers or schemas public.

#### `api/app.ts`

Export a documented `buildApi(options)` function that:

1. Creates the Fastify instance with structured logging and request IDs.
2. Installs the Zod validator and serializer compilers.
3. Registers common plugins in a deterministic order.
4. Registers unversioned operational and discovery routes.
5. Registers the v1 plugin under the configured prefix.
6. Returns the instance without calling `listen()`.

The composition root decides when to call `listen()` and `close()`.

#### `api/types.ts`

Define and document:

- `BuildApiOptions`, containing configuration and application dependencies.
- `AuthenticatedPrincipal`, containing at least `actorId`, `tenantId`, and the token subject
  needed by transport authorization.
- Fastify module augmentation for decorated services, authentication functions,
  execution-context binding, and `request.principal`.

Do not add optional request fields and then rely on non-null assertions. Public routes have no
principal; protected route hooks establish one before protected controllers execute.

### Common plugins

#### `api/plugins/index.ts`

Export a documented `registerApiPlugins()` plugin or function. Registration order must be
visible and intentional:

1. Error representation and shared schemas.
2. Security headers, CORS, cookies, and request policy.
3. Zod/OpenAPI support.
4. Authentication and execution-context decorators.

Dynamic OpenAPI generation must be registered before routes.

#### `api/plugins/openapi.ts`

Register `@fastify/swagger` for OpenAPI 3.1 and an API reference UI only when configured.
Configure the Zod-to-JSON-Schema transformation for Draft 2020-12. Expose the generated
document at a stable unversioned path such as `/openapi.json`.

Every exported plugin function and configuration helper requires JSDoc.

#### `api/plugins/security.ts`

Express the intended boundary for Helmet, explicit CORS origins, body limits, and rate
limiting. Implement only configuration that is understood and safe in this phase. Do not use
wildcard origins together with credentialed requests.

Route-specific credential rate limits belong on the password and OIDC route plugins, not in
URL-aware global logic.

#### `api/plugins/cookies.ts`

Register cookie parsing and centralize cookie options. The configuration contract must express
`HttpOnly`, `Secure`, `SameSite`, path, and the preferred `__Host-` name. Do not issue session
cookies until the token/session service is implemented.

#### `api/plugins/authentication.ts`

Define the transport authentication boundary and decorate Fastify with a documented
authentication hook. The skeleton may delegate to a token verification port and fail explicitly
when no implementation is provided.

Do not reproduce the legacy behavior that accepts either a cookie or bearer token on every
endpoint. Credential transport must be an explicit route or service configuration decision.

#### `api/plugins/execution-context.ts`

Bridge authenticated Fastify requests to the execution context implemented in `@slmdevs/core`.
This plugin is described in detail in the execution-context section below.

#### `api/plugins/errors.ts`

Install one documented error handler that maps known application errors, validation errors,
authentication errors, authorization errors, and intentionally unimplemented operations to RFC
9457 `application/problem+json` responses.

Unexpected errors are logged with the request ID. Responses must not expose stack traces,
secrets, raw provider responses, or database details.

### Common schemas

#### `api/schemas/common.ts`

Define documented reusable Zod schemas for IDs, timestamps, and shared headers.

#### `api/schemas/pagination.ts`

Define documented cursor request and response schemas. It is acceptable for v1 route handlers
not to implement pagination yet; the types should show the intended contract without inventing
data.

#### `api/schemas/problem.ts`

Define the documented RFC 9457 response schema and any intentionally supported extension
fields, such as a validation-error list or request ID.

### Unversioned routes

#### Health

`health/routes.ts` registers operational routes. `health/controller.ts` contains named,
documented handlers. `health/schemas.ts` declares complete response schemas. Health is
functional in this phase and does not require authentication.

Keep liveness independent from MongoDB. A readiness endpoint may report dependency state if the
composition root can provide it without adding a new monitoring framework.

#### Discovery and JWKS

`discovery/routes.ts` exposes the stable JWKS resource. `discovery/controller.ts` delegates to
a `SigningKeyProvider` port. `discovery/schemas.ts` defines public JWK and JWK Set response
shapes without private key members.

The skeleton may return an empty key set only when clearly configured as a development
skeleton. It must not generate ephemeral signing keys or publish a symmetric secret. The actual
signer, persistent key source, rotation, cache policy, and overlap period are review-follow-up
work.

Do not expose OAuth authorization-server or OpenID Provider metadata from auth unless a later
decision makes auth itself a provider. In the current design auth is an OIDC client of
`services/profiles`.

### Version 1 composition

#### `api/v1/index.ts`

Register v1 route families with explicit prefixes. Registration order must not be used as
hidden authorization policy.

#### Shared v1 hooks

`hooks/authenticate.ts` obtains and validates a principal through the decorated authentication
port. `hooks/authorize.ts` contains small, documented authorization hook factories such as
`requireRole()` or `requireScope()` only if the current domain contract already defines those
concepts.

Do not invent a role hierarchy merely to populate the skeleton. A minimal
`requireAuthenticatedPrincipal` boundary is enough for review.

### Feature module convention

For implemented route families, use these roles consistently:

- `routes.ts`: route declarations, prefixes, schemas, hooks, and handler binding.
- `controller.ts`: named thin HTTP handlers with no persistence or cryptography.
- `hooks.ts`: feature-specific Fastify hooks; never path-aware middleware.
- `schemas.ts`: Zod request and response schemas plus OpenAPI metadata.
- `types.ts`: HTTP-specific types not cleanly inferred at the route declaration.

Avoid creating an empty file only to satisfy the convention. If a feature has no
feature-specific hook or type yet, omit that file and add it when the responsibility exists.
The proposed layout shows expected architectural slots, not a requirement for meaningless
exports.

### Users routes

Create route definitions sufficient to show the resource boundary:

```text
GET    /users
POST   /users
GET    /users/{userId}
PATCH  /users/{userId}
DELETE /users/{userId}
```

Controllers delegate to `UsersService`. If the use cases are incomplete, return a typed
not-implemented problem. Never serialize `passwordHash` or reuse the stored entity schema as
the public user response.

Move transport-only request schemas out of entity modules when implementing their API
equivalents. Existing persisted entity schemas remain authoritative for stored records.

### Session routes

Create route definitions sufficient to express session ownership:

```text
GET    /sessions/current
DELETE /sessions/current
POST   /sessions/refresh
```

Session controllers depend on `SessionService`. Token issuance, refresh rotation, revocation,
and cookies remain explicit unimplemented operations unless an existing reviewed implementation
already supplies them through a port.

### Authentication-method plugins

#### Auth composition

`api/v1/routes/auth/index.ts` receives validated authentication configuration and registers a
known method plugin only when enabled. Disabled methods register no routes and therefore do not
appear in the generated OpenAPI document.

`auth/types.ts` defines the small common plugin contract. Do not force password and OAuth into a
large strategy interface when their ceremonies differ. Share only lifecycle and
result concepts that are genuinely common.

#### Password

The password plugin reserves a session-creation endpoint:

```text
POST /auth/password/sessions
```

Its controller delegates to `PasswordService`. Request and safe response schemas must be
complete, but credential checking, timing defenses, hashing, token issue, and cookie issue may
remain typed unimplemented behavior for the review build.

#### OIDC

The auth service acts as an OpenID Connect relying party/client. The first provider ID is
`profiles`, representing `services/profiles` as the future OpenID Provider.

Reserve these routes:

```text
POST /auth/oidc/{provider}/authorizations
GET  /auth/oidc/{provider}/callback
```

With the requested API prefix, the first route resolves to:

```text
POST /api/v1/auth/oidc/profiles/authorizations
```

`oidc/providers.ts` describes supported provider IDs. The infrastructure provider registry maps
`profiles` to typed issuer/discovery configuration. Do not hard-code the profiles host in a
controller.

The skeleton validates provider IDs and expresses the redirect/callback response contracts.
Discovery retrieval, Authorization Code exchange, PKCE, state, nonce, ID-token validation,
account linking, and provider-side endpoints are outside this phase. An authorization request
must fail explicitly until those protections exist; it must not construct a partially protected
redirect.

The provider-side implementation in `services/profiles` is a separate build plan. For profiles
to become a real provider, it will eventually need discovery, authorization, token, JWKS, and
user-information capabilities.

## Core execution-context integration

Core currently defines the following immutable context:

```ts
type ExecutionContext = Readonly<{
  authenticated: Readonly<{
    actorId: string;
    tenantId: string;
  }>;
  request: Readonly<{
    requestId: string;
  }>;
}>;
```

The integration must use these existing Core exports:

- `runWithExecutionCtx(context, callback)`: binds a context to the async work created by the
  callback.
- `getExecutionCtx()`: reads the bound context from entities and policies.
- `MissingExecutionContextError`: signals code running outside a bound scope.
- `dummyExecutionCtx`: development placeholder; forbidden in API runtime code.

### Required Fastify lifecycle

1. Fastify establishes `request.id`.
2. The authentication hook validates credentials and assigns a trusted `AuthenticatedPrincipal`
   to `request.principal`.
3. The execution-context hook builds:

   ```ts
   {
     authenticated: {
       actorId: request.principal.actorId,
       tenantId: request.principal.tenantId
     },
     request: {
       requestId: request.id
     }
   }
   ```

4. The hook invokes the continuation from inside
   `runWithExecutionCtx(executionContext, continuation)` so downstream hooks, controllers,
   application services, and entity calls inherit the `AsyncLocalStorage` scope.
5. Existing entities continue to call `getExecutionCtx()` and do not depend on Fastify types.

Implement the binding as a named, JSDoc-commented callback-style `preHandler` or an
equivalently verified wrapper. Calling `runWithExecutionCtx()` around only the body of an async
hook is insufficient if the downstream Fastify lifecycle is started after that callback
returns.

Protected route hook order must be visible:

```text
authenticate -> bind execution context -> authorize -> controller
```

Public routes do not receive a fabricated authenticated context. Background jobs that use
entities must independently call `runWithExecutionCtx()` with a trusted job actor, tenant, and
request/correlation ID.

## Application and infrastructure boundaries

### Application ports and services

`application/auth/ports.ts` defines the documented interfaces needed by transport code, such as
token verification, signing-key lookup, and OIDC provider lookup. Keep ports small and
consumer-oriented.

The service files express use-case ownership and constructor dependencies. Their methods may
throw a typed not-implemented error in this phase, but their public signatures must use
domain/application inputs rather than Fastify request types.

### Key infrastructure

`SigningKeyProvider` returns public JWK material for discovery and eventually provides
signing-key access through a separate private operation. The public JWKS model must exclude
private fields and symmetric keys by construction.

`infrastructure/keys/jwks.ts` may contain mapping helpers between the provider port and the
public response. It must not generate or persist keys in this phase.

### OIDC provider registry

`infrastructure/oidc/provider-registry.ts` implements typed provider lookup from validated
configuration. Unknown providers must produce a safe client error.

Provider identity must eventually be established by the stable issuer and subject pair, not by
email. That account-linking machinery is deferred.

## OpenAPI 3.1 contract

Use Fastify route schemas as the single source of truth. Since the service already uses Zod 4,
use a compatible Zod Fastify type provider and configure its OpenAPI transformation for Draft
2020-12.

The root document must declare OpenAPI 3.1. Each operation included in the skeleton must
define:

- A stable and unique `operationId`.
- Tags and a concise summary.
- Params, query, headers, and body schemas where applicable.
- Explicit response schemas for every returned status.
- `security` requirements, including an explicit empty requirement for public operations.
- RFC 9457 problem responses for errors.

Response schemas are security boundaries. Fastify serialization must prevent accidental output
of password hashes, provider tokens, internal MongoDB fields, or private JWK members.

The OpenAPI document should include declared security schemes even though token issuance is
deferred. Do not document an OAuth flow that this service does not yet implement. The external
OIDC provider relationship can be described in operation text until its protocol is functional.

## API prefix review item

The requested design uses `/api/v1`, including `/api/v1/auth/oidc/profiles/authorizations`. The
repository naming guide currently specifies `/{service}/v{N}`, which would produce
`/auth/v1/...`.

Do not spread either prefix through route files. Model the base prefix once in validated
configuration and register v1 through nested Fastify prefixes. Use the requested `/api/v1`
contract for the skeleton, then ask the architecture reviewer to decide whether the deployed
service or gateway must expose `/auth/v1` instead. The decision should require a configuration
change, not a route refactor.

## JSDoc standard for this build

JSDoc is part of the deliverable, not cleanup work. At minimum:

- Every function states its responsibility, parameters, return value, and relevant thrown
  errors.
- Every type alias and interface explains the boundary it models.
- Important type fields document trust or security semantics.
- Every class explains its role and dependency ownership.
- Every Fastify plugin explains its scope, decorators, and registration prerequisites.
- Every hook explains its lifecycle position and established guarantees.
- Every controller explains the HTTP operation it serves.
- Every Zod schema constant explains the request, response, or persisted shape it validates.

Prefer named functions over anonymous route handlers so documentation and stack traces identify
the architectural role. Comments must explain intent or contract, not restate a symbol name.

## Implementation sequence

### Phase 1: configuration and contracts

1. Extend service configuration types and add the parsed local JSON source with Zod validation
   and normalization.
2. Add API, application-port, error, and principal types.
3. Add the required Fastify and OpenAPI dependencies using versions compatible with Fastify 5
   and the installed Zod version.
4. Update TypeScript import aliases only where they clarify stable layer imports.

### Phase 2: API foundation

1. Build `api/app.ts` without listening on a socket.
2. Add shared problem schemas and the error handler.
3. Add security, cookie, OpenAPI, authentication, and execution-context plugin boundaries.
4. Add functional health and OpenAPI endpoints.
5. Add the JWKS port and discovery route without key generation.

### Phase 3: version and feature composition

1. Register the v1 plugin and feature prefixes.
2. Add thin users and sessions route/controller/schema skeletons.
3. Add password authentication route skeletons.
4. Add the OIDC provider registry, `profiles` configuration, and route skeletons.

### Phase 4: runtime integration

1. Update `bootstrap.ts` to construct application dependencies and the API.
2. Return a runtime that closes Fastify before closing the database connection.
3. Update `index.ts` to start the server and preserve graceful signal shutdown.
4. Remove no-longer-needed development shortcuts from the runtime path without altering
   unrelated entity behavior.

### Phase 5: architecture review handoff

1. Inspect the source tree for the intended layers, plugin scopes, policy links, and dependency
   direction.
2. Confirm that deferred operations are explicit placeholders rather than fake implementations.
3. Confirm that route schemas, OpenAPI metadata, authentication hooks, and execution-context
   binding are connected in the intended order.
4. Run only an isolated `@typescript-eslint/typedef` lint check. Do not run `pnpm lint` or load
   the rest of the project ESLint rules.
5. Record intentionally unimplemented operations and architecture-review decisions.

Do not add or run automated tests, TypeScript checks, full-project lint commands, formatting
commands, or build commands during this phase unless the user separately changes the scope. The
isolated `@typescript-eslint/typedef` check is the only lint exception.

## Definition of done

The structural build is ready for human architecture review when:

- The target layers and plugin scopes are recognizable in source paths.
- The API instance can be constructed and closed independently of the process entry point.
- Fastify registration order is deterministic.
- `/health` and `/openapi.json` work when enabled.
- The emitted document identifies itself as OpenAPI 3.1 and contains the skeleton routes
  actually registered.
- OIDC configuration recognizes `profiles`, but protected redirects and callbacks remain
  explicitly unimplemented.
- Protected-route declarations show authentication followed by execution-context binding.
- Entity calls can inherit `actorId`, `tenantId`, and `requestId` through Core's
  `AsyncLocalStorage` contract once a trusted principal exists.
- Public routes do not fabricate an execution context.
- Controllers do not access MongoDB, cryptographic libraries, or environment variables
  directly.
- Route response schemas cannot expose `passwordHash` or private key material.
- Every introduced function, type, class, plugin, hook, controller, and schema has meaningful
  JSDoc.
- The review handoff identifies any type, lint, formatting, or build issues without treating
  their resolution as a completion criterion.

## Human architecture review checklist

The review following this build should explicitly decide:

- Whether the public prefix is `/api/v1`, `/auth/v1`, or rewritten by a gateway.
- Whether application-service and entity-policy responsibilities are separated at the correct
  boundary.
- Whether Fastify decorators are appropriately narrow and encapsulated.
- Whether the Core execution-context binding is correct across the downstream Fastify
  lifecycle.
- Whether auth should own JWT issuance or delegate private-key operations to a separate
  security component or KMS.
- Whether `services/profiles` should own the full OpenID Provider responsibility.
- Whether session cookies, bearer tokens, or separate route profiles will be the supported
  credential transports.
- Which roles, scopes, and tenant policies are real domain concepts rather than transport
  inventions.
- Which intentionally unimplemented files should be completed, changed, or removed before
  production authentication work begins.

## Deferred work

The following work begins only after the architectural review:

- Password verification and password-hashing policy.
- JWT signing, strict validation, key persistence, JWKS caching, and key rotation.
- Session storage, refresh rotation, revocation, and secure cookie issuance.
- OIDC discovery, PKCE, state, nonce, callback validation, token exchange, and account linking.
- Provider-side OIDC implementation in `services/profiles`.
- Detailed authorization policies, audit events, production rate limits, and abuse controls.
- Automated tests, contract tests, security tests, and performance tests.
