---
title: Tools Library Roadmap
lastUpdate: 2026-10-24
lang: en
category: planning
status: active
---

# Tools Library Roadmap

Feature status for `tools/` (`@slmdevs/tools`). Line format, status tags, and editing rules
are defined in the platform [ROADMAP](../../ROADMAP.md#roadmap-rules). Paths in `refs` are
relative to `tools/`.

## TOOLS features

Small shared helpers with no application policy. Exports `@slmdevs/tools/shared` (browser
safe) and `@slmdevs/tools/node` (Node only).

- [implemented] TOOLS-001 `CustomError` and error names. (refs: `src/shared/errors.ts`)
- [implemented] TOOLS-002 `makeApiRequest` fetch helper. (refs: `src/shared/requests.ts`)
- [implemented] TOOLS-003 Type guards. (refs: `src/shared/typeguards.ts`)
- [implemented] TOOLS-004 `zodToMongoSchema`. (refs: `src/shared/transform.ts`)
- [implemented] TOOLS-006 JWT signing input and timing-safe hash comparison.
  (refs: `src/node/crypto.ts`)
- [implemented] TOOLS-007 `loadEnvVariables`: reads a config schema's `.meta({ env })`
  fields from `process.env`, validates each with its own field schema, and returns the
  resolved values without touching the config object, throwing `ENV_VAR_ERROR` on a missing
  or malformed variable. (refs: `src/node/environment.ts`)
- [proposed] TOOLS-005 `zodToMySqlSchema` real conversion: the current function is a
  placeholder. (refs: `docs/zod-to-mysql-schema.md`)
