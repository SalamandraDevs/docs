---
title: Build Plans
lastUpdate: 2026-11-13
lang: en
category: guide
status: published
---

## Purpose

House rules for writing build plans: the documents that describe a new build, such as a
new service, package, or capability, or a change to existing code, before it is
implemented, so a reviewer can approve the approach and an executing agent can carry it
out without guessing. This guide calls either one "the work". These rules apply to every
contributor, human or agent, and to every plan under `docs/plans/`.

A plan is grounded in the architecture and house documents. It does not restate them; it
cites them, applies them, and records any deliberate deviation:

- Architecture: [SOLID Principles](../architecture/solid-principles.md) (`SP`),
  [Design Patterns](../architecture/design-patterns.md) (`DP`), and
  [Development Good Practices](../architecture/development-good-practices.md) (`GP`).
- House: [Code Style](./code-style.md) (`CS`),
  [Documents Style Guide](./documents-style-guide.md) (`DS`),
  [Build Services](./build-services.md) (`BS`), and
  [Naming Conventions](./naming-conventions.md).

A plan is a Markdown document and follows the
[Documents Style Guide](./documents-style-guide.md) in full; this document adds only what
is specific to plans.

Each rule section carries a stable ID (`BP-NN`) for citation; IDs follow
[Naming Conventions, Document IDs][naming-document-ids].

The links in this guide are relative to `docs/en/house/`. The plan-side examples, inside
code blocks, use paths relative to `docs/plans/`; copy those, never the targets of this
guide's own links.

## Writing the Document

### BP-01 Location, name, and status

- Save plans in `docs/plans/` as `{YYMMDD}-{topic}-plan.md`, lowercase kebab-case
  (`261029-messages-plan.md`, `261030-session-plan.md`). `{YYMMDD}` is the date the plan
  was created, so file listings sort by age; it does not change when the plan is edited
  later, which `lastUpdate` records.
- A plan's `status` moves through these values, in this order, and never skips one:
  1. `proposed`: the plan is written and under review; its decisions may still change.
  2. `approved`: the review accepted the plan, before implementation starts. Implement
     only an approved plan; a change to a decision during implementation goes back
     through review and is recorded in the plan.
  3. `published`: the work is implemented, and the plan describes what was built.
  4. `archived`: the plan no longer guides work, for example because a later plan
     replaces its approach. A finished plan is `published`, not `archived`.
- Follow the frontmatter with one `#` heading that repeats the `title` value.

### BP-02 Required sections

Every plan has these top-level sections, in this order:

1. `## Current Status` with a `### What's the Problem` subsection.
2. `## Goal`, with a `### Decisions` subsection.
3. `## Implementation Details`.
4. `## Acceptance Criteria`.
5. `## Out of Scope`.
6. `## Follow-ups`.

Keep the section names exactly as written so plans read alike and can be searched. Add
subsections freely inside a section; do not add top-level sections between them. A
section that has nothing to say still appears, with one line stating so.

The link definitions block ([BP-03 Citing architecture and house rules][bp-03]) closes the
file, after the Follow-ups content. It is not a section and has no heading.

### BP-03 Citing architecture and house rules

A plan cites every architecture and house rule it relies on, so a reviewer can check each
claim against its source without guessing which rule is meant, and every citation is a
link that opens the rule.

- **First citation**: the rule's ID and title, exactly as its section heading reads,
  written as a reference-style link whose label is the ID in lowercase:
  `[GP-03 Contracts, composition-root wiring, and configured backends][gp-03]`.
- **Later citations**: the ID alone, as a shortcut link: `[GP-03]`. Labels match without
  regard to case, so it resolves to the same definition. Never write an ID without its
  link.
- **Definitions**: every label is defined once, in one block at the end of the plan,
  sorted by label. Definitions do not render, so their position does not matter to a
  reader; keeping them together means that moving or editing text never breaks a link.
- **Paths**: relative to the plan file, so they start with `../en/`:
  `[gp-03]: ../en/architecture/development-good-practices.md#gp-03-...`.
- **Rules without an ID**: cite them by document and section name, with a reference-style
  label made of the document's name and the section anchor:
  `[Naming Conventions, Service structure][naming-service-structure]`. Later citations
  may drop the document name and keep the label:
  `[Service structure][naming-service-structure]`.
- **Roadmap entries**: the entry ID as link text, with a label in lowercase:
  `[AUTH-011][auth-011]`, defined to the roadmap section that lists the entry.
- **Line length**: definition lines are exempt from the 92-character limit of
  [DS-01 Markdown style][ds-01], because their URLs cannot wrap. The exemption covers
  plans and this guide. Prose lines that cite stay within the limit; reference-style
  links keep them short.
- Separate several citations with semicolons, since some titles contain commas:
  `[GP-03]; [DP-01 Factory][dp-01]`.

A plan that cites one rule twice and another once:

```markdown
### What's the Problem

- The providers read a global config at module level; this breaks
  [GP-03 Contracts, composition-root wiring, and configured backends][gp-03].

## Goal

### Decisions

1. **Senders are built in the composition root**: `bootstrapMessaging` creates them.
   - Rules: [GP-03]; [DP-08 Composition Root][dp-08].

## Follow-ups

- ...

[dp-08]: ../en/architecture/design-patterns.md#dp-08-composition-root
[gp-03]: ../en/architecture/development-good-practices.md#gp-03-contracts-composition-root-wiring-and-configured-backends
```

Where citations are required:

- In `### What's the Problem`, next to every behavior of the current code that breaks a
  rule ([BP-04 Current Status][bp-04]).
- In every decision, on its `Rules:` line ([BP-05 Goal and decisions][bp-05]).
- In every decision that breaks a rule, on a `Deviation:` line, and again under
  Follow-ups ([BP-09 Follow-ups][bp-09]).
- Inline in Implementation Details and Acceptance Criteria, wherever a step or check
  applies a rule.

## Plan Sections

### BP-04 Current Status

Describes what is in place today, before the work starts: for a new build, the context
it lands in; for a change to existing code, how the related topic works. A plan may cover
both, such as a change to a service followed by its extraction into a new package; then
describe each part as below.

- State facts verified by reading the code, configuration, and deployment files, not
  assumptions.
- Reference code by path and symbol: `services/auth/src/infra/keys/factory.ts`,
  `createKeysProvider`. Paths start at the workspace root, the directory that holds
  `docs/`, `core/`, `tools/`, `services/`, and `legacy/`, because each code project under
  it is its own repository. Add
  `:line` only to the evidence of a defect or a violation in What's the Problem, where it
  pins the exact behavior; line numbers drift as soon as the code is edited.
- For a new build, such as a new service, package, or capability with no code yet,
  describe the context it lands in: the platform parts it will depend on or integrate
  with, the consumers waiting for it, the existing projects whose layout and patterns it
  should mirror, any legacy predecessor, and external constraints such as vendor APIs,
  quotas, or sandbox limits. State constraints as facts here; Acceptance Criteria states
  only the thresholds the finished work must meet ([BP-07 Acceptance Criteria][bp-07]).
- For a change to existing code, cover the parts the work will touch: modules and their
  contracts, data and storage, public interfaces (endpoints, commands, events, library
  exports), configuration and environment variables, deployment manifests, and any
  legacy code being replaced.
- For legacy code being replaced, describe what it does here; put its defects and rule
  violations in What's the Problem, and how the executing agent must treat it (for
  example, reference only, never copied) under Rules for the executing agent
  ([BP-06 Implementation Details][bp-06]).
- Name the existing patterns the work must follow, each with the file that shows it,
  cited as [BP-03] requires.
- Make no decisions here.

A pattern named in Current Status:

```markdown
- **Provider selection**: `createKeysProvider`
  (`services/auth/src/infra/keys/factory.ts`) selects the provider named by
  `SIGNING_KEY_PROVIDER`, following [DP-01 Factory][dp-01].
```

`### What's the Problem` states why the work is needed. Include the items that apply; a
new build or feature may fix no defect and break no rule:

- **New need or missing capability**: what is missing and the evidence, such as a roadmap
  entry (`[AUTH-011][auth-011]`).
- **Defect or gap**: the failing path or wrong behavior, with the evidence.
- **Rule violations**: when the current code breaks an architecture or house rule, cite
  it next to the behavior that breaks it, as [BP-03] requires.
- **Impact**: who or what is affected, and the cost of leaving it as it is.

State at least one of the first three items, and always state the impact.

A rule violation with its evidence:

```markdown
- Providers read a global `localConfig` at module level
  (`legacy/emailservice/src/providers/smtp.ts:11`); this breaks
  [GP-03 Contracts, composition-root wiring, and configured backends][gp-03].
```

### BP-05 Goal and decisions

`## Goal` opens with a short paragraph: the outcome the plan delivers and its boundary.

`### Decisions` is a numbered list. Each item is one decision, whether the human already
made it or the author proposes it; the plan is the single record of them, and the review
of the plan approves or changes them:

- For a new build, the first decisions are its founding ones: where it lives
  ([GP-01 One repository per package or service][gp-01]), its name and identifiers under
  [Naming Conventions](./naming-conventions.md), its contracts and boundaries with the
  rest of the platform, and the language, framework, and vendors it uses.
- Start each decision with a bold lead term and a colon, then the decision and its
  reason: `1. **Contracts live in the transport package**: ...`.
- End with a `Rules:` line listing every architecture and house rule the decision
  applies, cited as [BP-03] requires:

  ```markdown
  1. **Contracts live in the transport package**: decision and reason.
     - Rules: [SP-05 Dependency Inversion Principle (DIP)][sp-05]; [GP-02];
       [DP-02 Adapter][dp-02].
  ```

- Mention a rejected alternative in one sentence when it explains the choice; options
  are weighed before the plan is written, and the plan records the outcome.
- When a decision must break a rule, add a `Deviation:` line citing the rule and giving
  the reason, and list it again under Follow-ups for approval. Never break a rule
  silently.
- Number decisions once and never renumber them; later sections cite them as
  "decision 3".

### BP-06 Implementation Details

The section an executing agent follows step by step. It holds two fixed subsections, then
the stages:

1. `### Rules for the executing agent`: which documents to read, which existing files to
   mirror, dependency boundaries between layers, which project commands to run, how to
   treat legacy code, and when to stop and ask.
2. `### Interface layout`: each public interface the work adds or alters, across all
   stages (HTTP endpoints, CLI commands, events, library exports, configuration keys,
   environment variables), with its name or path, inputs, outputs, and errors. HTTP paths
   follow [Naming Conventions, APIs][naming-apis]. When the work changes no public
   interface, state so in one line.
3. One or more stages (`### Stage 1: senders inside auth`), each with ordered steps
   numbered from 1 (`#### Step 1: dependencies`). A plan with one stage still names it.

Steps follow dependency order inside each stage and cover, where they apply:

- **Scaffolding**: the first step of a stage that creates a new repository or service. It
  sets up the source layout, the dependency manifest and lockfile, the pinned toolchain,
  the quality gates in CI, the container image, and the publishing settings, before any
  feature work in that repository. When the new repository comes before all other work,
  its stage is `### Stage 0: scaffolding`. It applies [GP-01];
  [GP-05 Locked dependencies and pinned toolchain][gp-05];
  [GP-07 Supply-chain safeguards][gp-07]; and [GP-13 Quality gates run in CI][gp-13].
- **File layout**: in each stage that adds or moves files, the first step after any
  scaffolding; a directory tree of new and changed files, with the main element each one
  holds (a class, module, or function). Names follow
  [Naming Conventions](./naming-conventions.md).
- **Code examples**: enough to remove guesswork, starting each block with a comment
  naming its target file by full path. For a format without comments, such as JSON, name
  the target file in the sentence before the block. Examples are code the agent will
  copy, so they must already follow the code style and source-comment rules for the
  target language; for TypeScript, [Code Style](./code-style.md) and
  [DS-09 Doc comments and line comments][ds-09] to
  [DS-13 Cross-references with `{@link}`][ds-13]. Show one full example per pattern and
  excerpts for its repetitions.
- **Configuration and environment**: configuration schema or format changes, secret
  placeholders, new environment variables, and every deployment manifest that must pass
  them ([GP-08 Validated configuration, fail fast][gp-08];
  [GP-09 Secrets and credentials only from the environment][gp-09];
  [GP-10 Configuration varies per deployment, not per build][gp-10]).
- **Composition root**: where the new components are built and wired
  ([DP-08 Composition Root][dp-08];
  [GP-03 Contracts, composition-root wiring, and configured backends][gp-03]).
- **Documentation changes**: house, architecture, roadmap, and translation updates the
  work requires.

Validation commands and runtime checks belong in Acceptance Criteria, not in a final
step. Cite rules inline, as [BP-03] requires, where a step applies them. When an example
depends on a third-party API, tell the agent to check that API in the installed version
(its types, headers, or reference documentation) before copying the example.

### BP-07 Acceptance Criteria

Conditions a reviewer or an agent can check to call the work done. Each one is concrete:
a command, an observable behavior, or a measurable number.

- **Validation**: the project's format, lint, type-check or compile, and build commands
  that must pass, plus repository checks taken from the **Check:** line of every GP
  practice the plan applies, narrowed to the files, manifests, and packages the work adds
  or changes; for example, no transport-layer file imports from storage, for
  [GP-02 Layered service structure][gp-02]. A check that already fails for reasons that
  predate the work is not an acceptance criterion; list it under Follow-ups as a gap
  ([BP-09]).
- **Verification**: runtime behavior to confirm, such as the service refusing to start
  with a classified configuration error when a required variable is unset ([GP-08]), or
  an endpoint returning a given error response.
- **Timings and capacities**: latency budgets, timeouts, rate limits, quotas, or volumes
  the finished work must meet or respect, only when the work depends on them (for
  example, an endpoint's rate limit, [BS-01 Secure transport defaults][bs-01]). Refer to
  the vendor limits stated in Current Status instead of repeating them. When none
  applies, state so in one line.
- **Tests**: only when the human explicitly asked for them. Then name the tests, the
  contract seam they exercise, and the command that runs them
  ([GP-24 Tests at contract seams][gp-24]). Otherwise write one line stating that tests
  are out of scope.

### BP-08 Out of Scope

A list of what the plan deliberately does not do, each item with a one-line reason or a
pointer to where it is handled (another plan, a roadmap entry). Include work a reader
could reasonably expect and that the plan leaves out, such as related flows, tests not
requested, or legacy code left in place.

### BP-09 Follow-ups

What the agent considers pending for the human to review, decide, or schedule:

- Registry and documentation updates a new build requires, where a registry exists: a
  service id under [Naming Conventions, Services][naming-services], roadmap entries (and,
  for a repository outside the scope listed in [Roadmap Rules](../roadmap/README.md),
  that scope's extension), index links in the documentation, and agent instructions.
  When no registry covers the new build, such as a package name, state it as an open
  question.
- Open questions the plan could not resolve, and values the agent does not know (hosts,
  regions, sender addresses).
- Deviations from architecture or house rules that need approval ([BP-05]), each with
  the rule cited, or one line stating that there are none.
- Gaps found in existing code while writing the plan that are outside its scope, each
  with the rule it breaks cited, including repository checks that already fail
  ([BP-07]).
- Work that becomes possible or necessary after the plan lands: translations and
  operational steps such as provider access requests.

## Reviewing the Plan

### BP-10 Compliance review before hand-off

Before presenting a plan, the author contrasts it against the architecture and house
documents and fixes what does not comply:

- Every decision carries its `Rules:` line, every rule the current code breaks is cited
  in `### What's the Problem`, and every citation follows [BP-03], points to a rule that
  exists, and says what the plan claims.
- Every label has exactly one definition in the closing block, every definition is used,
  and every path starts with `../en/`.
- The document follows the [Documents Style Guide](./documents-style-guide.md), in
  particular [DS-01]; [DS-02 Wording][ds-02]; and
  [DS-04 Frontmatter for docs/ files][ds-04].
- The code examples follow the rules for the target language listed in [BP-06].
- Names follow [Naming Conventions](./naming-conventions.md) and the conventions of the
  target language and layout.
- Dependency direction, composition root, configuration, secrets, and errors follow
  [GP-02]; [GP-03]; [GP-08]; [GP-09]; and [GP-20 Classified errors][gp-20].
- The patterns already in the platform are mirrored, not reinvented.
- Remaining deviations are stated in their decision and listed under Follow-ups.

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

The context a new build lands in, or what exists today for a change, with path and
symbol references and the patterns the work must follow.

### What's the Problem

Why the work is needed, with evidence. Cite each rule the current code breaks next to the
behavior that breaks it: "... this breaks [XX-NN Title][xx-nn]".

## Goal

The outcome this plan delivers and its boundary.

### Decisions

1. **Lead term**: decision and reason.
   - Rules: [XX-NN]; [YY-NN Title][yy-nn].
   - Deviation: [XX-NN]; reason (only when the decision breaks a rule).

## Implementation Details

### Rules for the executing agent

### Interface layout

### Stage 1: {name}

#### Step 1: {name}

## Acceptance Criteria

- Validation:
- Verification:
- Timings and capacities:
- Tests: out of scope unless requested.

## Out of Scope

## Follow-ups

[xx-nn]: ../en/{architecture|house}/{document}.md#xx-nn-title
[yy-nn]: ../en/{architecture|house}/{document}.md#yy-nn-title
```

[bp-03]: #bp-03-citing-architecture-and-house-rules
[bp-04]: #bp-04-current-status
[bp-05]: #bp-05-goal-and-decisions
[bp-06]: #bp-06-implementation-details
[bp-07]: #bp-07-acceptance-criteria
[bp-09]: #bp-09-follow-ups
[bs-01]: ./build-services.md#bs-01-secure-transport-defaults
[dp-08]: ../architecture/design-patterns.md#dp-08-composition-root
[ds-01]: ./documents-style-guide.md#ds-01-markdown-style
[ds-02]: ./documents-style-guide.md#ds-02-wording
[ds-04]: ./documents-style-guide.md#ds-04-frontmatter-for-docs-files
[ds-09]: ./documents-style-guide.md#ds-09-doc-comments-and-line-comments
[ds-13]: ./documents-style-guide.md#ds-13-cross-references-with-link
[gp-01]: ../architecture/development-good-practices.md#gp-01-one-repository-per-package-or-service
[gp-02]: ../architecture/development-good-practices.md#gp-02-layered-service-structure
[gp-03]: ../architecture/development-good-practices.md#gp-03-contracts-composition-root-wiring-and-configured-backends
[gp-05]: ../architecture/development-good-practices.md#gp-05-locked-dependencies-and-pinned-toolchain
[gp-07]: ../architecture/development-good-practices.md#gp-07-supply-chain-safeguards
[gp-08]: ../architecture/development-good-practices.md#gp-08-validated-configuration-fail-fast
[gp-09]: ../architecture/development-good-practices.md#gp-09-secrets-and-credentials-only-from-the-environment
[gp-10]: ../architecture/development-good-practices.md#gp-10-configuration-varies-per-deployment-not-per-build
[gp-13]: ../architecture/development-good-practices.md#gp-13-quality-gates-run-in-ci
[gp-20]: ../architecture/development-good-practices.md#gp-20-classified-errors
[gp-24]: ../architecture/development-good-practices.md#gp-24-tests-at-contract-seams
[naming-apis]: ./naming-conventions.md#apis
[naming-document-ids]: ./naming-conventions.md#document-ids
[naming-services]: ./naming-conventions.md#services
