---
title: Documents Style Guide
lastUpdate: 2026-10-24
lang: en
category: guide
status: active
owner: OdairTrujillo
---

## Purpose

House conventions for documentation and source-level API comments across the Salamandra
Devs platform. These rules apply to every contributor, human or agent. Code style lives in
the [Code Style](./code-style.md) guide.

Each rule section carries a stable ID (`DS-NN`) for citation; IDs follow
[Document IDs](./naming-conventions.md#document-ids).

## Markdown Documents

Guidelines for writing and maintaining documentation, planning documents, and design
specifications across the platform.

### DS-01 Markdown style

- Wrap ordinary prose lines in Markdown files at a maximum of 92 characters.
- Preserve code blocks, tables, headings, and directory trees when reflowing text. This
  preservation rule takes precedence over the prose line-length rule.
- Prefer itemized lists (bullet points and numbered lists) over tables in documentation
  and planning documents. Use tables only when a structured comparison of multiple
  attributes across many items genuinely improves clarity; for example, configuration
  reference tables or provider feature matrices. When items have one to three key
  attributes, describe them in list form instead:
  `- Item Name: attribute 1, attribute 2, attribute 3.`
- Use semicolons instead of em-dashes (—), with no space before the semicolon. Prefer
  `text; more text` over `text — more text` for clarity and consistency in plain-text
  processing.
- When a list item starts with a lead term (a bold label, a name, or a code identifier)
  followed by its definition or description, separate them with a colon, not a semicolon:
  `- **Factory**: selects and builds a concrete implementation.` Keep semicolons for
  joining clauses inside a sentence.

### DS-02 Wording

- Avoid the word "canonical" (and the Spanish "canónico"); it is rarely used in Spanish
  and reads as jargon. Prefer a plain qualifier such as "service id", "reference", or
  "official", or drop the qualifier when the surrounding rule already carries the meaning.
  Keep it only when it is a literal identifier (a CLI flag, a field name) or the name of
  an external standard's term, such as the canonical UUID text form.
- Avoid the word "entity" (and the Spanish "entidad"); the platform no longer has an
  entity layer. Name the noun a route or record module is about a "resource" (`users`,
  `sessions`), and use `{Resource}` / `{resource}` in naming patterns
  (`{Resource}Record`, `{resource}ListQuerySchema`). Call persisted data a "record" and
  its storage class a "records handler". Do not use "subject" for this: it already means
  the authenticated principal (JWT `sub`, `SessionRecord.subject`). Keep "entity" only in
  literal names of earlier code (`entityId`, `Entities`, `#entities`), in quotations of
  external sources, and in vendor terms such as a Vault identity entity.

### DS-03 Plans and design requirements

- Do not create flow diagrams in plans or design requirements.
- Use descriptive headings and itemized lists to structure design documents.
- Reference architectural principles and existing patterns from `docs/en/architecture/`.
- Link related planning documents and implementation artifacts when appropriate.
- Maintain frontmatter metadata for searchability and document lifecycle tracking.

### DS-04 Frontmatter for docs/ files

Every Markdown file created in the `docs/` directory must include YAML frontmatter at the
top with the following required fields:

```yaml
---
title: Brief, descriptive title
lastUpdate: YYYY-MM-DD
lang: en or es
category: architecture, infrastructure, planning, or other
status: <one of the status values below>
owner: Name (optional, recommended for active documents)
---
```

Field descriptions:

- `title`: One-line description of the document's purpose. Examples: "Auth Service
  Architecture Build Plan", "SOLID Principles", "Durable Signing Key Provider
  Implementation Plan".
- `lastUpdate`: The date the document was last updated, in `YYYY-MM-DD` format. Use
  `today` for a placeholder when creating a new document; replace it with the actual date
  before committing.
- `lang`: The document language. Use `en` for English or `es` for Spanish. Maintain
  separate files for each language in `docs/en/` and `docs/es/` respectively.
- `category`: A classification for the document. Examples: `architecture`,
  `infrastructure`, `planning`, `guide`, `operations`. Use snake_case if multi-word.
- `status`: The current state of the document. See [Status values](#ds-05-status-values).
- `owner`: (Optional) The person or team responsible for maintaining this document.
  Recommended for active, evolving documents. Format: `FirstName LastName` or `Team Name`.

### DS-05 Status values

The `status` field in frontmatter uses one of the following values. Use these
consistently to signal document completeness and reliability.

Document lifecycle states:

- `published`: document is complete, reviewed, and ready for reference. Content is stable
  and accurate. Example: `auth-plan.md` with architecture decisions that have been
  implemented.
- `active`: document is current and actively maintained. Content may evolve as the
  platform changes. Used for living guides, operational runbooks, and roadmaps that stay
  synchronized with implementation. Example: `ROADMAP.md` tracking feature status across
  services.
- `draft`: document is incomplete and still being written. Not recommended for external
  reference; intended for team discussion only.
- `archived`: document describes an obsolete approach or completed initiative. Content is
  preserved for historical reference but should not guide new work.
- `proposed`: document describes a planned change or new feature under discussion. Content
  may be speculative and subject to revision before implementation begins.

Feature status tags (`implemented`, `planned`, `proposed`, `canceled`) are not frontmatter
values; they mark individual features inline. Their definitions, the roadmap line format,
and the roadmap editing rules live in [ROADMAP.md](../../ROADMAP.md#roadmap-rules).

### DS-06 Frontmatter examples

Architecture planning document:

```yaml
---
title: Auth Session Issuance Implementation Plan
lastUpdate: 2026-09-05
lang: en
category: architecture
status: published
owner: Odair Trujillo
---
```

Operational runbook:

```yaml
---
title: Mail Servers Backup and Sync
lastUpdate: 2026-09-20
lang: en
category: infrastructure
status: active
owner: DevOps Team
---
```

Spanish translation:

```yaml
---
title: Principios SOLID
lastUpdate: 2026-08-21
lang: es
category: arquitectura
status: published
owner: Odair Trujillo
---
```

Roadmap document:

```yaml
---
title: Salamandra Devs Platform Roadmap
lastUpdate: 2026-09-22
lang: en
category: planning
status: active
---
```

### DS-07 When to update lastUpdate

- Update `lastUpdate` whenever the document content changes materially.
- Minor formatting or typo fixes do not require updating the date.

### DS-08 Translating between languages

- Keep frontmatter field names in English (`title`, `lastUpdate`, `lang`, `category`,
  `status`, `owner`).
- Translate the `title` value into the document's language.
- Use the `lang` field to indicate the language (`en` or `es`).
- When translating an existing document, use the same `lastUpdate` date as the English
  version to show they are synchronized translations.
- If a translated version is updated after the English, bump its `lastUpdate` to the new
  date.

## TypeDoc Source Comments

TypeDoc generates the API-documentation site for the platform libraries from TSDoc
`/** */` comments on exported declarations.

### DS-09 Doc comments and line comments

- Write TSDoc `/** */` doc comments on every exported declaration (class, function,
  method, property, type, interface, enum) so the generated site documents the public
  surface.
- Only a `/** */` comment attached to a declaration is documentation. Implementation notes
  that live inside a function or method body stay `//` line comments; never convert an
  in-body note into a floating `/** */`, which attaches to no declaration and
  `eslint-plugin-jsdoc` flags.

```ts
export class DatabaseAdapter {
  /** Runs dataset and index DDL through the backend's `DatasetProvisioner`. */
  private readonly provisioner: IDatasetProvisioner<DatasetDef>;

  constructor(handle: DatabaseHandle) {
    // The discriminated `backend` tag selects the driver once; this body note stays a
    // line comment.
    switch (handle.backend) {
      // ...
    }
  }
}
```

- Private members (`private` or `#`) are excluded from the generated site
  (`excludePrivate: true`). Comment them only as source-reading aids and do not rely on
  them rendering.
- Comment lines are limited to 92 characters, including the leading indentation and
  comment markers.

### DS-10 Tags

Use the following tags as house style, consistent with the comments the repository
already writes:

- `@param name - description`, using the hyphen form.
- `@returns` (not `@return`) for a non-void return, without a hyphen.
- `@throws {CustomError} <condition>` for each failure path, naming the relevant
  `ErrorName` in the condition text. The `{CustomError}` type is required by the
  `jsdoc/require-throws-type` rule and matches the rule that every thrown error is a
  `CustomError` with a deliberate `ErrorName`.
- `@remarks` for detail beyond the one-line summary; `@example` with a fenced code block
  for usage.
- `{@link Symbol}` for cross-references instead of a bare identifier.
- `@category` or `@group` to organize the generated sidebar; `@internal` to keep an
  exported-but-not-public symbol out of the site.
- Do not restate a type TypeScript already encodes; never add a `@param {string}` type
  annotation.

### DS-11 Type and schema documentation

- For types not derived from a Zod schema, do not attach per-field JSDoc comments to
  individual properties inside a type. Instead, write a single JSDoc block above the type
  itself, with one bullet line per field summarizing its purpose in one line. Keep all
  documentation consolidated at the type level rather than scattered inline.

```ts
// Avoid
export type SessionGrant = Readonly<{
  /** Server-side session identifier. */
  sessionId: string;
  /** Local actor represented by the session. */
  actorId: string;
}>;

// Prefer
/**
 * Trusted identity reconstructed from a verified access token.
 * - sessionId: Server-side session identifier.
 * - actorId: Local actor represented by the session.
 */
export type SessionGrant = Readonly<{
  sessionId: string;
  actorId: string;
}>;
```

- As the Zod-specific exception to the preceding rule, put detailed field-level JSDoc on
  the schema constant and keep the inferred type JSDoc brief. The schema JSDoc carries the
  bullets that would otherwise have lived on the hard-coded type. Both feed TypeDoc as
  written.

```ts
/**
 * Item-route record identifier schema.
 * - recordId: Record identifier addressed by the route.
 */
export const recordParamsSchema = z
  .object({ recordId: z.uuid() })
  .strict();

/** Record path parameters accepted by item routes. */
export type RecordParams = z.infer<typeof recordParamsSchema>;
```

### DS-12 Description (summary) and `@remarks`

- The description is the leading untagged paragraph of the doc comment. TSDoc has no
  `@description` tag and the house style does not add one; the summary paragraph is the
  description. Keep it to one line that states what the symbol is or does.
- `@remarks` carries detail beyond that one-line summary: mechanism, invariants,
  lifecycle, idempotency, and the collaborators involved. Put extended explanation here,
  never in the summary.
- For a class that `implements` a contract, write `@remarks` role-first: lead with what
  the class does or how it works. Do not open with a restatement appositive such as "The
  shipped {@link IContract}." or "The backend-neutral {@link IContract}:". The
  `implements` clause, plus the interface's own pointer back to its implementation,
  already documents the class-to-contract binding, so repeating it in the class remarks is
  redundant.
- Keep the contract-to-implementation pointer on the interface side (in the interface's
  own `@remarks`, for example "Implemented by {@link DatabaseProvider}" or "{@link
  DatabaseProvider} is the shipped implementation"), not duplicated on the class.

```ts
// Avoid; restates the binding the `implements` clause already encodes.
/**
 * Provisioning workflow that converts schemas and coordinates dataset setup.
 *
 * @remarks
 * The backend-neutral {@link IDatasetProvisioner}: it converts a dataset's Zod schema
 * through the injected {@link SchemaConverter}, then drives an {@link IDataset}.
 */
export class DatasetProvisioner ... implements IDatasetProvisioner<TDef> {}

// Prefer; the summary names the role, `@remarks` leads with mechanism.
/**
 * Provisioning workflow that converts schemas and coordinates dataset setup.
 *
 * @remarks
 * Converts a dataset's Zod schema through the injected {@link SchemaConverter}, then
 * drives an {@link IDataset} to create the dataset and its indexes.
 */
export class DatasetProvisioner ... implements IDatasetProvisioner<TDef> {}
```

### DS-13 Cross-references with `{@link}`

- Use `{@link Symbol}` for any reference to another exported symbol, instead of a bare
  identifier or a backticked name, so TypeDoc renders a navigable link on the generated
  site.
- `{@link Symbol}` renders the symbol name; use `{@link Symbol | custom text}` only when
  the surrounding prose needs different link text.
- Link the collaborators a reader would actually jump to: the contract a class
  implements, the types a function consumes or returns, and the sibling that consumes its
  output.
- `{@link}` resolves exported symbols only. Do not link private members or non-exported
  locals; they are excluded from the generated site and the link will not resolve.

```ts
/**
 * Returns the connected, backend-tagged database handle.
 *
 * @returns Connected database handle.
 * @throws {CustomError} DB_BACKEND_ERROR when called before
 *   {@link DatabaseProvider.connect}.
 */
getHandle(): DatabaseHandle;
```
