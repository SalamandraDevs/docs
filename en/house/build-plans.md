---
title: Build Plans
lastUpdate: 2026-10-29
lang: en
category: guide
status: active
---

## Purpose

House rules for writing build plans: the documents that describe a change before it is
implemented, so a reviewer can approve the approach and an executing agent can carry it
out without guessing. These rules apply to every contributor, human or agent, and to every
plan under `docs/plans/`.

A plan is grounded in the architecture and house documents. It does not restate them; it
cites them, applies them, and records any deliberate deviation:

- Architecture: [SOLID Principles](../architecture/solid-principles.md) (`SP`),
  [Design Patterns](../architecture/design-patterns.md) (`DP`), and
  [Development Good Practices](../architecture/development-good-practices.md) (`GP`).
- House: [Code Style](./code-style.md) (`CS`),
  [Documents Style Guide](./documents-style-guide.md) (`DS`), and
  [Naming Conventions](./naming-conventions.md).

A plan is a Markdown document and follows the
[Documents Style Guide](./documents-style-guide.md) in full; this document adds only what
is specific to plans.

Each rule section carries a stable ID (`BP-NN`) for citation; IDs follow
[Document IDs](./naming-conventions.md#document-ids).

## Writing the Document

### BP-01 Location, name, and status

- Save plans in `docs/plans/` as `{DDMMYY}-{topic}-plan.md`, lowercase kebab-case
  (`291026-messages-plan.md`, `301026-session-plan.md`).
- A new plan uses `status: proposed`; move it to `published` once the change is
  implemented, and to `archived` when it no longer guides work.
- Follow the frontmatter with one `#` heading that repeats the `title` value.

### BP-02 Required sections

Every plan has these top-level sections, in this order:

1. `## Current Status` with a `### What's the Problem` subsection
   ([BP-04 Current Status][bp-04]).
2. `## Goal`, with a `### Proposals` subsection ([BP-05 Goal and proposals][bp-05]).
3. `## Implementation Details` ([BP-06 Implementation Details][bp-06]).
4. `## Acceptance Criteria` ([BP-07 Acceptance Criteria][bp-07]).
5. `## Out of Scope` ([BP-08 Out of Scope][bp-08]).
6. `## Follow-ups` ([BP-09 Follow-ups][bp-09]).

Keep the section names exactly as written so plans read alike and can be searched. Add
subsections freely inside a section; do not add top-level sections between them. A
section that has nothing to say still appears, with one line stating so.

[bp-04]: #bp-04-current-status
[bp-05]: #bp-05-goal-and-proposals
[bp-06]: #bp-06-implementation-details
[bp-07]: #bp-07-acceptance-criteria
[bp-08]: #bp-08-out-of-scope
[bp-09]: #bp-09-follow-ups

### BP-03 Citing architecture and house rules

A plan cites every architecture and house rule it relies on, so a reviewer can check each
claim against its source without guessing which rule is meant, and every citation is a
link that opens the rule.

- **First citation**: the rule's ID and title, exactly as its section heading reads,
  written as a reference-style link whose label is the ID in lowercase:
  `[GP-03 Contracts at boundaries, wiring at the composition root][gp-03]`.
- **Definition**: the label's target, written once, at the end of the section where the
  rule is first cited. The path is relative to the plan file:
  `[gp-03]: ../en/architecture/development-good-practices.md#gp-03-...`.
- **Later citations**: the ID alone, as a shortcut link: `[GP-03]`. Labels match without
  regard to case, so it resolves to the same definition. Never write an ID without its
  link.
- Separate several citations with semicolons, since some titles contain commas:
  `[GP-03]; [DP-01 Factory][dp-01]`.
- Cite a rule without an ID by document and section name, linked to that section:
  `[Naming Conventions, Symbol conventions](../en/house/naming-conventions.md#...)`.

A plan that cites GP-03 twice:

```markdown
### What's the Problem

- The providers read a global config at module level; this breaks
  [GP-03 Contracts at boundaries, wiring at the composition root][gp-03].

[gp-03]: ../en/architecture/development-good-practices.md#gp-03-contracts-at-boundaries-wiring-at-the-composition-root

## Goal

### Proposals

1. **Senders are built in the composition root**: `bootstrapMessaging` creates them.
   - Rules: [GP-03]; [DP-08 Composition Root][dp-08].

[dp-08]: ../en/architecture/design-patterns.md#dp-08-composition-root
```

Where citations are required:

- In `### What's the Problem`, next to every behavior of the current code that breaks a
  rule ([BP-04]).
- In every proposal, on its `Rules:` line ([BP-05]).
- In every proposal that breaks a rule, on a `Deviation:` line, and again under
  Follow-ups ([BP-09]).
- Inline in Implementation Details and Acceptance Criteria, wherever a step or check
  applies a rule.

## Plan Sections

### BP-04 Current Status

Describes what is in place and how it works today, before any change.

- State facts verified by reading the code, configuration, and deployment files, not
  assumptions. Reference files as `path/to/file.ts:line`.
- Cover the parts the change will touch: modules and their contracts, data and records,
  routes, configuration and environment variables, deployment manifests, and any legacy
  code being replaced.
- Name the existing patterns the change must follow, each with the file that shows it,
  cited as [BP-03 Citing architecture and house rules][bp-03] requires; for example,
  "selection follows [DP-01 Factory][dp-01] as in `auth/src/infra/keys/factory.ts`".
- Make no proposals here.

`### What's the Problem` states why the current situation must change:

- The defect, gap, or new need, with the evidence (a failing path, a missing capability,
  a roadmap entry such as AUTH-011).
- Every architecture or house rule the current code breaks, cited next to the behavior
  that breaks it, as [BP-03] requires. For example: "the providers read a global config
  at module level; this breaks
  [GP-03 Contracts at boundaries, wiring at the composition root][gp-03]".
- Who or what is affected, and the cost of leaving it as it is.

[bp-03]: #bp-03-citing-architecture-and-house-rules
[dp-01]: ../architecture/design-patterns.md#dp-01-factory
[gp-03]: ../architecture/development-good-practices.md#gp-03-contracts-at-boundaries-wiring-at-the-composition-root

### BP-05 Goal and proposals

`## Goal` opens with a short paragraph: the outcome the plan delivers and its boundary.

`### Proposals` is a numbered list. Each proposal is one decision:

- Start with a bold lead term and a colon, then the decision and its reason:
  `1. **Contracts live in the transport package**: ...`.
- End with a `Rules:` line listing every architecture and house rule the proposal
  applies, cited as [BP-03] requires:

  ```markdown
  1. **Contracts live in the transport package**: decision and reason.
     - Rules: [SP-05 Dependency Inversion Principle (DIP)][sp-05]; [GP-02];
       [DP-02 Adapter][dp-02].
  ```

- Record decisions the human already made as proposals, so the plan is the single record
  of them. Mention a rejected alternative in one sentence when it explains the choice.
- When a proposal must break a rule, add a `Deviation:` line citing the rule and giving
  the reason, and list it again under Follow-ups for approval. Never break a rule
  silently.
- Number proposals once and never renumber them; later sections cite them as "proposal
  3".

### BP-06 Implementation Details

The section an executing agent follows step by step. It may be split into stages
(`### Stage 1: senders inside auth`, `### Stage 2: extraction into packages`), each with
ordered steps (`#### Step 1: dependencies`).

Include what the change needs, in this order when it applies:

- **Rules for the executing agent**: which documents to read, which existing files to
  mirror, import boundaries, which package scripts to run, when to stop and ask.
- **File layout**: a directory tree of new and changed files, with the main symbol of
  each file. Names follow [Naming Conventions](./naming-conventions.md).
- **API layout**: each route as method, path, route def, hooks, request and response
  schemas, and problem responses. Paths follow
  [APIs](./naming-conventions.md#apis); hooks follow [CS-09 Shared hooks][cs-09] to
  [CS-12 Typed route-local hooks][cs-12].
- **Code examples**: enough to remove guesswork, starting each block with a comment
  naming its target file. Examples are code the agent will copy, so they must already
  obey the house rules: [CS-01 Imports][cs-01];
  [CS-02 Zod schemas and derived types][cs-02]; [CS-04 Errors][cs-04];
  [CS-05 Files and modules][cs-05]; [CS-06 Blank lines][cs-06]; and TSDoc from
  [DS-09 Doc comments and line comments][ds-09] to
  [DS-13 Cross-references with `{@link}`][ds-13]. Show one full example per pattern and
  excerpts for its repetitions.
- **Configuration and environment**: schema changes, configuration file blocks with
  `READ FROM ENV ...` placeholders, new environment variables, and every deployment
  manifest that must pass them ([GP-08 Validated configuration, fail fast][gp-08];
  [GP-09 Secrets and credentials only from the environment][gp-09];
  [GP-10 Configuration varies per deployment, not per build][gp-10]).
- **Composition root**: where the new objects are built and injected
  ([DP-08 Composition Root][dp-08]; [GP-03]).
- **Documentation changes**: house, architecture, roadmap, and translation updates the
  change requires.

Cite rules inline, as [BP-03] requires, where a step applies them. When an example depends
on a third-party API, tell the agent to check the installed typings before copying it.

[cs-01]: ./code-style.md#cs-01-imports
[cs-02]: ./code-style.md#cs-02-zod-schemas-and-derived-types
[cs-04]: ./code-style.md#cs-04-errors
[cs-05]: ./code-style.md#cs-05-files-and-modules
[cs-06]: ./code-style.md#cs-06-blank-lines
[cs-09]: ./code-style.md#cs-09-shared-hooks
[cs-12]: ./code-style.md#cs-12-typed-route-local-hooks
[ds-09]: ./documents-style-guide.md#ds-09-doc-comments-and-line-comments
[ds-13]: ./documents-style-guide.md#ds-13-cross-references-with-link
[dp-08]: ../architecture/design-patterns.md#dp-08-composition-root
[gp-08]: ../architecture/development-good-practices.md#gp-08-validated-configuration-fail-fast
[gp-09]: ../architecture/development-good-practices.md#gp-09-secrets-and-credentials-only-from-the-environment
[gp-10]: ../architecture/development-good-practices.md#gp-10-configuration-varies-per-deployment-not-per-build

### BP-07 Acceptance Criteria

Conditions a reviewer or an agent can check to call the work done. Each one is concrete:
a command, an observable behavior, or a measurable number.

- **Validation**: the package scripts that must pass (`pnpm format`, `pnpm lint`,
  `pnpm check`, `pnpm build`), plus repository checks taken from the **Check:** line of
  every GP practice the plan applies (for example, `grep -rn "#infra/" src/application`
  returns nothing for [GP-02 Layered service structure][gp-02]).
- **Verification**: runtime behavior to confirm, such as the service failing at startup
  with `ENV_VAR_ERROR` when a required variable is unset ([GP-08]), or a route returning
  a given problem response.
- **Timings and capacities**: latency budgets, timeouts, rate limits, quotas, or volumes,
  only when the change depends on them (for example, a provider's send rate or a route's
  `config.rateLimit`, [GP-23 Secure transport defaults][gp-23]).
- **Tests**: only when the human explicitly asked for them. Then name the tests, the
  contract seam they exercise, and the command that runs them
  ([GP-24 Tests at contract seams][gp-24]). Otherwise write one line stating that tests
  are out of scope.

[gp-02]: ../architecture/development-good-practices.md#gp-02-layered-service-structure
[gp-23]: ../architecture/development-good-practices.md#gp-23-secure-transport-defaults
[gp-24]: ../architecture/development-good-practices.md#gp-24-tests-at-contract-seams

### BP-08 Out of Scope

A list of what the plan deliberately does not do, each item with a one-line reason or a
pointer to where it is handled (another plan, a roadmap entry). Include work a reader
could reasonably expect and that the plan leaves out, such as related flows, tests not
requested, or code under `legacy/`.

### BP-09 Follow-ups

What the agent considers pending for the human to review, decide, or schedule:

- Open questions the plan could not resolve, and values the agent does not know (hosts,
  regions, sender addresses).
- Deviations from architecture or house rules that need approval ([BP-05]), each with
  the rule cited.
- Gaps found in existing code while writing the plan that are outside its scope, each
  with the rule it breaks cited.
- Work that becomes possible or necessary after the plan lands: roadmap entries,
  translations, operational steps such as provider access requests.

## Reviewing the Plan

### BP-10 Compliance review before hand-off

Before presenting a plan, the author contrasts it against the architecture and house
documents and fixes what does not comply:

- Every proposal carries its `Rules:` line, every rule the current code breaks is cited
  in What's the Problem, and every citation follows [BP-03], points to a rule that
  exists, and says what the plan claims.
- The document follows the [Documents Style Guide](./documents-style-guide.md), in
  particular [DS-01 Markdown style][ds-01]; [DS-02 Wording][ds-02]; and
  [DS-04 Frontmatter for docs/ files][ds-04].
- The code examples follow the CS and DS rules listed in [BP-06].
- Names follow [Naming Conventions](./naming-conventions.md), including the symbol table
  and import aliases.
- Dependency direction, composition root, configuration, secrets, and errors follow
  [GP-02]; [GP-03]; [GP-08]; [GP-09]; and [GP-20 Classified errors][gp-20].
- The patterns already in the codebase are mirrored, not reinvented.
- Remaining deviations are stated in their proposal and listed under Follow-ups.

[ds-01]: ./documents-style-guide.md#ds-01-markdown-style
[ds-02]: ./documents-style-guide.md#ds-02-wording
[ds-04]: ./documents-style-guide.md#ds-04-frontmatter-for-docs-files
[gp-20]: ../architecture/development-good-practices.md#gp-20-classified-errors

## Template

```markdown
---
title: {Topic} Implementation Plan
lastUpdate: YYYY-MM-DD
lang: en
category: architecture
status: proposed
---

# {Topic} Implementation Plan

## Current Status

What exists today, with file references and the patterns the change must follow.

### What's the Problem

Why it must change, with evidence. Cite each rule the current code breaks next to the
behavior that breaks it: "... this breaks [XX-NN Title][xx-nn]".

[xx-nn]: ../en/{architecture|house}/{document}.md#xx-nn-title

## Goal

The outcome this plan delivers and its boundary.

### Proposals

1. **Lead term**: decision and reason.
   - Rules: [XX-NN]; [YY-NN Title][yy-nn].
   - Deviation: [XX-NN]; reason (only when the proposal breaks a rule).

[yy-nn]: ../en/{architecture|house}/{document}.md#yy-nn-title

## Implementation Details

### Rules for the executing agent

### Stage 1: {name}

#### Step 1: {name}

## Acceptance Criteria

- Validation:
- Verification:
- Timings and capacities:
- Tests: out of scope unless requested.

## Out of Scope

## Follow-ups
```
