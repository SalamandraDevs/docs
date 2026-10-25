---
title: Salamandra Devs Documentation
lastUpdate: 2026-10-25
lang: en
category: guide
status: active
---

# Salamandra Devs Documentation

Community-maintained knowledge base for the **Salamandra Devs** community. Guides,
architecture notes, and operational runbooks live here so we can share what we learn,
improve it together, and keep practices aligned across projects.

This repository is open to contributions from members and collaborators. If you use these
docs, spot a gap, or want to document something new, you are welcome to participate.

**Other languages:** [Principios SOLID (español)](../es/arquitectura/principios-solid.md) ·
[Índice en español](../es/README.md)

## Table of contents

- [Topics](#topics)
  - [Architecture](#architecture)
  - [House](#house)
  - [Roadmap](#roadmap)
- [How to use this repository](#how-to-use-this-repository)
- [Contribute](#contribute)
  - [Ways to participate](#ways-to-participate)
  - [Contribution workflow](#contribution-workflow)
  - [Pull request guidelines](#pull-request-guidelines)
  - [Reporting issues](#reporting-issues)
  - [Writing guidelines](#writing-guidelines)

## Topics

### Architecture

Software design principles, patterns, and practices used across Salamandra Devs projects.

- [SOLID Principles](./architecture/solid-principles.md): definitions, terminology,
  TypeScript examples, benefits, trade-offs, and references for SRP, OCP, LSP, ISP, and DIP.
- [Design Patterns](./architecture/design-patterns.md): the creational, structural,
  behavioral, and application-architecture patterns the Salamandra Devs code uses today
  (DP-01 through DP-10), grouped by category with the problem each one solves.
- [Development Good Practices](./architecture/development-good-practices.md): general
  guidance on modularity, encapsulation, and maintainable system design, with practical
  examples.

### House

Platform-wide conventions for code, documentation, and source-level API comments.

- [Code Style](./house/code-style.md): TypeScript and general code style; imports, Zod
  schemas, wrappers, errors, modules, blank lines, and RESTful API route hooks.
- [Documents Style Guide](./house/documents-style-guide.md): Markdown style, wording,
  frontmatter, status values, and TypeDoc source comments, with worked examples.
- [Naming Conventions](./house/naming-conventions.md): the single naming standard for
  services, routes, records, and symbols, with abbreviation rules and the document-ID
  scheme.

### Roadmap

Feature status per package, tracked inline with status tags.

- [Core Library Roadmap](./roadmap/CORE-ROADMAP.md): feature status for `core/`.
- [Tools Library Roadmap](./roadmap/TOOLS-ROADMAP.md): feature status for `tools/`.
- [Auth Service Roadmap](./roadmap/AUTH-ROADMAP.md): feature status for `services/auth/`.

## How to use this repository

- Browse topics by area using the lists above.
- Prefer linking to an existing guide instead of duplicating content in project READMEs.
- Treat docs as living material: when a practice changes in code or operations, update the
  relevant guide here.
- Open an issue when something is unclear, outdated, or missing.

## Contribute

We want this repository to reflect real community experience. Contributions can be small
fixes, new guides, clearer examples, or corrections when something no longer matches current
practice.

### Ways to participate

- **Improve existing docs**: fix typos, clarify wording, update outdated steps, or add
  examples.
- **Add new guides**: document architecture decisions, onboarding flows, runbooks, or
  tooling setup.
- **Review pull requests**: help validate accuracy, readability, and consistency with
  existing material.
- **Open issues**: report gaps, ask questions, or propose new topics before writing a large
  change.

### Contribution workflow

1. **Fork** this repository to your GitHub account.
2. **Clone** your fork locally:

   ```bash
   git clone https://github.com/<your-username>/<repo-name>.git
   cd <repo-name>
   ```

3. **Create a branch** for your change:

   ```bash
   git checkout -b docs/your-topic
   ```

4. **Make your changes** in Markdown. Keep edits focused and easy to review.
5. **Commit** with a clear message that explains what changed and why.
6. **Push** your branch to your fork:

   ```bash
   git push -u origin docs/your-topic
   ```

7. **Open a pull request** against the main branch of the upstream repository.
8. **Respond to review feedback** and update the PR until it is ready to merge.

If you do not have write access, this fork-and-PR flow is the standard path for all
contributions.

### Pull request guidelines

- Keep one logical change per pull request when possible.
- Link related issues in the PR description when applicable.
- Summarize what changed and why reviewers should care.
- Prefer precise, practical writing over generic advice.
- Include commands, config snippets, or examples when they help readers apply the guide.
- Avoid unrelated formatting-only changes unless they improve readability in the files you
  are already editing.

### Reporting issues

Use GitHub Issues to:

- Report incorrect or outdated instructions
- Request documentation for a missing topic
- Ask for clarification on an existing guide
- Propose structural improvements to the docs set

When opening an issue, include:

- The document or topic involved
- What you expected to find
- What is unclear, missing, or wrong
- Any relevant context such as environment, tool versions, or project area

For larger proposals, opening an issue before starting work helps align scope and avoid
duplicate effort.

### Writing guidelines

- Write for practitioners: concrete steps, clear terminology, and real-world context.
- Use Markdown consistently with existing documents in this repository.
- Prefer stable concepts and named patterns over project-specific jargon when the guide is
  meant to be shared broadly.
- Cross-link related guides instead of repeating large sections.
- Add or update translations under `docs/es/` when contributing Spanish content.
- Keep filenames aligned across locales when possible (for example, `solid-principles.md` /
  `principios-solid.md`).
- Mark proposals, drafts, or environment-specific assumptions clearly when they are not yet
  standard practice.
- Do not commit secrets, credentials, private hostnames, or customer-specific data.

Thank you for helping Salamandra Devs build documentation the community can trust and
improve together.
