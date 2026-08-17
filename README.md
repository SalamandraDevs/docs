# Salamandra Devs Documentation

Community-maintained knowledge base for the **Salamandra Devs** community. Guides, architecture notes, and operational runbooks live here so we can share what we learn, improve it together, and keep practices aligned across projects.

This repository is open to contributions from members and collaborators. If you use these docs, spot a gap, or want to document something new, you are welcome to participate.

## Table of contents

- [Topics](#topics)
  - [Architecture](#architecture)
  - [Infrastructure](#infrastructure)
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

| Topic | Description |
| --- | --- |
| [SOLID Principles](./architecture/solid-principles.md) | Definitions, terminology, TypeScript examples, benefits, trade-offs, and references for SRP, OCP, LSP, ISP, and DIP. |
| [Current Design Patterns in the Mongo Entity Stack](./architecture/current-patterns.md) | How the shared Mongo layer maps to established patterns, with examples of sound and broken implementations. |
| [Domain-orthogonal infrastructure wiring](./architecture/domain-orthogonal-infrastructure-wiring.md) | Ports-and-adapters approach for separating domain logic from persistence and wiring dependencies at the composition root. |
| [Development Good Practices](./architecture/development-good-practices.md) | General guidance on modularity, encapsulation, and maintainable system design, with practical examples. |

### Infrastructure

Operational guides for local development, mail services, and platform setup.

| Topic | Description |
| --- | --- |
| [Mail servers backup and sync](./infrastructure/MAIL-SERVERS-BACKUP-AND-SYNC.md) | Mailcow backup strategy, cold standby synchronization, systemd timers, and retention policy. |
| [Local HTTPS setup for `mylocalhost.com`](./infrastructure/mylocalhost-https-setup-en.md) | Apache and mkcert setup on Ubuntu for trusted local HTTPS development. |
| [CI/CD diagrams](./infrastructure/CI_CD/) | Excalidraw diagrams documenting CI/CD flows and related infrastructure decisions. |

## How to use this repository

- Browse topics by area using the table above.
- Prefer linking to an existing guide instead of duplicating content in project READMEs.
- Treat docs as living material: when a practice changes in code or operations, update the relevant guide here.
- Open an issue when something is unclear, outdated, or missing.

## Contribute

We want this repository to reflect real community experience. Contributions can be small fixes, new guides, clearer examples, or corrections when something no longer matches current practice.

### Ways to participate

- **Improve existing docs** — fix typos, clarify wording, update outdated steps, or add examples.
- **Add new guides** — document architecture decisions, onboarding flows, runbooks, or tooling setup.
- **Review pull requests** — help validate accuracy, readability, and consistency with existing material.
- **Open issues** — report gaps, ask questions, or propose new topics before writing a large change.

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

If you do not have write access, this fork-and-PR flow is the standard path for all contributions.

### Pull request guidelines

- Keep one logical change per pull request when possible.
- Link related issues in the PR description when applicable.
- Summarize what changed and why reviewers should care.
- Prefer precise, practical writing over generic advice.
- Include commands, config snippets, or examples when they help readers apply the guide.
- Avoid unrelated formatting-only changes unless they improve readability in the files you are already editing.

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

For larger proposals, opening an issue before starting work helps align scope and avoid duplicate effort.

### Writing guidelines

- Write for practitioners: concrete steps, clear terminology, and real-world context.
- Use Markdown consistently with existing documents in this repository.
- Prefer stable concepts and named patterns over project-specific jargon when the guide is meant to be shared broadly.
- Cross-link related guides instead of repeating large sections.
- Mark proposals, drafts, or environment-specific assumptions clearly when they are not yet standard practice.
- Do not commit secrets, credentials, private hostnames, or customer-specific data.

Thank you for helping Salamandra Devs build documentation the community can trust and improve together.
