---
title: HTTP Library Roadmap
lastUpdate: 2026-11-18
lang: en
category: planning
status: active
---

# HTTP Library Roadmap

Feature status for `http/` (`@salamandradevs/http`). Paths in `refs` are relative to
`http/`; plan documents under `docs/plans/` are relative to the workspace root.

## HTTP features

Shared Fastify transport: the plugins, hook types, transport schemas, and outbound client
that `auth` and `gateway` register and consume.

- [implemented] HTTP-001 Errors plugin (`http-errors`): maps `CustomError` names to RFC 9457
  problem responses so every boundary reports a stable, documented body. (refs:
  `src/plugins/errors.ts`, `src/schemas/problems.ts`,
  `docs/plans/261112-http-package-plan.md`)
- [implemented] HTTP-002 Security plugin (`http-security`): secure transport defaults and an
  `onRoute` guard that requires exactly one rate-limit tier per route, with counters in the
  injected `options.store` (in-memory by default, Redis when shared). (refs:
  `src/plugins/security.ts`, `src/schemas/rate-limit.ts`,
  `docs/plans/261112-http-package-plan.md`)
- [implemented] HTTP-003 Authentication plugin (`http-authentication`): verifies the bearer
  token and decorates `authenticateRequest` and `request.sessionGrant`. (refs:
  `src/plugins/authentication.ts`, `docs/plans/261112-http-package-plan.md`)
- [implemented] HTTP-004 Authorization plugin (`http-authorization`): decorates
  `requireAdminRole` and `requireDefaultTenantSession`. (refs:
  `src/plugins/authorization.ts`, `docs/plans/261112-http-package-plan.md`)
- [implemented] HTTP-005 Execution-context plugin (`http-execution-context`): binds the
  per-request `ExecutionContext` to the async scope and decorates `bindExecutionContext`.
  (refs: `src/plugins/execution-context.ts`, `docs/plans/261112-http-package-plan.md`)
- [implemented] HTTP-006 Record cursor codec plugin (`http-record-cursor-codec`): decorates
  `request.decodeCursor` and `reply.encodeCursor` for opaque pagination cursors. (refs:
  `src/plugins/record-cursor-codec.ts`, `docs/plans/261112-http-package-plan.md`)
- [implemented] HTTP-007 Transport schemas: `problemDetailsSchema`, `problemResponses`,
  `paginationQuerySchema`, `createPageSchema`, `recordParamsSchema`, and
  `rateLimitPolicySchema`, with their inferred types. (refs: `src/schemas/`,
  `docs/plans/261112-http-package-plan.md`)
- [implemented] HTTP-008 Hook-array types: `PreHandlerHooks` and `OnRequestHooks`, which the
  service composers specialize into their route chains. (refs: `src/hooks/`,
  `docs/plans/261112-http-package-plan.md`)
- [implemented] HTTP-009 Outbound client `makeApiRequest`: the shared fetch helper for
  service-to-service calls, forwarding the bound request id as `x-request-id` when the
  caller passes its `executionCtx` provider, and parsing problem responses against
  `problemDetailsSchema`.
  Moved here from `tools` (formerly TOOLS-002). (refs: `src/client/`,
  `docs/plans/261112-http-package-plan.md`)
- [proposed] HTTP-010 Contract-seam test suite: the package currently ships without tests
  (decision 14), so the transport plugins, schemas, and client are exercised only through
  their consumers. Add a package test suite once the platform test strategy lands. (refs:
  `docs/plans/261029-tests-suite-plan.md`, `docs/plans/261112-http-package-plan.md`)
