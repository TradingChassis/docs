# TradingChassis — Trading Infrastructure Documentation

[![Docs](https://img.shields.io/badge/docs-MkDocs%20Material-blue)](#)
[![Deploy Docs](https://github.com/TradingChassis/docs/actions/workflows/deploy.yaml/badge.svg)](https://github.com/TradingChassis/docs/actions/workflows/deploy.yaml)
[![Diagrams: Mermaid](https://img.shields.io/badge/diagrams-Mermaid-ff3670)](#)
[![Made with Markdown](https://img.shields.io/badge/made%20with-Markdown-1f425f)](#)
[![License: CC BY 4.0](https://img.shields.io/badge/license-CC%20BY%204.0-green)](https://creativecommons.org/licenses/by/4.0/)

This repository contains the documentation for TradingChassis's infrastructure. It is the canonical reference for architectural decisions, concepts, implementation-facing Stack descriptions, and operational model.

> **Terminology note:** This README follows the [TradingChassis terminology](https://tradingchassis.github.io/docs/latest/00-guides/terminology/). Capitalized terms are used according to the canonical definitions in the documentation.

---

## 🎯 Purpose

The goal of this repository is to make the structure, reasoning, and constraints of the infrastructure explicit and durable. It captures:

- canonical infrastructure concepts and semantic models
- architectural boundaries and design decisions
- implementation-facing Stack realizations
- operational behavior and Runtime model
- infrastructure evolution context

Documentation here is organized to remain accurate over time by separating what the infrastructure is *conceptually* from how it is *realized* in implementation.

---

## 👥 Audience

This documentation is intended for:

- infrastructure architects and quantitative developers working on or reasoning about this infrastructure project
- infrastructure and platform engineers who need to understand subinfrastructure boundaries
- contributors who need shared vocabulary and design context before making changes

---

## 🗂️ Repository Structure

```text
.
├── docs/
│   ├── 00-guides/          # Orientation, vocabulary, reading aids
│   ├── 10-architecture/    # Architecture views, principles, and ADRs
│   ├── 20-concepts/        # Canonical semantic models
│   ├── 30-stacks/          # Implementation-facing Stack documents
│   ├── 40-operations/      # Operational model and runbook-facing material
│   ├── 50-evolution/       # Roadmap, milestones, and development history
│   ├── assets/             # Logos and static assets
│   ├── overrides/          # MkDocs theme overrides
│   ├── stylesheets/        # Custom CSS
│   └── javascripts/        # Custom JavaScript
├── mkdocs.yml              # Site configuration
├── requirements-dev.txt    # Python dependencies
├── CONTRIBUTING.md
└── README.md
```

---

## 📚 Documentation Model

The documentation is numbered to encode reading flow from foundational orientation toward implementation and operational detail.

| Section | Role |
| ----------------- | ----------------- |
| `00-guides`       | Entry point — architecture map, documentation structure, terminology, and philosophy. Start here. |
| `10-architecture` | Infrastructure-level architecture views, logical and physical structure, infrastructure flows, and Architecture Decision Records (ADRs). |
| `20-concepts`     | Canonical semantic definitions — the Event model, State model, Determinism model, Time model, Order lifecycle, Queue semantics, and related invariants. This is the authoritative source for exact meaning. |
| `30-stacks`       | Implementation-facing realizations of the conceptual model, organized by Stack. Stack documents describe what each subinfrastructure does, how it is structured, and what its boundaries are, without redefining canonical semantics. |
| `40-operations`   | Operational model, monitoring, recovery, and runbook-facing material. |
| `50-evolution`    | Roadmap, milestones, and development history. |

The `20-concepts` section defines what the infrastructure *means*. The `30-stacks` section documents how those meanings are realized. Other sections orient, record decisions, or describe operational and evolutionary context.

Each Stack in `30-stacks` follows a standard document structure: overview, scope and role, interfaces, internal structure, operational behavior, and implementation notes.

---

## 🛠️ Local Development

The site is built with MkDocs and the Material for MkDocs theme.

### Install dependencies

```bash
pip install -r requirements-dev.txt
```

### Serve locally

```bash
mkdocs serve
```

Then open `http://127.0.0.1:8000`.

### Build the static site

```bash
mkdocs build
```

---

## 🚢 Publishing

The site is published to GitHub Pages as a versioned documentation site using `mike`.

### Release workflow

Documentation is deployed from Git tags or manually, not from every push to `main`.

Typical flow:

1. Create your documentation changes in a feature branch.
2. Open and merge a pull request into `main`.
3. Create and push a version tag, for example:

```bash
git checkout main
git pull
git tag 0.1.0
git push origin 0.1.0
```

Pushing the tag triggers the GitHub Actions workflow, which deploys that documentation version with `mike`, updates the `latest` alias, and sets `latest` as the default version.

### Manual local deployment

To deploy a version manually from your local environment:

```bash
mike deploy --update-aliases <version> latest
mike set-default latest
```

Example:

```bash
mike deploy --update-aliases 0.1.0 latest
mike set-default latest
```

### Local preview of versioned docs

To preview the versioned site locally:

```bash
mike serve
```

### Notes

- Use version strings like `0.1.0` rather than `v0.1.0` for documentation versions unless you intentionally want your documentation version names to match Git tag names with a `v` prefix.
- The GitHub Pages site is published from the `gh-pages` branch managed by `mike`.

---

## 🤝 Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for full guidance.

In brief:

- Contributions should improve clarity, correctness, or completeness of the documentation.
- Canonical semantic definitions live in `20-concepts`. Stack and operational documents should apply those definitions, not redefine them.
- Architectural decisions of significant scope should be documented as ADRs under `10-architecture/adr/`.
- Keep commits small and well scoped. Pull requests should explain what changed and why it improves the documentation.
