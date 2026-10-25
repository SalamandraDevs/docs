---
title: Core Library Roadmap
lastUpdate: 2026-10-24
lang: en
category: planning
status: active
---

# Core Library Roadmap

Feature status for `core/` (`@slmdevs/core`). Line format, status tags, and editing rules
are defined in the platform [ROADMAP](../../ROADMAP.md#roadmap-rules). Paths in `refs` are
relative to `core/`.

## CORE features

Shared record and database foundation: backend-neutral contracts, adapters, tenant-aware
record handling, and request execution context.

- [implemented] CORE-001 Database contracts: connection lifecycle, CRUD, and dataset
  contracts. (refs: `src/database/contracts.ts`)
- [implemented] CORE-002 MongoDB adapter: connection, CRUD, dataset provisioning.
  (refs: `src/database/mongo/`)
- [implemented] CORE-003 MySQL adapter: connection, CRUD, dataset provisioning.
  (refs: `src/database/mysql/`)
- [implemented] CORE-004 Dataset provisioner: creates datasets and indexes from Zod-based
  definitions. (refs: `src/database/dataset-provisioner.ts`)
- [implemented] CORE-005 `RecordsHandler`: tenant scope, `recordId`, audit metadata, and
  deterministic cursor pagination ordered by `createdAt` then `recordId`, both descending.
  (refs: `src/records/records-handler.ts`)
- [implemented] CORE-006 Execution context: AsyncLocalStorage carrying actor, tenant, and
  request id. (refs: `src/utils/context.ts`)
- [implemented] CORE-008 Database provider and backend-bound adapters: `DatabaseProvider`
  (`IDatabaseProvider`) owns backend selection and connection lifecycle, exposing a
  discriminated `DatabaseHandle`, and `DatabaseAdapter` selects the backend once to
  `provision` dataset definitions (via `IDatasetProvisioner`) and `createCrud` `ICrud`
  adapters by name.
  (refs: `src/infra/database/database-provider.ts`,
  `src/infra/database/database-adapter.ts`)
- [proposed] CORE-007 MySQL schema conversion: depends on
  [TOOLS-005](./TOOLS-ROADMAP.md#tools-features).
