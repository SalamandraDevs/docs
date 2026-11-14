---
title: English-to-Spanish Translation Mappings
lastUpdate: 2026-11-13
lang: en
category: guide
status: active
---

# English-to-Spanish Translation Mappings

Reference for translating `docs/en/` into `docs/es/`. It records the directory names, file
names, frontmatter values, and terminology each Spanish document uses, so translations and
updates stay consistent over time.

**English is always the source of truth.** Translate from `docs/en/` into `docs/es/`, never
the reverse. When the two disagree, the English document is correct and the Spanish one is
realigned to it.

## Translation rules

- Mirror the `en/` structure with the Spanish folder and file names in the tables below. One
  English document maps to exactly one Spanish document.
- Keep verbatim, never translated: code identifiers, type and symbol names, route paths, CLI
  commands, config keys, environment variable names, package names, and string literals
  inside code.
- Keep the section IDs (`SP-NN`, `DP-NN`, `GP-NN`, `CS-NN`, `DS-NN`, `BS-NN`) identical
  across languages; only the heading text after the ID is translated.
- Translate prose, code comments, and TSDoc summary/`@remarks` text. Keep TSDoc tags
  (`@param`, `@returns`, `@throws`, `@remarks`, `{@link}`) and the symbols they reference.
- Frontmatter: keep field names in English, translate `title`, set `lang: es`, keep `status`
  as the English literal (see [DS-05](./en/house/documents-style-guide.md)), and set
  `lastUpdate` to the same date as the English version (DS-08).
- Use semicolons, not em-dashes (DS-01).
- Prefer the established English programming term when one is in common Spanish usage; see
  [Terms kept in English](#terms-kept-in-english). Translate the rest; see
  [Translated terminology](#translated-terminology).

## Pending translations

Spanish documents that lag behind their English source. Update the translation, then
remove its line here.

- `arquitectura/buenas-practicas-desarrollo.md`: behind the 2026-11-13 English update;
  GP-16, GP-19, and GP-23 moved to BS; GP-06, GP-11, and GP-22 merged into GP-05, GP-03,
  and the new GP-25, whose titles changed; GP-09, GP-12, GP-15, GP-18, GP-20, GP-21, GP-24,
  and the closing sections were rewritten.
- `normas/construir-servicios.md`: not created yet; translate
  `house/build-services.md`.
- `normas/estilo-codigo.md`: behind the 2026-11-13 English update; CS-09 to CS-12 moved
  to Build Services as BS-05 to BS-08, and the Purpose section changed.
- `arquitectura/patrones-diseno.md`: behind the 2026-11-13 English update; DP-06 links to
  BS-07 instead of CS-11.
- `normas/convenciones-nombres.md`: behind the 2026-11-13 English update; the Document IDs
  section adds the `BS` prefix and the retired-ID rule.
- `README.md`: behind the 2026-11-13 English update; the House index adds Build
  Services.

## Directory names

| English (`docs/en/`) | Spanish (`docs/es/`) |
| --- | --- |
| `architecture/` | `arquitectura/` |
| `house/` | `normas/` |
| `roadmap/` | `roadmap/` |
| `infra/` | `infra/` |

The word **roadmap** is kept in Spanish content (it is in common use), so the directory and
the document titles use `roadmap`, not a translation.

## File names

| English | Spanish |
| --- | --- |
| `README.md` | `README.md` |
| `architecture/solid-principles.md` | `arquitectura/principios-solid.md` |
| `architecture/design-patterns.md` | `arquitectura/patrones-diseno.md` |
| `architecture/development-good-practices.md` | `arquitectura/buenas-practicas-desarrollo.md` |
| `house/code-style.md` | `normas/estilo-codigo.md` |
| `house/documents-style-guide.md` | `normas/guia-estilo-documentos.md` |
| `house/naming-conventions.md` | `normas/convenciones-nombres.md` |
| `house/build-services.md` | `normas/construir-servicios.md` |
| `infra/packaging.md` | `infra/empaquetado.md` |
| `roadmap/README.md` | `roadmap/README.md` |
| `roadmap/AUTH-ROADMAP.md` | `roadmap/AUTH-ROADMAP.md` |
| `roadmap/CORE-ROADMAP.md` | `roadmap/CORE-ROADMAP.md` |
| `roadmap/TOOLS-ROADMAP.md` | `roadmap/TOOLS-ROADMAP.md` |

The `{REPO}-ROADMAP.md` names are identifiers and stay as-is across languages.

## Frontmatter `category` values

The `category` value is translated (the field name stays `category`).

| English | Spanish |
| --- | --- |
| `architecture` | `arquitectura` |
| `guide` | `guia` |
| `planning` | `planificacion` |
| `operations` | `operaciones` |
| `infrastructure` | `infraestructura` |

## Links and anchors

- Cross-document links use the mapped Spanish path plus the Spanish heading anchor. Compute
  the anchor from the translated heading (GitHub slug: lowercase, spaces to hyphens,
  punctuation dropped, accented letters kept, for example `#ids-de-documento`).
- Design-pattern headings keep the English pattern name (see below), so their anchors stay
  English and match `en/` (for example `#dp-08-composition-root`).
- Links to documents that have no Spanish translation (for example `architecture-review.md`
  or anything under `en/howto/`) point to the English file unchanged.

## Design-pattern names

Proper names of GoF and application patterns stay in English in the `DP-NN` headings and
when naming the pattern itself: **Factory, Adapter, Bridge, Facade, Strategy, Chain of
Responsibility, Repository, Composition Root, Ambient Context, Single Flight**. A generic,
non-proper reference may use the common Spanish noun in lowercase (for example `factoría`,
`fachada`, `estrategia`, `adaptador`).

## Terms kept in English

Common in Spanish technical writing; left untranslated. Render them with ordinary Spanish
articles and agreement (for example "el backend", "los hooks", "la release").

- backend, frontend
- build, release, rollback, deploy (verb; the noun is `despliegue`)
- staging, production, devel (branch and environment names)
- roadmap
- wrapper
- hook, endpoint, middleware
- token, tenant, scope (npm package scope)
- cache / caché, log / logs, runtime, stdout
- lockfile, toolchain, dataset, payload, bootstrap
- fake(s), mock
- commit, push, fork, pull request
- readiness, liveness
- Single Flight (the pattern name)

## Translated terminology

Concepts translated consistently across the Spanish documents.

| English | Spanish |
| --- | --- |
| record | registro |
| records handler | manejador de registros |
| composition root | raíz de composición |
| layer; layered | capa; en capas |
| backing service | servicio de respaldo |
| attached resource | recurso adjunto |
| contract | contrato |
| convenience wrapper | wrapper de conveniencia |
| the caller | quien llama |
| discriminated union | unión discriminada |
| quality gate | puerta de calidad |
| type guard | guarda de tipo |
| placeholder | marcador de posición |
| request coalescing | coalescencia de peticiones |
| contract seam | costura de contrato |
| working tree | árbol de trabajo |
| snapshot | instantánea |
| trigger | disparador |
| flag | bandera |
| barrel file | archivo barril |
| stateless | sin estado |
| least privilege | mínimo privilegio |
| supply chain | cadena de suministro |
| safeguard | salvaguarda |
| graceful stop | parada ordenada |
| disposability | descartabilidad |
| correlation id | id de correlación |
| parity | paridad |
| schema | esquema |
| dataset provisioning | aprovisionamiento de datasets |
| index | índice |
| deployment | despliegue |
| branch | rama |
| provenance | procedencia |
| dead weight | peso muerto |
| event stream | flujo de eventos |
| house (rules / style) | de la casa |

## Feature status tags

Inline roadmap status tags are translated (they are prose, not frontmatter values):

| English | Spanish |
| --- | --- |
| `implemented` | `implementado` |
| `planned` | `planificado` |
| `proposed` | `propuesto` |
| `canceled` | `cancelado` |

Frontmatter `status` values are not translated; they stay as the English literals defined in
DS-05 (`published`, `active`, `draft`, `archived`, `proposed`).
</content>
