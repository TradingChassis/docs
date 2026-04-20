# Ingest Frequency vs Decision Frequency

---

## Purpose

This document explains the distinction between **ingest frequency** and **decision frequency** in the Infrastructure and how both remain grounded in the canonical event-driven model.

It establishes:

- what each concept means at a semantic level;
- why they need not be equal;
- how lower decision frequency coexists with higher ingest frequency without introducing a separate runtime clock, hidden scheduler truth, or out-of-band State transitions;
- what constraints apply to any difference between the two.

Capitalized terms are used as in [Terminology](../00-guides/terminology.md).

---

## Scope

**In scope:**

- The semantic distinction between Event arrival rate and Strategy evaluation rate
- How decision cadence is expressed within the canonical Event-driven model
- Constraints ensuring that reduced decision frequency does not break determinism or replayability

**Out of scope:**

- Specific batching or throttling implementations
- Timer architectures, polling intervals, or scheduler designs
- Queue Processing, Risk evaluation, or Execution Control cadence — those are downstream concerns addressed in [Queue Processing](queue-processing.md)
- Specific Event categorization or filtration rules (implementation choices)

---

## Core distinction: ingest frequency vs decision frequency

### Ingest frequency

**Ingest frequency** is the rate at which **Events** enter the **Event Stream** and are processed by the Infrastructure. It reflects the arrival rate of external occurrences — market data updates, execution reports, control signals — as represented by their **Processing Order** positions in the stream.

Every Event in the stream is applied in **Processing Order**. Each applied Event updates derived **State**. Ingest frequency therefore sets the rate at which derived **State** is updated.

### Decision frequency

**Decision frequency** is the rate at which **Strategy** is evaluated — i.e., the rate at which the Infrastructure invokes Strategy to read derived **State** and potentially emit **Intents**.

Strategy evaluation does **not** occur on every Event. Strategy emits zero or more **Intents** per evaluation, and evaluation itself occurs on a subset of processing steps. That subset is determined by semantic relevance: which Events, under which derived State conditions, warrant Strategy being consulted.

### The relationship

Ingest frequency and decision frequency are **independent** of each other. A infrastructure may process thousands of Events per second (high ingest frequency) while Strategy evaluates on only a fraction of those processing steps (lower decision frequency).

This follows from the model: Strategy produces commands only when relevant conditions exist, not on every State update.

---

## Relationship to Event processing and State derivation

All Events in the **Event Stream** are processed in **Processing Order** regardless of whether any given Event triggers Strategy evaluation. Derived **State** is updated at every stream position:

`State(n) = f(Event Stream[0..n], Configuration)`

This means that between Strategy evaluations, derived **State** continues to advance through every processed Event. When Strategy next evaluates, it reads **fully current derived State** — a projection that reflects all Events up to and including the triggering Event. No State is stale; no Event is skipped for the purposes of State derivation.

Strategy evaluation is a **step that may or may not occur within a given processing step**, not a parallel process running at its own rate. Whether it occurs is a semantic property of the current Event and current derived State under **Configuration** — not a property of wall-clock time.

---

## Decision cadence as a semantic choice over derived State

Whether Strategy is evaluated on a given processing step is determined by **what the Event represents** and **what derived State reveals at that position** — both of which are fully defined by the Event Stream and Configuration.

**Canonical-compatible forms of reduced decision cadence:**

- **Event-type relevance:** Strategy evaluates only on Events of a specific semantic category (e.g., market snapshot Events rather than every incremental update). Which categories are decision-relevant is a **Configuration** choice.

- **State-condition relevance:** Strategy evaluates when a derived State condition has changed in a way relevant to the Strategy's logic (e.g., an order book condition threshold crossed). The condition itself is derived from the Event Stream and is evaluated deterministically.

- **Derived position tracking:** If Strategy is to evaluate every Nth Event, a counter maintained as **derived State** (incremented by each processed Event as part of State derivation) can serve as the condition. This counter is part of derived State — not a hidden mutable truth — and its progression is fully deterministic and replayable.

**The common property of all valid forms:** the trigger for Strategy evaluation is a **deterministic function of the Event Stream and Configuration**. It is an emergent property of the processing, not an external clock signal.

**What is not permitted:**

- A background timer or `sleep`-based interval that fires "evaluate Strategy every N milliseconds." This is a separate runtime tick — explicitly forbidden by the canonical model ([Invariants: EC2](invariants.md), [Time Model](time-model.md)).
- A wall-clock check ("if `now() - last_decision_time > threshold`, evaluate") that makes the decision depend on real-time elapsed duration. Wall-clock time must not determine internal causality or trigger State transitions ([Time Model: Causality rule](time-model.md#causality-rule)).
- Any hidden mutable state, maintained outside the Event Stream and Configuration derivation path, that tracks when the next decision should occur. Hidden mutable truth is forbidden ([Invariants: D3](invariants.md)).

---

## Compatibility with replayability and determinism

Reduced decision cadence is compatible with determinism and replayability if and only if the trigger for Strategy evaluation is itself a **deterministic function of Event Stream + Configuration**.

**If it is:**

Replay of the same Event Stream under the same Configuration produces Strategy evaluation at identical stream positions. Decision frequency is a property of the replay input — not of the environment in which replay runs. Backtesting and Live will reach the same decision points at the same stream positions, since both apply the same triggering logic to equivalent Event Streams.

**If it is not:**

If Strategy evaluation is triggered by wall-clock time or any mechanism outside the Event Stream, two replays of the same stream may evaluate Strategy at different positions, producing different Intents and ultimately different derived State. This breaks determinism.

**Normative statement:** Decision frequency must be expressed as a property of the Event Stream and Configuration. It must not be an independent time-based property of the runtime environment.

---

## Constraints and non-goals

### Constraints

**I1 — Strategy evaluation must not be driven by an independent clock.**
No timer, scheduler, or wall-clock interval may serve as the authority for when Strategy is evaluated. The only valid trigger is the occurrence of a specific Event or a specific derived State condition, both of which are determined by the Event Stream and Configuration.

**I2 — Derived State must be current at the point of Strategy evaluation.**
When Strategy evaluates, it reads derived State that is up to date through the triggering Event's **Processing Order** position. The Infrastructure must not present Strategy with a stale State projection from a prior position as if it were current.

**I3 — Decision cadence must not alter Processing Order.**
Reduced decision frequency does not imply that non-triggering Events are held back, batched, or reordered. Every Event is applied in **Processing Order** and updates derived State. Decision cadence only affects when Strategy is consulted — not how Events are sequenced.

**I4 — Hidden state tracking decision timing is forbidden.**
Any mutable store that tracks "time since last decision" or "Events since last evaluation" outside of derived State is hidden mutable truth ([Invariants: D3](invariants.md)). Where such tracking is needed, it must be maintained as derived State — a value computed from the Event Stream and Configuration.

### Non-goals

- This document does not define which specific Event categories trigger Strategy evaluation in the Infrastructure. That is a Configuration-level implementation choice.
- This document does not prescribe minimum or maximum decision frequencies. These are Strategy-specific and context-specific design decisions.
- This document does not address how Queue Processing, Risk evaluation, or Venue Adapter pacing relate to ingest frequency. Those concerns are addressed in their respective documents.

---

## Boundaries to other documents

| Document | Relationship |
| -------- | ------------ |
| [Time Model](time-model.md) | Defines Processing Order as the causal axis; prohibits wall-clock-driven State transitions; establishes Event Time as metadata |
| [State Model](state-model.md) | Defines `State = f(Event Stream, Configuration)`; State is updated at every stream position regardless of decision frequency |
| [Infrastructure Flows](../10-architecture/infrastructure-flows.md) | Defines Strategy evaluation as step 3 of the processing sequence; Strategy "emits zero or more Intents" per step |
| [Determinism Model](determinism-model.md) | Defines what breaks determinism; wall-clock-dependent branching and hidden mutable state are the relevant failure modes here |
| [Invariants](invariants.md) | E1 (Events sole source of State transitions), E4 (Processing Order), D2 (no wall-clock-dependent branching), D3 (no hidden mutable state) |
| [Queue Processing](queue-processing.md) | Downstream execution-control cadence; distinct from decision cadence and outside the scope of this document |

---

## Open boundaries intentionally left abstract

1. **Which Event categories trigger Strategy evaluation.** The canonical model defines that Strategy evaluation is triggered by Events under Configuration, but the specific categories or conditions that constitute a "decision-relevant" Event for a given Strategy are implementation and Strategy-design choices.

2. **How decision-relevance is encoded in Configuration.** Whether triggering rules are expressed as Event type filters, derived State predicates, derived counters, or other Configuration structures is an implementation decision. The canonical constraint is that the mechanism must be a deterministic function of Event Stream + Configuration.

3. **Multiple Strategies with different decision cadences.** The canonical model supports Strategy as a logical role; whether multiple Strategy instances run concurrently with different cadences, and how their outputs are composed, is an implementation and architectural decision beyond this document's scope.
