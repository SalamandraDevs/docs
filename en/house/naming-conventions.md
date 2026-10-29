---
title: Naming Conventions
lastUpdate: 2026-10-29
category: architecture
lang: en
status: published
---

## Purpose

Define the single global naming standard for SalamandraDevs engineering and DevOps.

This document is authoritative for names and paths. When code or other docs disagree, follow
this document first, then align implementation.

## Document IDs

House and architecture documents give each section a stable ID, `XX-NN`: a two-letter
document prefix and a two-digit number (`GP-05`). An ID-bearing section uses the heading
`### XX-NN Title`. Never reuse or renumber an ID; add new sections with the next free
number. Prefixes: `SP` SOLID principles, `DP` design patterns, `GP` development good
practices, `CS` code style, `DS` documents style, `BP` build plans.

## Engineering

### Services

- **Service ids:** single word lowercase name.
- **Defined ids:** `auth`.
- **Repository directory:** `services/{service}/` where `{service}` equals the service id.
- **Package name:** `@salamandradevs/{service}` (for example `@salamandradevs/auth`).

### Service structure

Each service source tree under `services/{service}/src/` is split into these main sections:

- `application/`; use-case services and the contracts they publish or consume. Required.
  - `{plural}-service/` per use case (`users-service/`, `sessions-service/`), holding
    `{plural}-service.ts`, `schema.ts`, `types.ts`, and `index.ts`.
  - `contracts.ts` at the section root for the application contracts (`ISessionsService`,
    `ISigningKeyProvider`, `IOAuthProvider`).
  - `config/` for the application configuration schema and types.
- `config/`; loads and validates the service configuration file (`config.ts`, `schema.ts`,
  `types.ts`). The deployment files themselves live in `services/{service}/config/`
  (`devel.json`, `prod.json`). Required.
- `infra/`; adapters for external systems, one folder per concern (`keys/`, `oauth/`), with
  one subfolder per vendor (`keys/jose/`, `keys/openbao/`, `oauth/google/`). Only when the
  service integrates external systems.
- `runtime/`; the composition root. `runtime.ts` exposes `createRuntime`, and each bootstrap
  step lives in a file named for its topic (`database.ts`, `keys.ts`, `oauth.ts`,
  `services.ts`, `admin.ts`). Required.
- `storage/`; record modules and the dataset collection (`datasets.ts`). Only when the
  service persists records. See [Record modules](#record-modules).

The process entry point is `src/index.ts`.

### TypeScript import aliases (services)

| Alias | Maps to |
| --- | --- |
| `#root/*` | `src/*` |
| `#api/*` | `src/api/*` |
| `#application/*` | `src/application/*` |
| `#config/*` | `src/config/*` |
| `#infra/*` | `src/infra/*` |
| `#storage/*` | `src/storage/*` |

- Declare every alias twice: in `package.json` `imports` (mapping to `./dist/...`) and in
  `tsconfig.json` `paths` (mapping to `src/...`).
- Use `.js` extension in relative and package import specifiers (NodeNext).

### APIs

- **Format:** `/{service}/v{N}/{resource}/...` with lowercase path segments.
- **Service segment:** service id, configured as `http.apiPrefix` (`/auth`).
- **Version segment:** `v1`, `v2`, etc.
- **Examples:** `/auth/v1/users`, `/auth/v1/users/:recordId`, `/auth/v1/session`,
  `/auth/v1/session/refresh`, `/auth/v1/oauth/:provider/callback`.
- **Segments:** lowercase single words; no uppercase, mixed case, or kebab-case.
- **Path parameters:** camelCase matching the record field (`:recordId`, `:provider`).
- **Unversioned routes:** service-level endpoints sit outside the version segment
  (`/health`, `/openapi.json`, `/docs`, `/.well-known/jwks.json`).

### Records

- **Record type:** PascalCase singular with a `Record` suffix (`UserRecord`,
  `SessionRecord`).
- **Records handler class:** PascalCase singular with a `RecordsHandler` suffix, extending
  core `RecordsHandler` (`UserRecordsHandler`, `SessionRecordsHandler`).
- **Handler name:** the `handlerName` literal equals the dataset name (`'users'`,
  `'sessions'`).
- **Fields:** camelCase in TypeScript types and Zod schemas (`tenantId`, `recordId`,
  `handlerName`, `createdBy`, `updatedAt`).
- **Status enums:** lowercase single-word value literals (`active`, `suspended`,
  `terminated`), exported as `{resource}RecordStatusSchema` with the inferred type
  `{Resource}RecordStatus`.
- **Never:** PascalCase field names; snake_case field names in application types or Zod
  record schemas.

### Record modules

Each record module under `{service}/src/storage/{plural}/`:

| File | Naming |
| --- | --- |
| `{singular}-records-handler.ts` | records handler class (`user-records-handler.ts` → `UserRecordsHandler`) |
| `schema.ts` | Zod record and status schemas |
| `datasetdef.ts` | `DatasetDef` + `IndexDef` entries, always including `recordIndexDef` |
| `types.ts` | types inferred from the record schemas |
| `index.ts` | barrel re-exports |

**Folder `{plural}`:** lowercase plural matching the dataset name (`users`, `sessions`).

### Symbol conventions (per record)

| Symbol kind | Pattern | Example |
| --- | --- | --- |
| Record type | `{Resource}Record` | `UserRecord` |
| Record Zod schema | `{resource}RecordSchema` | `userRecordSchema` |
| Record status Zod schema | `{resource}RecordStatusSchema` | `userRecordStatusSchema` |
| Record status type | `{Resource}RecordStatus` | `UserRecordStatus` |
| Records handler class | `{Resource}RecordsHandler` | `UserRecordsHandler` |
| Dataset definition | `{resource}DatasetDef` | `userDatasetDef` |
| Application service class | `{Resources}Service` | `UsersService` |
| Application service contract | `I{Resources}Service` | `ISessionsService` |
| Application input Zod schema | `{action}{Resource}InputSchema` | `createUserInputSchema` |
| Application input type | `{Action}{Resource}Input` | `CreateUserInput` |
| Application view Zod schema | `{resource}ViewSchema` | `userViewSchema` |
| Application view type | `{Resource}View` | `UserView` |
| Request body Zod schema | `{action}{Resource}BodySchema` | `createUserBodySchema` |
| Request body type | `{Action}{Resource}Body` | `CreateUserBody` |
| List query Zod schema | `{resource}ListQuerySchema` | `userListQuerySchema` |
| List query type | `{Resource}ListQuery` | `UserListQuery` |
| Response Zod schema | `{resource}ResponseSchema` | `userResponseSchema` |
| Response type | `{Resource}Response` | `UserResponse` |
| List response Zod schema | `{resource}ListResponseSchema` | `userListResponseSchema` |
| Problem responses map | `{resource}ProblemResponses` | `userProblemResponses` |
| Route def | `{Action}{Resource}Request` | `UpdateUserRequest` |

- Find operations over a collection use the plural resource (`findUsersInputSchema`,
  `FindUsersInput`).
- A route def is the Fastify route generic (`{ Params; Body }`) shared by the route, its
  hooks, and its controller; it is not a payload type.
- Path parameter schemas shared across route groups use the record field name
  (`recordParamsSchema`, `RecordParams`).

### Contracts

- Interfaces carry an `I` prefix followed by a capability or role name (`ICrud`,
  `IDataset`, `IDatabaseProvider`, `ISigningKeyProvider`, `IOAuthProvider`).
- Implementations are named for their vendor and role (`MongoCrud`, `MySqlDataset`,
  `JoseSigningKeyProvider`, `GoogleOAuthProvider`).

### Core library (`@salamandradevs/core`)

Established forms (full words remain valid under [Abbreviations](#abbreviations)):

- **`Db`** for database connection types (`IDbConnection`, `MongoDbConnection`,
  `MySqlDbConnection`).
- **`Dataset` not abbreviated** for dataset contracts and classes (`IDataset`,
  `DatasetProvisioner`, `MongoDataset`).
- **`Def`** for definition records (`DatasetDef`, `IndexDef`, `recordIndexDef`,
  `{resource}DatasetDef`).
- **Implementation files:** `{backend}-{name}.ts` kebab-case inside the backend folder
  (`mongo/mongo-crud.ts`, `mongo/mongo-db-connection.ts`, `mysql/mysql-crud.ts`,
  `mysql/mysql-dataset.ts`).
- **Records layer:** `storage/` holds `RecordsHandler`, `BaseRecord`, `baseRecordSchema`,
  and `recordIndexDef`.

### Dataset and index names

Defined in each record module `datasetdef.ts`; the same definitions provision MongoDB
collections and MySQL tables.

| Item | Convention | Examples |
| --- | --- | --- |
| `DatasetDef.name` | lowercase plural dataset name | `users`, `sessions` |
| `IndexDef.name` | snake_case descriptive name | `tenant_record`, `tenant_username`, `tenant_email`, `tenant_created_record` |
| `IndexDef.fields[].name` | camelCase field names matching record fields | `{ name: 'tenantId', direction: 'asc' }` |

### Abbreviations

Technical abbreviations are optional: prefer the full word when the name stays clear. Only
established industry shortenings are permitted, and only when they keep a compound symbol
readable.

**Word count**

Count only full words when deciding whether a name is long enough to abbreviate. Prefer
shortening when the full compound exceeds **two counted words**; conventional standalone
terms and suffixes (`Id`, `Config`, `Crud`) are exempt.

Standard acronyms do **not** count as words: `HTTP`, `API`, `JSON`, `SQL`, `URI`, `JWT`,
`JWK`, `OAuth`, `CRUD`, and similar industry forms. Render them according to identifier
casing (`IOAuthProvider`, `MySqlCrud`).

Examples:

- `Mongo` + `Database` + `Connection` → three words → `MongoDbConnection`
- `user` + `Dataset` + `Definition` → three words → `userDatasetDef`
- `OAuth` + `Provider` → one counted word (`Provider`) → keep `IOAuthProvider`; do not
  abbreviate further

**When to abbreviate**

- Abbreviate only when the full spelling is unnecessarily long; the spelled-out form
  remains valid (`MongoDatabaseConnection`, `userDatasetDefinition`).
- Shorten **at most one** listed word per compound name.
- Do not abbreviate short or already-clear names (`Dataset`, `User`, `Session`,
  `Connection`).
- Do not invent project-specific shortenings (`Usr`, `Prov`, `Cfg`).

**Consecutive abbreviations**

Two abbreviations must not appear back to back in the same symbol.

- Allowed: `MongoDbConnection` (`Db` only), `IndexDef` (`Def` only)
- Not allowed: `DbCrud`, `CrudDef`, `DbDef`

**Allowed abbreviations**

| Full word | Abbreviation | Example |
| --- | --- | --- |
| Database | `Db` | `IDbConnection`, `MongoDbConnection` |
| Definition | `Def` | `DatasetDef`, `userDatasetDef` |
| Identifier | `Id` | `recordId`, `tenantId`, `actorId` (field suffix) |
| Configuration | `Config` | `config/`, `ServiceConfig` |
| Context | `Ctx` | `getExecutionCtx`, `runWithExecutionCtx` |
| Create, Read, Update, Delete | `Crud` | `ICrud`, `MongoCrud` |

**Never**

- Two or more listed abbreviations in one symbol, especially adjacent (`DbCrud`).
- Ad-hoc truncations not listed above.
- Abbreviations in service ids or API path segments.

### Files and source files

- **Directories and filenames:** lowercase; multi-word segments use kebab-case
  (`users-service/`, `user-records-handler.ts`). The fixed module filename `datasetdef.ts`
  is the only concatenated exception.
- **TypeScript classes, interfaces, types:** PascalCase.
- **Functions, constants, variables:** camelCase.
- **Never:** `camelCase` or `PascalCase` directory names; spaces in paths.

## DevOps

### Branches

Flow branches: `devel`, `main`, `staging`, `production`.

| Branch | Role |
| --- | --- |
| `devel` | continuous development; ahead of `main` |
| `main` | frozen development state; fixes and minor features; staging candidate |
| `staging` | deployment target on staging infra (may share prod infra); non-production domain |
| `production` | deployment target on production infra; principal domain |

Feature and fix work uses `{featureBranch}` or `{fixBranch}` names chosen per team
convention.

- **Flow branch names:** lowercase single words only; no uppercase, separators, or mixed
  casing.
