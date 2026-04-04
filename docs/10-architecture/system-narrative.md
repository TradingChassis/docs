# System Narrative

---

## Purpose

The System Narrative provides architectural context for the trading infrastructure described in this documentation.

While the concept documents define the formal models governing System behavior (Events, State, Time, Determinism), this document explains the broader architectural motivation and tells the **story of how the System works** from Event recording through State derivation, Strategy decision, policy evaluation, Execution Control, and Venue feedback.

This document is intentionally narrative in style. Normative definitions and invariants belong in the concept documents it references; this document makes the overall model coherent, readable, and grounded.

---

## Problem Space

Modern trading systems must simultaneously satisfy several competing requirements:

- **reproducibility** of Research results
- **deterministic behavior** under identical inputs
- **reliable interaction** with Venues
- **operational safety** under Live Execution conditions
- **scalable recording** and Analysis of market data

A trading architecture may evolve organically, possibly resulting in systems where Research diverges significantly from Live trading. Such divergence leads to Strategies behaving differently in simulation than in production, making Research results unreliable as predictors of Live behavior.

This System addresses the problem by enforcing a **unified conceptual model** across all Runtimes. Backtesting and Live Execution share the same Core Runtime semantics. The System guarantees that if a Strategy behaves a certain way against a given Event Stream in Backtesting, it will behave the same way against the same Event Stream in Live — because the processing model is identical.

---

## Microstructure Context

The System is designed primarily for microstructure-driven trading Strategies.

Microstructure refers to the behavior of markets at the level of the order book and individual trade Events, including:

- limit Orders and cancellations
- market Orders and liquidity consumption
- bid/ask spread dynamics
- order book depth and position within the book
- short-term liquidity imbalances

Strategies operating in this domain depend heavily on realistic modeling of Execution behavior, latency and Event ordering, Order lifecycle transitions, and market interaction through order books.

Because of this, the infrastructure must accurately reproduce the interaction between trading logic and the market. The System architecture therefore prioritizes **deterministic Event processing** and **realistic Execution modeling** — the two properties that make Research results transferable to Live.

---

## System Scope

The System provides the infrastructure required to support trading Research and Live Execution.

Its responsibilities include:

- market data recording, validation, and normalization
- deterministic Backtesting infrastructure
- event-driven Execution from Strategy output through Venue interaction
- policy enforcement via the Risk Engine
- operational monitoring and Analysis

The System deliberately does not define Strategies. Strategy logic is an external Component that reads State projections and emits commands into the System through well-defined interfaces.

Venues represent system boundaries: the System sends outbound requests to Venues and ingests their responses as Events.

---

## How the System works

The System is deterministic and event-driven. Its behavior is fully defined by two canonical inputs: the **Event Stream** and **Configuration**. Everything else — State, dispatch decisions, Order projections — is derived from these inputs.

### Events and State

Everything begins with an **Event**: an immutable record of something the System must process. Events arrive in **Processing Order** — the strict internal sequence that defines causality. Event Time (the timestamp an Event carries) is metadata; it does not determine when the System processes the Event.

When the System applies an Event, it derives updated **State**:

`State = f(Event Stream, Configuration)`

State is not a mutable store owned by any Component. It is a **deterministic projection** of the Event Stream and Configuration, recomputable at any point by replaying the Event Stream and Configuration from the beginning. No Component holds parallel mutable truth. The Event Stream, together with Configuration, is the only authoritative source of System history.

State is organized into three domains: **Market State** (current market conditions), **Execution State** (Orders, fills, positions, and execution-control substate), and **Control State** (operational flags and Runtime signals). These three domains together constitute the full derived condition of the System at any Event Stream position.

### Strategy and Intents

With current State derived, **Strategy** evaluates its position and decides what it wants to do. Strategy reads projections of derived State — market conditions, current Order exposure, positions — and produces **Intents**: ephemeral commands expressing desired trading actions (create a new Order, replace an existing one, cancel).

An Intent is a **command**, not an Event. It is not appended to the Event Stream. It does not change State. It exists only as transient input to the current processing step. If Strategy produces no Intent, the processing step continues without one; if Strategy produces an Intent, it moves immediately to the next stage.

This distinction matters: because Intents are ephemeral commands rather than persistent records, they carry no history, create no State on their own, and impose no commitment on the System until they have passed through policy evaluation and Execution Control.

### Risk: policy only

Every Intent is evaluated by the **Risk Engine** before anything further happens. The Risk Engine answers exactly one question: **is this Intent admissible under policy?**

The answer is binary: **allowed** or **denied**. An allowed Intent proceeds; a denied Intent does not.

The Risk Engine does **not** decide when to send the Intent, in what order, or whether current rate limits permit transmission. Those are execution-control questions, not policy questions. Risk stops at admissibility. The boundary between policy and Execution Control is strict, and the System enforces it.

Where a policy decision must be part of the canonical record (for replay or audit), the outcome appears as an **Intent-related Event** on the Event Stream.

### Execution Control: Queue and Queue Processing

An allowed Intent does not go directly to the Venue. It enters **Execution Control** — the **Queue** and **Queue Processing** subsystem — which schedules and transmits allowed work.

The **Queue** is **derived execution-control substate** within **Execution State**: a projection of current effective pending outbound work, recomputable from the Event Stream and Configuration ([Queue Semantics](../20-concepts/queue-semantics.md)).

**Queue Processing** runs as a deterministic computation **within Event processing**. There is no separate runtime tick, no background scheduling loop, no independent clock. When the System processes an Event, that same processing step also evaluates the Queue: it applies dominance rules (ensuring at most one effective pending command per logical Order key), checks inflight status, evaluates rate limits, and selects which allowed commands may be dispatched in this step.

The result is that every execution-control decision is a pure, deterministic function of the current Event Stream and Configuration. Given the same inputs, the System will always produce the same dispatch decisions at the same Event Stream position.

### Orders begin at submission

When Queue Processing selects an Intent for dispatch, it is handed to the **Venue Adapter** for outbound transmission. At this moment — **submission** — an **Order** comes into existence in **Execution State**. `Submitted` is the first Order state.

Nothing before dispatch constitutes an Order. Intent generation, Risk acceptance, and Queue residency are all pre-submission phases of the **Intent lifecycle**. The **Order lifecycle** is separate and begins only at this boundary.

After submission, Order state evolves exclusively through **Execution Events** from the Venue: acknowledgements, fills, partial fills, rejections, cancellations. These Events advance an already-existing Order; they do not create it. The Order existed from the moment of dispatch; Venue responses update its lifecycle.

### Venue feedback and re-entry

The Venue returns execution feedback. The **Venue Adapter** surfaces this as **Execution Events** that are appended to the Event Stream. On a subsequent processing step, those Events derive updated Execution State — Order lifecycle transitions, position changes, balance updates — and the cycle continues.

There is no parallel path by which Venue responses update State directly. All State evolution, including Order lifecycle progression, flows through the Event Stream.

---

## Determinism and replayability

The event-driven model described above is the foundation of the System's determinism guarantee.

Because `State = f(Event Stream, Configuration)`, and because all dispatch decisions are deterministic functions of derived State and Configuration, the entire history of the System can be reproduced exactly by replaying the Event Stream under the same Configuration. This applies to Backtesting and Live equally.

This has practical consequences:

- **Research results transfer.** A Strategy validated in Backtesting behaves identically in Live, because both run the same processing model against equivalent Event Streams.
- **Failures recover cleanly.** State after a failure can be reconstructed from the Event Stream without relying on snapshots or parallel mutable stores.
- **Execution can be audited.** Any historical dispatch decision can be reproduced and explained from the Event Stream.

Determinism does not happen by accident. It requires that no component holds hidden mutable State, that no State transition occurs outside Event processing, and that Queue Processing runs within Event processing rather than on an independent clock. These are not implementation constraints — they are **invariants** of the canonical model.

---

## Backtesting and Live

Backtesting and Live are two implementations of the same canonical model.

Both Runtimes apply `State = f(Event Stream, Configuration)`, process the same `Strategy → Risk → Queue → Venue Adapter` chain, and maintain the same Order lifecycle beginning at submission.

What differs is infrastructure: the source of the Event Stream (historical datasets vs live Venue feeds), the Venue implementation (simulated vs real), and the operational characteristics (batch experiments vs continuous operation). The semantic model does not differ.

This equivalence is not accidental. It is the architectural goal that makes Research useful: because the Runtime semantics are identical, a Strategy that behaves well in Backtesting is evaluated against the same logical model it will encounter in Live.

---

## Relationship to other documents

This document tells the architectural story. For normative definitions, precise rules, and invariants, see:

- [Terminology](../00-guides/terminology.md) — canonical definitions of all terms used here
- [Logical Architecture](logical-architecture.md) — component responsibilities and hard boundaries
- [System Flows](system-flows.md) — step-by-step canonical Runtime sequencing
- [Event Model](../20-concepts/event-model.md) — formal definition of Events and the Event Stream
- [State Model](../20-concepts/state-model.md) — `State = f(Event Stream, Configuration)` and State domains
- [Intent Lifecycle](intent-lifecycle.md) — Intent stages from generation to terminal disposition
- [Order Lifecycle](../20-concepts/order-lifecycle.md) — Order stages from submission onward
- [Queue Semantics](../20-concepts/queue-semantics.md) — Queue as derived execution-control substate
- [Queue Processing](../20-concepts/queue-processing.md) — deterministic execution-control evaluation
- [Determinism Model](../20-concepts/determinism-model.md) — what determinism requires and what breaks it
