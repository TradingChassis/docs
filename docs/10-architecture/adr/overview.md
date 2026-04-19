# Architecture Decision Records (ADRs)

---

## Purpose

Architecture Decision Records capture the **architectural decisions** that shape the Infrastructure — the problems that required a decision, what was decided, and why.

System documentation (Architecture, Concepts, Stacks) defines **what the Infrastructure is**: its semantics, component boundaries, processing rules, and realization. ADRs complement that documentation by recording **why** the architecture takes the form it does — the pressures, trade-offs, and constraints that led to each decision.

ADRs are the authoritative source for architectural rationale. They are not duplicated in system docs, and system docs do not replace them.

---

## How to read ADRs

ADRs are **decision records**, not tutorials or system overviews. Each ADR follows a consistent structure:

| Section | What it contains |
| ------- | ---------------- |
| **Context** | The architectural pressure or problem that required a decision |
| **Decision** | The chosen rule, structure, or constraint — stated normatively |
| **Consequences** | What the decision enables, constrains, or requires |
| **Trade-offs** | Genuine costs accepted and alternatives foregone |

**When to read ADRs:**

- **Understanding rationale.** When you need to know *why* the Infrastructure is structured a particular way, not just *what* it does.
- **Evaluating changes.** Before proposing a change that affects an area covered by an ADR, read the ADR to understand the constraints and trade-offs that shaped the current design.
- **Onboarding.** ADRs provide the reasoning layer that system documentation does not repeat. Reading them alongside architecture and concept documents gives a complete picture of the Infrastructure's design.

---

## System Foundations

Decisions that establish the structural and semantic foundations of the Infrastructure.

| ADR | Decision |
| --- | -------- |
| [ADR-001: Two-Axis Documentation and Architecture Structure](system-foundations/ADR-001-two-axis-structure.md) | Documentation and architecture are organized along two orthogonal axes — conceptual and implementation — with the conceptual axis authoritative over the implementation axis, preventing semantic drift. |
| [ADR-002: Canonical Storage as Authoritative Dataset Layer](system-foundations/ADR-002-canonical-storage.md) | Validated datasets are promoted through a staged pipeline to an immutable Canonical Storage layer that all Stacks reference as the single authoritative dataset source — distinct from the Runtime Event Stream. |
| [ADR-003: Event-Derived State Model](system-foundations/ADR-003-event-vs-state-model.md) | The Event Stream is canonical history; State is a deterministic projection of Event Stream and Configuration — not primary truth. This is the structural foundation for deterministic replay and Backtesting/Live parity. |

## Execution Engine

Decisions governing the Core Runtime's execution pipeline — how Intents are processed, validated, scheduled, dispatched, and tracked.

| ADR | Decision |
| --- | -------- |
| [ADR-004: Derived Execution-Control Queue](execution-engine/ADR-004-outbound-queue.md) | Outbound work is reconciled through a derived execution-control Queue that holds at most one effective command per logical order key, using dominance, inflight gating, and rate-limited dispatch within deterministic Event processing. |
| [ADR-005: Mandatory Risk Validation Before Execution Control](execution-engine/ADR-005-risk-validation.md) | Every Intent must pass through the Risk Engine — a mandatory policy gate deciding admissibility only (allowed / denied) — before entering execution-control substate. Risk does not manage dispatch timing or scheduling. |
| [ADR-006: Explicit Lifecycle State Machines](execution-engine/ADR-006-explicit-state-machines.md) | Intent lifecycle and Order lifecycle are modeled as two explicit, non-overlapping state machines separated at submission, with execution-control mechanisms operating within the Intent lifecycle rather than as separate top-level state machines. |
| [ADR-007: Layered Runtime Architecture of the Core](execution-engine/ADR-007-layered-runtime-architecture.md) | The Core Runtime is organized into four layered responsibilities — Strategy, Risk, Execution Control, Venue Adapter — each answering exactly one control question (what / whether / when / how) with strict boundaries. |

## Research Infrastructure

Decisions about tooling and infrastructure that support Strategy Research and Backtesting.

| ADR | Decision |
| --- | -------- |
| [ADR-008: hftbacktest as Simulated Venue](research-infrastructure/ADR-008-hftbacktest-as-venue.md) | Backtesting uses hftbacktest as the simulated Venue behind the Venue Adapter boundary, providing microstructure-level simulation while preserving the same architectural boundary shape as Live. |

---

## Relationship to other documentation

ADRs and system documentation serve different roles:

| Documentation | Role |
| ------------- | ---- |
| **Architecture documents** | Define component boundaries, processing chains, and structural organization — **what** the Infrastructure is. |
| **Concept documents** | Define canonical semantics — Events, State, Determinism, Lifecycles, Invariants — **what** the rules are. |
| **Stack documents** | Define how canonical models are realized in specific operational contexts — **how** it is built. |
| **ADRs** | Record **why** the architecture takes its current form — the decisions, pressures, and trade-offs behind it. |

System documents describe the current architecture without repeating decision rationale. ADRs record decision rationale without restating system definitions. The two are complementary; neither replaces the other.
