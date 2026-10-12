---
title: Jose Signing Key Provider and Neutral Bootstrap Implementation Plan
lastUpdate: 2026-10-01
lang: en
category: architecture
status: published
owner: Odair Trujillo
---

# Jose Signing Key Provider and Neutral Bootstrap Implementation Plan

## Goal

The deliverable is `JoseSigningKeyProvider`, a Node.js-native signing-key provider for local
development. It generates its own ES256 key, stores the private material in an encrypted
file keyed from the environment, and satisfies the existing `ISigningKeyProvider` contract,
so a developer can bootstrap the service with minimum safety and no external key manager.

Introducing a second provider is only clean if the composition root already depends on the
`ISigningKeyProvider` abstraction rather than on a concrete class. Today it does not; the
bootstrap step and a few runtime types name `OpenBaoSigningKeyProvider` directly. The
neutral bootstrap is therefore a consequence of adding Jose: the changes to
`src/runtime/keys.ts` and the new selection factory exist only to let a provider plug in by
configuration alone, without touching token issuance, verification, or the JWKS route.

## Guiding principles

- Dependency inversion; the composition root and `TokensService` depend on the
  `ISigningKeyProvider` abstraction, and each concrete provider owns its own environment,
  configuration, and storage. See `docs/en/architecture/` for the SOLID baseline.
- Open for extension, closed for modification; a new provider is a new module plus one
  `case` in the selection factory, with no change to the bootstrap step or the services that
  consume the provider.
- The contract carries the lifecycle; `initialize()` is documented as "loads durable state
  and ensures that an active key exists," which each provider satisfies differently. OpenBao
  validates operator-provisioned keys and fails when the document is absent; Jose treats an
  absent key file as first run and generates one. The bootstrap step never branches on that
  difference.

## Scope and non-goals

In scope:

- `JoseSigningKeyProvider` with encrypted-file storage and first-run key generation.
- A provider-selection factory and a neutral bootstrap step that let it plug in by name.
- A configuration discriminated union that chooses the provider by name.
- Promotion of the neutral signing-key schemas out of the OpenBao module.

Out of scope:

- Key rotation for Jose; the provider generates exactly one active key and never rotates.
  The lifecycle types still fit, with the key array holding one entry. Rotation remains the
  OpenBao provider's responsibility and the production path.
- Any change to token issuance, verification, or the JWKS route; these already depend on the
  interface and stay untouched.
- Tests; none are added or modified by this plan.

## JoseSigningKeyProvider

The provider lives in `infra/keys/jose/` with `provider.ts`, `schema.ts`, `types.ts`,
`keyfile.ts` (the file and crypto concern), and `index.ts`. There is no database storage
layer and no connection, so the `ISigningKeyProvider` contract keeps its original four
methods and needs no teardown hook.

### Configuration

`joseSigningKeysConfigSchema` lives in `infra/keys/jose/schema.ts` and is self-contained, so
the provider directory owns its whole storage story without the factory ever handling a file
handle or a secret:

```ts
export const joseSigningKeysConfigSchema = z
  .object({
    name: z.literal('jose'),
    keyNamePrefix: z.string().min(1),
    storage: z
      .object({
        filePath: z.string().min(1),
        secretEnv: z.string().min(1)
      })
      .strict()
  })
  .strict();
```

- `keyNamePrefix`; prefix for the generated `kid` and provider key identifier.
- `storage.filePath`; path to the encrypted key file, which must be listed in `.gitignore`.
- `storage.secretEnv`; name of the environment variable holding the file passphrase, read by
  the provider itself. This mirrors the established pattern where `bootstrapServices` reads
  the refresh-token hash secret from the variable named by `refreshToken.hashSecretEnv`
  rather than from the configuration object.

### Construction and lifecycle

The constructor is cheap and cannot throw; it stores configuration only. All input and
output happens in `initialize()`, the interface's existing async hook, via a private helper.
A separate public `initStorage()` is intentionally avoided, keeping one entry point and
matching the OpenBao provider's shape:

```ts
export class JoseSigningKeyProvider implements ISigningKeyProvider {
  // Built once in initialize(); no rotation means no refresh machinery.
  private snapshot: SigningKeySnapshot | null = null;

  constructor(
    private readonly config: JoseSigningKeysConfig,
    private readonly algorithm: 'ES256',
    private readonly cacheMaxAgeSecs: number
  ) {}
}
```

The in-process snapshot mirrors the OpenBao projection but also retains the active private
key for signing:

```ts
type SigningKeySnapshot = Readonly<{
  state: SigningKeyState;
  keys: readonly PublicJwk[];
  byKid: ReadonlyMap<string, CryptoKey>;
  activeSigner: Readonly<{ kid: string; privateKey: CryptoKey }>;
}>;
```

`SigningKeySnapshot` is a provider-internal type in `jose/types.ts`, named to match the
OpenBao provider's equivalent. The two are different shapes in separate module barrels, so
the shared name does not collide, provided `infra/keys/index.ts` exposes only the neutral
surface (the factory, configuration, and promoted schema) rather than star-exporting both
provider barrels; otherwise the two identically named types would meet in one scope.

### Encrypted file storage

The file concern lives in `infra/keys/jose/keyfile.ts` and reuses the `jose` library's
password-based JWE, which bakes in the salt and key derivation, so no AES or scrypt plumbing
is written by hand. The whole payload is encrypted, so the file leaks nothing, not even
which key identifiers exist:

```ts
// payload = { state: SigningKeyState, privateJwks: Record<kid, PrivateJwk> }
const jwe: string = await new CompactEncrypt(encoder.encode(JSON.stringify(payload)))
  .setProtectedHeader({ alg: 'PBES2-HS256+A128KW', enc: 'A256GCM' })
  .encrypt(encoder.encode(secret));

// Exclusive create with owner-only permissions.
await writeFile(filePath, jwe, { mode: 0o600, flag: 'wx' });

const { plaintext } = await compactDecrypt(
  await readFile(filePath, 'utf8'),
  encoder.encode(secret)
);
const payload: unknown = JSON.parse(decoder.decode(plaintext));
```

The `wx` flag plus `0o600` gives atomic, exclusive creation with owner-only permissions. It
also arbitrates two processes racing on one host; one wins the create, and the other catches
`EEXIST` and reads the existing file instead of generating a second key.

### initialize(): load or generate

```
secret = process.env[config.storage.secretEnv]?.trim()
  → CONFIG_ERROR when missing or blank   (mirrors the refresh hash-secret check)
payload = await readKeyFile(filePath, secret)         // null on ENOENT
if (!payload):
  payload = await createKeyFile(filePath, secret)     // generate, encrypt, wx write;
                                                       // on EEXIST, re-read
publishSnapshot(payload)                              // validate, import public and private
```

`createKeyFile` is the generate-on-empty step:

```
const { publicKey, privateKey } = await generateKeyPair('ES256', { extractable: true });
const kid = `${config.keyNamePrefix}-${randomUUID()}`;
const publicJwk  = { ...(await exportJWK(publicKey)), kid, alg: 'ES256', use: 'sig' };
const privateJwk = await exportJWK(privateKey);        // carries `d`
const entry: SigningKeyEntry = {
  kid, providerKeyId: kid, status: 'active', publicJwk,
  activateAfter: now, activatedAt: now, retiredAt: null, verifyUntil: null
};
const state: SigningKeyState = { currentKid: kid, keys: [entry] };
await writeKeyFile(filePath, secret, { state, privateJwks: { [kid]: privateJwk } });
```

`publishSnapshot` parses `state` against the neutral `signingKeyStateSchema`, enforces the
same public-JWK invariants the OpenBao provider enforces through a shared neutral validator,
imports each public JWK into `byKid`, and the active private JWK into `activeSigner`.
It assigns `this.snapshot` once, so readers always see a complete projection.

### Development key reset

There is no reset operation; removing the key file and restarting is the reset. An absent
file re-enters the generate-on-empty path in `initialize()` and mints a fresh key, so the
`ENOENT` branch doubles as the reset mechanism and needs no extra code.

Two behaviors matter in practice:

- The file is read once in `initialize()`, so deleting it while the service runs changes
  nothing until the next startup; the running provider keeps serving the key already in its
  snapshot. The developer loop is stop, delete the file, start. The planned `flag: 'wx'`
  write means the file must be deleted, not truncated, for the next create to succeed.
- It is a hard reset, not a rotation. The previous `kid` leaves JWKS at once, so every
  issued access token stops verifying and sessions must re-authenticate. This is acceptable
  in development and is the opposite of the `activateAfter`/`verifyUntil` overlap that makes
  true rotation seamless; a grace window for the old key would need the rotation machinery,
  not deletion.

A decrypt failure is never treated as an empty file. A wrong or changed passphrase makes
`compactDecrypt` throw, and that fails loudly rather than regenerating, so a typo cannot
silently discard the keys.

When the provider is implemented, this behavior is documented on `initialize()` as a TSDoc
comment, so the reset path is discoverable from the source and the generated API site:

```ts
/**
 * Loads the encrypted key file and publishes the first snapshot, generating a key on first
 * run when the file is absent.
 * @returns Completed provider initialization.
 * @throws {CustomError} CONFIG_ERROR when the passphrase environment variable is unset or
 * blank.
 * @throws {CustomError} KEYS_PROVIDER_ERROR when the file cannot be decrypted or violates a
 * signing-key invariant.
 * @remarks
 * Development key reset: delete the key file and restart. An absent file re-enters the
 * generate-on-empty path and mints a fresh key, so there is no separate reset operation.
 * The file is read once here, so deleting it while the process runs takes effect only on
 * the next initialize(); the running provider keeps serving the key already in memory. This
 * is a hard reset, not a rotation: the previous kid leaves JWKS at once and every issued
 * access token stops verifying. A decrypt failure fails loudly and is never treated as an
 * empty file, so a wrong passphrase cannot discard the keys.
 * @category Signing Keys
 */
async initialize(): Promise<void> {
  // ...
}
```

### Interface methods

```ts
async getCurrentSigner(): Promise<SigningKeyHandle> {
  const { activeSigner } = this.currentSnapshot();

  return {
    kid: activeSigner.kid,
    algorithm: this.algorithm,
    // Web Crypto ECDSA P-256 with SHA-256 returns the raw 64-byte r||s pair, identical to
    // the OpenBao Transit path and to what the JWS signing input expects.
    sign: async (input: Uint8Array): Promise<Uint8Array> =>
      new Uint8Array(
        await crypto.subtle.sign(
          { name: 'ECDSA', hash: 'SHA-256' },
          activeSigner.privateKey,
          input
        )
      )
  };
}

async getVerificationKey(kid: string): Promise<CryptoKey | null> {
  return this.currentSnapshot().byKid.get(kid) ?? null;
}

async listPublicKeys(): Promise<readonly PublicJwk[]> {
  return this.currentSnapshot().keys;
}
```

`getCurrentSigner` returns the same `SigningKeyHandle` shape as OpenBao, so `TokensService`
needs no change. There is no per-call synchronize, because the key never rotates; this is a
deliberate simplification over OpenBao, not an omission.

### Security posture and limitations

- Minimum safety by design; the private key is encrypted at rest with a passphrase held in
  the environment, not on disk. A leaked or accidentally committed key file is ciphertext to
  anyone without the passphrase.
- The passphrase and the file leak together if they share a host with no secret segregation;
  in that threat model the right answer is the OpenBao provider, where the private key never
  enters the process at all.
- Single host; a file is not shared between machines, so separate hosts would each
  generate a key and publish a divergent JWKS. Multi-replica key sharing is the OpenBao
  provider's role. The loud startup warning frames the Jose provider as development-only,
  so this is a stated constraint rather than a defect.

## Composition-root consequences

Adding the provider forces the composition root to depend on the abstraction. Because the
design inverts that dependency, the required changes are small and localized; none of them
touch token issuance, verification, or the JWKS route.

### Provider selection by name

`signingKeysConfigSchema.provider` becomes a discriminated union on a `name` field, so the
provider is chosen by name in configuration:

```ts
provider: z.discriminatedUnion('name', [
  openBaoSigningKeysConfigSchema,
  joseSigningKeysConfigSchema
]);
```

The existing OpenBao block renames its discriminator field from `type` to `name`
(`z.literal('openbao')`), and `config/local.json` follows by renaming `provider.type` to
`provider.name`. No OpenBao runtime code reads the discriminator, so the rename is confined
to the schema and the configuration file.

### Neutral schema promotion

`signingKeyStateSchema` and `publicJwkSchema` currently live in
`infra/keys/openbao/schema.ts`, but they describe the neutral lifecycle document rather than
an OpenBao wire shape. They move to a new `infra/keys/schema.ts`. The OpenBao module keeps
only its own response and environment schemas and imports the neutral state schema from the
parent. Jose and its file storage then validate against the same contract.

### The provider factory

`infra/keys/factory.ts` performs selection and construction, and is barrelled by a new
`infra/keys/index.ts`. Construction is synchronous and side-effect-free; all input and
output is deferred to `initialize()`. The `switch` is the single extension point for future
providers:

```ts
export function createSigningKeyProvider(config: SigningKeysConfig): ISigningKeyProvider {
  switch (config.provider.name) {
    case 'openbao': {
      const env: OpenBaoEnvironment = readOpenBaoEnvironment();

      return new OpenBaoSigningKeyProvider(
        config.provider,
        config.algorithm,
        config.jwks.cacheMaxAgeSecs,
        env.BAO_ADDR,
        env.BAO_TOKEN
      );
    }

    case 'jose': {
      // Loud warning at selection time, the earliest and clearest point. Bootstrap has no
      // Fastify logger yet, so this surfaces through process warnings / stderr.
      process.emitWarning(
        'JoseSigningKeyProvider is a development provider; private keys live in a local ' +
          'encrypted file on a single host. Use the OpenBao provider in production.',
        { code: 'DEV_SIGNING_KEY_PROVIDER' }
      );

      return new JoseSigningKeyProvider(
        config.provider,
        config.algorithm,
        config.jwks.cacheMaxAgeSecs
      );
    }

    default: {
      // Exhaustiveness; a new union member nobody wired here fails loudly at startup.
      throw new CustomError({
        name: 'CONFIG_ERROR',
        message: `Unknown signing-key provider '${JSON.stringify(config.provider)}'`
      });
    }
  }
}
```

The OpenBao `case` moves the three construction lines and the environment read out of the
old `keys.ts`; its behavior is unchanged. `readOpenBaoEnvironment` stays in the OpenBao
module.

### The neutral bootstrap step

`src/runtime/keys.ts` is reduced to composing a provider by contract and starting it. It
names no environment variable, no storage, and no concrete class:

```ts
import { createSigningKeyProvider } from '#infra/keys/index.js';
import type { ServiceConfig } from '#config/types.js';
import type { ISigningKeyProvider } from '#application/contracts.js';

export async function bootstrapSigningKeys(
  config: ServiceConfig
): Promise<ISigningKeyProvider> {
  // The factory owns provider selection, environment, and storage wiring; this step only
  // triggers the interface lifecycle, so a new provider never changes this function.
  const provider: ISigningKeyProvider = createSigningKeyProvider(config.signingKeys);

  await provider.initialize();

  return provider;
}
```

### Type neutralization

The only concrete coupling to `OpenBaoSigningKeyProvider` outside the OpenBao module lives
in two files, because `api/types.ts`, the JWKS discovery controller, `runtime/services.ts`,
and `TokensService` already depend on `ISigningKeyProvider`:

- `runtime/types.ts`; change `ServiceDependencies.signingKeys` to `ISigningKeyProvider` and
  swap the import.
- `runtime/runtime.ts`; drop the concrete import and type the local value as
  `ISigningKeyProvider`.

No teardown wiring is needed, since neither provider holds a resource the runtime must close
beyond the database connection it already manages.

## File manifest

New files:

- `infra/keys/jose/provider.ts`; `JoseSigningKeyProvider`.
- `infra/keys/jose/schema.ts`; `joseSigningKeysConfigSchema` and the private-JWK schema.
- `infra/keys/jose/keyfile.ts`; encrypted-file read, write, and crypto.
- `infra/keys/jose/types.ts`; `SigningKeySnapshot` (provider-internal) and inferred config
  types.
- `infra/keys/jose/index.ts`; barrel for the Jose provider.
- `infra/keys/factory.ts`; the selection factory.
- `infra/keys/index.ts`; barrel for the keys infrastructure.
- `infra/keys/schema.ts`; promoted neutral `signingKeyStateSchema` and `publicJwkSchema`.

Modified files:

- `runtime/keys.ts`; neutral bootstrap step.
- `runtime/runtime.ts` and `runtime/types.ts`; interface-typed signing-key provider.
- `infra/keys/config/schema.ts`; discriminated union on `name`.
- `infra/keys/openbao/schema.ts`; rename `type` to `name`, import promoted neutral schema.
- `infra/keys/openbao/provider.ts`; import the promoted neutral schema.
- `config/local.json`; rename `provider.type` to `provider.name`.
- `.gitignore`; ignore the Jose key file path.

## Implementation order

1. Promote the neutral schemas, rename the discriminator to `name`, and introduce the
   configuration discriminated union. OpenBao continues through its unchanged path.
2. Add `infra/keys/factory.ts`, reduce `runtime/keys.ts`, and apply the two composition-root
   type fixes. Verify that OpenBao boots through the factory with no behavior change.
3. Add `infra/keys/jose/keyfile.ts` and the Jose schema and types.
4. Add `JoseSigningKeyProvider` and wire the loud warning in the factory `case`.
5. Switch `config/local.json` to the Jose provider in a development environment and confirm
   first-run generation, restart reuse, and JWKS publication.
