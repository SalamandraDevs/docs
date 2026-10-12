---
title: Zod To MySQL Schema
lastUpdate: 2026-09-08
category: tools
lang: en
status: proposed
---

# Zod To MySQL Schema

## Purpose

Define `zodToMySqlSchema` in `@slmdevs/tools/shared` as the Zod-to-MySQL conversion
boundary used by `DatasetProvisioner`.

The converter receives an entity Zod object schema and produces only the definitions that
belong inside a `CREATE TABLE` statement. `MySqlDataset` owns table-name escaping, statement
assembly, connection use, and execution. The converter does not inspect or alter an existing
database table.

## Current Placeholder

The initial implementation intentionally returns the fixed storage-owned key definition:

```sql
`id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY
```

This preserves the `SchemaConverter<MySqlSchema>` flow while MySQL dataset provisioning is
being connected. It does not translate entity fields and must not be used as the final table
definition for an entity.

## Target Contract

`MySqlSchema` represents a table definition without a table name:

```ts
type MySqlSchema = {
  ddl: string;
};
```

The target converter signature remains:

```ts
function zodToMySqlSchema(schema: z.ZodType): MySqlSchema;
```

For example, an entity schema with `tenantId`, `entityId`, and `createdAt` should produce a
comma-separated DDL fragment such as:

```sql
`id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
`tenantId` VARCHAR(255) NOT NULL,
`entityId` CHAR(36) NOT NULL,
`createdAt` DATETIME(3) NOT NULL
```

`MySqlDataset.create(name, desired)` then constructs the statement using the escaped `name`:

```sql
CREATE TABLE `users` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `tenantId` VARCHAR(255) NOT NULL,
  `entityId` CHAR(36) NOT NULL,
  `createdAt` DATETIME(3) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

The converter must never interpolate a table name. It emits column and table-constraint
fragments only.

## Supported Zod Subset

The first complete implementation supports only the constructs needed by the current entity
schemas:

- Root `z.object(...)` schemas.
- Required and optional properties.
- Nullable properties.
- Strings, including UUID-formatted strings.
- Dates.
- Booleans.
- Integers and finite numbers when a type choice is explicitly supported.
- String enums.
- SQL-representable literal defaults.

The converter rejects unsupported constructs rather than guessing an incompatible storage
representation. This includes unions, intersections, discriminated unions, transforms,
arbitrary refinements, recursive schemas, records, maps, sets, functions, promises, and
custom types.

Arrays and nested objects are also rejected initially. A later design can add explicit JSON
storage semantics, including the distinction between Zod validation and MySQL JSON checks.

## Type Mapping

| Zod input | MySQL type | Notes |
| --- | --- | --- |
| `z.string()` | `VARCHAR(255)` | Default bounded text policy. |
| `z.string().max(n)` | `VARCHAR(n)` | Valid only when `n` is within the chosen limit. |
| `z.uuid()` | `CHAR(36)` | Stores canonical UUID text. |
| `z.date()` | `DATETIME(3)` | Preserves JavaScript millisecond precision. |
| `z.boolean()` | `BOOLEAN` | MySQL aliases this to `TINYINT(1)`. |
| `z.number().int()` | `INT` | Use `BIGINT` only after defining JavaScript handling. |
| `z.number()` | `DOUBLE` | Reject NaN and infinity at validation boundaries. |
| `z.enum([...])` | `VARCHAR(255)` | Add a `CHECK` constraint in the complete implementation. |

The storage-owned `id` is always added first and is not inferred from the entity schema:

```sql
`id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY
```

MySQL `BIGINT` values can exceed JavaScript's safe integer range. `MySqlCrud` treats this id as
an adapter-private value and does not return it as part of backend-neutral entity records.

## Nullability And Defaults

Required, non-nullable properties produce `NOT NULL`. Both `.optional()` and `.nullable()`
produce a nullable MySQL column in the initial mapping. This is a storage decision: MySQL does
not distinguish an omitted value from an explicit `NULL` once data is stored.

The full converter may emit a SQL `DEFAULT` only for literals safely represented in SQL:

- strings, with SQL escaping;
- finite numbers;
- booleans;
- `NULL`; and
- recognized database expressions deliberately represented by converter metadata.

It must reject JavaScript functions and dynamically evaluated Zod defaults. Application-side
defaults are not equivalent to database defaults.

## Conversion Strategy

Use Zod 4's `z.toJSONSchema` as the stable structural conversion pass where it accurately
represents the schema. It provides the root object shape, required fields, string bounds, UUID
format, enum values, and simple defaults.

Use focused Zod node inspection only for JavaScript types that JSON Schema cannot represent,
especially `z.date()`. This matches the existing MongoDB converter. Keep any access to Zod
runtime internals contained in the converter so a future Zod upgrade has one compatibility
surface.

Before emitting DDL, require the converted root to be an object. Read properties in their
declared order, translate each supported property, determine nullability from the original Zod
wrappers and JSON Schema required list, and append the storage-owned `id` before all entity
columns.

## Constraints And Indexes

The converter is responsible for column definitions and intrinsic column constraints, such as
`NOT NULL`, `DEFAULT`, and enum `CHECK` constraints. It does not infer application indexes.

Dataset definitions remain responsible for named, compound, unique, and ordering-sensitive
indexes. `MySqlDataset.createIndex` translates those backend-neutral declarations to MySQL DDL
after the table has been created. This preserves index names and makes pagination indexes
visible at the entity-definition layer.

Foreign keys also remain out of the first implementation. They need explicit relationship
metadata; property names and UUID formats are not enough to safely infer a referenced table,
column, or delete policy.

## Errors

Wrap conversion errors in `CustomError` with the existing
`DB_SCHEMA_CONVERTION_ERROR` name. Include a clear message that identifies the unsupported
Zod construct or property path, and retain the original error as `cause`.

Failing conversion prevents provisioning from creating a table with a guessed or incomplete
shape. Provisioning remains create-only: a changed Zod schema does not modify an existing
MySQL table. Such changes require an explicit migration.

## Implementation Sequence

1. Keep the fixed `id` fragment until `MySqlDataset.create` executes `MySqlSchema.ddl`.
2. Move `MySqlSchema` into `core/src/database/mysql/types.ts` and export it from the MySQL
   module. `tools` may retain a structurally identical return type until package ownership is
   finalized.
3. Add root-object validation and the required scalar mappings.
4. Add literal defaults and enum checks after deciding the repository's MySQL version floor.
5. Add explicit JSON, foreign-key, or migration policies only with corresponding metadata and
   backend-neutral contracts.
