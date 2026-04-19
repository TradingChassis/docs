# TradingChassis Documentation

This documentation is the technical reference for a deterministic, event-driven trading system.

It focuses on architecture, semantics, implementation structure, and operations.  
It does not focus on proprietary strategy logic or alpha design.

---

## What this system is

At a high level, the Infrastructure is designed for:

- event-driven runtime behavior with deterministic processing
- Research and Backtesting grounded in the same conceptual model as Live operation
- explicit separation of architecture concerns (semantics, runtime architecture, stack realization, operations)

For the architecture map of the full system, start with [Architecture Map](00-guides/architecture-map.md).

---

## What this documentation contains

The repository is organized into six major sections:

- **Guides** (`00-guides`) - orientation, terminology, and navigation aids
- **Architecture** (`10-architecture`) - system-level structure, boundaries, and ADRs
- **Concepts** (`20-concepts`) - canonical semantic models and invariants
- **Stacks** (`30-stacks`) - implementation-facing stack overviews
- **Operations** (`40-operations`) (work in progress) - monitoring, recovery, incident, and runbook-oriented material
- **Evolution** (`50-evolution`) (work in progress) - roadmap, milestones, and development trajectory

For a dedicated structure explanation, see [Documentation Structure](00-guides/documentation-structure.md).

---

## Start here by goal

### Understand the Infrastructure conceptually

1. [Architecture Map](00-guides/architecture-map.md)
2. [System Narrative](10-architecture/system-narrative.md)
3. [Architecture Overview](10-architecture/architecture-overview.md)

### Understand runtime architecture

1. [Logical Architecture](10-architecture/logical-architecture.md)
2. [System Flows](10-architecture/system-flows.md)
3. [Physical Architecture](10-architecture/physical-architecture.md)
4. [Architecture Principles](10-architecture/architecture-principles.md)

### Understand core semantics (authoritative definitions)

1. [Terminology](00-guides/terminology.md)
2. [Concepts Overview](20-concepts/concepts-overview.md)
3. [Time Model](20-concepts/time-model.md)
4. [Event Model](20-concepts/event-model.md)
5. [State Model](20-concepts/state-model.md)
6. [Determinism Model](20-concepts/determinism-model.md)
7. [Invariants](20-concepts/invariants.md)

### Find implementation, stack details and operations material
 
1. relevant stack section under [Stacks](30-stacks/stacks-overview.md)
2. related ADRs under [Architecture](10-architecture/adr/system-foundations/ADR-001-two-axis-structure.md)
3. monitoring / recovery / incident and runbook documents under [Operations](40-operations/operations-overview.md)

---

## How to use this documentation effectively

- Use overview pages to orient quickly.
- Use concept documents for exact semantic definitions.
- Use architecture documents for component boundaries and sequencing.
- Use stack and operations documents for applied implementation and operating context.

When in doubt, treat concept documents as semantic authority and use overview pages as navigational summaries.
