---
title: Durable Signing Key Provider Implementation Plan
lastUpdate: 2026-09-30
lang: en
category: architecture
status: published
---

# Durable Signing Key Provider Implementation Plan

## Goal

Replace the in-memory signing key with a durable provider whose private key material remains
outside the auth process. Use OpenBao Transit as the local key-management service and preserve
an application contract that can later be implemented by another signing-key provider.

The implementation must retain the established signing-key lifecycle:

1. Load durable lifecycle state and its active provider key reference during service startup.
2. Create the first key when no signing state exists.
3. Publish a new public key before using the corresponding private key.
4. Atomically activate the new key and retire the previous key.
5. Keep retired public keys in JWKS until their `verifyUntil` timestamp.
6. Remove expired public keys and destroy their private material.

This plan deliberately avoids a signing-key entity, repository, deposit abstraction, background
scheduler, and provider factory. OpenBao owns the durable key material and a small lifecycle
state document. The auth service depends only on `ISigningKeyProvider`.

## Current Status

Implemented, then reduced to an operator-owned lifecycle (AUTH-023). The running service
performs only the four `ISigningKeyProvider` operations: it reads one KV lifecycle document,
validates it strictly, signs with OpenBao Transit, and serves verification and JWKS from an
in-process snapshot. It never creates, rotates, activates, retires, or destroys a key, and it
never writes lifecycle state.

Every mutation the original design gave the provider is now a human operator action, driven
through the OpenBao CLI and documented in `docs/en/howto/openbao/README.md`, which is the
procedure of record. The current signing key is the one an operator marks `active` in the
document, selected by status and never by recency, KV version, or Transit key version; see
`## Operator-Owned Lifecycle` for the full reconciliation.

The following sections describe the original automated engine and are retained as design
history, not as current service behavior: `### Opportunistic maintenance`, first-key creation
in `### Initialization`, `### Creating a key`, `### Rotation and activation`, the destruction
half of `### JWKS retention and destruction`, `### Startup preflight`, and the compare-and-set
write protocol referenced throughout. Each now reads as what an operator does by hand. Sign-in,
verification, and JWKS publication were never automated mutations, so their sections stay
current.

## Mandatory Implementation Rule

When implementation requires behavior beyond this plan, ask the developer before choosing a new
architecture or adding another interface, class, persistence model, or runtime service.

## Current Starting Point

The current implementation has these relevant pieces:

- `EphemeralSigningKeyProvider` generates one ES256 key pair with `jose` and keeps it in
  memory.
- Restarting the auth process invalidates access tokens signed by the previous process.
- Multiple auth replicas would generate unrelated signing keys and publish different JWKS data.
- `TokenService` depends on `TokenSigningKeyProvider` and receives a private `CryptoKey`.
- `TokenService` uses `SignJWT` for access-token construction and `jwtVerify` for verification.
- `SigningKeyProvider` exposes only `listPublicKeys()` to the discovery API.
- `/.well-known/jwks.json` already publishes the provider's `PublicJwk` values.
- Signing and JWKS configuration currently live under `AuthConfig`, but durable provider
  details must move to infrastructure-owned key configuration.

The private-key-returning contract is incompatible with managed signing backends. Durable
providers keep private keys under their own custody and expose signing operations instead.

## Architecture Decisions

### One application contract

Create one `ISigningKeyProvider` contract in `application/auth/contracts.ts`. The
`OpenBaoSigningKeyProvider` and a future provider can implement it.

Do not add an `ISigningKeysDeposit`, key repository, or generic managed-key client interface.
The OpenBao implementation may use private helper functions and private transport types
without exposing its storage design to the application.

### Issuer-scoped keys

Signing keys belong to the configured access-token issuer, not to a tenant or session. All
tenants whose tokens share the same `iss` value use the same signing-key set.

No signing-key field is added to `SessionRecord`. A JWT carries its `kid` in the protected
header, and the JWKS maps that identifier to the corresponding public key.

### External private-key custody

OpenBao Transit generates ES256-compatible `ecdsa-p256` keys and performs signing. The auth
service must never receive, serialize, persist, cache, or log a private signing key.

A future `ISigningKeyProvider` implementation must follow the same rule. It may use its own
key-management backend and metadata store internally, but those details remain behind
`ISigningKeyProvider`.

### OpenBao owns lifecycle state

Use OpenBao Transit for key material and OpenBao KV v2 for one lifecycle state document. The KV
document provides the compare-and-set revision needed for atomic activation across auth
replicas.

This state is infrastructure metadata, not an auth entity. It does not use Core `Entities`,
MongoDB, the request execution context, or tenant scoping.

### One backend key per rotation

Create a separate OpenBao Transit key for every rotation instead of coupling the application
model to OpenBao key-version semantics. Each lifecycle entry maps a public `kid` to one private
OpenBao Transit key name.

A future `ISigningKeyProvider` implementation can map the same lifecycle model to its own
provider-owned key identifiers.

### Opportunistic maintenance

> Superseded; see `## Current Status`. The running service runs no lifecycle maintenance; the
> transitions here are operator CLI actions in `docs/en/howto/openbao/README.md`.

Do not add a scheduler in this phase. Run lifecycle maintenance:

- once during `initialize()` at service startup;
- before selecting a signer for access-token issuance; and
- before listing public keys for JWKS.

This makes rotation and cleanup occur on the next relevant operation. A dedicated scheduled
worker can be introduced later if rotation must happen without auth or JWKS traffic.

## Expected Module Layout

Create or change these modules. Supporting barrel exports and imports may also change when
required by the listed moves; do not add another architectural layer.

```text
services/auth/
├── config/
│   └── local.json                                  # auth, OAuth, and signing keys config
├── src/
│   ├── application/
│   │   ├── config/schema.ts                        # provider-neutral auth policy
│   │   ├── config/types.ts                         # provider-neutral auth policy
│   │   ├── contracts.ts                            # ISigningKeyProvider, ITokenService
│   │   ├── sessions-service/                       # signing-failure consistency
│   │   └── tokens-service/                         # renamed TokenService implementation
│   ├── infra/keys/
│   │   ├── config/schema.ts                        # shared signing config validation
│   │   ├── config/types.ts                         # shared signing config types
│   │   ├── types.ts                                # shared lifecycle key types
│   │   └── openbao/
│   │       ├── index.ts                            # provider module barrel
│   │       ├── provider.ts                         # durable provider implementation
│   │       ├── schema.ts                           # config and response validation
│   │       └── types.ts                            # OpenBao config and transport types
│   ├── api/
│   │   ├── plugins/errors.ts                       # safe infrastructure error logging
│   │   ├── routes/discovery/controller.ts          # JWKS response and cache header
│   │   └── types.ts                                # ApiServices signing key reference
│   ├── config/
│   │   ├── schema.ts                               # composed config validation
│   │   └── types.ts                                # global config composition
│   └── runtime/
│       ├── keys.ts                                 # provider construction and initialization
│       └── runtime.ts                              # composition root
└── docs/en/howto/openbao/                          # operator runbook and local runtime
    ├── docker-compose.yml                          # local OpenBao and durable volume
    ├── config.hcl                                  # non-development server configuration
    ├── auth-signing-policy.hcl                     # least-privilege auth policy
    └── README.md                                   # provisioning and troubleshooting runbook
```

The implemented layout uses `src/infra/` rather than `src/infrastructure/`, keeps application
services in per-service directories directly under `src/application/`, and splits the
composition root across `src/runtime/` instead of a single `bootstrap.ts`. Provider classes are
imported through the module barrel (`#infra/keys/openbao/index.js`), matching how
`#infra/oauth/index.js` is consumed; configuration schemas stay deep-imported.

Delete `application/auth/token-service.ts` after renaming it to `tokens-service.ts`. Replace
the contents of `infrastructure/keys/signing-key-provider.ts`; either remove that file or use
it only as an index export to `openbao/provider.ts`. Do not retain
`EphemeralSigningKeyProvider` as a production fallback. Treat this refactor as a breaking
change with no backward compatibility.

## Types and Contracts

### Signing key handle

Add `SigningKeyHandle` to `application/auth/types.ts`:

```ts
/**
 * Signing capability bound to one selected provider-managed key.
 * - kid: Identifier written to the JWT protected header.
 * - algorithm: JOSE algorithm supported by the selected key.
 * - sign: Signs an exact JWS signing input without exposing private key material.
 */
export type SigningKeyHandle = Readonly<{
  kid: string;
  algorithm: 'ES256';
  sign: (input: Uint8Array) => Promise<Uint8Array>;
}>;
```

`application/auth/types.ts` currently contains per-property JSDoc. Because this implementation
changes that file, consolidate the documentation for every type in it into one type-level JSDoc
block with one one-line bullet per field, as required by `AGENTS.md`. Do not leave a mixture of
the two documentation shapes in the same file.

The handle captures one selected provider key. If rotation occurs after the handle is
returned, its `sign()` function must continue signing with the captured key rather than
silently switching keys. This guarantees that the signature matches the `kid` already written
to the protected header.

### Signing provider contract

Replace `SigningKeyProvider` and `TokenSigningKeyProvider` with this single contract in
`application/auth/contracts.ts`:

```ts
/** Durable signing-key lifecycle and cryptographic operations boundary. */
export interface ISigningKeyProvider {
  /**
   * Loads durable state and ensures that a current signing key exists.
   * @throws {CustomError} When provider state cannot be initialized safely.
   */
  initialize(): Promise<void>;

  /**
   * Selects the active provider-managed signing key.
   * @returns Signing capability bound to the selected key.
   */
  getCurrentSigner(): Promise<SigningKeyHandle>;

  /**
   * Resolves one eligible public verification key.
   * @param kid - Exact key identifier read from a JWT protected header.
   * @returns Matching public key, or null when no eligible key exists.
   */
  getVerificationKey(kid: string): Promise<CryptoKey | null>;

  /**
   * Lists public verification keys eligible for JWKS publication.
   * @returns Pending, active, and unexpired retired public keys.
   */
  listPublicKeys(): Promise<readonly PublicJwk[]>;

  // Future implementation: run lifecycle maintenance explicitly when a scheduler exists.
  // maintenance(): Promise<void>;
}
```

The commented `maintenance()` method documents a future scheduler-facing contract. In the
shipped service there is no lifecycle maintenance to run: the provider only reads, validates,
signs, and publishes, and every transition is an operator action (see `## Current Status`).
Initialization, signer selection, verification, and JWKS publication are the whole surface.

`getVerificationKey()` returns `null` only when a successfully loaded state does not contain an
eligible key with the requested `kid`. Provider communication, state validation, and JWK import
failures must reject the promise. This keeps an expected lookup miss separate from an
infrastructure failure. Represent rejected provider operations with `CustomError` and an
appropriate `ErrorName`.

### Shared provider lifecycle types

Keep provider lifecycle state in `infra/keys/types.ts`. These types describe the auth service's
durable signing-key lifecycle, not OpenBao transport details. They are shared inside
`infra/keys` so a future `ISigningKeyProvider` implementation can reuse the same lifecycle
model.

Do not add a `JwksDocument` type or a `createJwksDocument()` factory. The discovery route
already declares `jwksDocumentSchema` as its response contract, so a second one-field wrapper
would restate that contract without validating or transforming anything; the controller sends
`{ keys }` directly.

```ts
/** Lifecycle status of one provider-managed signing key. */
export type SigningKeyStatus = 'pending' | 'active' | 'retired';

/**
 * Durable metadata for one provider-managed signing key.
 * - kid: Public identifier written to JWT protected headers.
 * - providerKeyId: Opaque identifier understood only by the provider implementation.
 * - status: Current lifecycle status.
 * - publicJwk: Public verification key published through JWKS.
 * - activateAfter: Earliest time when a pending key may become active.
 * - activatedAt: Time when the key became active, or null while pending.
 * - retiredAt: Time when the key became retired, or null otherwise.
 * - verifyUntil: Last time when a retired key remains eligible for verification.
 */
export type SigningKeyEntry = Readonly<{
  kid: string;
  providerKeyId: string;
  status: SigningKeyStatus;
  publicJwk: PublicJwk;
  activateAfter: string;
  activatedAt: string | null;
  retiredAt: string | null;
  verifyUntil: string | null;
}>;

/**
 * Durable lifecycle state shared by all auth service replicas.
 * - currentKid: Identifier of the only active key, or null before initial key creation.
 * - keys: Complete provider-managed signing-key lifecycle.
 */
export type SigningKeyState = Readonly<{
  currentKid: string | null;
  keys: readonly SigningKeyEntry[];
}>;
```

`providerKeyId` is an opaque provider-owned identifier. The OpenBao provider stores the Transit
key name there. A future `ISigningKeyProvider` implementation can store its own key identifier
without changing the shared lifecycle model.

Do not export these types from the auth application package surface. They are infrastructure
implementation types, not `ISigningKeyProvider` contract types.

Use ISO 8601 strings in the durable lifecycle document and convert them to dates only while
evaluating policy. The OpenBao KV response metadata version is the compare-and-set revision and
does not need to be stored inside the state document.

Validate the complete state before publishing it to any reader:

- `kid` and `providerKeyId` values are non-empty and unique;
- every public JWK is an ES256 EC P-256 public key with matching `kid`, `alg`, and `use`
  values;
- `currentKid` identifies the only active entry;
- there is exactly one active entry and at most one pending entry after initialization;
- pending entries have no activation, retirement, or verification-expiry timestamps;
- active entries have `activatedAt` and no retirement or verification-expiry timestamps; and
- retired entries have `activatedAt`, `retiredAt`, and `verifyUntil` timestamps in
  chronological order.

The only valid state without an active key is a missing document observed during first-key
initialization. Once a state document exists, missing or contradictory active-key data is an
error and must not trigger automatic replacement.

### OpenBao transport types

Keep OpenBao-specific API types in `infrastructure/keys/openbao/types.ts`. They must not be
exported from the auth application package surface or from shared key infrastructure modules.

Add narrowly typed OpenBao response types and matching Zod schemas only for endpoints that the
provider calls:

- KV state read and write responses;
- Transit key read response containing its public key;
- Transit signing response containing the signature; and
- OpenBao error response.

Parse response bodies as `unknown` and validate them before use. Do not cast unvalidated JSON
to the response types. Keep the response schemas in `openbao/schema.ts` beside the
configuration schema. Do not build a general OpenBao SDK or expose these types outside the
OpenBao provider directory.

Define each response schema once and infer its TypeScript response type with `z.infer`. Do not
maintain a handwritten response shape beside an equivalent schema. Document inferred object
types with one type-level JSDoc block and one one-line bullet per returned field.

### OpenBao response contract

These are properties of the OpenBao HTTP API, not preferences. Each one silently breaks a
request path if a schema contradicts it, so verify against the linked documentation before
changing a response schema.

- Response schemas must not be `strict`. OpenBao wraps every logical response in an envelope
  carrying `request_id`, `lease_id`, `lease_duration`, `renewable`, `warnings`, `wrap_info`,
  `auth`, and `mount_type`, and KV version metadata adds `created_time`, `custom_metadata`,
  `deletion_time`, and `destroyed`. Strict object schemas reject all of it. Reserve `.strict()`
  for documents this service writes itself: the provider configuration and the lifecycle state.
- A KV v2 read and a KV v2 write return different shapes. A read answers
  `{ data: { data, metadata } }`; a write answers the new version metadata directly under
  `data`, with no nested `data` or `metadata`. They need two schemas.
- A Transit read of an asymmetric key maps each version to an object holding `public_key`,
  `name`, `certificate_chain`, and `creation_time`. Only symmetric key types map a version to a
  bare creation timestamp, so a `Record<string, string>` shape never matches `ecdsa-p256`.
- `deletion_allowed` is accepted only by `POST /transit/keys/:name/config`. The create-key
  endpoint ignores it, and the key then refuses `DELETE` with a `400`, which the 404-tolerant
  cleanup path does not absorb.
- A compare-and-set conflict is a `400` whose error reads `check-and-set parameter did not
  match the current version`. Match that text specifically: `check-and-set parameter required
  for this call` and `error parsing check-and-set parameter` share the prefix but are mount or
  request faults that no retry can resolve.
- `marshaling_algorithm: 'jws'` returns the concatenated `r||s` form JWT requires, encoded with
  URL-safe base64 rather than standard base64.
- Transit reports a missing key inconsistently: a read answers `404`, while `DELETE` and
  `/config` answer `400`, with `could not delete key; not found` and `no existing key named
  ...` respectively. Cleanup must treat those `400` bodies as success, or a maintenance pass
  aborts whenever another replica destroyed the same expired key first.
- A trailing `*` in a policy path is a prefix match that crosses `/`, so
  `transit/keys/auth-signing-*` already grants the `/config` subpath used for
  `deletion_allowed`. No separate policy rule is required.

References: [Transit secrets engine API](https://openbao.org/api-docs/secret/transit/),
[KV v2 secrets engine API](https://openbao.org/docs/secrets/kv/kv-v2/), and
[Policies](https://openbao.org/docs/concepts/policies/).

## Configuration Ownership

Keep `application/auth/config` provider-neutral. It should contain only policies that
authentication application services consume directly:

- default tenant and actor identifiers used by the public execution context;
- access-token issuer, audience, lifetime, and accepted clock tolerance;
- refresh-token lifetime and cookie settings; and
- configured authentication methods.

Move `SigningKeysConfig` to `infra/keys/config/types.ts` along with its `schema.ts`
counterpart. That module owns signing-key lifecycle policy, JWKS publication cache policy, and
composition of the selected provider configuration.

Move `OpenBaoSigningKeysConfig` to `infra/keys/openbao/types.ts`. Move its
provider-specific schema to `infra/keys/openbao/schema.ts`. OpenBao connection
settings belong to the OpenBao provider boundary, not the shared signing config module.

The global service configuration composes both modules:

```ts
/**
 * Validated configuration consumed by the auth composition root.
 * - dbServerUri: Connection URI passed to the configured database driver.
 * - databaseName: Database selected when constructing persistence.
 * - http: Listener and route-prefix settings used during API startup.
 * - openApi: API-document generation and endpoint-exposure policy.
 * - auth: Provider-neutral authentication and token policy.
 * - signingKeys: Infrastructure signing-provider and JWKS policy.
 * - oauth: Provider-specific OAuth configuration keyed by provider name.
 */
export type ServiceConfig = Readonly<{
  dbServerUri: string;
  databaseName: string;
  http: HttpConfig;
  openApi: OpenApiConfig;
  auth: AuthConfig;
  signingKeys: SigningKeysConfig;
  oauth: OAuthConfig;
}>;
```

This keeps the dependency direction clean. `application/auth` does not import
`infrastructure/keys`. The composition root imports both because it is responsible for wiring
runtime dependencies.

### Infrastructure signing config

Create shared signing-key config in `infra/keys/config/schema.ts` and infer its type in the
sibling `types.ts`:

```ts
/**
 * Provider-neutral signing lifecycle and JWKS policy schema.
 * - algorithm: Only JOSE signing algorithm supported by this service.
 * - rotationIntervalSecs: Number of seconds between active-key rotations.
 * - verificationOverlapSecs: Time a retired key remains available for verification.
 * - jwks: Public JWKS cache policy.
 * - provider: Configuration for the only provider implemented in this phase.
 */
export const signingKeysConfigSchema = z
  .object({
    algorithm: z.literal('ES256'),
    rotationIntervalSecs: z.number().int().positive(),
    verificationOverlapSecs: z.number().int().positive(),
    jwks: z.object({ cacheMaxAgeSecs: z.number().int().positive() }).strict(),
    provider: openBaoSigningKeysConfigSchema
  })
  .strict();
```

Do not give the `jwks` section a separate exported `JwksConfig` alias. It is one field read in
one place, and `SigningKeysConfig['jwks']` already names it where a signature needs it.

Create OpenBao signing-key config in `infra/keys/openbao/types.ts`:

```ts
/**
 * Connection and key namespace used by the OpenBao signing provider.
 * - type: Provider discriminator.
 * - transitMount: Transit secrets-engine mount name.
 * - kvMount: KV v2 secrets-engine mount name.
 * - statePath: KV v2 path containing the lifecycle state document.
 * - keyNamePrefix: Prefix used to derive provider-managed Transit key names.
 */
export type OpenBaoSigningKeysConfig = Readonly<{
  type: 'openbao';
  transitMount: string;
  kvMount: string;
  statePath: string;
  keyNamePrefix: string;
}>;
```

Keep the matching OpenBao schema beside that provider module and export it for the shared
signing config schema to compose. The shared schema should not define OpenBao connection
fields directly.

`SigningKeysConfig.algorithm`, the `SigningKeyHandle.algorithm` literal, Transit key creation,
persisted public JWK metadata, and the verification allowlist must all resolve to `ES256`.
Validate that agreement at the infrastructure boundary. `TokensService` takes the algorithm
from the selected handle during issuance and retains its explicit ES256 allowlist during
verification; it does not read infrastructure signing configuration directly.

For the initial implementation, the provider discriminator accepts only `openbao`. Do not add a
provider factory or configuration union before a second provider implementation exists.

Apply these validations in the owning schemas:

- `BAO_ADDR` is an HTTP or HTTPS URL without embedded credentials;
- mount names, state path, and key prefix are non-empty strings;
- lifecycle durations and `cacheMaxAgeSecs` are positive integers;
- `rotationIntervalSecs` is greater than `jwks.cacheMaxAgeSecs`; and
- `verificationOverlapSecs` is at least the sum of the access-token lifetime, accepted clock
  tolerance, and JWKS cache duration.

The last validation crosses application and infrastructure configuration. Perform it once in
`src/config/schema.ts` after `authConfigSchema` and `signingKeysConfigSchema` have parsed their
own sections. Do not duplicate that cross-section rule inside either child schema.

Read the connection from `BAO_ADDR` and `BAO_TOKEN`, the names the OpenBao CLI and SDK use
natively, so one pair of variables serves the shell, the container and this service. Honour no
`VAULT_*` fallback: the CLI accepts one for upstream compatibility, and that fallback is how a
process silently authenticates with a token issued for a different server.

Neither value belongs in `local.json`; the address is deployment specific and the token is a
secret. Never include the token in an error message or log request headers. A missing, blank or
malformed value is a startup `CONFIG_ERROR` naming the variable but never its value.

## OpenBao Local Runtime

### Server mode

Implementation includes the repository-owned Compose file, server configuration, policy file,
and operator runbook, which live in `./docs/en/howto/openbao`. Starting, initializing,
unsealing, and provisioning the running OpenBao instance are human operator tasks; an agent
must not execute those operations or store the resulting credentials.

`./docs/en/howto/openbao/README.md` is the single operator reference. Provisioning steps,
periodic-token issuance, unsealing, clearing the lifecycle document, policy notes, and the
mapping from each startup failure to its remedy belong there rather than in this plan.

Run OpenBao in normal server mode, not development mode. Configure durable file storage for the
first local implementation and mount its data directory through a named container volume.

Bind the local listener only where the auth service can reach it. Plain HTTP is acceptable only
on the isolated local container network. Production-like or remote environments must use TLS.

The operator runbook at `./docs/en/howto/openbao/README.md` must document these steps without
committing generated credentials:

1. Start the OpenBao container.
2. Initialize the server once.
3. Store the returned unseal keys and root token outside the repository.
4. Unseal the server after a container or host restart.
5. Enable the Transit secrets engine at the configured mount.
6. Enable KV v2 at the configured metadata mount.
7. Install the auth signing policy.
8. Create a service token using that policy. The service does not renew it, so issue it with a
   lifetime the operator manages out of band; see `docs/en/howto/openbao/README.md`.
9. Export that token through the configured environment variable.

Do not automate storage of the root token or unseal keys in repository files.

### Minimal policy permissions

Implementation includes writing the least-privilege policy file. Installing that policy and
creating the service token are human operator tasks.

> Superseded; see `## Current Status`. The broad set below belonged to the automated engine;
> the shipped `auth-signing-policy.hcl` is narrower.

The shipped runtime token requires permission only to:

- read the lifecycle KV document (`read` on `secret/data/auth/signing-lifecycle`); and
- sign with Transit keys under its prefix (`update` on `transit/sign/auth-signing-*`).

It has no `transit/keys/*` capability at all, including `read`, so a compromised auth service
cannot destroy key material or rewrite its own lifecycle state. Key creation, configuration,
deletion, and KV writes are operator capabilities exercised with an administrative token, not
the runtime token; see `docs/en/howto/openbao/README.md`.

Limit the policy to the exact Transit sign prefix and KV state path. The runtime token must not
have root or general secret-management permissions.

## `OpenBaoSigningKeyProvider` Behavior

### Transport boundary

Keep OpenBao HTTP handling private to `openbao/provider.ts`. Use one private request method
that adds the OpenBao token, applies a finite timeout, parses JSON as `unknown`, validates the
operation-specific response schema, and converts failures to sanitized `CustomError` values.
This method contains provider-specific behavior and is not a dead-weight wrapper.

The request method may return an operation-specific absent result only where absence is an
expected state, such as a missing lifecycle document or an already deleted Transit key. All
other non-success statuses are provider failures. Keep this distinction private to the
provider; do not add it to `ISigningKeyProvider`.

Do not create an OpenBao client interface or a general OpenBao SDK. Do not move the request
method to `./tools`. The existing `makeApiRequest()` response normalization is intended for
service-style response bodies and does not provide the strict redaction and response shapes
required here.

### Initialization

> Partly superseded; see `## Current Status`. The service does not create the first key or
> write initial state; an absent document fails startup, and first-key provisioning is an
> operator task in `docs/en/howto/openbao/README.md`. The load, validate, and publish steps
> remain current.

`initialize()` must complete before the API accepts traffic:

1. Read the KV v2 lifecycle document and its metadata version.
2. If the document is absent, create the first Transit key and write an initial state
   containing that key as active. Use KV v2 compare-and-set version `0` so the write succeeds
   only when the document still does not exist.
3. If another replica wins initial creation, apply the losing-key reconciliation rules from the
   key-creation section and reload the winning state.
4. Validate the complete lifecycle state and required provider material before publishing it.
5. Run lifecycle maintenance against the validated state.
6. Import the retained public JWKs and publish one complete in-process snapshot.
7. Return only when an active signer and the corresponding public verification key are ready.

The initial key does not need a pending interval because no token can be issued before
bootstrap finishes. Its public JWK must still be persisted in the same successful state write
before the key is selected for signing.

### Creating a key

> Superseded; see `## Current Status`. Key creation is an operator CLI procedure, not a service
> operation; see `docs/en/howto/openbao/README.md`.

Use one private key-creation operation for both initial creation and later rotation. It must:

1. Generate a random `kid` with `randomUUID()`.
2. Derive a safe Transit key name from `keyNamePrefix` and the generated identifier.
3. Ask OpenBao Transit to create a non-exportable `ecdsa-p256` key.
4. Enable deletion for lifecycle cleanup through a second call to the key's `/config` endpoint;
   `deletion_allowed` is not a create-key parameter.
5. Read the key and require `latest_version` to be `1`, then take version `1`'s PEM public key.
   A freshly created key has exactly one version, so anything else means the name was already
   in use and its published public half would not match what Transit signs with.
6. Convert the public key to a Web Crypto `CryptoKey` and then `PublicJwk` with `jose`.
7. Set `kid`, `use: 'sig'`, and `alg: 'ES256'` on the public JWK.

The operation returns the new `kid`, Transit key name, and public JWK to its caller; it does
not write lifecycle state itself. Initialization writes the first entry as active with
compare-and-set version `0`. Rotation writes a later entry as pending with the KV metadata
version read with the current state. This keeps lifecycle transitions in one state-update path
instead of duplicating them inside key generation.

OpenBao KV v2 arbitrates concurrent lifecycle updates through compare-and-set: a successful
write identifies the winning auth service replica, while an explicit CAS conflict identifies
a losing replica. After a conflict, the loser reloads the persisted state and destroys only
the Transit key created by its failed attempt if that key is not referenced by the winning
state.

A timeout or other ambiguous write response must not be treated as a CAS conflict. The
provider must reload the lifecycle state and destroy its generated Transit key only after
confirming that its `providerKeyId` is not referenced by the persisted state.

Capture one timestamp immediately before the initial compare-and-set write and use it for the
first key's `activateAfter` and `activatedAt`. A later rotation key has a null `activatedAt`
and remains pending until its `activateAfter`, calculated as the creation time plus
`jwks.cacheMaxAgeSecs`.

The entry carries no separate `publishedAt`. Publication is decided by `status` and
`verifyUntil`, activation by `activateAfter`, and the next rotation by `activatedAt`, so a
fourth timestamp would be written but never read.

### Signing

`getCurrentSigner()` must synchronize lifecycle state, run due maintenance, and capture the
active entry. The returned handle remains bound to that entry's `kid` and `providerKeyId`; it
must not look up the current key again inside `sign()`.

The captured `sign()` function must:

1. Standard-base64-encode the exact JWS signing input bytes for the OpenBao request.
2. Call the Transit sign endpoint for the captured entry's `providerKeyId`.
3. Request `sha2-256`, `prehashed: false`, and `marshaling_algorithm: 'jws'`.
4. Validate the `vault:v1:` response prefix. Transit key version `1` is required because this
   design creates a separate Transit key instead of rotating versions within a key.
5. Extract and base64url-decode the JWS-formatted signature.
6. Require the decoded ES256 signature to contain exactly 64 bytes.
7. Return the signature as `Uint8Array`.

OpenBao's `jws` marshaling mode already produces the ECDSA signature representation required
by JWT. The provider still removes the OpenBao response wrapper and returns backend-neutral
bytes. Do not call the Transit verification endpoint during normal JWT verification.

### Rotation and activation

> Superseded; see `## Current Status`. The service performs no rotation, activation,
> retirement, or compare-and-set writes. It keeps one private state-synchronization operation
> that only reads and republishes the snapshot; all transitions are operator actions in
> `docs/en/howto/openbao/README.md`.

Implement one private state-synchronization operation used by `initialize()`,
`getCurrentSigner()`, and `listPublicKeys()`. Each attempt loads state and its KV metadata
version, performs due transitions with compare-and-set, validates the final state, and replaces
the in-process snapshot. Do not call a separate reload after successful synchronization.

Share one in-flight synchronization promise across all calls within a provider instance and
clear it after success or failure. This prevents the same replica from performing duplicate
reads or competing lifecycle writes while KV compare-and-set remains the cross-replica guard.

Clear only the promise the finishing call joined. Every caller awaiting the shared promise runs
its own completion path, so a caller that clears the field unconditionally can discard a newer
synchronization another caller has already started, which reintroduces exactly the duplicate
work the shared promise exists to prevent.

For each loaded state, maintenance must evaluate transitions in this order:

1. If the pending entry has reached `activateAfter`, atomically make it active and make the
   previous active entry retired in the same state write. Use one captured activation time for
   `activatedAt`, `retiredAt`, and calculating `verifyUntil`.
2. If no pending entry exists and the current time has reached the active key's `activatedAt`
   plus `rotationIntervalSecs` minus `jwks.cacheMaxAgeSecs`, create and persist one pending
   key.
3. Make a persisted pending key immediately available to `listPublicKeys()` and the local
   verification cache.
4. Remove expired retired entries according to the destruction rules.

Validate each candidate state before its compare-and-set write. After a successful write, use
the written state and returned KV version as the new local values. On an explicit CAS conflict,
discard the candidate state, reload the winner, recompute due transitions, and retry a bounded
number of times. Apply the separate ambiguous-write reconciliation rule instead of treating a
timeout as a CAS conflict.

At most one lifecycle entry may have `active` status. Treat missing active state, multiple
active entries, an unknown algorithm, or missing provider material for a pending, active, or
unexpired retired key as a startup or signing-key configuration failure. An already missing
Transit key is allowed only for an expired retired entry awaiting state cleanup. Do not
silently generate replacement material for inconsistent existing state.

The deferred public `maintenance()` method refers only to future scheduler access. The internal
maintenance behavior in this section is not performed by the shipped service; it is the
operator procedure of record in `docs/en/howto/openbao/README.md` (see `## Current Status`).

### JWKS retention and destruction

> Partly superseded; see `## Current Status`. The retention rules for `listPublicKeys()` are
> current. The maintenance-driven Transit key destruction below is not performed by the
> service; an operator deletes retired keys per `docs/en/howto/openbao/README.md`.

`listPublicKeys()` must return:

- pending keys that have already been published;
- the active key; and
- retired keys whose `verifyUntil` timestamp is still in the future.

It must never return private JWK members. When a retired entry passes `verifyUntil`,
maintenance must first delete its Transit key and then remove it from the KV state with
compare-and-set. Treat an already missing Transit key as a successful deletion. This ordering
allows a later maintenance call to complete cleanup if the state update loses a race.

Destruction must never target an active or pending key.

### In-process verification cache

Treat the provider's in-process snapshot as a projection of durable lifecycle state, not as an
independent source of truth. It contains the state-load time, the publishable public JWK list,
and a map from `kid` to the imported public `CryptoKey`, status, and `verifyUntil`. Build it
with `jose.importJWK()` only after validating the complete state.

Build a complete replacement cache before publishing it to readers. If validation or any JWK
import fails, keep the previous valid cache and report the reload failure. Replacing the cache
as one snapshot prevents concurrent verification calls from observing a partially refreshed
key set. The private synchronization operation refreshes the snapshot after initialization,
every successful local lifecycle change, and every durable reload.

`getVerificationKey(kid)` must:

1. Look up the exact `kid`; it must never fall back to the active key.
2. Return a pending or active key, or a retired key whose `verifyUntil` is still in the future.
3. Return `null` for a retired key after `verifyUntil`, even when durable cleanup has not run.
4. On a missing `kid`, return `null` without reloading when the snapshot is younger than
   `jwks.cacheMaxAgeSecs`.
5. When the snapshot is older, run state synchronization once, retry the exact lookup, and
   return the key or `null`.

The bounded reload on a cache miss allows one auth service replica to discover a key created by
another without letting arbitrary unknown `kid` values cause an OpenBao request per bearer
token. Concurrent misses share one in-flight synchronization, including misses for different
key identifiers. If synchronization fails, reject with `EXTERNAL_API_ERROR`; do not return
`null` and misrepresent an OpenBao outage as an unknown key.

`CryptoKey` is the type exported by `jose` for its runtime Web Crypto key representation:

```ts
import type { CryptoKey } from 'jose';
```

It models a Web Crypto key and is not an OpenBao-specific key type. The cache contains public
verification material only. Do not cache private material, Transit signing responses, bearer
tokens, or verification results.

## Final `TokensService` Shape

Rename `TokenService` to `TokensService` because the class issues both access and refresh
tokens. Keep the interface name `ITokenService` as requested; do not introduce
`ITokensService`.

```ts
export class TokensService implements ITokenService {
  constructor(
    private readonly config: AuthConfig,
    private readonly signingKeys: ISigningKeyProvider
  ) {}

  verify(credential: BearerToken): Promise<SessionGrant>;

  issueAccessToken(candidate: SessionCandidate, sessionId: string): Promise<AccessToken>;

  issueRefreshToken(): Promise<RefreshToken>;
}
```

Keep the existing class, constructor, and public-method JSDoc shape when renaming the file and
class. Update descriptions that still refer to a local JOSE private-key provider. Follow the
consolidated type-level field documentation rule from `AGENTS.md` for every changed type.

### Access-token issuance

`issueAccessToken()` must no longer use `SignJWT.sign(privateKey)`. Before returning a signer,
`getCurrentSigner()` performs lifecycle maintenance and selects an active key whose public JWK
has already been persisted and made available to `listPublicKeys()`. `TokensService` therefore
never handles private key material or activates a key itself.

The `JWTPayload` mentioned here is the type exported directly by `jose`:

```ts
import type { JWTPayload } from 'jose';
```

Do not add a local wrapper or intersection for custom claims. `JWTPayload` permits additional
properties through its index signature. Verification must still narrow `sid`, `actorId`, and
`tenantId` at runtime because their values are typed as `unknown` when the JWT is decoded.

`issueAccessToken()` must:

1. Obtain a `SigningKeyHandle` from `signingKeys.getCurrentSigner()`.
2. Construct the protected header with the handle's `algorithm`, the handle's `kid`, and
   `typ: 'JWT'`. Do not independently select either `alg` or `kid` from configuration.
3. Calculate `iat` and `exp` as integer NumericDate values in seconds. Derive `exp` from `iat`
   plus the configured access-token lifetime.
4. Construct a value typed as `JWTPayload` with `sid`, `actorId`, `tenantId`, `sub`, `iss`,
   `aud`, `iat`, and `exp`.
5. Use the shared Node crypto helper to JSON-serialize and base64url-encode the protected
   header and payload, then join them with `.` as the exact JWS signing input.
6. UTF-8 encode that signing input and pass the resulting bytes to `handle.sign()`.
7. Base64url-encode the returned ES256 JWS signature bytes without changing their format.
8. Append the encoded signature to the signing input and return the compact token and
   configured lifetime as `AccessToken`.

The handle remains bound to the selected key for the entire operation. If rotation occurs
after the handle is returned, it continues signing with the key identified by the protected
header. The provider retains that key's public material through `verifyUntil`, so the issued
token remains verifiable during the overlap period.

Issuance does not read or update the verification cache directly. A local lifecycle change
refreshes that cache inside the provider, while another replica reloads durable state if it
later encounters the new `kid`. A signing or provider failure must abort issuance; the service
must never return an unsigned or partially assembled token.

Use `jose` base64url utilities where practical inside the shared helper. Keep the helper scoped
to producing the signing input; do not add a new JWS service or interface.

### Access-token verification

`verify()` continues using `jose.jwtVerify()` and the `JWTPayload` type imported directly from
`jose`. `JWTPayload` is already the default payload type for `jwtVerify()`, so the explicit
`jwtVerify<JWTPayload>()` generic is unnecessary. The protected header is untrusted until
signature verification succeeds, so decoding it is only a key-selection step.

`verify()` must:

1. Accept a `BearerToken` with a defined `token` selected by the route transport boundary.
2. Decode the protected header and require non-empty `kid`, `alg: 'ES256'`, and `typ: 'JWT'`
   values. Do not use any other header field to select provider behavior.
3. Call `signingKeys.getVerificationKey(kid)` and fail authentication if it returns `null`.
4. Pass the public `CryptoKey` to `jose.jwtVerify()` with the explicit
   `algorithms: ['ES256']` allowlist and the configured issuer, audience, and clock tolerance.
5. Require `sid`, `sub`, `actorId`, and `tenantId` to be non-empty strings after verification.
6. Build and return `SessionGrant` only from the verified and narrowed payload.

Missing credentials are rejected before `verify()` is called. Malformed tokens, unknown keys,
disallowed algorithms, invalid signatures, invalid registered claims, and incomplete custom
claims must all produce the same existing `AUTHENTICATION_FAILED` response. Do not reveal which
check failed.

A durable-state reload failure while resolving an unknown `kid` is different: preserve the
existing infrastructure-facing `EXTERNAL_API_ERROR` so the API returns `502`, rather than
misrepresenting an OpenBao outage as invalid client credentials. `TokensService` must not wrap
that error in `AUTHENTICATION_FAILED`. Logs may retain a sanitized internal cause but must not
contain the bearer token.

No new error class is required. `getVerificationKey()` returns `null` when the `kid` remains
unknown and throws `EXTERNAL_API_ERROR` when OpenBao prevents synchronization. `TokensService`
maps `null` and JOSE parsing or verification failures to `AUTHENTICATION_FAILED`, while
preserving both `EXTERNAL_API_ERROR` and `CONFIG_ERROR`.

The provider raises only these two names. `verify()` must name both in its pass-through check;
a check written against any other `ErrorName` silently reports every provider failure as a
`401` on a valid credential, because the generic branch below it wraps whatever it does not
recognize.

The provider supplies public keys only. A cache hit performs no OpenBao request. An unknown-key
lookup follows the age-gated reload behavior defined by the verification-cache section.

### Refresh-token issuance

`issueRefreshToken()` remains independent of the signing-key provider. It must generate an
opaque credential with Node's cryptographically secure randomness and return it only to
`SessionService`. The token carries no claims, key identifier, or expiration timestamp;
`SessionRecord` remains the source of truth for its hash, lifetime, rotation, and revocation.

Do not pass a refresh token to `ISigningKeyProvider`, encode it as a JWT, persist its plaintext
value, or log it. Keep the existing asynchronous `ITokenService` method signature even though
the initial random generation is synchronous.

### Final `ITokenService` contract

The public contract keeps its current three methods and types:

```ts
/** Token verification and issuance boundary. */
export interface ITokenService {
  /**
   * Verifies one bearer access credential.
   * @param credential - Bearer credential selected by the route transport policy.
   * @returns Trusted session grant reconstructed from the credential.
   */
  verify(credential: BearerToken): Promise<SessionGrant>;

  /**
   * Issues a short-lived access credential for an active session.
   * @param candidate - Trusted identity bound to the session.
   * @param sessionId - Identifier embedded in the issued access credential.
   * @returns Newly issued access credential and lifetime.
   */
  issueAccessToken(candidate: SessionCandidate, sessionId: string): Promise<AccessToken>;

  /**
   * Issues an opaque refresh credential for one session.
   * @returns Newly issued refresh credential.
   */
  issueRefreshToken(): Promise<RefreshToken>;
}
```

Do not add provider initialization, JWKS, rotation, or key-management methods to
`ITokenService`. Those operations remain behind `ISigningKeyProvider` and are not application
token operations.

## API and Bootstrap Changes

### Discovery API

Change `ApiServices.signingKeys` to `ISigningKeyProvider`. The JWKS controller continues
calling `listPublicKeys()` and sends the result as `{ keys }` under the route's declared
`jwksDocumentSchema`. Before
returning keys, `listPublicKeys()` runs the provider's single state-synchronization operation
and reads the public JWK list from its resulting snapshot. Do not reload the same state again.
This ensures that a discovery request handled by a replica which did not perform the rotation
still publishes the pending key.

Return only pending, active, and unexpired retired public JWKs according to the retention
rules. Keep their persisted `kid`, `alg`, and `use` members, and never include lifecycle
metadata or a private or symmetric JWK member. An OpenBao read or validation failure must
preserve the last valid in-process snapshot but fail the current discovery request with the
existing infrastructure-facing error; it must not serve stale state as though the refresh
succeeded.

Add this response header using `config.signingKeys.jwks.cacheMaxAgeSecs`:

```http
Cache-Control: public, max-age=<cacheMaxAgeSecs>
```

Return `200` with the existing JWKS response schema. Do not add a version prefix to
`/.well-known/jwks.json`, redirect the endpoint, or expose provider lifecycle state, OpenBao
paths, or provider key identifiers. The configured pending-key publication interval must remain
long enough for previously returned JWKS documents to reach their advertised maximum age before
the pending key becomes active.

### Startup preflight

> Superseded; see `## Current Status`. No preflight, seal-status probe, token lookup, or
> background token renewal shipped; bootstrap calls only `initialize()`. This describes a
> design that was not built. Startup errors surface from the first KV read, classified by the
> transport boundary.

Bootstrap must verify the instance before any lifecycle work, so an operator problem produces a
message naming its cause rather than an opaque failure on the first KV read. Two endpoints make
this possible because they answer where the normal paths do not:

- `sys/seal-status` is unauthenticated, so it responds while the instance is sealed and while
  the token is expired. It separates "OpenBao is down or sealed" from "the token is wrong".
- `auth/token/lookup-self` needs a valid token but no policy grant, so success here followed by
  a later `403` means the policy is missing a capability rather than the token being invalid.

Preflight must reject an uninitialized instance, a sealed instance, and a rejected token, each
with its own message. It must also report the token's remaining lifetime, warning when the
token is not renewable, or is renewable without a period and so still capped by its maximum
time-to-live. Bootstrap runs before the Fastify logger exists, so that warning goes to the
console; it is a standing operational condition, not a per-request event.

The provider then renews the token in the background for the process lifetime. The timer
re-arms at half the granted lifetime, is bounded at both ends, and is unreferenced so it
never holds the process open. It retries on failure because the usual cause is a transient
outage. Renewal cannot rescue a non-periodic token past its maximum time-to-live.

Every transport failure carries the operation label and requested path in its cause, and the
status is classified before it reaches the message: `503` distinguishes a sealed instance from
an inactive node, `403` names token rejection against a specific path, `429` names a rate-limit
quota, and the two `400` cases the lifecycle depends on are a lost compare-and-set race and a
Transit key that was never made deletable. Response bodies may name mounts and policies, so
only the status category and the requested path ever escape into an error.

### Composition root

Update `bootstrap.ts` in this order:

1. Read the OpenBao token from the environment variable named by the validated provider config.
   Reject a missing or blank value without including it in the error.
2. Construct one `OpenBaoSigningKeyProvider` from the validated signing-key configuration and
   token.
3. Await `signingKeys.initialize()` before constructing the API or accepting traffic.
4. Construct `TokensService` with `config.auth` and the initialized
   `ISigningKeyProvider` instance.
5. Inject that same provider instance into `TokensService` and `ApiServices.signingKeys` so
   signing, verification, and discovery share one coherent local snapshot.
6. Decorate the Fastify instance with the validated service config so discovery can set the
   configured JWKS cache header without adding signing policy to `ApiServices`.
7. Preserve the existing session, password, OAuth, user, and route composition.

If OpenBao is unavailable, sealed, unauthorized, or inconsistent during startup, bootstrap must
fail. Do not fall back to an ephemeral key because that would silently invalidate existing
tokens. The existing bootstrap failure path must still close the database connection before
propagating the startup error.

## Session Consistency During Signing Failures

Remote signing introduces an expected infrastructure failure point. Keep the adjustment
minimal and do not introduce a distributed transaction abstraction.

During session creation, `SessionService.create()` must:

1. Generate the refresh token and persist only its hash in the new active `SessionRecord`.
2. Request the access token using the persisted session identifier.
3. Return the session and both credentials only after signing succeeds.
4. If signing fails, delete the record as a compensating operation and return no credential.

If compensating deletion also fails, propagate the signing failure and retain the sanitized
cleanup failure in its causal context so the existing server error path can log both failures.
Do not add a logger dependency or a new error subclass to `SessionService`; use `CustomError`
and the applicable `ErrorName`. Do not convert either failure to an authentication error. The
stored record is not usable by a client because its plaintext refresh token and access token
were never returned, but it becomes an orphan that requires later operational cleanup. Do not
claim that compensation provides database-level atomicity.

During refresh, `SessionService.refresh()` must:

1. Validate the presented refresh token and load the active record.
2. Generate the replacement refresh token without persisting it yet.
3. Issue a new access token for the current session identifier.
4. Compare-and-set the stored refresh-token hash to the replacement hash.
5. Return the new credentials only when that state update succeeds.

If signing fails, leave the existing refresh hash unchanged and propagate the infrastructure
failure. If compare-and-set loses a concurrent rotation race, discard the unreturned access and
refresh tokens and return the existing generic authentication failure. Never return a refresh
token whose hash was not committed.

## Error Handling and Security

Use `CustomError` for classified application and infrastructure failures. Reuse an existing
`ErrorName` whenever its semantics match. If no existing entry accurately represents a required
failure, add one narrowly scoped literal to `ErrorName` and update the Fastify error mapping
when that failure requires a distinct HTTP response. Do not create specialized `Error`
subclasses.

Keep client-visible failures separated by responsibility:

- Map missing or invalid credentials, unknown `kid` values, invalid signatures, rejected JWT
  claims, and refresh-token races to the existing generic `AUTHENTICATION_FAILED` response.
- Wrap OpenBao timeouts, network failures, authorization failures, sealed state, and malformed
  responses in the existing infrastructure-facing `EXTERNAL_API_ERROR`. The API maps it to
  `502` without exposing the provider response body.
- Represent invalid durable lifecycle invariants with the existing `CONFIG_ERROR`. Startup must
  fail, runtime calls map it through the existing `500` path, and neither path may silently
  create replacement keys.
- Do not introduce a provider-specific `ErrorName`. `EXTERNAL_API_ERROR` and `CONFIG_ERROR`
  already carry the two classifications this provider needs, and the API error plugin maps them
  explicitly to `502` and `500`. A `KEYS_PROVIDER_ERROR` literal previously existed in
  `tools/src/shared/errors.ts` with no mapping and no thrower; it was removed, because anything
  raised under it would have reached clients as a generic `500`.
- Propagate signing infrastructure failures through `SessionService`; do not convert them to
  `401` or `403` responses.

Apply these security rules throughout the provider and token service:

- Validate every OpenBao response before reading fields from it.
- Set a finite timeout on every OpenBao request.
- Never log bearer tokens, refresh tokens, OpenBao tokens, JWS signing inputs, signatures,
  private key material, authorization headers, or raw provider response bodies.
- Log only sanitized operation names, provider status categories, key identifiers when safe,
  request correlation data, and error causes that contain no secret response data.
- Permit bounded automatic retries for safe reads and explicit KV compare-and-set conflicts.
- Do not retry key creation, state mutation, deletion, or signing after an ambiguous response
  unless the operation-specific reconciliation rules first establish whether it completed.

Update the Fastify error plugin to log sanitized `500` and `502` failures at error level. Keep
the current warning behavior for `401` and `403`. This ensures runtime provider failures and
failed session compensation are observable while the RFC 9457 response remains generic.

A repeated ES256 signing request may produce a different valid signature and does not itself
corrupt persisted state. Automatic signing retries are nevertheless deferred in this phase so
timeout behavior and provider load remain explicit rather than hidden inside the provider.

## Implementation Sequence

1. Move signing-key and JWKS config from application auth config to infrastructure key config,
   then compose and validate both config sections at the service root.
2. Add `SigningKeyHandle`, shared lifecycle types, and validated private OpenBao transport
   types.
3. Replace both current signing-key contracts with `ISigningKeyProvider` and update dependent
   type imports.
4. Implement the private OpenBao transport boundary and lifecycle document validation.
5. Implement initialization, first-key creation, CAS conflict reconciliation, and public-key
   conversion.
6. Implement the shared state-synchronization operation, pending-key publication, activation,
   retirement, and expired-key cleanup.
7. Implement current-key selection and remote ES256 JWS signing through a captured
   `SigningKeyHandle`.
8. Implement atomic snapshot replacement and age-gated verification-key cache misses.
9. Rename `TokenService` to `TokensService`, assemble compact JWS access tokens through the
   signing handle, and enforce verification claim narrowing and the ES256 allowlist.
10. Apply session-creation compensation and move the refresh-token compare-and-set after
    successful access-token signing.
11. Preserve infrastructure failures through token and session services while keeping invalid
    credentials behind the generic authentication failure.
12. Update global config composition, bootstrap initialization, API service types, error
    logging, JWKS state synchronization, cache headers, imports, and exports.
13. Add the repository-owned local OpenBao container configuration, policy, and operator
    README. Initialization, unsealing, and credential handling remain human operator tasks.
14. Run formatting and static type checking for `services/auth` without building or running
    tests.

## Completion Criteria

> These were acceptance criteria for the original automated engine; see `## Current Status`.
> The criteria covering key creation, rotation, activation, retirement, and Transit destruction
> are now operator responsibilities verified against `docs/en/howto/openbao/README.md`, not
> service behavior. The read, sign, verification, JWKS, and failure-handling criteria remain
> current.

- Restarting the auth process reloads durable state and preserves verification of access tokens
  whose signing keys remain inside their `verifyUntil` window.
- Multiple auth replicas converge on the same active key through compare-and-set state updates.
- A discovery request synchronizes durable state, so replicas publish the same retained JWKS
  key set.
- The auth process never receives or persists a private signing key.
- Initial startup creates one key only when durable signing state does not exist.
- A rotation key appears in JWKS for at least the configured publication interval before it
  signs tokens.
- Activation retires exactly one prior active key through an atomic state update.
- Retired public keys remain published through their configured verification overlap.
- Expired retired keys disappear from JWKS and their Transit key material is destroyed.
- Verification uses only ES256, exact `kid` lookup, registered-claim validation, and runtime
  narrowing of required custom claims.
- An unknown-key lookup reloads durable state at most once after the snapshot reaches its cache
  age; ordinary cache hits require no OpenBao request.
- `TokensService` receives only `ISigningKeyProvider` and contains no OpenBao-specific code.
- `ITokenService` retains only token verification and issuance operations.
- OpenBao failure during bootstrap does not cause an ephemeral fallback.
- Runtime OpenBao failures return the existing infrastructure error and are not reported as
  invalid credentials.
- Session creation returns no credentials and compensates its new record when signing fails.
- Refresh-token state changes only after access-token signing and before either new credential
  is returned.
- `/.well-known/jwks.json` returns only retained public keys and the configured cache header.
- OpenBao credentials and cryptographic material are absent from API responses and logs.

## Operator-Owned Lifecycle

Implemented, AUTH-023. Everything the provider did beyond the four `ISigningKeyProvider`
operations was a mutation, and every mutation is now an operator action. Removing them removed
the write path with them: compare-and-set retries, ambiguous-write reconciliation,
losing-candidate cleanup and the conflict classification existed only to make concurrent writes
safe, and with a single writer none of it is needed.

What remains is one KV read, one Transit sign, strict validation of the document that read
returns, and the in-process snapshot those serve. Login, JWKS publication and verification were
untouched, because they already used only those two calls.

This supersedes several earlier sections of this plan. Key creation, rotation and activation,
destruction, the compare-and-set write protocol and the startup preflight describe behaviour
the service no longer performs; they now describe what an operator does, and
`docs/en/howto/openbao/README.md` is the procedure of record. The provider configuration keeps
only `algorithm`, `jwks.cacheMaxAgeSecs` and the provider block, since
`rotationIntervalSecs` and `verificationOverlapSecs` had no reader once the lifecycle engine
was gone.

Two consequences followed:

- The policy narrowed to `read` on the state path and `update` on `transit/sign/*`, dropping
  `transit/keys/*` entirely. A compromised auth service can neither destroy key material nor
  rewrite its own lifecycle state.
- Rotation stopped happening rather than moving. Scheduled rotation was considered and rejected
  (AUTH-015): rotation is an OpenBao CLI procedure, so nothing in the service schedules it.
  Validation matters more than before, not less, because the document it guards is now written
  by hand.

## Deferred Work

- A scheduled maintenance worker for traffic-independent rotation.
- Production OpenBao high availability, automatic unseal, TLS, backup, and disaster recovery.
- Metrics, alerts, and administrative key-rotation endpoints.
- Emergency compromise workflows that invalidate a key before normal `verifyUntil`.
- Reconciliation of orphaned session records when compensating deletion fails.
- An explicit bounded retry policy for remote signing requests.

## Backend-Neutral Provider Selection

Deferred. The API layer already depends only on `ISigningKeyProvider`, and after the
operator-owned reduction (see `## Current Status`) the provider exposes only the four contract
operations; the token-renewal and preflight methods this section once tracked no longer exist,
so bootstrap calls only `initialize()` and shutdown never touches the provider. What remains is
a small leak in the composition root: `bootstrapSigningKeys` returns the concrete
`OpenBaoSigningKeyProvider`, and `runtime/types.ts` types the composed service's `signingKeys`
field as that class. Closing that gap is what lets a second provider arrive without touching
bootstrap.

One change is pure de-leaking and does not need a second provider to exist:

- Return and store the interface. Have `bootstrapSigningKeys` return `ISigningKeyProvider` and
  `runtime/types.ts` type the composed service as the interface, so nothing downstream of
  construction names the concrete class.

The remaining changes are gated by the rule above, which forbids a factory or a configuration
union before a second provider implementation exists:

- Widen `signingKeys.provider` from the OpenBao schema to a `type` discriminated union.
- Add a provider factory that switches on `provider.type`, with each branch calling its own
  module's environment reader, so the composition root constructs nothing concrete and learns
  no provider's environment variable names.

Only then does construction select a provider by configuration rather than naming
`OpenBaoSigningKeyProvider` directly.

## Future Signing Key Providers

Future providers should be designed when they are implemented, using the
`ISigningKeyProvider` contract rather than adding speculative types or dependencies now.

- `HsVaultSigningKeyProvider`: use HashiCorp Vault Transit for private-key custody and remote
  signing. The provider should keep Vault transport details inside
  `infrastructure/keys/hsvault/` and return backend-neutral signing bytes through
  `SigningKeyHandle.sign()`.
- `AwsKmsSigningKeyProvider`: use AWS KMS asymmetric signing keys for private-key custody and
  remote signing. The provider can keep AWS-specific metadata, aliases, and key identifiers
  inside `infrastructure/keys/awskms/`.
