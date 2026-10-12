---
title: Package Publishing and Consumption
lastUpdate: 2026-10-12
lang: en
category: runbook
status: active
owner: haldrik
---

## Purpose

How Salamandra Devs shared libraries are published to npm and consumed by other
packages, services, and apps. It records the conventions each publishable package must
follow, the release pipeline, the install rules for consumers, the versioning policy, and
the gotchas learned while migrating the `@salamandradevs` scope onto npm.

The reference packages are `@salamandradevs/tools` and `@salamandradevs/core`. The first
real consumers are `services/auth` (installs both) and the `yesenia` site.

## Table of contents

- [Scope and ownership](#scope-and-ownership)
- [Rules for a publishable package](#rules-for-a-publishable-package)
- [Publishing pipeline](#publishing-pipeline)
- [Consuming published packages](#consuming-published-packages)
- [Versioning (semver) policy](#versioning-semver-policy)
- [Release checklist](#release-checklist)
- [Gotchas and lessons learned](#gotchas-and-lessons-learned)

## Scope and ownership

- All shared packages publish under the **`@salamandradevs`** npm scope, which mirrors the
  `SalamandraDevs` GitHub org.
- The scope must be an npm org **we own**. We originally used `@slmdevs`, but that scope is
  already owned by an unrelated account (`@slmdevs/celeraone`), so publishing there fails.
  Verify scope ownership before naming a package.
- Packages are **public**. Public scope + public GitHub repo is also what enables build
  provenance.

## Rules for a publishable package

Each package is its own GitHub repo under `SalamandraDevs/` (they are not a single
workspace; `~/dev/slmdevs` is just a parent folder). A publishable `package.json` carries:

- `"name"`: `@salamandradevs/<pkg>`.
- `"publishConfig": { "access": "public", "provenance": true }` — `access: public` is
  mandatory for scoped packages; `provenance: true` records a signed build attestation.
- `"repository"`, `"homepage"`, `"bugs"` pointing at the exact GitHub repo. **`repository.url`
  must match the repo exactly** or provenance is rejected.
- `"files": ["dist", "types"]` — only the build output ships.
- `"exports"`: explicit subpaths only; do not declare a subpath that has no built output.
  For example `@salamandradevs/tools` exposes `./browser`, `./node`, `./shared` and has **no
  root (`.`) export** — consumers import the specific entry they need.
- `"prepublishOnly": "pnpm run build"` — guarantees a fresh build before packing.
- Build is `tsc --build`, emitting `dist/` (JS) and `types/` (`.d.ts`).

**Built artifacts are committed.** `dist/` and `types/` are intentionally **not** gitignored.
This lets a developer copy a dependency's built output straight into another package's
`node_modules` during local work, avoiding a publish-and-reinstall round-trip.

## Publishing pipeline

Publishing is tokenful but staged, so no version goes live without a human. Each repo has
`.github/workflows/release.yml`:

1. **Trigger:** push a semver tag, e.g. `git tag v0.1.0 && git push origin v0.1.0`.
2. **Build:** checkout, `pnpm install`, `pnpm run build`. The job installs `npm@latest`
   because staged publishing needs **npm >= 11.15.0** (and Node >= 22.14).
3. **Guard:** refuse to proceed if the version is already on npm.
4. **Stage:** `npm stage publish`, authenticated by the `NPM_TOKEN` secret. This does **not**
   make the version live — it parks the release in npm's staging queue.
5. **Provenance:** attached automatically because the workflow has `permissions: id-token:
   write` and the repo + package are public. No `--provenance` flag is needed.
6. **Approve (manual, human):** a maintainer promotes the staged release with 2FA. The web
   UI + passkey works; the `npm stage approve` CLI path does not when the second factor is a
   passkey. Approve from the package page on npmjs.com.
7. **Verify:** `npm view @salamandradevs/<pkg> version` returns the new version.

### The `NPM_TOKEN` secret

- It is a **granular npm access token** scoped to `@salamandradevs` with **write (staged)**
  permission, created at npmjs.com, then stored per repo as the GitHub Actions secret
  `NPM_TOKEN` (`gh secret set NPM_TOKEN -R SalamandraDevs/<repo>`).
- It is an npm credential, not a GitHub one. `secrets.GITHUB_TOKEN` cannot publish to npm.
- npm caps granular write tokens at ~90 days, so expect to rotate it.

### Dependency order

Publish bottom-up: a package must be live on npm before anything that depends on it can
build in CI. Current order: **`tools` → `core` → consumers** (`core` depends on `tools`;
`auth` depends on both).

## Consuming published packages

- Depend on the **npm version range** (`^0.1.0`), never a `file:../...` path. `file:` paths
  only resolve inside the monorepo layout and break any off-monorepo CI (this is exactly
  what blocked Netlify from building `yesenia`). `core` consumes `@salamandradevs/tools`
  this way, and `services/auth` consumes both `core` and `tools` this way.
- **Local development** against unpublished changes: build the dependency and copy its
  `dist/`/`types/` into the consumer's `node_modules/@salamandradevs/<pkg>/`. This is why
  the artifacts are committed — no version bump or publish is required to try a change.
- **When a consumer needs only a sliver**, prefer localizing over taking the dependency.
  `yesenia` used only the Winston logger from `tools`, so the logger was copied into the app
  (`src/lib/globals/logger.ts`) and the dependency dropped entirely — the site now installs
  from its own repo with zero private-scope deps.

## Versioning (semver) policy

We follow [semver](https://semver.org). The version in `package.json` is the source of
truth; the release is cut by tagging `v<version>`.

- **Pre-1.0 (`0.y.z`), where we are now:** the API is still settling. Bump **minor** (`0.1.0
  → 0.2.0`) for any breaking change or notable feature, and **patch** (`0.1.0 → 0.1.1`) for
  backward-compatible fixes. Consumers should pin with `^0.y.z`, understanding that under
  0.x a caret only allows patch updates.
- **1.0 and after:** **major** for breaking changes, **minor** for backward-compatible
  additions, **patch** for backward-compatible fixes.
- **Release = tag.** Bump the version in `package.json`, commit, then `git tag vX.Y.Z` and
  push the tag. The guard step blocks a tag whose version is already published, so bump
  before tagging.
- Never republish a version. If a staged release is wrong, reject it and cut the next patch.

## Release checklist

1. Land the change on `main`; make sure `pnpm run build` and lint pass.
2. Bump `version` in `package.json` per the policy above; commit.
3. `git tag vX.Y.Z && git push origin main --tags`.
4. Watch the `Release` workflow; it should end at "staged with id ...".
5. Approve the staged release on npmjs.com with 2FA.
6. `npm view @salamandradevs/<pkg> version` to confirm it is live.
7. Publish dependents only after their dependencies are live.

## Gotchas and lessons learned

- **Scope ownership is checked at publish, opaquely.** Staging to a scope you do not own
  fails with `400 Bad Request ... Version uniqueness check failed unexpectedly` — not an
  obvious "forbidden". If you see it, confirm the org owns the scope.
- **Provenance preconditions:** public repo, public package, `id-token: write`, and
  `repository.url` matching the repo exactly. Miss any and provenance silently will not
  attach (or the publish errors).
- **Token restriction notice:** npm is restricting tokens that bypass 2FA for direct
  publishing — staged publishing (CI stages, human approves) is the supported path.
- **Framework-matched adapters/plugins:** pin versions to the consumer's framework major.
  `yesenia` on SvelteKit 2 needs `@sveltejs/adapter-netlify@^6` (7.x targets kit 3 and
  fails with `builder.config.files` undefined). `latest` is not always the right pick.
- **Pre-push hooks rewrite files.** These repos run `pnpm format && pnpm lint` on
  `pre-push`, which edits the tree *after* your commit, leaving it dirty and the pushed
  commit unformatted. Run format + lint and commit before pushing so the hook is a no-op.
