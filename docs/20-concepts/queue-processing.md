# Queue Processing

---

## Purpose and scope

This document defines **Queue Processing**: the deterministic evaluation logic that selects which **allowed** pending outbound work may be transmitted in the current processing step.

Queue Processing is **Execution Control** only. It does not decide **policy** (that is **Risk**). It operates over **derived Execution Control substate** (the **Queue**) and relevant projections of **Execution State**, both of which are fully derived from the **Event Stream** and **Configuration**.

This document does **not**:

- restate formal **Event** or **State** semantics ([Event Model](event-model.md), [State Model](state-model.md));
- replace the canonical glossary ([Terminology](../00-guides/terminology.md));
- restate full **Runtime** sequencing ([Infrastructure Flows](../10-architecture/infrastructure-flows.md)) or component boundaries ([Logical Architecture](../10-architecture/logical-architecture.md));
- define **Queue** structure or **dominance** algorithms ([Queue Semantics](queue-semantics.md), [Intent Dominance](intent-dominance.md));
- define **Intent** lifecycle stages ([Intent Lifecycle](intent-lifecycle.md)) or **Order** lifecycle ([Order Lifecycle](order-lifecycle.md)).

Capitalized terms are used as in [Terminology](../00-guides/terminology.md).

---

## Role of Queue Processing

**Queue Processing** determines, for the current processing step, **which** reconciled **allowed** pending Intents may be handed to the **Venue Adapter** for dispatch, and **in what order**.

Its role is strictly bounded:

| Responsibility | Owner |
| -------------- | ----- |
| **Policy admissibility** (allowed / denied) | **Risk Engine** |
| **Transmission timing, ordering, inflight gating, rate-compliant selection** | **Queue Processing** |

Queue Processing **never** re-runs policy. It operates only on work that **Risk** has already **allowed**; it cannot reinstate **denied** Intents and cannot admit new Intents on its own.

**Queue Processing is not a separate runtime tick, autonomous loop, or independently clocked scheduler.** It is a **deterministic computation** executed as part of **Event processing**—the same sequential step that updates **Market State**, **Execution State**, and **Control State** also evaluates and advances **Execution Control substate** ([Infrastructure Flows](../10-architecture/infrastructure-flows.md)).

---

## Inputs to Queue Processing

Queue Processing reads **only** derived quantities; it holds **no** independent authoritative state.

| Input | Source |
| ----- | ------ |
| **Derived Queue substate** | Execution Control substate: reconciled allowed pending Intents per logical order key, derived from **Event Stream + Configuration** |
| **Execution State projections** | Current **Order** states, inflight status, fills — derived **Execution State** |
| **Rate-capacity projection** | Remaining outbound capacity at this processing position, derived from rate rules and prior Event Stream history under **Configuration** |
| **Ordering / priority rules** | Specified in **Configuration**; applied deterministically |

Because all inputs are derived from **Event Stream + Configuration**, the output of Queue Processing is also deterministic: identical Event Stream history and Configuration always produce the same transmission decisions at the same Event Stream position.

---

## Deterministic evaluation rules

Within a single processing step, Queue Processing applies the following rules in order. Each rule is a **pure function** of current derived State; none requires a separate **Event** unless **canonical history** explicitly demands recording the outcome ([Terminology: Intent visibility](../00-guides/terminology.md#intent-visibility)).

### 1. Dominance and reconciliation

Before evaluating eligibility, the Queue substate holds at most one **effective** pending Intent per logical order key (per [Intent Dominance](intent-dominance.md) rules applied at admission time). Queue Processing reads the **already-reconciled** effective work; it does not re-apply dominance as a separate step, though it may confirm that the effective Intent remains valid under current State.

### 2. State validity

An Intent remaining in **Pending dispatch** substate is valid only if current derived State is consistent with its execution: the referenced Order (if any) must still exist in a compatible state and the Intent must not have been superseded since admission. Invalid pending work is **not** dispatched; it is treated as **Closed** in the Intent lifecycle sense ([Intent Lifecycle](intent-lifecycle.md)).

### 3. Inflight gating

At most one outbound request per logical order key may be **inflight** at a time. Queue Processing reads the inflight projection from **Execution State**: if a request for the same order key is already inflight, no further request for that key is dispatched in this step regardless of eligibility.

### 4. Rate-limit evaluation

Queue Processing reads the **rate-capacity projection** derived from prior Event Stream history under Configuration (e.g. token-bucket rules, outbound counts against Venue limits). If dispatching the next eligible Intent would exceed available capacity, that Intent **remains in Pending dispatch** and is re-evaluated in a future processing step.

Rate-limit state is **derived** from the **Event Stream** and Configuration. Rate limits and capacity recovery **must not** introduce unnecessary separate **Event types** ([Terminology](../00-guides/terminology.md#intent-visibility)); where a capacity change is relevant to determinism, it is expressed through deterministic rules tied to **Processing Order** and existing Event Stream history.

### 5. Eligibility and ordering

Among work that passes the above gates, Queue Processing applies **ordering rules** from Configuration (e.g. priority by Intent type, FIFO within priority, order-level sequencing) to produce a **deterministic** ordered candidate set for this step.

Eligibility is a **derived predicate** over current State—not a separate named lifecycle state that requires its own **Event** unless canonical history demands it.

### 6. Dispatch selection

Queue Processing selects the maximal sendable prefix of the candidate set given remaining rate capacity and inflight slots. Selected Intents are handed to the **Venue Adapter** for transmission; their status transitions from **Pending dispatch** to **Dispatched** in the Intent lifecycle ([Intent Lifecycle](intent-lifecycle.md)).

---

## Flush behavior

A single processing step may dispatch **multiple** Intents if rate capacity and inflight rules permit. This is a natural consequence of the evaluation rules applied to all pending work in one step; it requires no special mechanism beyond the evaluation rules above.

---

## When Queue Processing runs

Queue Processing runs **within** every **Event processing** step that includes a Queue evaluation phase, as defined by the processing model. It does **not** run as a background loop or on a separate clock.

Re-evaluation occurs whenever a new **Event** advances the processing position—including **Execution Events** (e.g. fill confirming a previous dispatch, clearing an inflight slot), **Market Events** (if Configuration routes them through Queue evaluation), **Control-Time Events**, and any other **Events** that update derived State in ways relevant to Execution Control.

Where current **State** and **Configuration** imply a future relevant control-time re-evaluation point—for example, a rate-limit window that will recover while allowed outbound work remains pending—the Core may derive a **Control Scheduling Obligation**. The **Runtime** realizes this obligation by injecting a **Control-Time Event** into the **Event Stream**. That Event is canonical and is processed within **Event processing** in the same way as any other Event; Queue Processing runs within that step. **Control-Time Events** are **sparse and deadline-style**: each corresponds to a specific derived obligation, not to an independent timer, a periodic tick, or an autonomous loop. They are **Control State** semantics and do not originate from a **Venue** or **Venue Adapter**.

---

## Relationship to Risk, Queue, and Execution State

**Risk Engine ➝ Queue Processing boundary:**

- **Risk** evaluates **admissibility** of each **Intent** as it arises during Strategy evaluation, before Queue residency. **Risk** produces **allowed** or **denied** only.
- **Queue Processing** evaluates **sendability** of already-**allowed** work.
- There is **no** feedback path where Queue Processing sends work back to **Risk** for re-evaluation of policy.

**Queue (derived substate) ➝ Queue Processing:**

- The **Queue** is **derived Execution Control substate**, fully recomputable from **Event Stream + Configuration** ([State Model](state-model.md)).
- Queue Processing reads this substate; it **does not** own it as independent truth.
- Outcomes of Queue Processing (dispatch decisions, inflight slot consumption) are reflected in **Execution State** derivation going forward, consistent with **Event processing**.

**Queue Processing ➝ Execution State and Order lifecycle:**

- When Queue Processing selects an Intent for dispatch, the **Venue Adapter** transmits the request. **At dispatch**, an **Order** enters **Execution State** at **Submitted**—this is the entry point of the **Order lifecycle** ([Order Lifecycle](order-lifecycle.md)). The **Order** exists from this moment, before any **Venue** response arrives.
- Subsequent **Venue** responses enter the **Event Stream** as **Execution Events** and **evolve the already-existing Order** (e.g. from **Submitted** to **Accepted**, **Filled**, **Rejected**, or **Cancelled**). **Venue** responses do **not** create the **Order** for the first time; they advance it.

---

## Processing invariants

1. **Execution Control** only: Queue Processing decides **when** to send among **allowed** work; it does **not** decide **whether** work is admissible (that is **Risk**).
2. **Derived inputs only:** All inputs are projections of **Event Stream + Configuration**; Queue Processing holds no independent authoritative state.
3. **Part of Event processing:** Queue Processing is a **deterministic computation** within the canonical **Event processing** step—**not** a separate tick, loop, or independently clocked subinfrastructure.
4. **Determinism:** Given identical **Event Stream** and **Configuration**, Queue Processing produces identical dispatch decisions at every **Processing Order** position.
5. **Internal derivations are not Events:** Dominance, eligibility, inflight gating, scheduling, and rate-limit bookkeeping are **internal derivations**; they do not require separate **Event types** unless canonical history explicitly demands records for replay or audit. **Control-Time Events**, by contrast, are canonical **Events** injected by the **Runtime** when a **Control Scheduling Obligation** is realized; they are not ad hoc internal bookkeeping records.
6. **Sequencing after Risk:** Queue Processing always operates on work that **Risk** has already **allowed** in the current or prior processing steps; it **never** precedes or replaces **Risk** evaluation for newly generated **Intents**.
7. **Order lifecycle begins at dispatch:** When Queue Processing dispatches an Intent, an **Order** enters **Execution State** at **Submitted**. **Execution Events** from the **Venue** then advance that already-existing **Order** through subsequent lifecycle states. **Orders** do **not** exist before **submission**, and **Venue** responses do **not** create **Orders** for the first time.

---

## Relationship to other documents

- [Terminology](../00-guides/terminology.md) — canonical terms, including **Intent visibility** and Execution Control definitions.
- [Logical Architecture](../10-architecture/logical-architecture.md) — component boundaries; Queue Processing responsibility defined there.
- [Infrastructure Flows](../10-architecture/infrastructure-flows.md) — canonical sequencing of Queue Processing within **Event processing**.
- [Intent Lifecycle](intent-lifecycle.md) — Intent stage progression through `Pending dispatch ➝ Dispatched ➝ Inflight ➝ Closed`.
- [Intent Pipeline](../10-architecture/intent-pipeline.md) — submission boundary and dispatch-triggered **Order** creation.
- [Order Lifecycle](order-lifecycle.md) — **Order** evolution from **Submitted** onward.
- [Queue Semantics](queue-semantics.md) — Queue structure and admission rules (not Queue Processing evaluation).
- [Intent Dominance](intent-dominance.md) — reconciliation rules applied at Queue admission; referenced here for context.
