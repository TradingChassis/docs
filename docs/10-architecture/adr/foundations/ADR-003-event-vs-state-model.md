---
title: Event-Derived State Model
updated: 2026-04-01
adr_status: Accepted
---

# ADR-003: Event-Derived State Model

Status: {{ page.meta.adr_status }}  
Updated: {{ page.meta.updated }}

---

## Context

A trading infrastructure must process a continuous stream of occurrences — market data updates, Venue execution reports, control signals — and maintain a coherent internal condition based on those occurrences. The architectural question is: **what is the authoritative representation of infrastructure history and current condition?**

Two fundamental approaches exist:

**Mutable state as primary truth.** Components directly mutate an internal state store in response to incoming signals. The current state *is* the Infrastructure's truth; history is either not retained or maintained separately as a secondary concern.

**Event stream as primary truth.** Occurrences are recorded as immutable Events in a canonical stream. State is a derived projection: a deterministic function of the Event stream and configuration. The stream is the Infrastructure's truth; current state is recomputable from it.

The mutable-state-as-truth approach creates specific problems for a infrastructure that must be deterministic, replayable, and operationally auditable:

- **Reconstruction is unreliable.** If state is the primary store, reconstructing historical state requires either complete snapshots at every point of interest or reverse-engineering state from incomplete logs. Neither is guaranteed to reproduce the exact condition the Infrastructure was in at a given moment.
- **Replay is structurally unsupported.** Replaying the same inputs to reproduce the same outputs requires that all state evolution be accounted for by those inputs. When components mutate state directly — through ad hoc writes, hidden caches, timer-driven updates, or out-of-band corrections — the inputs alone are insufficient to reproduce the result.
- **Backtesting and Live diverge.** If the Infrastructure permits state mutations outside the input-processing path, Backtesting (which replays historical inputs) cannot reproduce the state mutations that occurred during Live execution. The two Runtimes evaluate different infrastructures.
- **Causality is ambiguous.** When multiple components can mutate state through different mechanisms, the question "why is the Infrastructure in this state?" has no single authoritative answer. Debugging requires correlating disparate mutation sources rather than reading a single ordered history.
- **Consistency across domains is fragile.** Market State, Execution State, and Control State must remain mutually consistent. Direct mutation across domains without a single sequencing mechanism creates windows where one domain reflects an occurrence that another has not yet processed.

The Infrastructure requires deterministic behavior, exact replayability, and semantic parity between Backtesting and Live. These properties are structurally incompatible with a mutable-state-as-truth model.

---

## Decision

**Events are the canonical record of infrastructure history. State is a deterministic projection of the Event Stream under Configuration. State is not primary truth.**

Formally:

`State = f(Event Stream, Configuration)`

This decision imposes the following architectural rules:

1. **Events are the only source of State transitions.** No component may change derived State by any mechanism other than processing a canonical Event. Spontaneous, timer-driven, and out-of-band State changes are forbidden.

2. **The Event Stream is the authoritative history.** Events are immutable once appended. The stream is totally ordered by Processing Order — the strict internal causal sequence that determines when each Event is applied. Event Time (the external timestamp an Event carries) is metadata; it does not override Processing Order.

3. **State is derived, not owned.** No component holds State as independent mutable truth. Components read projections of derived State; they do not maintain authoritative copies that diverge from what the Event Stream would produce.

4. **State is reconstructible.** Given an identical Event Stream and identical Configuration, the Infrastructure produces identical State at every Processing Order position. State can be reconstructed at any point by replaying the stream under the same Configuration.

5. **All inputs that influence State must be canonical.** Any data that affects derived State must be either part of the Event Stream or part of explicit, versioned Configuration. Hidden stores, caches, and out-of-band writes that influence State evolution are forbidden.

### State domains

Derived State is organized into exactly three top-level domains:

| Domain | What it derives from |
| ------ | -------------------- |
| **Market State** | Market Events (order book updates, trades, market data) |
| **Execution State** | Execution Events (Venue reports), Intent-related Events where canonical history requires them |
| **Control State** | Infrastructure Events, Control Events (configuration changes, operational signals) |

There is no fourth top-level domain. Execution-control substate (Queue contents, inflight tracking, rate-limit bookkeeping) is part of Execution State — derived, not independently authoritative.

---

## Consequences

**Deterministic replay is a structural property, not an aspiration.** Because State is fully determined by Event Stream + Configuration, and because no mutation path exists outside Event processing, replay of the same stream under the same Configuration is guaranteed to produce the same State at every position. This holds for all State domains including Execution Control substate.

**Backtesting and Live share the same derivation model.** Both Runtimes apply the same `State = f(Event Stream, Configuration)` rule. Infrastructure differs (historical vs live inputs, simulated vs real Venue); the derivation semantics do not. This equivalence is a direct consequence of the event-derived model — it is not achievable if either Runtime permits state mutations outside the Event processing path.

**All derived entities are projections, not independent truth.** Orders, positions, Queue contents, and Execution Control bookkeeping exist only as projections maintained during Event processing. None is an authoritative store that can diverge from the Event Stream. This rules out designs where, for example, a Queue maintains its own truth independently of the derivation path, or where Order state is updated through a mechanism other than processing Execution Events.

**Processing Order is the causal axis.** Because State derivation is sequential over the Event Stream, the strict ordering of Events — Processing Order — defines the Infrastructure's internal causal history. Event Time is preserved as external metadata but does not determine when an Event is applied. This eliminates timing-dependent ambiguity in state evolution.

**No separate runtime tick.** All advancement of State — including Execution Control substate — occurs within Event processing. There is no background timer, polling loop, or autonomous scheduler that advances State independently. This follows directly from the decision: if State is derived from the Event Stream, anything that changes State outside Event processing introduces a mutation path that the stream cannot account for.

**Failure recovery by replay.** State after a failure can be reconstructed from the Event Stream without relying on snapshots or parallel mutable stores. Snapshots, if used, are a pure optimization — they must not contradict what replay would produce.

---

## Trade-offs

**Implementation must enforce the derivation discipline.** Every state-changing operation must flow through Event processing. Components cannot take shortcuts by mutating state directly, even when that would be simpler in the short term. This requires consistent architectural enforcement across all components and all Runtimes.

**Event Stream must be retained and ordered.** The Event Stream is the canonical history; losing or corrupting it means losing the authoritative record from which State is derived. Stream integrity, retention, and correct Processing Order assignment are infrastructure requirements imposed by this decision.

**Derived state has latency relative to the Event Stream.** State at any moment reflects only Events processed up to the current Processing Order position. This is inherent to sequential derivation and is the same in Backtesting and Live. It is not a deficiency — it is the mechanism that makes determinism possible — but implementation must not attempt to compensate by introducing out-of-band state updates.

---

## Summary

The Infrastructure treats the Event Stream as canonical history and State as a derived projection. State is not primary truth; it is a deterministic function of Event Stream and Configuration. This decision is the structural foundation for deterministic replay, Backtesting/Live semantic parity, and the prohibition of hidden mutable truth throughout the Infrastructure.
