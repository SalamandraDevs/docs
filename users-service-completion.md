---
title: Users Service Completion Plan
lastUpdate: 2026-09-08
category: architecture
lang: en
status: implemented
---

# Users Service Completion Plan

## Goal

Complete the user application boundary so HTTP user routes talk only to `UsersService`, while
`UserRecordsHandler` remains an internal persistence dependency composed in `bootstrapService`.

The desired boundary is:

- API controllers depend on `request.server.services.users`.
- `UsersService` owns user create, register, read, update, delete, and find use cases.
- `UserRecordsHandler` remains available to lower-level application collaborators that need
  credential or identity lookup, but it is no longer exposed through Fastify API services.
- HTTP response serialization stays in the API route layer through `toUserResponseSchema`.

## Current State

`UsersService` currently owns only:

- `create(input: CreateUserInput): Promise<UserView>`
- `register(input: RegisterUserInput): Promise<UserView>`

The user API controller calls `request.server.services.userRecordsHandler` directly for:

- `readUser`
- `updateUser`
- `deleteUser`
- `findUsers`

This weakens the application boundary because controllers can bypass user policy and operate on
durable `UserRecord` values directly. It also makes `ApiServices` expose both the high-level
`users` service and its lower-level record dependency.

## Boundary Decisions

Keep these responsibilities separate:

- `UserRecordsHandler`: tenant-scoped persistence operations and record metadata policy.
- `UsersService`: user application use cases and password-safe `UserView` projection.
- User route controllers: HTTP request/response concerns, cursor transport, and response
  serialization with `toUserResponseSchema`.

Do not move `toUserResponseSchema` into `UsersService`. `UsersService` should return `UserView`
with `Date` values. The API layer should serialize those values into `UserResponse`.

Do not remove `UserRecordsHandler` from service construction. `PasswordsService`,
`OAuthService`, bootstrap administrator setup, and `UsersService` still need it as an internal
composition dependency.

## Implementation Plan

### 1. Add application input types for read, update, delete, and find

Add application-layer schemas and inferred types in `application/users-service/schema.ts` and
`application/users-service/types.ts`.

Suggested schemas:

- `readUserInputSchema`: `{ userId: z.uuid() }`
- `deleteUserInputSchema`: `{ userId: z.uuid() }`
- `updateUserInputSchema`: `{ userId: z.uuid(), roles?: ..., status?: ... }`
- `findUsersInputSchema`: `{ username?: ..., email?: ..., cursor: ..., limit: ... }`

Prefer reusing route schemas only when doing so does not create an API-to-application import
direction. Application schemas must not import from `api/routes`.

For find options, use the core cursor type/schema as the application boundary input:

- `cursor: RecordFindCursor | null`
- `limit: number`

The controller should continue decoding the opaque HTTP cursor with
`request.decodeCursor(...)`. That keeps opaque cursor transport in the API layer and passes
only core pagination input to `UsersService`.

### 2. Add `UsersService` methods that wrap `UserRecordsHandler`

Add methods to `UsersService`:

- `read(input: ReadUserInput): Promise<UserView | null>`
- `update(input: UpdateUserInput): Promise<UserView | null>`
- `delete(input: DeleteUserInput): Promise<UserView | null>`
- `find(input: FindUsersInput): Promise<FindResult<UserView>>`

Each method should parse its input through the application schema, call `UserRecordsHandler`,
and project returned records through `userViewSchema.parse(...)`.

For `find`, preserve pagination metadata:

```ts
const result: FindResult<UserRecord> = await this.userRecordsHandler.find(criteria, options);

return {
  ...result,
  records: result.records.map((user: UserRecord): UserView => userViewSchema.parse(user))
};
```

This is not a convenience wrapper because it establishes the application boundary and strips
sensitive fields before returning to callers.

### 3. Update user controllers to depend only on `services.users`

Change user route handlers so they call:

- `request.server.services.users.read({ userId: request.params.userId })`
- `request.server.services.users.update({ userId: request.params.userId, ...request.body })`
- `request.server.services.users.delete({ userId: request.params.userId })`
- `request.server.services.users.find({ username, email, cursor, limit })`

Controllers should still:

- translate `null` results to `RESOURCE_NOT_FOUND`
- call `reply.encodeCursor(result)` for paginated results
- serialize route payloads with `toUserResponseSchema.parse(...)`

After this change, controllers should no longer import `UserRecord` or access
`request.server.services.userRecordsHandler`.

### 4. Remove `userRecordsHandler` from decorated API services

Update `ApiServices` in `api/types.ts` to remove the public `userRecordsHandler` property.

Update `bootstrapService` so `userRecordsHandler` is still constructed and injected into:

- `PasswordsService`
- `UsersService`
- `OAuthService`
- bootstrap administrator setup

But do not include it in the `services` object passed to `buildApi`.

This keeps record persistence available to application collaborators while preventing routes
from bypassing `UsersService`.

### 5. Review remaining direct record-handler usage

After removing the decoration, search for:

```sh
rg "services\\.userRecordsHandler|userRecordsHandler" services/auth/src
```

Expected remaining usages:

- construction and internal composition in `bootstrap.ts`
- constructor dependencies in `UsersService`, `PasswordsService`, and `OAuthService`
- direct internal credential/identity lookups where those services legitimately own that policy

Unexpected usages:

- route controllers
- API plugins except where a plugin is explicitly an application-service adapter
- exported API service types

## Boundary Risks

### Pagination Cursor Shape

`reply.encodeCursor(result)` requires `FindResult<T extends BaseRecord>` because it reads
`createdAt` and `recordId` from the final record.

`UserView` currently includes `createdAt` and `recordId`, so `FindResult<UserView>` should
still work. Keep `UserView` as a base-record-compatible shape with `Date` timestamps. Do not
serialize dates inside `UsersService`; date serialization belongs in `toUserResponseSchema`.

### Update Input Scope

The current route update schema allows `roles` and `status`. When moving the update use case
into `UsersService`, keep the allowed update fields explicit. Do not accept arbitrary
`Partial<UserRecord>` at the service boundary.

### Not Found Handling

`UsersService` can either return `null` or throw `RESOURCE_NOT_FOUND`. Prefer returning `null`
initially to preserve current controller behavior and avoid moving HTTP-facing error semantics
into the application service prematurely.

### API Schema Duplication

Some API schemas and application schemas may temporarily describe similar shapes. Avoid
importing API route schemas into the application layer. If duplication becomes painful, promote
shared schema pieces downward into the application layer and let API route schemas compose from
them.

## Verification

- Run build/type and lint verification only:
```sh
pnpm run check
node_modules/.bin/eslint src/application/users-service/schema.ts \
  src/application/users-service/types.ts \
  src/application/users-service/users-service.ts \
  src/api/types.ts \
  src/api/routes/v1/users/controller.ts
```
- Do not add or run tests unless explicitly requested.
- Make a second iteration over ../AGENTS.md to verify that the generated code or modified files meet the defined rules.
