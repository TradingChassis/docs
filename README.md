# TradingChassis — Trading Infrastructure Documentation

This repository contains the architecture documentation for the TradingChassis trading infrastructure. It is the canonical reference for the infrastructure's concepts, architectural decisions, implementation-facing stack descriptions, and operational model.

---

## Purpose

The goal of this repository is to make the structure, reasoning, and constraints of the Infrastructure explicit and durable. It captures:

- canonical infrastructure concepts and semantic models
- architectural boundaries and design decisions
- implementation-facing stack realizations
- operational behavior and runtime model
- infrastructure evolution context

Documentation here is organized to remain accurate over time by separating what the Infrastructure is *conceptually* from how it is *realized* in implementation.

---

## Audience

This documentation is intended for:

- infrastructure architects and quantitative developers working on or reasoning about the Infrastructure
- infrastructure and platform engineers who need to understand subinfrastructure boundaries
- contributors who need shared vocabulary and design context before making changes

---

## Repository Structure

```text
.
├── docs/
│   ├── 00-guides/          # Orientation, vocabulary, reading aids
│   ├── 10-architecture/    # Architecture views, principles, and ADRs
│   ├── 20-concepts/        # Canonical semantic models
│   ├── 30-stacks/          # Implementation-facing stack documents
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

## Documentation Model

The documentation is numbered to encode reading flow from foundational orientation toward implementation and operational detail.

| Section           | Role                                                                                                                                                                                                                          |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `00-guides`       | Entry point — architecture map, documentation structure, terminology, and philosophy. Start here.                                                                                                                             |
| `10-architecture` | Infrastructure-level architecture views, logical and physical structure, infrastructure flows, and Architecture Decision Records (ADRs).                                                                                                      |
| `20-concepts`     | Canonical semantic definitions — the Event model, State model, Determinism model, Time model, Order lifecycle, Queue semantics, and related invariants. This is the authoritative source for exact meaning.                   |
| `30-stacks`       | Implementation-facing realizations of the conceptual model, organized by Stack. Stack documents describe what each subinfrastructure does, how it is structured, and what its boundaries are, without redefining canonical semantics. |
| `40-operations`   | Operational model, monitoring, recovery, and runbook-facing material.                                                                                                                                                         |
| `50-evolution`    | Roadmap, milestones, and development history.                                                                                                                                                                                 |

The `20-concepts` section defines what the Infrastructure *means*. The `30-stacks` section documents how those meanings are realized. Other sections orient, record decisions, or describe operational and evolutionary context.

Each Stack in `30-stacks` follows a standard document structure: overview, scope and role, interfaces, internal structure, operational behavior, and implementation notes.

---

## Local Development

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

## Publishing

The site is published to GitHub Pages as a versioned documentation site using `mike`.

### Release workflow

Documentation is deployed from Git tags, not from every push to `main`.

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

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for full guidance.

In brief:

- Contributions should improve clarity, correctness, or completeness of the documentation.
- Canonical semantic definitions live in `20-concepts`. Stack and operational documents should apply those definitions, not redefine them.
- Architectural decisions of significant scope should be documented as ADRs under `10-architecture/adr/`.
- Keep commits small and well scoped. Pull requests should explain what changed and why it improves the documentation.

---

## License

Documentation content is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) unless otherwise stated.

> This repository documents infrastructure architecture and engineering concepts. It does not constitute financial advice.
