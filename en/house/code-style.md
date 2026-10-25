---
title: Code Style
lastUpdate: 2026-10-24
lang: en
category: guide
status: published
---

## Purpose

House code style for TypeScript across the Salamandra Devs platform, including how RESTful
API services compose Fastify route hooks. These rules apply to every contributor, human or
agent. Source comment rules live in the [Documents Style Guide](./documents-style-guide.md).

Each rule section carries a stable ID (`CS-NN`) for citation; IDs follow
[Document IDs](./naming-conventions.md#document-ids).

## TypeScript

### CS-01 Imports

- Keep import declarations on a single line. When an import would exceed the configured
  Prettier line length, split it into multiple single-line import declarations from the
  same module instead of using a multi-line import block.
- When editing imports, preserve the repository's current grouping preference: external
  imports first, then external type imports, then a blank line, then internal value
  imports, then type imports, prioritizing alias imports. This is house style for
  readability, not a TypeScript recommendation.

```ts
import Fastify from 'fastify';
import { serializerCompiler, validatorCompiler } from 'fastify-type-provider-zod';
import { CustomError } from '@salamandradevs/tools/shared';
import type { FastifyBaseLogger, FastifyInstance } from 'fastify';
import type { RawReplyDefaultExpression } from 'fastify';
import type { BaseRecord } from '@salamandradevs/core';

import { OAuthService } from '#application/oauth-service/oauth-service.js';
import { UsersService } from '#application/users-service/users-service.js';
import { UserRecordsHandler } from '#storage/users/index.js';
import type { IPasswordsService } from '#application/contracts.js';
import type { ServiceConfig } from '../config/types.js';
```

- Do not import classes with `import type`, even when they are currently used only in type
  positions. Import classes with ordinary value imports so their runtime nature remains
  clear.

```ts
// Avoid
import type { UsersService } from '../application/users-service/users-service.js';

// Prefer
import { UsersService } from '../application/users-service/users-service.js';
```

### CS-02 Zod schemas and derived types

- Do not add explicit TypeScript type annotations to Zod schema constants in schema files.
  Schema files must rely on the global ESLint override that disables
  `@typescript-eslint/typedef` for schema definitions. If the global exception is not
  present, use the file-scoped rule `/* eslint-disable @typescript-eslint/typedef */`.

```ts
// Avoid
export const recordParamsSchema: z.ZodType<RecordParams> = z
  .object({ recordId: z.uuid() })
  .strict();

// Prefer
export const recordParamsSchema = z
  .object({ recordId: z.uuid() })
  .strict();
```

- Prefer deriving types from Zod schemas with `z.infer` instead of hard-coding matching
  TypeScript shapes when the schema is the runtime validation source. Avoid circular
  imports: schema files must not import the types that are inferred from those same
  schemas.

```ts
// schema.ts
export const recordParamsSchema = z
  .object({ recordId: z.uuid() })
  .strict();

// types.ts
import type { z } from 'zod';

import type { recordParamsSchema } from './schema.js';

/** Record path parameters accepted by item routes. */
export type RecordParams = z.infer<typeof recordParamsSchema>;
```

- Use Zod for validation, projection, normalization, and shape-level serialization when a
  schema is the runtime source of truth. Use regular functions for behavior,
  orchestration, and policy. Avoid convenience wrappers that only rename, forward, or parse
  through an existing schema without adding real transformation or validation.

### CS-03 Wrappers

- Do not create a wrapper function whose only purpose is to give an existing function a
  different name. If a wrapper calls another function with the same arguments, same
  return type, and no added logic, it is dead weight; call the original function directly
  instead. This is a TypeScript-specific application of the general
  [convenience-wrapper rule](#cs-07-convenience-wrappers).

```ts
// Avoid
export async function hashPassword(password: string): Promise<string> {
  return argon2.hash(password);
}

// Prefer
const passwordHash: string = await argon2.hash(password);
```

- Do not create wrapper types that only rename another type without adding domain
  meaning, validation, or structural constraints. Use the original type directly.

```ts
// Avoid
export type UserId = string;

// Prefer
const recordId: string = request.params.recordId;
```

### CS-04 Errors

- As the error-specific application of the general convenience-wrapper rule, do not
  create wrapper functions to throw errors; throw a `new CustomError()` instead.

```ts
// Avoid
function throwAuthenticationFailed(): never {
  throw new CustomError({
    name: 'AUTHENTICATION_FAILED',
    message: 'Authenticated session grant is required'
  });
}

// Prefer
throw new CustomError({
  name: 'AUTHENTICATION_FAILED',
  message: 'Authenticated session grant is required'
});
```

- Every error that application code throws or passes to a callback must be a
  `CustomError` with a deliberate `ErrorName`. Do not rely on an API error handler's
  fallback mapping to classify an error. A plain `Error` is permitted only inside a `try`
  block when its enclosing `catch` immediately converts it into a `CustomError` and retains
  it as the cause.

```ts
try {
  const response: Response = await fetch(url);

  if (!response.ok) {
    throw new Error('Provider request failed');
  }
} catch (error: unknown) {
  throw new CustomError({
    name: 'EXTERNAL_API_ERROR',
    message: 'OAuth provider request failed',
    cause: error
  });
}
```

### CS-05 Files and modules

- Do not write type declarations in the same file that defines a class. Put those types in
  a sibling `types.ts` file and import them from the class implementation file.

```ts
// users-service.ts
import type { CreateUserInput, UserView } from './types.js';

export class UsersService {
  async create(input: CreateUserInput): Promise<UserView> {
    // ...
  }
}

// types.ts
export type CreateUserInput = z.infer<typeof createUserInputSchema>;
```

- Every module `index.ts` must be a barrel file using `export * from './module.js'`. Do
  not use individual value or type re-exports in module index files.

```ts
// Avoid
export { UsersService } from './users-service.js';
export type { UserView } from './types.js';

// Prefer
export * from './users-service.js';
export * from './types.js';
```

## General code style rules

### CS-06 Blank lines

- Conditional blocks (if, switch, etc.), loop blocks (for, for...of, while, etc.),
  try-catch, and function declaration blocks must be preceded and followed by a blank line
  to visually separate them from surrounding code. Treat an if-else chain or
  try-catch-finally statement as one construct: add blank lines before and after the
  complete construct, not between its connected clauses. A blank line is not required
  between a construct and its immediately enclosing brace or a file boundary. Return
  statements must always be preceded by a blank line, except when they are the sole
  statement inside their enclosing block.

```ts
const sessionId: string | undefined = request.sessionGrant?.sessionId;

if (!sessionId) {
  throw new CustomError({
    name: 'AUTHENTICATION_FAILED',
    message: 'Authenticated session grant is required'
  });
}

const session: SessionView = await services.sessions.read(sessionId);

return session;
```

### CS-07 Convenience wrappers

- Do not create convenience wrappers that only hide a property access, return an input
  unchanged, or forward to another function without adding validation, transformation, or
  policy. Inline the direct operation at the call site instead. The
  [Wrappers](#cs-03-wrappers) and [Errors](#cs-04-errors) sections provide specific
  applications for rename-only and error-throwing wrappers.
- As an exception, a method that forwards to a handler or another service solely to
  satisfy a published contract (an interface) across an architectural boundary is not a
  convenience wrapper, even with the same arguments, same return type, and no added logic;
  the delegation itself is the boundary, and inlining it at the call site would break the
  layering. An example is an application-service method that fulfills its interface by
  delegating to a records handler, so a consumer depends on the service contract rather
  than on storage.

```ts
// Avoid
function readSessionId(request: FastifyRequest): string {
  return request.sessionGrant.sessionId;
}

// Prefer
const sessionId: string = request.sessionGrant.sessionId;
```

### CS-08 Delegated operation names

- Do not rename a delegated operation. When a method or function only forwards to another
  operation without adding validation, transformation, policy, or orchestration, give it
  the same name as the operation it delegates to; do not introduce a synonym that renames
  the concept at the boundary. One operation keeps one name as it crosses layers, the way
  a `recordId` keeps its name from transport to storage, so a reader can trace it and
  concepts are not silently remapped; a `deactivateItem` that merely calls `deleteRow`
  hides that it deletes. This complements the boundary-delegation exception above: the
  delegation is allowed, and it must stay transparent. When the method genuinely adds
  behavior, name it for that behavior instead.

```ts
// Avoid
async revokeAllForSubject(subject: string): Promise<number> {
  return this.sessionRecordsHandler.deleteBySubject(subject);
}

// Prefer
async deleteBySubject(subject: string): Promise<number> {
  return this.sessionRecordsHandler.deleteBySubject(subject);
}
```

## RESTful API hooks

How RESTful API services compose Fastify route hooks: where shared hooks live, when a
route group may define its own, and how route-local hooks stay typed without assertions.

### CS-09 Shared hooks

- Cross-cutting or reusable route hooks must be published as Fastify plugin decorations.
  This includes authentication, execution-context binding, rate/tenant/security hooks, and
  common authorization hooks. Do not export free-floating helper functions from shared hook
  modules and pass them directly as Fastify lifecycle hooks.
- Use `api/routes/v1/hooks/` as the preferred source for shared v1 hook composition. These
  files should compose decorated Fastify hooks into ordered arrays for shared route
  policies.

```ts
// api/routes/v1/hooks/authenticate.ts
export function protectionHooks(fastify: FastifyInstance): PreHandlerHooks {
  return [fastify.authenticateRequest, fastify.bindExecutionContext];
}
```

### CS-10 Route-local hooks

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

### CS-11 Controllers

- Controllers must assume declared hooks already established route preconditions. Do not
  repeat authentication, execution-context, tenant, or common authorization checks in
  controllers unless the controller is handling a distinct runtime state that hooks cannot
  establish.

### CS-12 Typed route-local hooks

- A route-local hook that needs typed access to route-specific request parts (`params`,
  `body`, `querystring`, `headers`) is typed through a route def type, never through
  `FastifyRequest` augmentation and never through assertions. Augmentation is reserved for
  route-invariant decorations such as `sessionGrant` or cursor codecs.
- Type the hook parameter as `FastifyRequest<TRouteDef>`, compose the hook array through
  the shared `PreHandlerHooks<TRouteDef>` helper from `api/routes/v1/hooks/types.ts`, and
  reuse the same route def at route registration and in the controller so the request
  contract has one source of truth. Define `TRouteDef` in `api/routes/v1/[slug]/types.ts`.
- Keep shared, route-invariant composers on the default `PreHandlerHooks` generic and put
  the specialization on the route-local composer; a default-generic hook composes into a
  specialized array, never the reverse.
- Clue: if a hook needs asserting `request.params`, `body`, or `query` (`as ...`) to
  typecheck, the route def approach was not applied; introduce a route def and thread it
  through the composer instead of asserting.

```ts
// api/routes/v1/hooks/types.ts
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
