---
title: Salamandra Devs Platform Roadmap
lastUpdate: 2026-10-25
lang: en
category: planning
status: active
---

# Salamandra Devs Platform Roadmap

Brief map of what each repository does and the status of its features. Verified against the
last update date source code; where a plan document disagrees with the code, the code wins
and the disagreement is listed under [Known discrepancies](#known-discrepancies).

This file holds the roadmap rules, the repository index, and the sections that do not have
their own file yet. CORE, TOOLS, and AUTH features live in `en/roadmap/`.

## Roadmap rules

These rules apply to this file and to every `en/roadmap/{REPO}-ROADMAP.md` file.

### Line format

Every feature is one line with a stable ID, so people and agents can grep and cite it:

```text
- [status] ID Name: what it does. (refs: source files or plan documents)
```

Example:

```markdown
- [implemented] AUTH-004 Sessions: issue, read, terminate, and refresh with rotating opaque
  refresh tokens stored as HMAC hashes. (refs: `docs/session-plan.md`,
  `src/application/sessions-service/`)
```

### Status tags

These tags mark individual features inline in roadmaps and planning documents; they are not
frontmatter `status` values.

- `implemented`: feature is complete and present in the code on the main working tree.
  Behavior is verified and available for use. The implementing plan document, when one
  exists, is listed among the item's `(refs: ...)` alongside the code paths.
- `planned`: feature has an approved plan document and is ready for implementation. Design
  review is complete; implementation order is decided.
- `proposed`: feature is inferred, under discussion, or designed but not yet approved. A
  design document may exist and be open for feedback; implementation order is not decided.
- `canceled`: feature was rejected or indefinitely deferred. Recorded for context but
  requires no implementation work.

### Scope and paths

- Scope: only `core/`, `tools/`, `services/auth`, `services/profiles`, `infra/` and `docs/`.
- Paths are relative to each repository root. Route paths are relative to the auth service
  root, before any gateway prefix.

### Editing rules

- Change a status only after checking the code; cite the file or plan in `refs`.
- Never reuse or renumber an ID. Add new IDs at the end of the repository's section or
  roadmap file.
- When synchronizing the status of implemented features, update `lastUpdate` in every
  roadmap file you change to reflect the current snapshot.

## Repositories

| ID | Path | Package | Role | Depends on | Roadmap |
| --- | --- | --- | --- | --- | --- |
| CORE | `core/` | `@slmdevs/core` | Backend-neutral records layer | TOOLS | [CORE-ROADMAP](./en/roadmap/CORE-ROADMAP.md) |
| TOOLS | `tools/` | `@slmdevs/tools` | Shared errors, guards, crypto | none | [TOOLS-ROADMAP](./en/roadmap/TOOLS-ROADMAP.md) |
| AUTH | `services/auth/` | `@slmdevs/auth` | Authentication service | CORE, TOOLS | [AUTH-ROADMAP](./en/roadmap/AUTH-ROADMAP.md) |
| PROFILES | `services/profiles/` | `@slmdevs/profiles` | Identity and entitlements | CORE, TOOLS | [below](#profiles-slmdevsprofiles) |
| INFRA | `infra/` | none | Shared deployment stack; network, MongoDB, OpenBao, operator tooling | none | [below](#infra-infra) |
| DOCS | `docs/` | none | Documentation and plans | none | [below](#docs-docs) |

Dependency direction: services depend on `core` and `tools`; `core` depends on `tools`.
Architecture rules (ports and adapters, SOLID, naming) live in `docs/en/architecture/`.

## PROFILES: `@slmdevs/profiles`

Planned identity and entitlements service (person and organization profiles, capabilities).
It starts after the auth service works standalone. Only the persistence scaffold exists.

- [implemented] PROFILES-001 Records and datasets: `persons`, `organizations`, and
  `outbox` with schemas, and dataset provisioning at startup. Uncommitted at the time of
  writing. (refs: `src/entities/`, `src/index.ts`)
- [proposed] PROFILES-002 Service runtime: configuration loading and an HTTP API following
  the auth layering. Nothing exists yet.
- [proposed] PROFILES-003 Person-to-user linking: `Person.userId` refers to an auth user.
  (refs: `src/entities/persons/schema.ts`)
- [proposed] PROFILES-004 Capabilities and entitlements: what an identity may do, evaluated
  per tenant and organization. Inferred; no design yet.
- [proposed] PROFILES-005 Outbox publisher: deliver `outbox` events, for example
  `profile.person.updated`. (refs: `docs/es/arquitectura/convenciones-nombres.md`)
- [proposed] PROFILES-006 First internal OIDC provider for auth. (refs: `docs/auth-plan.md`)

## INFRA: `infra/`

Shared deployment stack that platform services depend on. It owns the Docker `services`
network, the backing MongoDB and OpenBao instances, and the signing-key operator tooling.
Services join the external `services` network and never provision this infrastructure
themselves. Its own git repository, initialized without remotes.

- [implemented] INFRA-001 Shared stack compose: `docker-compose.yml` (production) and
  `docker-compose.dev.yml` (development) create the `services` network and run MongoDB and
  OpenBao (file storage, Shamir seal, config in `openbao/openbao.hcl`). Production enables
  MongoDB authentication and expects a real OpenBao auto-unseal; development runs MongoDB
  unauthenticated. (refs: `infra/docker-compose.yml`, `infra/docker-compose.dev.yml`,
  `infra/openbao/openbao.hcl`, `infra/README.md`)
- [implemented] INFRA-002 Signing-key operator tooling: the OpenBao lifecycle scripts
  (`unseal.sh`, `rotate.sh` and its steps) and the jose dev-keyfile provisioner
  (`provision.sh`, which copies the encrypted keyfile into the auth service so local dev
  reads it by default) are owned here, not by the auth service, which only consumes signing
  material. (refs: `infra/operator/openbao/`, `infra/operator/jose/provision.sh`, AUTH-023)

## DOCS: `docs/`

- [implemented] DOCS-001 Architecture guides in English and Spanish. (refs:
  `en/architecture/`, `es/arquitectura/`)
- [implemented] DOCS-002 Plan documents for auth. (refs: `auth-plan.md`, `session-plan.md`,
  `signing-plan.md`, `users-service-plan.md`, `entity-cursor-plugin.md`, `tests-suite.md`)
- [implemented] DOCS-003 Front matter and status conventions. (refs:
  `en/house/documents-style-guide.md`)
- [implemented] DOCS-005 TypeDoc API documentation: TSDoc `/** */` source comments rendered
  to a per-package HTML site via `pnpm docs`, wired for `core`, `tools`, and `services/auth`
  (auth uses an expand entry-point strategy over `src`), with the source-comment convention
  codified in `en/house/documents-style-guide.md`. Output is local and git-ignored;
  publishing/hosting is not yet set up. (refs: `en/house/documents-style-guide.md`,
  `core/typedoc.json`, `tools/typedoc.json`, `services/auth/typedoc.json`)
- [proposed] DOCS-004 Reconcile stale plan documents with the code. (refs: see below)
- [implemented] DOCS-006 House rules moved out of `CLAUDE.md`: code style, documentation
  and TSDoc rules, and API hook rules now live in `en/house/code-style.md`,
  `en/house/documents-style-guide.md`, and `en/architecture/api-hooks.md`, which
  `CLAUDE.md` imports. The API hook rules later moved into `en/house/code-style.md`.
  (refs: `en/house/`)
- [implemented] DOCS-007 Per-repository roadmaps: CORE, TOOLS, and AUTH features moved to
  `en/roadmap/{REPO}-ROADMAP.md`, with the roadmap rules kept in this file. (refs:
  `en/roadmap/`)
- [implemented] DOCS-008 Document IDs: every section of the SOLID, design-pattern,
  good-practice, code-style, and documents-style documents carries a stable `XX-NN` ID
  (`SP`, `DP`, `GP`, `CS`, `DS`), defined in the naming conventions. (refs:
  `en/house/naming-conventions.md`)

## Known discrepancies

Read these before trusting a plan document.

- `docs/entity-cursor-plugin.md` and parts of `docs/session-plan.md` say `entityId` and
  `Entities`; the code uses `recordId` and `RecordsHandler`.
- Early plan documents and comments mention `userId`; as of 2026-09-24 the code uses
  `recordId` consistently throughout transport, route, and application layers
  (`/v1/users/:recordId`).
- `docs/es/arquitectura/convenciones-nombres.md` names the service `profile`, the package
  `@slmdevs/profile`, and the resource `Org` with singular route segments such as
  `/auth/v1/user`. The repository uses `services/profiles`, `@slmdevs/profiles`,
  `organizations`, and plural routes such as `/v1/users`. The naming document is the stated
  authority, so the code is the side that would change.
- `docs/session-plan.md` refers to `multi-tenant-auth-design.md`, which is not in `docs/`.
