---
title: Tools Library Roadmap
lastUpdate: 2026-11-18
lang: en
category: planning
status: active
---

# Tools Library Roadmap

Feature status for `tools/` (`@salamandradevs/tools`). Paths in `refs` are relative to
`tools/`; plan documents under `docs/plans/` are relative to the workspace root.

## TOOLS features

Small shared helpers with no application policy. Exports `@salamandradevs/tools/shared`
(browser safe) and `@salamandradevs/tools/node` (Node only).

- [implemented] TOOLS-001 `CustomError` and error names. (refs: `src/shared/errors.ts`)
- [implemented] TOOLS-003 Type guards. (refs: `src/shared/typeguards.ts`)
- [implemented] TOOLS-004 `zodToMongoSchema`. (refs: `src/shared/transform.ts`)
- [implemented] TOOLS-006 JWT signing input and timing-safe hash comparison.
  (refs: `src/node/crypto.ts`)
- [implemented] TOOLS-007 `loadEnvVariables`: reads a config schema's `.meta({ env })`
  fields from `process.env`, validates each with its own field schema, and returns the
  resolved values without touching the config object, throwing `ENV_VAR_ERROR` on a missing
  or malformed variable. (refs: `src/node/environment.ts`)
- [proposed] TOOLS-005 `zodToMySqlSchema` real conversion: the current function is a
  placeholder. (refs: `docs/plans/zod-to-mysql-schema.md`)
- [implemented] TOOLS-008 `RATE_LIMIT_EXCEEDED` error name: adds the classification the auth
  rate limiter uses for its 429 response. (refs: `src/shared/errors.ts`,
  `docs/plans/261031-rate-limit-plan.md`)
- [proposed] TOOLS-009 Dataset maintenance tools: programmatic helpers to prune database and
  Redis records (such as expired API key records and stale cache entries), invoked
  explicitly by each service's cron module rather than by automatic store-side policies such
  as TTL indexes. Added once the services run together. (refs: pending; consumed by
  per-service cron modules)
