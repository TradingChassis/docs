# Documentation Structure

---

## Purpose

This document explains how the documentation repository is organized and how to navigate it.

It describes section roles, reading paths, and document types. It does **not** redefine semantics.

---

## Repository map

The documentation is organized into numbered top-level groups:

- `00-guides/` - orientation, shared vocabulary, and repository-level reading aids
- `10-architecture/` - architecture views, boundaries, and design records
- `20-concepts/` - canonical conceptual models and semantic definitions
- `30-stacks/` - implementation-facing stack realizations of the conceptual model
- `40-operations/` - operational model, monitoring, recovery, and runbook-facing material
- `50-evolution/` - roadmap, milestones, and development-history context

Numbering encodes reading flow from foundational orientation to implementation, operations, and future change.

---

## Section roles

### `00-guides` - Orientation and navigation

Use this section to enter the documentation set:

- [architecture-map.md](architecture-map.md) for macro navigation across architecture topics
- [documentation-structure.md](documentation-structure.md) for repository layout and reading paths
- [documentation-philosophy.md](documentation-philosophy.md) for documentation governance principles
- [terminology.md](terminology.md) for shared term definitions used across documents

This section reduces onboarding cost and keeps cross-document language aligned.

### `10-architecture` - Infrastructure architecture views

Use this section to understand architectural composition and responsibility boundaries:

- overview and narrative documents for high-level framing
- logical and physical architecture for component boundaries and deployment shape
- flow and lifecycle-adjacent architecture documents for runtime sequencing context
- ADRs under `10-architecture/adr/` for decision history and rationale

This section explains how the Infrastructure is organized without replacing canonical concept definitions.

### `20-concepts` - Canonical semantics

Use this section for authoritative definitions of core models (for example, Event, State, Time, Determinism, Invariants, lifecycle semantics, and related canonical concept documents).

This is the primary reference layer when exact meaning matters. Other sections should apply these definitions, not compete with them.

### `30-stacks` - Implementation-facing realization

Use this section to see how conceptual and architectural models are realized per Stack (data recording, storage, Backtesting, Live, analysis, and adjacent Stack domains).

Stack documents are applied views: they explain implementation shape and constraints while remaining aligned with canonical concepts.

### `40-operations` - Operating the Infrastructure

Use this section for operational concerns: observability, monitoring, recovery, incident handling, and runbook-oriented material.

Operations documents are runtime-practice references, not the source of semantic definitions.

### `50-evolution` - Planned and historical change

Use this section for forward and backward context:

- where the Infrastructure is heading (roadmap, milestones)
- what changed over time (development logs and evolution notes)

This section provides planning and change traceability without redefining current canonical behavior.

---

## Document role types

Not all documents serve the same function. Use role type to decide authority:

- **Canonical definitions** - exact semantic meaning and invariants (primarily in `20-concepts/`, plus shared vocabulary in [terminology.md](terminology.md))
- **Conceptual bridges** - connect canonical concepts to architectural reasoning (for example, architecture principles and architecture-level syntheses)
- **Overviews and navigation aids** - orient readers and summarize structure; these should not introduce new semantics
- **Applied implementation documents** - stack-level realization under concrete constraints (`30-stacks/`)
- **Operational documents** - operating, monitoring, and recovery practices (`40-operations/`)
- **Decision-history documents** - ADRs and evolution artifacts (`10-architecture/adr/`, `50-evolution/`)

When a statement appears in multiple role types, canonical definition documents take precedence for semantic interpretation.

---

## Reading paths

### New reader path

Start here for a full understanding:

1. [architecture-map.md](architecture-map.md)
2. [architecture-overview.md](../10-architecture/architecture-overview.md)
3. [logical-architecture.md](../10-architecture/logical-architecture.md)
4. [concepts-overview.md](../20-concepts/concepts-overview.md)
5. core concept documents in `20-concepts/` (Time, Event, State, Determinism, Invariants)
6. relevant stack and operations sections as needed

### Precision reference path

When you need exact definitions:

1. [terminology.md](terminology.md)
2. target canonical document in `20-concepts/`
3. architecture document in `10-architecture/` for boundary context
4. stack or operations document only for applied behavior

### Contributor path

Before editing:

1. identify the target document role type
2. verify authoritative source documents for any reused concepts
3. update adjacent overview/bridge documents only if role-consistent
4. confirm no cross-document contradiction was introduced

---

## How structure prevents duplication and semantic drift

The structure controls drift by assigning each concern to a primary home:

- semantic definitions live in canonical concept documents;
- architecture documents apply and compose those definitions;
- Stack and operations documents describe realization and operation;
- overviews summarize and route readers.

Because roles are separated, contributors can update implementation or operational material without accidentally redefining core semantics. Conversely, semantic changes can be propagated intentionally rather than leaking through summary text.

This is the main function of the repository structure: stable meaning with evolving implementation.

---

## Boundaries of this document

This document explains information architecture and navigation only. It does not:

- define semantics in detail;
- duplicate [documentation-philosophy.md](documentation-philosophy.md);
- replace architecture, concept, stack, or operations source documents.
