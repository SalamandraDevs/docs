---
title: Auth Entity Cursor Pagination Plugin Plan
lastUpdate: 2026-09-13
category: architecture
lang: en
status: implemented
---

# Auth Entity Cursor Pagination Plugin Plan

## Goal

Move the HTTP-specific entity cursor lifecycle out of individual collection controllers and
into one reusable Fastify plugin in `services/auth`. The plugin will decode a requested cursor
before a handler runs and append an encoded `nextCursor` before the response is serialized.

The entity and database layers remain responsible for deterministic ordering and pagination.
The plugin only translates the cursor across the HTTP boundary.

## Current Starting Point

`Entities.find()` accepts an `EntityFindOptions` value, orders records by `createdAt DESC` and
`entityId DESC`, and returns `FindResult<T>` with `records` and `hasNextPage`. The Core cursor
schema accepts a `Date` or coercible date value and a UUID entity ID.

The users controller currently owns all transport work:

- It base64url-decodes and JSON-parses `request.query.cursor`.
- It validates the parsed value with `entityFindCursorSchema`.
- It converts the last returned `User` into the next cursor.
- It base64url-encodes the cursor and conditionally adds `nextCursor` to the response.

This will be duplicated by every future cursor-paginated collection endpoint unless it is moved
to one API component.

## Scope and Decisions

### Plugin name and location

Create `services/auth/src/api/plugins/entity-cursor-codec.ts` and export
`entityCursorCodecPlugin`.

The name intentionally describes the full lifecycle: request cursor decoding, pagination
metadata, and response cursor encoding.

### Fastify lifecycle

Use `preSerialization` for the outgoing response:

- `preSerialization` adds `nextCursor` to an object response before Fastify's response
  serializer processes it.

Incoming decoding is an explicit request decorator call, not a lifecycle hook. This lets a
controller choose whether it uses cursor pagination without a route configuration marker.

Do not use `onSend` as the response hook. At that point the payload may already be serialized
into a string or buffer, making an object-level `nextCursor` addition brittle and
serializer-dependent.

### Cursor wire contract

Keep the existing base64url JSON representation:

```json
{
  "createdAt": "2026-09-13T15:42:00.000Z",
  "entityId": "5ff51c42-951e-4ecb-a665-0e19e8f43eb2"
}
```

`JSON.stringify()` serializes an `EntityFindCursor.createdAt` `Date` as an ISO timestamp.
During decoding, pass the parsed value directly to Core's `entityFindCursorSchema.safeParse`;
its `z.coerce.date()` field produces the `Date` required by `Entities.find()`.

No API-specific cursor schema, JSON reviver, or resource-specific cursor type will be
introduced.

### Error behavior

On malformed base64url/JSON text or an invalid Core cursor schema result, the plugin throws:

```ts
new CustomError({
  name: "VALIDATION_ERROR",
  message: "Invalid entity cursor",
});
```

The existing API error plugin maps this to the documented `400` problem response. Do not
include the supplied cursor or Zod details in the public error; they may be logged only if that
becomes an explicit observability requirement.

## Plugin Contract

### Request decorator

Add a Fastify request method named `decodeCursor`:

```ts
decodeCursor(cursor: string | undefined): EntityFindCursor | null
```

It returns `null` for an absent cursor. For a supplied value, it base64url-decodes,
JSON-parses, and validates it through `entityFindCursorSchema`. Controllers call it at the
point where an HTTP cursor becomes an `Entities.find()` input.

### Reply decorator

Add a Fastify reply method named `encodeCursor`:

```ts
encodeCursor<T extends Entity>(result: FindResult<T>): void
```

It checks `hasNextPage`, derives the final returned record's `{ createdAt, entityId }`, and
encodes that cursor immediately. The method stores the resulting string privately for the
current reply. It stores nothing when there is no following page.

### Response hook behavior

`preSerialization` behaves as follows:

1. If `reply.encodeCursor(...)` was not called or did not prepare a next cursor, return the
   payload unchanged.
2. If an encoded cursor exists, return a shallow copy of the successful object payload with
   `nextCursor` added.
3. Never append a cursor to an error response. If an error replaces a successful payload after
   `encodeCursor` ran, return the problem response unchanged.

The hook must not overwrite a `nextCursor` already present in a controller payload. Treat that
as an implementation error during development rather than silently choosing one value.

## Implementation Steps

### 1. Add cursor codec plugin types

Create a sibling `types.ts` file for plugin-private types. Define the stored page metadata and
the generic reply-decorator signature there. Import `Entity`, `EntityFindCursor`, and
`FindResult` from Core rather than duplicating their structures.

Extend the `fastify` module declaration in `services/auth/src/api/types.ts` with:

- `FastifyRequest.decodeCursor`;
- `FastifyReply.encodeCursor`.

The plugin is safe for non-paginated routes because neither decorator changes state until a
controller calls it.

### 2. Implement the plugin

In `entity-cursor-codec.ts`:

1. Decorate request and reply with the contract above.
2. Use a reply-local metadata store that cannot leak across requests. A
   `WeakMap<FastifyReply, ...>` is appropriate if metadata is not exposed as a public reply
   property.
3. Implement `request.decodeCursor`. It returns `null` for `undefined`; otherwise it decodes
   the value with `Buffer`, parses JSON, and validates it through
   `entityFindCursorSchema.safeParse`.
4. Implement `reply.encodeCursor`. When `hasNextPage` is true, it requires a final record,
   creates an `EntityFindCursor`, encodes it with `JSON.stringify` and `Buffer`, and stores the
   string for the current reply. Throw an internal error if `hasNextPage` is true without a
   final record.
5. Add `preSerialization`. Read the prepared string and return an augmented successful object
   payload only when it exists. Return all other payloads unchanged.
6. Wrap the implementation with `fastify-plugin`, declare its name as
   `auth-entity-cursor-codec`, and depend on `auth-errors` so validation errors reach the
   configured RFC 9457 error boundary.

Keep the base64url encode/decode helpers private functions in this plugin. They are
implementation details, not route helpers or Core contracts.

### 3. Register the plugin

Import and register `entityCursorCodecPlugin` in `services/auth/src/api/plugins/index.ts`,
after `errorsPlugin` and before route registration.

The registration order must ensure the API error handler exists before the cursor hook can
throw. No database, entity, or service construction changes are needed.

### 4. Migrate the users list controller

No change is needed in `services/auth/src/api/routes/v1/users/routes.ts`. Retain the existing
query and response schemas; they continue to document the string `cursor` request parameter and
optional string `nextCursor` response property.

In the users controller:

1. Remove `Buffer`, `entityFindCursorSchema`, `EntityFindCursor`, and all local cursor
   encode/decode helpers.
2. Pass `request.decodeCursor(request.query.cursor)` to `users.find`.
3. After `find` resolves, call `reply.encodeCursor(result)`.
4. Return/send a response containing only the typed `items` mapping. The plugin supplies
   `nextCursor` when appropriate.

Keep explicit callback typing in the item mapping as currently required by the service style.

The resulting controller has no transport-specific cursor code:

```ts
export async function listUsers(
  request: FastifyRequest<{ Querystring: UserListQuery }>,
  reply: FastifyReply,
): Promise<void> {
  const criteria: Partial<User> = {
    ...(request.query.username ? { username: request.query.username } : {}),
    ...(request.query.email ? { email: request.query.email } : {}),
  };

  const result: FindResult<User> = await request.server.services.users.find(criteria, {
    cursor: request.decodeCursor(request.query.cursor),
    limit: request.query.limit ?? 20,
  });

  reply.encodeCursor(result);

  const response: UserListResponse = {
    items: result.records.map((item: User): UserResponse => toUserResponse(item)),
  };

  void reply.code(200).send(response);
}
```

### 5. Preserve pagination semantics

The plugin must not change any of these existing guarantees:

- A missing cursor means the first page and reaches `Entities.find` as `null`.
- A cursor contains only the entity policy's ordering fields: `createdAt` and `entityId`.
- `nextCursor` is emitted only if the persistence result reports `hasNextPage`.
- The cursor describes the final returned record, never the look-ahead record used to determine
  `hasNextPage`.
- Tenant filtering and ordering stay inside `Entities.find`; the plugin never accepts tenant
  IDs, order definitions, or database criteria from HTTP input.

The plugin also does not bind filters such as `username` and `email` into a cursor. Clients
must continue to send identical filters when retrieving later pages. Add filter binding only as
an explicit API contract change, not as an implicit plugin behavior.

## Verification

Perform the following checks after implementation:

1. Run `pnpm run build` in `core`, because auth consumes Core's generated package output.
2. Run `pnpm run check` in `services/auth` to verify Fastify decorator augmentation, plugin
   hook signatures, and the migrated controller types.
3. Manually request `GET /v1/users?limit=1` against a tenant with at least two users. Confirm
   the response has one item and a `nextCursor`.
4. Send that `nextCursor` back as `cursor`; confirm the next page begins after the prior final
   item and has no duplicate item.
5. Request the final page; confirm it has no `nextCursor` property.
6. Send malformed base64url text, invalid JSON, an invalid date, an invalid UUID, and an object
   with an extra property. Confirm each receives the existing `400` validation problem
   response.
7. Confirm a non-paginated route remains unaffected when neither decorator is called.

No automated tests are included in this plan. Add them only when explicitly requested.

## Non-Goals

- Signing or encrypting cursors.
- Offset/page-number pagination.
- Moving HTTP/base64url behavior into Core.
- Supporting custom order definitions supplied by clients.
- Automatically applying cursor behavior to routes that do not call the codec decorators.
