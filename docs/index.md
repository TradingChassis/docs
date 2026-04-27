# TradingChassis Documentation

This documentation is the technical reference for TradingChassis:  
an open-source trading infrastructure project for building small-scaled professional Research-to-Production trading systems.

It focuses on architecture, semantics, implementation structure, operations, and project evolution.  
It does not focus on proprietary Strategy logic, alpha design, trading signals, or trading performance claims.

---

## What this infrastructure is

TradingChassis is designed as infrastructure for professional Research-to-Production trading systems.

At a high level, the infrastructure is organized around:

- deterministic Event processing and explicit Processing Order
- State derivation from Event Stream + Configuration
- canonical data flows from raw market data toward validated and reusable datasets
- Research, Backtesting, Analysis, and Live as connected infrastructure contexts
- explicit separation of semantics, Runtime behavior, Stack realization, and operations
- reproducibility, auditability, observability, and operational maintainability
- runbooks, monitoring, recovery context, and maintenance procedures as part of the infrastructure model

Backtesting is treated as part of Research. Live is a different operational context, not a separate conceptual universe. The goal is to make behavior explainable across contexts by grounding it in shared semantics, explicit architecture, and documented operational boundaries.

For the architecture map of the full infrastructure, start with [Architecture Map](00-guides/architecture-map.md).

---

## What this documentation contains

The repository is organized into six major sections:

- **Guides** (`00-guides`) - orientation, terminology, documentation structure, and navigation aids
- **Architecture** (`10-architecture`) - infrastructure structure, boundaries, flows, principles, and Architecture Decision Records
- **Concepts** (`20-concepts`) - canonical semantic models and invariants
- **Stacks** (`30-stacks`) - implementation-facing Stack overviews and subinfrastructure boundaries
- **Operations** (`40-operations`) - monitoring, recovery, incident handling, runbooks, maintenance, and operational context
- **Evolution** (`50-evolution`) - roadmap, milestones, development logs, and architectural progress

For a dedicated structure explanation, see [Documentation Structure](00-guides/documentation-structure.md).

The documentation separates what the infrastructure *means* from how it is *realized* and how it is *operated*:

- **Concepts** define canonical semantics.
- **Architecture** explains boundaries, structure, and sequencing.
- **Stacks** describe implementation-facing realizations.
- **Operations** explains how the infrastructure is monitored, maintained, recovered, and used operationally.
- **Evolution** records how the project changes over time.

---

## Start here by goal

### Understand the infrastructure conceptually

1. [Architecture Map](00-guides/architecture-map.md)
2. [Infrastructure Narrative](10-architecture/infrastructure-narrative.md)
3. [Architecture Overview](10-architecture/architecture-overview.md)

### Understand architecture and boundaries

1. [Logical Architecture](10-architecture/logical-architecture.md)
2. [Infrastructure Flows](10-architecture/infrastructure-flows.md)
3. [Physical Architecture](10-architecture/physical-architecture.md)
4. [Architecture Principles](10-architecture/architecture-principles.md)

### Understand core semantics

1. [Terminology](00-guides/terminology.md)
2. [Concepts Overview](20-concepts/concepts-overview.md)
3. [Time Model](20-concepts/time-model.md)
4. [Event Model](20-concepts/event-model.md)
5. [State Model](20-concepts/state-model.md)
6. [Determinism Model](20-concepts/determinism-model.md)
7. [Invariants](20-concepts/invariants.md)

### Understand Research, Backtesting, Analysis, and Live context

1. [Architecture Map](00-guides/architecture-map.md)
2. [Infrastructure Narrative](10-architecture/infrastructure-narrative.md)
3. relevant Stack documents under [Stacks](30-stacks/stacks-overview.md)
4. related semantic models under [Concepts](20-concepts/concepts-overview.md)

### Find implementation and Stack details

1. relevant Stack section under [Stacks](30-stacks/stacks-overview.md)
2. related architecture views under [Architecture](10-architecture/architecture-overview.md)
3. related ADRs under [Architecture Decision Records](10-architecture/adr/foundations/ADR-001-two-axis-structure.md)

### Understand operations, monitoring, and maintenance

1. [Operations Overview](40-operations/operations-overview.md)
2. monitoring, recovery, incident, and runbook-oriented documents under [Operations](40-operations/operations-overview.md)
3. related Stack documents under [Stacks](30-stacks/stacks-overview.md)
4. related architecture boundaries under [Architecture](10-architecture/architecture-overview.md)

### Understand project direction and evolution

1. [Roadmap](50-evolution/roadmap.md)
2. milestones and development logs under [Evolution](50-evolution/roadmap.md)
3. relevant ADRs under [Architecture Decision Records](10-architecture/adr/foundations/ADR-001-two-axis-structure.md)

---

## How to use this documentation effectively

Use this documentation as a layered technical reference:

- Use **Guides** to orient quickly.
- Use **Terminology** and **Concepts** for exact semantic definitions.
- Use **Architecture** documents for structure, sequencing, and boundaries.
- Use **Stacks** for implementation-facing subinfrastructure views.
- Use **Operations** for monitoring, recovery, runbooks, maintenance, and operating context.
- Use **Evolution** to understand roadmap, milestones, and historical direction.

When in doubt, treat concept documents as semantic authority and use overview pages as navigational summaries.

Stack and operations documents should apply canonical semantics; they should not redefine them locally.
