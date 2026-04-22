# Architecture Principles

---

## Purpose and scope

This document states the **governing architectural principles** of the Infrastructure.

Each principle expresses an enduring design rule that constrains how the Infrastructure is built, extended, and evolved. Principles are normative: they are not aspirations and not preferences. Violating a principle compromises architectural integrity.

This document does not replace the formal concept documents (Event Model, State Model, Determinism Model, Invariants). For normative definitions and invariant statements, see those documents. The principles here express *why* those rules exist and what they commit the architecture to.

Capitalized terms are used as in [Terminology](../00-guides/terminology.md).

---

## P1 — Events are the only source of State Transitions

**All State evolution is caused by processing canonical Events. No other mechanism may change derived State.**

The Infrastructure is event-driven in a strict sense: not merely that it reacts to Events, but that Events are the *sole* causal mechanism for State change. Spontaneous updates, wall-clock triggers, timer-driven branches, and out-of-band writes to State projections are forbidden.

This principle makes behavior fully observable: every State transition has a corresponding Event in the Event Stream that explains it.

➝ [Event Model](../20-concepts/event-model.md), [Invariants: E1](../20-concepts/invariants.md)

---

## P2 — State is a deterministic projection, not owned truth

**Derived State is fully determined by Event Stream + Configuration. No component owns State as independent mutable truth.**

Formally: `State = f(Event Stream, Configuration)`. State is a projection recomputable from the Event Stream under the same Configuration. Components read projections of State; they do not hold authoritative copies that diverge from the stream.

This principle eliminates a class of consistency bugs where a component's internal state disagrees with what the Event Stream would derive. There is one canonical answer at any Event Stream position, and it is always the derived one.

➝ [State Model](../20-concepts/state-model.md), [Determinism Model](../20-concepts/determinism-model.md)

---

## P3 — No hidden mutable truth

**All data that influences canonical behavior must be part of the Event Stream or explicit versioned Configuration. Hidden stores, caches, and out-of-band writes that affect derived State are forbidden.**

This principle is the enforcement mechanism behind P2. It rules out:

- private component state that shadows derived State and feeds future decisions;
- Configuration changes that take effect silently without being part of the canonical input;
- execution-control bookkeeping (inflight tracking, rate-limit state, Queue contents) maintained as authoritative primary stores outside the derivation path.

Queue contents, Order projections, and all execution-control substate are **derived** — recomputable, not primary. A Runtime that cannot reconstruct its State by replaying the Event Stream has violated this principle.

➝ [Determinism Model](../20-concepts/determinism-model.md), [Invariants: D3](../20-concepts/invariants.md)

---

## P4 — Determinism is non-negotiable

**Given identical Event Stream and identical Configuration, the Infrastructure must produce identical State at every stream position — without exception, across all domains, and across all Runtimes.**

Determinism is not a property that holds approximately or under normal conditions. It holds always: for Market State, Execution State (including Orders and execution-control substate), and Control State. Any behavior that depends on wall-clock time, OS scheduling, thread interleaving, or Runtime timing violates this principle.

Determinism is the basis for reproducible Research, reliable failure recovery by replay, and trusted equivalence between Backtesting and Live results.

➝ [Determinism Model](../20-concepts/determinism-model.md), [Invariants: D1–D4](../20-concepts/invariants.md)

---

## P5 — Processing Order is the canonical internal causal order

**State transitions follow Processing Order — the strict internal sequencing of Events in the Event Stream. No external timing mechanism defines the causal sequence.**

Processing Order is not wall-clock time, not arrival order at a network boundary, and not thread-scheduling order. It is the Infrastructure's authoritative determination of which Event is logically next. Every State transition occurs at a specific position in Processing Order, and that position is the canonical identifier of where in the Infrastructure's history that transition belongs.

This principle has two direct consequences:

- **Replay is defined over Processing Order, not wall-clock time.** Two replays of the same Event Stream produce identical State at every Processing Order position, regardless of how quickly each replay runs in real time.
- **Event Time is external metadata, not internal causality.** A Market Event carrying a timestamp from an external feed is processed at its Processing Order position; it does not retroactively reorder prior State transitions.

Processing Order is what makes the Infrastructure's causal history unambiguous. Without it, the same set of Events could produce different State depending on when they were applied in real time — breaking both determinism and replayability.

➝ [Time Model](../20-concepts/time-model.md), [Determinism Model](../20-concepts/determinism-model.md), [Invariants: E4](../20-concepts/invariants.md)

---

## P6 — Policy and Execution Control are strictly separated

**The Risk Engine decides admissibility only. Queue and Queue Processing decide execution timing and ordering only. Neither crosses into the other's domain.**

Policy (Risk) answers: *Is this Intent allowed under the given rules?* The answer is binary: **allowed** or **denied**. Risk does not decide when to send, in what order, whether rate limits permit transmission, or whether inflight constraints block the request.

Execution Control (Queue + Queue Processing) answers: *Of the allowed work, what can be dispatched in this step, and in what order?* Execution Control does not re-evaluate policy and cannot reinstate denied Intents.

This separation ensures that safety constraints (policy) and operational efficiency (scheduling) remain independently auditable, independently evolvable, and cannot silently compensate for each other's failures.

➝ [Logical Architecture](logical-architecture.md), [Queue Processing](../20-concepts/queue-processing.md)

---

## P7 — Intent lifecycle is distinct from Order lifecycle

**An Intent is a pre-submission command. An Order is a post-submission derived entity. The two lifecycles must not be collapsed.**

An Intent is produced by Strategy as an ephemeral command. It is not an Event, not persistent, and not an Order. It exists only as transient input to a processing step. Nothing in the Intent lifecycle — generation, Risk evaluation, Queue residency — constitutes an Order.

An Order comes into existence in Execution State at **submission**. `Submitted` is the first Order state. After submission, Order state evolves through Execution Events. Venue responses advance an already-existing Order; they do not create it.

Collapsing these lifecycles produces documentation and implementation ambiguity about when the Infrastructure is committed to an outbound action and what can be revoked before that commitment.

➝ [Intent Lifecycle](intent-lifecycle.md), [Order Lifecycle](../20-concepts/order-lifecycle.md)

---

## P8 — Execution Control runs within Event processing; there is no separate tick

**Queue Processing is a deterministic computation executed as part of canonical Event processing. There is no independent scheduling loop, background timer, or runtime tick that advances execution-control state outside Event processing.**

Every advancement of Queue state, every dispatch decision, and every re-evaluation of eligibility, inflight status, and rate limits happens within the same sequential Event-processing step that updates all derived State. There is one advancement mechanism: the Event Stream.

This principle ensures that Queue Processing decisions are deterministic, replayable, and free of timing-dependent behavior. A module with a separate scheduler tick cannot guarantee that Backtesting produces the same dispatch decisions as Live.

➝ [Queue Processing](../20-concepts/queue-processing.md), [Infrastructure Flows](infrastructure-flows.md), [Invariants: EC2](../20-concepts/invariants.md)

---

## P9 — Backtesting and Live share the same canonical runtime semantics

**Backtesting and Live are two infrastructure configurations of the same Core Runtime model. The semantic model does not differ between them.**

Both Runtimes apply `State = f(Event Stream, Configuration)`, process the same `Strategy ➝ Risk ➝ Queue ➝ Venue Adapter` chain, maintain the same Order lifecycle beginning at submission, and treat Queue as derived execution-control substate.

What differs is infrastructure: data source (historical vs live), Venue (simulated vs real), and operational mode (batch vs continuous). The semantic model — the rules that govern State derivation, dispatch decisions, and lifecycle transitions — is identical.

This principle is the architectural basis for trusting Backtesting results as predictors of Live behavior. If the two Runtimes diverged semantically, Research would evaluate different than the one deployed in production.

➝ [Architecture Overview](architecture-overview.md), [Infrastructure Narrative](infrastructure-narrative.md)

---

## P10 — Design for microstructure granularity

**The Infrastructure is designed to operate at the granularity of individual market events, order book updates, and individual order lifecycle transitions. Coarser-grained usage is a projection of fine-grained data, not a separate architectural mode.**

Strategies operating at microstructure granularity (market making, short-term liquidity trading) require the highest-fidelity representation of market events, execution feedback, and order book state. The Infrastructure is designed to meet these requirements natively.

Strategies operating at coarser granularity use projections of the same data. This avoids the common failure mode of building a second, lower-fidelity infrastructure path that accumulates semantic divergence over time.

---

## P11 — Extend at boundaries; preserve the Core

**The Core Runtime — Event processing, State derivation, Strategy evaluation, Risk, and Execution Control — evolves slowly and only for fundamental improvements. Extension points exist at infrastructure boundaries: Venue Adapters, data recording connectors, simulated Venues, and Strategy modules.**

The Core Runtime implements the canonical rules that all other architecture depends on. Changes to it are high-consequence and must be evaluated against every principle above. Boundary components (Adapters, connectors, Strategy logic) can be added or replaced without modifying Core semantics.

This principle allows the Infrastructure to support new Venues, asset classes, and trading styles without re-litigating foundational semantic decisions.

➝ [Logical Architecture](logical-architecture.md)

---

## P12 — Components are separated by design; boundaries are load-bearing

**Strategy, Risk, Execution Control, and Venue Adapter are distinct components with hard responsibility boundaries. These boundaries are intentional and structural — not incidental to the current implementation.**

The processing chain — `Strategy ➝ Risk ➝ Queue + Queue Processing ➝ Venue Adapter` — is not a monolith that happens to have named stages. Each component has a defined responsibility, communicates through specified interfaces, and makes no assumptions about the internals of adjacent components. Strategy emits Intents without knowledge of dispatch scheduling. Risk evaluates admissibility without knowing dispatch order. Execution Control schedules allowed work without re-evaluating policy. The Venue Adapter translates protocol without deciding policy or scheduling.

This separation carries concrete architectural consequences:

- **Independent evolution.** Risk policy can be redesigned without changing Strategy logic or Execution Control rules. A Venue Adapter can be replaced for a new Venue without modifying the upstream chain.
- **Testability.** Each component can be verified against its defined contract without instantiating the full processing chain.
- **Runtime portability.** The same component definitions apply in Backtesting and Live (P9). Swapping a simulated Venue for a real one changes only the Venue boundary — not the components upstream of it.
- **Reduced coupling.** No component holds knowledge of another's internal state. Coupling is limited to defined interfaces: Intents from Strategy to Risk, allowed Intents to Execution Control, dispatch decisions to the Venue Adapter.
- **Localized reasoning.** Each component can be analyzed against its contract in isolation. Failures are traceable to specific boundaries rather than requiring full diagnosis.

This principle generalizes what P6 establishes for Policy vs Execution Control: the entire processing chain maintains strict responsibility separation, and that separation is a first-class architectural commitment.

➝ [Logical Architecture](logical-architecture.md), [Architecture Overview](architecture-overview.md)

---

## Relationship to other documents

These principles are enforced by the formal rules in the concept documents:

| Principle | Primary enforcement document |
| --------- | ---------------------------- |
| P1, P2, P3 | [Invariants](../20-concepts/invariants.md), [Determinism Model](../20-concepts/determinism-model.md) |
| P4 | [Determinism Model](../20-concepts/determinism-model.md) |
| P5 | [Time Model](../20-concepts/time-model.md), [Determinism Model](../20-concepts/determinism-model.md) |
| P6 | [Logical Architecture](logical-architecture.md), [Queue Processing](../20-concepts/queue-processing.md) |
| P7 | [Intent Lifecycle](intent-lifecycle.md), [Order Lifecycle](../20-concepts/order-lifecycle.md) |
| P8 | [Queue Processing](../20-concepts/queue-processing.md), [Infrastructure Flows](infrastructure-flows.md) |
| P9 | [Architecture Overview](architecture-overview.md), [Infrastructure Narrative](infrastructure-narrative.md) |
| P10, P11 | [Architecture Overview](architecture-overview.md), [Logical Architecture](logical-architecture.md) |
| P12 | [Logical Architecture](logical-architecture.md), [Architecture Overview](architecture-overview.md) |
