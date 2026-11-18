---
title: Core Library Roadmap
lastUpdate: 2026-11-18
lang: en
category: planning
status: active
---

# Core Library Roadmap

Feature status for `core/` (`@salamandradevs/core`). Paths in `refs` are relative to
`core/`; plan documents under `docs/plans/` are relative to the workspace root.

## CORE features

Shared record and database foundation: backend-neutral contracts, adapters, tenant-aware
record handling, and request execution context.

- [implemented] CORE-001 Database contracts: connection lifecycle, CRUD, and dataset
  contracts. (refs: `src/infra/database/contracts.ts`)
- [implemented] CORE-002 MongoDB adapter: connection, CRUD, dataset provisioning.
  (refs: `src/infra/database/mongo/`)
- [implemented] CORE-003 MySQL adapter: connection, CRUD, dataset provisioning.
  (refs: `src/infra/database/mysql/`)
- [implemented] CORE-004 Dataset provisioner: creates datasets and indexes from Zod-based
  definitions. (refs: `src/infra/database/dataset-provisioner.ts`)
- [implemented] CORE-005 `RecordsHandler`: tenant scope, `recordId`, audit metadata, and
  deterministic cursor pagination ordered by `createdAt` then `recordId`, both descending.
  (refs: `src/storage/records-handler.ts`)
- [implemented] CORE-006 Execution context: the `IExecutionCtxProvider` contract and the
  `ExecutionCtxProvider` implementation over a service-owned `AsyncLocalStorage`, carrying
  actor, tenant, and request id; injected into `RecordsHandler`, with no module-level
  instance. (refs: `src/runtime/`)
- [implemented] CORE-008 Database provider and backend-bound adapters: `DatabaseProvider`
  (`IDatabaseProvider`) owns backend selection and connection lifecycle, exposing a
  discriminated `DatabaseHandle`, and `DatabaseAdapter` selects the backend once to
  `provision` dataset definitions (via `IDatasetProvisioner`) and `createCrud` `ICrud`
  adapters by name.
  (refs: `src/infra/database/database-provider.ts`,
  `src/infra/database/database-adapter.ts`)
- [proposed] CORE-007 MySQL schema conversion: depends on
  [TOOLS-005](./TOOLS-ROADMAP.md#tools-features).
- [implemented] CORE-009 Keys providers: `ISigningKeyProvider`, `IVerificationKeyProvider`,
  and `IKeysProvider`; the Jose, OpenBao, and remote-JWKS providers; the signing lifecycle
  schemas; and `createKeysProvider`. Moved from `services/auth` so services share one
  implementation, each with its own keys.
  (refs: `src/keys/`, `docs/plans/261106-gateway-plan.md`)
- [implemented] CORE-010 Access-token verification: `IAccessTokenVerifier`,
  `AccessTokenVerifier`, and `accessTokenClaimsSchema`, so the issuer and every verifier
  share one claim definition and one verification flow.
  (refs: `src/tokens/`, `docs/plans/261106-gateway-plan.md`)
- [implemented] CORE-011 Unpaged cross-tenant find-all: `ICrud.findAll` and
  `RecordsHandler.findAll` return every matching record without pagination, deliberately
  bypassing tenant scope and the execution context for process-wide scans such as the
  gateway's revocation rebuild; `FindAllOptions` carries a `number | null` cap, optional
  order, and backend-neutral `greaterThan`/`lessThan` bounds translated per adapter, and
  `assertFindAllFilter` rejects an unfiltered scan.
  (refs: `src/infra/database/contracts.ts`, `src/infra/database/mongo/mongo-crud.ts`,
  `src/infra/database/mysql/mysql-crud.ts`, `src/storage/records-handler.ts`,
  `src/infra/database/utils.ts`)
- [implemented] CORE-012 API key verification: `IApiKeyVerifier`, `ApiKeyVerifier`,
  `apiKeyPlanSchema`, `apiKeyVerifyConfigSchema`, and `apiKeyPayloadSchema`, so the gateway
  issuer and every verifier share one key format, payload schema, and verification flow;
  the verifier reads the prefix and length ceiling from configuration and makes no
  revocation decision.
  (refs: `src/tokens/api-key-verifier.ts`,
  `docs/plans/261111-api-key-verification-plan.md`)
