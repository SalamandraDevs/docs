---
title: Roadmap Rules
lastUpdate: 2026-10-25
lang: en
category: planning
status: active
---

# Roadmap Rules

These rules apply to every `{REPO}-ROADMAP.md` file in this directory (`AUTH`, `CORE`,
`TOOLS`).

## Line format

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

## Status tags

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

## Scope and paths

- Scope: `core/`, `tools/`, and `services/auth/`, each with its own roadmap file in this
  directory.
- Paths are relative to each repository root. Route paths are relative to the auth service
  root, before any gateway prefix.

## Editing rules

- Change a status only after checking the code; cite the file or plan in `refs`.
- Never reuse or renumber an ID. Add new IDs at the end of the repository's roadmap file.
- When synchronizing the status of implemented features, update `lastUpdate` in every
  roadmap file you change to reflect the current snapshot.
