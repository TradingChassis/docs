---
title: Explicit Lifecycle State Machines
updated: 2026-04-01
adr_status: Accepted
---

# ADR-006: Explicit Lifecycle State Machines

Status: {{ page.meta.adr_status }}  
Updated: {{ page.meta.updated }}

---

## Context

The Infrastructure processes Intents (ephemeral commands from Strategy), evaluates them against policy (Risk), schedules them for dispatch (Queue Processing), transmits them to Venues (Venue Adapter), and tracks the resulting Orders through Venue feedback (Execution Events). This processing chain involves multiple components, each responsible for a different stage.

Without explicit lifecycle definitions, the following problems arise:

- **Fragmented lifecycle logic.** If no single model defines the valid states and transitions for an Intent or an Order, each component must independently encode its assumptions about what states exist and which transitions are legal. These assumptions diverge over time.
- **Ambiguous boundaries.** Without a clear statement of where one lifecycle ends and another begins, the distinction between pre-submission Intent handling and post-submission Order tracking erodes. Components begin treating Intent stages as Order states or vice versa — a confusion that produces incorrect behavior and misleading diagnostics.
- **Inconsistent transition enforcement.** If lifecycle transitions are implicit in conditional logic distributed across Strategy, Risk, Queue Processing, and Venue Adapter, there is no single point where a transition can be validated as legal. Invalid transitions may go undetected until they produce downstream failure.
- **Harder invariant validation.** The Infrastructure's determinism and replayability depend on every lifecycle transition being a deterministic function of the Event Stream and Configuration. If lifecycle rules are implicit, verifying this property requires auditing every component rather than validating a single explicit model.

The architecture requires explicit lifecycle state machines that define, for each relevant domain, the valid states, valid transitions, and the boundary conditions that separate one lifecycle from another.

---

## Decision

The Infrastructure models two primary lifecycle domains as explicit state machines: the **Intent lifecycle** and the **Order lifecycle**. These are distinct, non-overlapping lifecycles with a clear boundary at submission.

### Intent lifecycle

The Intent lifecycle defines the progression of an ephemeral command from Strategy output through policy decision, execution-control residency, dispatch, and terminal disposition.

**Stages:**

| Stage | Meaning |
| ----- | ------- |
| **Generated** | Strategy has emitted the command. |
| **Policy decided** | Risk has classified the command as allowed or denied. |
| **Pending dispatch** | Allowed and held in derived execution-control substate (Queue), awaiting selection. |
| **Dispatched** | Handed to the Venue Adapter for outbound transmission. |
| **Inflight** | Outbound request is outstanding; awaiting completion. |
| **Closed** | Terminal — no further progression for this Intent arc. |

**Terminal outcomes:** Policy rejection (denied → Closed), supersession by dominance (Pending dispatch → Closed), or completion after dispatch (Inflight → Closed).

The Intent lifecycle is a **pre-submission** state machine. No Order exists during any Intent stage. The boundary between the two lifecycles is **submission**: the moment the Venue Adapter transmits the outbound request.

### Order lifecycle

The Order lifecycle defines the progression of a derived entity in Execution State from submission through Venue interaction to terminal disposition.

**Stages:**

| Stage | Meaning |
| ----- | ------- |
| **Submitted** | The outbound request has been transmitted; the Order exists in Execution State. |
| **Accepted** | The Venue has acknowledged the Order. |
| **Partially Filled** | One or more fills have been applied; residual quantity remains. |
| **Filled** | Fully executed. Terminal. |
| **Cancelled** | Terminated before full execution. Terminal. |
| **Rejected** | The Venue did not accept the Order after submission. Terminal. |

**Entry condition:** `Submitted` is the first Order state. An Order comes into existence at submission — not at Intent generation, not at Risk evaluation, not at Queue admission. Prior stages belong to the Intent lifecycle.

**Transition driver:** All post-submission transitions are driven by Execution Events (Venue reports) processed in Processing Order. No Order transition occurs outside Event processing.

### The two lifecycles are distinct

Intent lifecycle and Order lifecycle must not be collapsed. No Intent stage is an Order state. No Order state is an Intent stage. The boundary between them is submission: the Intent arc may reach Dispatched and then Closed; the Order arc begins at Submitted and progresses through Venue feedback. Invariants L2, L3, and L5 enforce this separation.

### Execution-control mechanisms are not separate lifecycle state machines

Queue residency, dominance, eligibility, inflight gating, rate-limit evaluation, and scheduling are **execution-control derivations** within Event processing. They are important mechanisms that govern how Intents progress through the Intent lifecycle (specifically through the Pending dispatch stage), but they are not elevated as separate top-level lifecycle state machines alongside Intent and Order. They operate within the Intent lifecycle's execution-control stages, and they are derived from Event Stream + Configuration — not independently authoritative.

---

## Consequences

**Lifecycle transitions are enumerable and validatable.** Each lifecycle defines a finite set of valid states and transitions. A transition that does not appear in the model is invalid by definition. This makes it possible to validate lifecycle correctness by checking every transition against the explicit model, rather than auditing every component's conditional logic.

**The Intent/Order boundary is architecturally enforced.** Because the two lifecycles are explicit and non-overlapping, components cannot accidentally treat an Intent stage as an Order state. Strategy, Risk, and Queue Processing operate on Intent stages; Execution State projections reflect Order stages. The boundary at submission is unambiguous.

**Venue-specific behavior is abstracted.** The Order lifecycle defines canonical internal states. Venues may use different state representations, status codes, or transition semantics. The Venue Adapter translates Venue-specific responses into canonical Execution Events that drive the Order lifecycle. The explicit model ensures that Order semantics are consistent across Venues and across Runtimes.

**Deterministic replay covers both lifecycles.** Because both state machines advance only through Event processing (Intent-related Events for the Intent lifecycle where canonical history requires them; Execution Events for the Order lifecycle), replay of the same Event Stream under the same Configuration reproduces identical lifecycle state at every Processing Order position.

**Execution-control logic is scoped within the Intent lifecycle.** Queue Processing, dominance, inflight gating, and rate-limit evaluation operate within the Pending dispatch stage of the Intent lifecycle. They do not form a parallel lifecycle with their own state machine. This keeps the overall lifecycle model simple: two explicit state machines with a clear boundary, and execution-control derivations scoped within one of them.

---

## Trade-offs

**Lifecycle definitions must be maintained as explicit models.** Every valid state and transition must be defined and kept consistent across documentation, implementation, and validation. This is more upfront effort than leaving lifecycle logic implicit — but implicit lifecycle logic accumulates inconsistencies that are harder to detect and more costly to repair.

**The two-lifecycle model requires a clear submission boundary.** The decision to separate Intent and Order at submission means every component must respect this boundary. A component that conflates pre-submission and post-submission handling violates the lifecycle model. This is a constraint on implementation, but it is also the mechanism that prevents the ambiguity the decision was designed to eliminate.

---

## Summary

The Infrastructure models Intent lifecycle and Order lifecycle as two explicit, non-overlapping state machines separated at submission. The Intent lifecycle covers command progression from Strategy output through policy, execution control, and dispatch. The Order lifecycle covers execution progression from submission through Venue feedback to terminal disposition. Queue Processing and related execution-control mechanisms operate within the Intent lifecycle, not as separate top-level state machines. All lifecycle transitions are driven by Event processing, making both lifecycles deterministic and replayable.
