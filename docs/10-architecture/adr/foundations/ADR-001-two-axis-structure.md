---
title: Two-Axis Documentation and Architecture Structure
updated: 2026-04-01
adr_status: Accepted
---

# ADR-001: Two-Axis Documentation and Architecture Structure

Status: {{ page.meta.adr_status }}  
Updated: {{ page.meta.updated }}

---

## Context

The Infrastructure's architecture comprises two fundamentally different kinds of knowledge:

1. **Semantic models** — formal definitions of Events, State, Processing Order, Determinism, Intents, Orders, Execution Control, and the invariants that govern them. These define what the Infrastructure *is* and how it *behaves* at a logical level. They are stable across all Runtimes, all Venues, and all deployment configurations.

2. **Implementation structures** — concrete Stacks, deployment topologies, technology choices, operational procedures, and infrastructure constraints. These define how the semantic models are *realized* in specific contexts. They change as infrastructure evolves, as new Venues are added, or as operational requirements shift.

Without an explicit organizational decision, documentation and architectural reasoning mix these two categories. This creates specific problems:

- A change to the Data Storage Stack is misread as a change to Event Model or State derivation rules.
- Stack-specific operational detail leaks into concept definitions, coupling them to a particular deployment.
- Semantic invariants (determinism, Processing Order, `State = f(Event Stream, Configuration)`) become unclear when interleaved with Stack-specific discussion of how a particular Runtime realizes them.
- The boundary between "what must hold everywhere" and "what applies to this Stack" erodes, making it impossible to determine from the documentation alone whether a proposed change is semantic (affecting all Runtimes) or implementation-local.

The core risk is **semantic drift**: the canonical model loses its authority as implementation detail accumulates around it, and the question "is this a rule of the Infrastructure or a property of this Stack?" becomes unanswerable.

This risk is not hypothetical. Trading infrastructures that grow organically tend to produce documentation where the same concept is restated differently in each Stack's context, subtle contradictions accumulate, and no single document is definitively authoritative for a given rule.

---

## Decision

The Infrastructure's documentation and architecture are organized along **two orthogonal axes**.

### Conceptual Axis

Documents on this axis define the Infrastructure's **semantic foundation**: the formal models, invariants, and behavioral rules that hold across all Runtimes and all implementations.

This axis comprises:

- **Concept documents** — Time Model, Event Model, State Model, Determinism Model, Invariants, Queue Semantics, Queue Processing, Intent Dominance, Order Lifecycle, Failure Semantics, Snapshot-Driven Inputs, Ingest vs Decision Frequency
- **Architectural semantics** — Logical Architecture, Infrastructure Flows, Intent Pipeline, Intent Lifecycle, Architecture Principles

Documents on the conceptual axis **must not** depend on or assume any specific Stack, deployment topology, technology choice, or operational procedure. They define what is true of the Infrastructure regardless of how it is deployed.

### Implementation Axis

Documents on this axis define **how the conceptual models are realized** in specific operational contexts.

This axis comprises:

- **Stack documentation** — Data Recording, Data Quality, Data Storage, Backtesting, Live, Analysis, Monitoring
- **Physical Architecture** — deployment topology and infrastructure
- **Operations** — procedures, runbooks, monitoring configuration

Documents on the implementation axis **may reference** conceptual-axis documents. They **must not** redefine or contradict semantic rules established on the conceptual axis.

### The relationship is asymmetric

The conceptual axis is authoritative. Implementation-axis documents derive from and conform to it. The conceptual axis does not derive from or depend on any specific implementation.

When a semantic rule and a Stack-specific description appear to conflict, the conceptual-axis document governs.

---

## Consequences

**Semantic stability under implementation change.** Replacing a storage technology, adding a new Venue Adapter, or restructuring the Backtesting Stack does not require revision of concept documents. The Event Model, Determinism Model, and Invariants remain untouched because they do not reference the implementation being changed.

**Change classification is explicit.** Any proposed change can be classified: does it affect a conceptual-axis document (semantic change, all-Runtimes scope) or only an implementation-axis document (localized within a Stack)? This classification determines review scope and change risk.

**Concept documents are auditable in isolation.** Because concept documents do not contain Stack-specific detail, they can be reviewed for internal consistency, completeness, and correctness without filtering out implementation context that does not belong there.

**Implementation documents have a stable reference point.** Stack documentation references concept documents for semantic rules rather than restating them. This prevents duplicated and potentially divergent restatements of the same invariant in multiple Stacks.

**Cross-referencing is directional.** Implementation-axis documents reference conceptual-axis documents. The reverse direction is avoided: concept documents do not reference specific Stacks, preserving their independence from any particular realization.

---

## Trade-offs

**Bridging documents require editorial judgment.** Documents that span both axes — Architecture Overview, Infrastructure Narrative, Physical Architecture — require case-by-case decisions about which content is semantic and which is implementation-specific. This boundary is not always obvious and demands ongoing discipline.

**Stack documentation is less self-contained.** A reader approaching the Infrastructure from a single Stack must follow references to concept documents for semantic rules rather than finding everything restated locally. This is the intended trade-off: self-containedness at the Stack level would reintroduce the duplication and drift that this decision prevents.

---

## Summary

The documentation is structured along two orthogonal axes — conceptual and implementation — with an asymmetric authority relationship: the conceptual axis defines semantic rules; the implementation axis conforms to them.

This structure prevents semantic drift by ensuring that canonical models are defined once, in concept documents that do not depend on any Stack, and that Stack-level documentation references rather than restates those models.
