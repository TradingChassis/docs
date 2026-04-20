# Invariants

---

## Purpose and scope

This document states the **non-negotiable properties** that must always hold across the Infrastructure.

Each invariant is a normative constraint. Violation places the Infrastructure in an invalid State.

This document does not describe architecture, implementation, or process. For those, see [Logical Architecture](../10-architecture/logical-architecture.md), [Infrastructure Flows](../10-architecture/infrastructure-flows.md), and the concept documents referenced below.

Capitalized terms are used as in [Terminology](../00-guides/terminology.md).

Invariants are grouped by theme: [Event and State](#event-and-state-invariants) · [Execution Control](#execution-control-invariants) · [Lifecycle](#lifecycle-invariants) · [Risk](#risk-invariants) · [Determinism](#determinism-invariants).

---

## Event and State invariants

**E1 — Events are the only source of State transitions.**
No component may change derived **State** by any mechanism other than processing a canonical **Event**. Spontaneous, timer-driven, or out-of-band State changes are forbidden.

**E2 — State is fully determined by Event Stream and Configuration.**
Formally: `State = f(Event Stream, Configuration)`. No third input exists. Any data that influences derived **State** must either be part of the **Event Stream** or be explicit, versioned **Configuration**.

**E3 — Events are immutable once appended.**
An **Event** in the **Event Stream** must not be modified, deleted, or reordered after it is written. The stream is append-only.

**E4 — Events are processed in Processing Order.**
No component may apply **Events** out of the sequence defined by **Processing Order**. **Processing Order** is the canonical causal axis; **Event Time** is metadata and does not override it.

**E5 — Identical inputs produce identical State.**
Given an identical **Event Stream**, identical **Configuration**, and the same **Processing Order**, the Infrastructure must produce identical **State** at every stream position without exception.

---

## execution-control invariants

**EC1 — The Queue is derived execution-control substate.**
The **Queue** is a deterministic projection of **Event Stream + Configuration**, not an independent source of truth. It is part of **Execution State** and is not a fourth top-level State domain. Any apparent **Queue** content must be recomputably derivable by replay.

**EC2 — Queue Processing is part of Event processing.**
There is no separate runtime tick, background timer, or autonomous loop that advances execution-control state. **Queue Processing** runs as a deterministic step within canonical **Event processing** only.

**EC3 — execution-control derivations must not introduce independent authoritative state.**
Dominance, eligibility, inflight gating, scheduling evaluation, and rate-limit bookkeeping are deterministic functions of current derived **State** and **Configuration**. None may maintain a primary store that is not recomputable from **Event Stream + Configuration**.

**EC4 — At most one effective pending command per logical order key.**
For every logical order key, the execution-control substate holds at most one effective pending outbound command. Superseded commands are eliminated by dominance before any new command is dispatched.

Formally: `∀ key: count(effective_pending(key)) ≤ 1`

**EC5 — At most one inflight execution request per logical order key.**
For every logical order key, at most one outbound request may be inflight at any time. A subsequent command targeting the same key must not be dispatched until the prior request has reached a terminal outcome.

Formally: `∀ key: count(inflight(key)) ≤ 1`

**EC6 — execution-control derivations are not canonical Events unless history requires them.**
Dominance resolution, eligibility changes, scheduling decisions, and inflight-gating evaluations must not produce **Events** in the canonical stream unless there is an explicit requirement that they be part of replayable history. Internal derivation must remain internal.

---

## Lifecycle invariants

**L1 — Intent is a command, not an Event, and not a persistent entity.**
An **Intent** is produced by **Strategy** as an ephemeral command. It is not appended to the **Event Stream** as an **Intent** object. Where **Intent** processing must be visible in canonical history, that visibility is recorded through specific **Intent-related Events**, not by persisting the **Intent** itself.

**L2 — Intent lifecycle and Order lifecycle are distinct.**
**Intent** lifecycle covers command creation through policy decision and execution-control disposition. **Order** lifecycle begins at submission. One is not a stage of the other; the two lifecycles must not be collapsed.

**L3 — Order lifecycle begins at submission.**
`Submitted` is the first **Order** state. **Queue** residency, **Risk** acceptance, and **Venue** acknowledgment do not constitute **Order** creation. An **Order** comes into existence in **Execution State** at the moment of submission; that moment is part of canonical history.

**L4 — Order transitions are driven by canonical Events.**
After creation at submission, every **Order** state transition must be caused by a canonical **Execution Event** in **Processing Order**. No **Order** state change may occur out of sequence or outside **Event processing**.

**L5 — Intent is not an Order state.**
No stage of the **Intent** lifecycle (Generated, Policy decided, Queue-resident, Dispatched) is a state of an **Order**. The boundary between the two lifecycles is submission.

---

## Risk invariants

**R1 — All Intents must pass through the Risk layer before Queue admission.**
An **Intent** that has not received a policy decision from the **Risk Engine** must not enter the execution-control **Queue**. There is no path from **Strategy** to **Queue** that bypasses **Risk**.

**R2 — Risk determines admissibility only.**
The **Risk Engine** makes a binary policy decision: allowed or denied. It does not schedule outbound work, set transmission timing, manage inflight state, or apply rate limits. Those responsibilities belong exclusively to **Queue Processing**.

**R3 — Risk enforcement must not be bypassed.**
No component may cause an outbound execution action without a prior allowed policy decision from the **Risk Engine** for the corresponding **Intent**.

---

## Determinism invariants

**D1 — The Infrastructure must be fully replayable.**
Given the same **Event Stream** and the same **Configuration**, every prior **State** and every prior dispatch decision must be exactly reproducible. No execution path may depend on information not present in **Event Stream + Configuration**.

**D2 — Wall-clock time and runtime timing must not affect canonical behavior.**
Wall-clock time, OS scheduling, thread interleaving, network delivery timing, and any other runtime timing factor must not influence **State transitions** or dispatch decisions. Time-dependent behavior must be expressed through **Events** or **Configuration** so that it is part of the canonical input and is therefore replayable.

**D3 — Hidden mutable state is forbidden.**
Any data that influences a **State transition** or a dispatch decision must be part of **Event Stream + Configuration**. Private mutable stores, caches whose mutations are not traceable to the **Event Stream**, and out-of-band side effects that affect derived **State** are forbidden.

**D4 — Determinism applies equally to Backtesting and Live.**
Both **Runtimes** must apply the same deterministic processing rules to the same **Event Stream + Configuration** and produce the same **State**. Infrastructure may differ; semantic behavior must not.

---

## Violation

If any invariant is violated, the Infrastructure is in an **invalid State**.

A component that detects an invariant violation must halt further processing, emit diagnostic information, and prevent propagation of inconsistent **State**. Invariant violations are critical infrastructure faults and must not be silently ignored.

---

## Relationship to other documents

- [Terminology](../00-guides/terminology.md) — canonical definitions.
- [Event Model](event-model.md) — Event immutability, Event Stream, Processing Order.
- [State Model](state-model.md) — `State = f(Event Stream, Configuration)`; State domains.
- [Time Model](time-model.md) — Processing Order as causal axis; Event Time as metadata.
- [Determinism Model](determinism-model.md) — formal definition of determinism and what breaks it.
- [Queue Semantics](queue-semantics.md) — Queue as derived execution-control substate.
- [Queue Processing](queue-processing.md) — deterministic execution-control evaluation within Event processing.
- [Intent Lifecycle](../10-architecture/intent-lifecycle.md) — Intent lifecycle stages and terminal outcomes.
- [Order Lifecycle](order-lifecycle.md) — Order lifecycle starting at submission.
- [Intent Dominance](intent-dominance.md) — deterministic reconciliation of pending pre-submission work.
