---
title: Build Services
lastUpdate: 2026-11-14
lang: en
category: guide
status: published
---

## Purpose

House rules for building Fastify services on the Salamandra Devs platform: how a service
applies the general [Development Good Practices][gp] (`GP`) to its transport layer, route
hooks, health endpoints, request correlation, and error responses. These rules apply to
every contributor, human or agent, working in `services/` or in a package that ships
Fastify plugins.

Each rule names the GP practice or DP pattern it applies. That practice or pattern states
the platform-wide principle; the BS rule states how a Fastify service meets it. When the
two seem to disagree, the practice or pattern wins and the BS rule is corrected.

BS-05 to BS-08 were CS-09 to CS-12 in [Code Style](./code-style.md) until 2026-11-13; the
CS IDs are retired and never reused.

Each rule section carries a stable ID (`BS-NN`) for citation; IDs follow
[Document IDs](./naming-conventions.md#document-ids). Every section ends with a
**Check:** line, a condition a reviewer or an agent can verify in the repository. Examples
are trimmed excerpts that start with a comment naming their source file; counter-examples
are marked `Avoid` and are hypothetical.

## Transport

### BS-01 Secure transport defaults

HTTP services enable security headers, explicit CORS, rate limits, and secure cookie
attributes. Every route must declare its rate-limit tier, and the service refuses to start
when a route declares none, so a new route is never unlimited by accident; a route is
unlimited only when it explicitly declares the exempt tier. Tiers count by the narrowest
identity known at each point: the gateway meters anonymous traffic by client IP (public
routes and requests whose API key failed) and keyed traffic by the tenant its API key
names; each service behind the gateway meters public traffic by client IP, the
authenticated subject once a token is verified, and the account named in the request on
credential routes. Tier values come from configuration, never literal numbers in code.
Behind a proxy, the service trusts only the proxy's addresses, so limits count real
clients. The shared security plugin from `@salamandradevs/http` enforces the base-tier
guard in each service behind the gateway; the gateway's own API key plugin enforces its
base tier, which also admits the API key hook.

```typescript
// services/auth/src/api/routes/v1/session/routes.ts
onRequest: tenantHooks(fastify),
preHandler: accountRateLimitHooks(fastify, 'username'),

// services/auth/src/api/routes/health/routes.ts
config: { tenant: false },
onRequest: exemptRateLimitHooks(fastify),
```

A route without a tier stops startup with a classified `CONFIG_ERROR` ([GP-08][gp-08]),
and an exceeded limit answers a classified 429 with a `retry-after` header
([BS-04][bs-04]). The refresh cookie is `__Host-` prefixed with `secure`, `httpOnly`, and
`sameSite`.

```typescript
// Avoid: a route with no tier, a limit fixed in code, and a proxy trusted for any
// caller.
fastify.get('/', { preHandler: protectionHooks(fastify) }, readCurrentSession);
config: { rateLimit: { max: 5, timeWindow: '1 minute' } },
Fastify({ trustProxy: true });
```

Applies: [GP-25][gp-25]; [GP-10][gp-10]; [GP-14][gp-14].

**Check:** every route declares exactly one base tier (public or exempt), enforced at
startup; at the gateway every keyed route declares the API key hook as its base tier, and
every tenant-scoped service route declares the tenant hook, each refusing to start
otherwise; no route declares limit values in code; credential routes add a per-account
limit; `trustProxy` is never `true`.

## Operations

### BS-02 Liveness and readiness

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

Applies: [GP-15][gp-15]; [GP-03][gp-03].

**Check:** a readiness endpoint exists and the Dockerfile or compose file declares a
`HEALTHCHECK` or `healthcheck`.

### BS-03 Request correlation

Every request carries a correlation id that is accepted from the caller in the
`x-request-id` header when present, attached to every log line, and bound into the
execution context for downstream code. Fastify fills `request.id` from that header
through the `requestIdHeader` option, and the execution-context plugin copies it into
`ExecutionContext.request.requestId`.

```typescript
// http/src/plugins/execution-context.ts
const executionContext: ExecutionContext = {
  authenticated: { actorId: request.sessionGrant.actorId, /* ... */ },
  request: { requestId: request.id }
};

options.executionCtx.run(executionContext, done);
```

```typescript
// Avoid: a call to another platform service that drops the correlation id.
await fetch(`${profilesUrl}/v1/persons/${recordId}`);
```

A call to another platform service forwards the id through the shared client
`makeApiRequest` from `@salamandradevs/http`. The `executionCtx` option is omitted by
default, so a third-party call never carries the platform's id; a platform peer call
passes the service's execution context provider, whose bound request id is sent as
`x-request-id`.

```typescript
// Prefer: forward the correlation id to a platform peer.
await makeApiRequest(profilesUrl, `/v1/persons/${recordId}`, { method: 'GET' }, {
  executionCtx
});
```

Applies: [GP-18][gp-18]; [DP-09][dp-09].

**Check:** the Fastify instance sets `requestIdHeader: 'x-request-id'`, and outbound calls
to other platform services forward `x-request-id`.

### BS-04 Problem responses

Transport maps every `ErrorName` to an RFC 9457 problem response with a stable problem
`type` URI. Each URI is a slug resolved against one base built from the configured API
origin, so no host is written in code.

```typescript
// services/auth/src/api/plugins/index.ts
problemTypeBase: new URL('/problems/', options.config.publicUrls.api)

// http/src/plugins/errors.ts
} else if (error.name === 'RESOURCE_NOT_FOUND') {
  status = 404;
  slug = 'resource-not-found';
// ...
type: new URL(slug, options.problemTypeBase).toString(),
```

```typescript
// Avoid: a problem type with a host fixed in code.
type = 'https://api.example.com/problems/resource-not-found';
```

Applies: [GP-20][gp-20]; [GP-10][gp-10].

**Check:** every problem `type` URI is resolved against one base built from configuration,
and no source file contains a problem-type host.

## Route hooks

How RESTful API services compose Fastify route hooks: where shared hooks live, when a
route group may define its own, and how route-local hooks stay typed without assertions.
The hook arrays are chains of responsibility ([DP-06][dp-06]).

### BS-05 Shared hooks

- Cross-cutting or reusable route hooks must be published as Fastify plugin decorations.
  This includes authentication, execution-context binding, rate/tenant/security hooks, and
  common authorization hooks. Do not export free-floating helper functions from shared hook
  modules and pass them directly as Fastify lifecycle hooks.
- Use `api/routes/v1/hooks/` as the preferred source for shared v1 hook composition. These
  files should compose decorated Fastify hooks into ordered arrays for shared route
  policies.
- A shared composer may take parameters when it builds a hook from a plugin decoration,
  such as `fastify.rateLimitHook(policy)`. Each call creates a new hook instance, which is
  how a configured policy gets its own state per route. The parameters configure the
  decorated hook; they never replace it with a free-floating function.

```ts
// api/routes/v1/hooks/authenticate.ts
export function protectionHooks(fastify: FastifyInstance): PreHandlerHooks {
  return [
    fastify.authenticateRequest,
    fastify.authenticatedRateLimit,
    fastify.bindExecutionContext
  ];
}

// api/routes/v1/hooks/rate-limit.ts
export function accountRateLimitHooks(
  fastify: FastifyInstance,
  field: 'username' | 'email'
): PreHandlerHooks {
  const policy = fastify.config.http.rateLimit.credential.perAccount;

  return [fastify.rateLimitHook(policy /* , keyGenerator from request.ip + field */)];
}
```

Applies: [DP-06][dp-06]; [GP-02][gp-02].

**Check:** no shared hook module exports a free function that a route passes directly as a
lifecycle hook; composers in `api/routes/v1/hooks/` return arrays of `fastify.*`
decorations.

### BS-06 Route-local hooks

- In `api/routes/v1/[slug]/`, add a local `hooks.ts` only when the route needs
  slug-specific hook composition or highly local route policy. Local hook files may compose
  shared v1 hooks with decorated Fastify hooks, and may define private route-local hooks
  only when the policy is not shared outside that route group.

```ts
// api/routes/v1/users/hooks.ts
function requireUserOwnerOrAdmin(request: FastifyRequest): void {
  // Route-local policy that is not shared by other route groups.
}

export function userOwnerProtectionHooks(fastify: FastifyInstance): PreHandlerHooks {
  return [...protectionHooks(fastify), requireUserOwnerOrAdmin];
}
```

- When a local route policy becomes useful outside one route group, promote it to a
  Fastify plugin decoration and compose it through `api/routes/v1/hooks/` instead of
  importing it as a shared helper.

Applies: [DP-06][dp-06].

**Check:** every hook defined in a route-local `hooks.ts` is used only by its own route
group.

### BS-07 Controllers

- Controllers must assume declared hooks already established route preconditions. Do not
  repeat authentication, execution-context, tenant, or common authorization checks in
  controllers unless the controller is handling a distinct runtime state that hooks cannot
  establish.

Applies: [DP-06][dp-06].

**Check:** no controller re-checks authentication, execution context, tenant, or roles that
its route's hooks already establish.

### BS-08 Typed route-local hooks

- A route-local hook that needs typed access to route-specific request parts (`params`,
  `body`, `querystring`, `headers`) is typed through a route def type, never through
  `FastifyRequest` augmentation and never through assertions. Augmentation is reserved for
  route-invariant decorations such as `sessionGrant` or cursor codecs.
- Type the hook parameter as `FastifyRequest<TRouteDef>`, compose the hook array through
  the shared `PreHandlerHooks<TRouteDef>` helper from `@salamandradevs/http`, and
  reuse the same route def at route registration and in the controller so the request
  contract has one source of truth. Define `TRouteDef` in `api/routes/v1/[slug]/types.ts`.
- Keep shared, route-invariant composers on the default `PreHandlerHooks` generic and put
  the specialization on the route-local composer; a default-generic hook composes into a
  specialized array, never the reverse.
- Clue: if a hook needs asserting `request.params`, `body`, or `query` (`as ...`) to
  typecheck, the route def approach was not applied; introduce a route def and thread it
  through the composer instead of asserting.

```ts
// http/src/hooks/types.ts
export type PreHandlerHooks<
  TRouteDef extends RouteGenericInterface = RouteGenericInterface
> = preHandlerHookHandler<
  RawServerDefault,
  RawRequestDefaultExpression,
  RawReplyDefaultExpression,
  TRouteDef
>[];

// api/routes/v1/users/hooks.ts
function authorizeUserUpdate(request: FastifyRequest<UpdateUserRequest>): void {
  const targetRecordId: string = request.params.recordId; // inferred, no assertion
}

export function userUpdateProtectionHooks(
  fastify: FastifyInstance
): PreHandlerHooks<UpdateUserRequest> {
  return [...protectionHooks(fastify), authorizeUserUpdate];
}
```

Applies: [DP-06][dp-06]; [GP-04][gp-04].

**Check:** no hook or controller asserts `request.params`, `body`, or `query` with `as`;
route-specific request parts are typed through a route def in the slug's `types.ts`.

## References

- Nottingham, Mark, Erik Wilde, and Sanjay Dalal.
  [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457). IETF,
  2023.

[bs-04]: #bs-04-problem-responses
[dp-06]: ../architecture/design-patterns.md#dp-06-chain-of-responsibility
[dp-09]: ../architecture/design-patterns.md#dp-09-ambient-context
[gp]: ../architecture/development-good-practices.md
[gp-02]: ../architecture/development-good-practices.md#gp-02-layered-service-structure
[gp-03]: ../architecture/development-good-practices.md#gp-03-contracts-composition-root-wiring-and-configured-backends
[gp-04]: ../architecture/development-good-practices.md#gp-04-schemas-as-the-single-source-of-truth
[gp-08]: ../architecture/development-good-practices.md#gp-08-validated-configuration-fail-fast
[gp-10]: ../architecture/development-good-practices.md#gp-10-configuration-varies-per-deployment-not-per-build
[gp-14]: ../architecture/development-good-practices.md#gp-14-stateless-processes
[gp-15]: ../architecture/development-good-practices.md#gp-15-disposability-fast-start-graceful-stop
[gp-18]: ../architecture/development-good-practices.md#gp-18-logs-as-an-event-stream
[gp-20]: ../architecture/development-good-practices.md#gp-20-classified-errors
[gp-25]: ../architecture/development-good-practices.md#gp-25-secure-by-default-and-least-privilege
