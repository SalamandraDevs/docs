---
title: Naming Conventions
owner: OdairTrujillo
lastUpdate: 2026-08-21
category: architecture
lang: en
status: published
---

## Purpose

Define the single global naming standard for slmdevs engineering and DevOps.

This document is authoritative for names and paths. When code or other docs disagree, follow this document first, then align implementation.

## Engineering

### Services

- **Canonical service ids:** single word lowercase name.
- **Defined ids:** `auth`, `profile`, `gateway`.
- **Examples:** `messages`, `students`.
- **Repository directory:** `services/{service}/` where `{service}` equals the canonical id.
- **Package name:** `@slmdevs/{service}` (for example `@slmdevs/auth`, `@slmdevs/profile`).

### APIs

- **Format:** `/{service}/v{N}/...` with lowercase path segments.
- **Service segment:** canonical service id.
- **Version segment:** `v1`, `v2`, etc.
- **Examples:** `api.slmdevs.com/profile/v1/person`, `/auth/v1/user`.
- **Segments:** lowercase single words; no uppercase, mixed case, or kebab-case.

### Entities

- **Entity record type:** PascalCase singular (`User`, `Person`, `Org`, `OutboxEvent`).
- **Policy class (service module):** PascalCase plural (`Users`, `Persons`, `Orgs`, `OutboxEvents`).
- **Fields:** camelCase in TypeScript types and Zod schemas (`tenantId`, `entityId`, `createdBy`, `updatedAt`).
- **Status enums:** PascalCase type name, camelCase value literals, and `{entity}StatusSchema` Zod export.
- **Never:** PascalCase field names; snake_case field names in application types or Zod entity schemas.

### Entity modules

Each aggregate under `{service}/src/entities/{plural}/`:

| File | Naming |
| --- | --- |
| `{plural}.ts` | policy class file (`users.ts` → `Users`) |
| `schema.ts` | Zod entity + request schemas |
| `datasetdef.ts` | `MongoDatasetDef` + indexes |
| `types.ts` | records, commands, queries |
| `index.ts` | barrel re-exports |

**Folder `{plural}`:** lowercase or kebab-case plural matching the Mongo collection or domain plural (`users`, `persons`, `orgs`, `outbox-events`).

**Organization aggregate:** `Org` is the registered entity token, not a technical abbreviation. Canonical names: record `Org`, policy class `Orgs`, module directory `orgs/`, Mongo collection `orgs`, event subject `org`. Use `Organization` / `organizations` only in prose when clearer.

### Events (NATS)

- **Format:** `{service}.{subject}.{action}` — dot-separated, lowercase tokens.
- **Subject:** aggregate or domain noun in singular lowercase (`user`, `person`, `org`, `outbox`).
- **Action:** past tense verb (`created`, `updated`, `deleted`, `published`).
- **Examples:** `auth.user.created`, `profile.person.updated`, `profile.org.created`.
- **Never:** slashes in event names; variable ids in the subject (put ids in payload or headers); uppercase tokens.

### Symbol conventions (per aggregate)

| Symbol kind | Pattern | Example |
| --- | --- | --- |
| Entity record type | `{Entity}` | `Person` |
| Policy class | `{Entities}` plural | `Persons` |
| Entity Zod schema | `{entity}Schema` | `personSchema` |
| Status Zod schema | `{entity}StatusSchema` | `personStatusSchema` |
| Create request Zod schema | `create{Entity}RequestSchema` | `createPersonRequestSchema` |
| Create request DTO | `Create{Entity}Request` | `CreatePersonRequest` |
| Create command | `Create{Entity}Command` | `CreatePersonCommand` |
| Update request Zod schema | `update{Entity}RequestSchema` | `updatePersonRequestSchema` |
| Update request DTO | `Update{Entity}Request` | `UpdatePersonRequest` |
| Update command | `Update{Entity}Command` | `UpdatePersonCommand` |
| Response DTO | `{Entity}Response` | `PersonResponse` |
| Dataset definition | `{entity}DatasetDef` | `personDatasetDef`, `orgDatasetDef` |

Second aggregate example (`Org`): record `Org`, policy `Orgs`, schemas `orgSchema` / `orgStatusSchema`, requests `CreateOrgRequest`, commands `CreateOrgCommand`, responses `OrgResponse`.

### TypeScript import aliases (services)

| Alias | Maps to |
| --- | --- |
| `#root/*` | `src/*` |
| `#config/*` | `src/config/*` |
| `#entities/*` | `src/entities/*` |

Use `.js` extension in relative and package import specifiers (NodeNext).

### Core library (`@slmdevs/core`)

Established forms (full words remain valid under [Abbreviations](#abbreviations)):

- **`Db`** for database connection types (`IDbConnection`, `MongoDbConnection`).
- **`Dataset` not abbreviated** for the abstract class and methods (`Dataset`, `ensureDataset()`).
- **`Def`** for definition records (`MongoDatasetDef`, `MongoIndexDef`, `{entity}DatasetDef`).
- **Mongo implementation files:** `mongo-{name}.ts` kebab-case (`mongo-crud.ts`, `mongo-db-connection.ts`).
- **MySQL implementation files:** `{name}.ts` without extra prefix where folder is already `mysql/` (`mysqlcrud.ts`, `mysqldataset.ts`).

### Mongo dataset and index names

Defined in each entity `datasetdef.ts`:

| Item | Convention | Examples |
| --- | --- | --- |
| `MongoDatasetDef.name` | lowercase plural collection name | `users`, `persons`, `orgs`, `events_outbox`, `contractors` |
| `MongoIndexDef.name` | snake_case descriptive name | `tenant_username`, `tenant_email`, `tenant_person` |
| Index keys | camelCase field names matching entity fields | `{ tenantId: 1, username: 1 }` |

### Abbreviations

Technical abbreviations are optional: prefer the full word when the name stays clear. Only established industry shortenings are permitted, and only when they keep a compound symbol readable. Registered entity tokens such as `Org` are canonical names and are outside these rules.

**Word count**

Count only full words when deciding whether a name is long enough to abbreviate. Prefer shortening when the full compound exceeds **two counted words**; conventional standalone terms and suffixes (`Id`, `Config`, `Crud`) are exempt.

Standard acronyms do **not** count as words: `HTTP`, `API`, `JSON`, `BSON`, `SQL`, `NATS`, `URI`, `DTO`, `CRUD`, and similar industry forms. Render them according to identifier casing (`HttpApiClient`).

Examples:

- `Mongo` + `Database` + `Connection` → three words → `MongoDbConnection`
- `person` + `Dataset` + `Definition` → three words → `personDatasetDef`
- `HTTP` + `API` + `Client` → one counted word (`Client`) → keep `HttpApiClient`; do not abbreviate further

**When to abbreviate**

- Abbreviate only when the full spelling is unnecessarily long; the spelled-out form remains valid (`MongoDatabaseConnection`, `personDatasetDefinition`).
- Shorten **at most one** listed word per compound name.
- Do not abbreviate short or already-clear names (`Dataset`, `Person`, `User`, `Connection`).
- Do not invent project-specific shortenings (`Pers`, `Prov`, `Cfg`).

**Consecutive abbreviations**

Two abbreviations must not appear back to back in the same symbol.

- Allowed: `MongoDbConnection` (`Db` only), `MongoIndexDef` (`Def` only; spell out `Index` when `Def` is also present)
- Not allowed: `MongoDbIdxDef`, `DbCrud`, `CrudDto`, `IdxDef`

**Allowed abbreviations**

| Full word | Abbreviation | Example |
| --- | --- | --- |
| Database | `Db` | `IDbConnection`, `MongoDbConnection` |
| Definition | `Def` | `MongoDatasetDef`, `personDatasetDef` |
| Index | `Idx` | `searchResultIdx` |
| Identifier | `Id` | `entityId`, `tenantId`, `eventId` (field suffix) |
| Configuration | `Config` | `config/`, `ServiceConfig` |
| Create, Read, Update, Delete | `Crud` | `ICrud`, `MongoCrud` |
| Data Transfer Object | `DTO` | DTO role; use semantic names such as `CreatePersonRequest` |

**Never**

- Two or more listed abbreviations in one symbol, especially adjacent (`MongoDbIdxDef`).
- Ad-hoc truncations not listed above.
- Abbreviations in service ids or API path segments.
- Ad-hoc entity tokens; `Org` is the only registered shortened entity token.

### Files and source files

- **Directories and filenames:** lowercase; multi-word segments use kebab-case when needed.
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

Feature and fix work uses `{featureBranch}` or `{fixBranch}` names chosen per team convention.

- **Flow branch names:** lowercase single words only; no uppercase, separators, or mixed casing.

