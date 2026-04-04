# Failure Semantics

---

## Purpose

This document defines what **failure** means in the System and what constraints failure handling must satisfy within a deterministic, event-driven architecture.

It answers:

- what constitutes a failure in the canonical model;
- where failures occur in the processing chain;
- when failure outcomes must enter canonical history;
- what failure handling is forbidden because it would break replayability or State integrity.

This document does **not** describe recovery procedures, retry policies, alerting, incident response, or operational tooling. Those are implementation and operations concerns that must themselves conform to the constraints defined here.

Capitalized terms are used as in [Terminology](../00-guides/terminology.md).

---

## Scope

**In scope:**

- Semantic definition of failure in the canonical model
- Failure categories across the processing chain
- Constraints on how failures affect canonical history and derived State
- Relationship between failure handling and determinism

**Out of scope:**

- Specific failure recovery implementations
- Venue-specific reconnection or retry logic
- Monitoring, alerting, or runbook procedures
- Named Event types for failure records (defined at implementation time)

---

## Core definition

A **failure** is any condition in which a processing step cannot produce its expected outcome, or in which an expected external input does not arrive.

In the canonical model, this definition has a precise implication:

> A failure must not produce **silent** or **out-of-band State changes**. Any failure that has State-relevant consequences must expose those consequences through the **Event Stream**, not through parallel mutable mechanisms.

This constraint follows directly from the System's foundational rule: **Events are the only source of State transitions** ([Invariants: E1](invariants.md)). A failure that silently mutates derived State violates this invariant and breaks determinism and replayability.

Failures therefore divide into two classes based on their relationship to canonical State:

| Class | Description | State consequence |
| ----- | ----------- | ----------------- |
| **State-invisible failure** | A failure whose outcome has no effect on canonical derived **State** | No canonical record required |
| **State-visible failure** | A failure whose outcome changes or constrains what the **Event Stream** must represent | Must appear as an **Event** |

The central discipline of failure handling in this System is determining which class a given failure belongs to, and responding accordingly.

---

## Failure categories

### 1. Input and recording failures

Input failures occur when an external source supplies data the System cannot process: a malformed market update, a missing sequence number, a feed interruption, a corrupted record.

**Semantic constraint:** A gap or discontinuity in the observed event feed may itself need to be recorded as a **System Event** or **Control Event** on the canonical stream, if downstream processing depends on the System's known state of data completeness. The absence of an expected market update does not automatically become an Event — but if the System takes a definitive action based on that absence (e.g., pauses evaluation, marks the feed as degraded), that action and its effect on derived **State** must enter the stream.

A feed interruption that is transparently recovered — input resumes without any gap visible to the processing model — has no canonical State consequence and requires no record.

### 2. Strategy evaluation failures

Strategy evaluation failures occur when **Strategy** logic cannot produce a result during a processing step: runtime error, evaluation timeout, or invalid strategy output that does not conform to the expected command interface.

**Semantic constraint:** Strategy produces **Intents** — ephemeral commands with no persistent existence. A Strategy evaluation failure that prevents Intent generation leaves derived **State** unchanged. No **Order** is affected; no canonical State transition has occurred.

A Strategy failure is **state-invisible** unless the System takes a definitive action (e.g., halting a Strategy, disabling it) that must be reflected in **Control State** through **Events**.

Because Intents are not Events and are not part of canonical history, a failed Intent generation leaves no trace in the stream and requires no recovery for deterministic replay.

### 3. Risk evaluation failures

Risk evaluation failures occur when the **Risk Engine** cannot produce a policy decision for an **Intent**: internal error, unavailable configuration, or indeterminate evaluation result.

**Semantic constraint:** The Risk Engine's role is policy admissibility only (**allowed** / **denied**). A failure that prevents a definitive policy decision must be treated as a conservative **denied** outcome: the **Intent** must not proceed to **Execution Control** without a clear **allowed** decision.

This conservative treatment is **state-invisible**: a denied or indeterminate Intent leaves no mark on derived **State** (no **Order** exists, no **Queue** entry is made). If the policy outcome must enter canonical history for audit, it does so through an **Intent-related Event** per the intent-visibility rules ([Terminology: Intent visibility](../00-guides/terminology.md#intent-visibility)), not by any special failure pathway.

### 4. Execution Control failures

Execution Control failures occur during **Queue Processing**: inability to evaluate the Queue, corrupted derivation of inflight state, or failures in applying dominance, eligibility, or rate-limit rules.

**Semantic constraint:** The **Queue** is **derived execution-control substate** — not an independent source of truth ([Queue Semantics](queue-semantics.md)). A transient failure in Execution Control that leaves the Queue in an uncertain state does not permanently corrupt canonical **State**, because the Queue is fully recomputable from the **Event Stream** and **Configuration**. The Queue holds no authoritative state that cannot be recovered by re-derivation.

A failure in Queue Processing that prevents dispatch in a given step means work remains in **Pending dispatch** for the next processing step. No hidden fallback state is created; the next step re-derives the Queue correctly.

If a Queue Processing failure causes the System to take a definitive action that must be part of canonical history (e.g., withdrawing a pending Intent as part of a controlled shutdown), that action must be represented as an **Event**.

### 5. Venue and external interaction failures

Venue interaction failures are the most consequential category because they occur **after submission**, when an **Order** already exists in **Execution State**.

Failure modes include:

- **Dispatch failure:** The outbound request to the Venue could not be transmitted.
- **Missing acknowledgment:** The request was transmitted but no Venue response has arrived.
- **Explicit Venue rejection:** The Venue returned a rejection after submission.
- **Connection interruption:** The Venue feed or control connection was lost after one or more requests were in flight.

**Semantic constraints:**

**Dispatch failure (pre-acknowledgment).** If dispatch fails before the request reaches the Venue — i.e., transmission did not occur — there is no successful dispatch record in canonical history. Therefore no **Order** enters **Execution State**, and no **Order** lifecycle begins, since the lifecycle starts only at submission ([Order Lifecycle](order-lifecycle.md)).

**Missing acknowledgment (Order in Submitted).** An **Order** dispatched to the Venue remains in `Submitted` state until an **Execution Event** advances it. The System **must not** silently change the Order's state. No out-of-band reconciliation may alter the derived Order projection without going through the stream. The Order stays in `Submitted` — canonically and correctly — until an Event arrives that resolves its status.

**Definitive disposition under persistent uncertainty.** If the System ultimately determines that an unacknowledged Order must be given a definitive terminal disposition (e.g., treated as unknown or failed), that determination must be expressed as an **Event** appended to the stream. Only then does the Order lifecycle advance. The specific policy governing when and how that determination is made is an implementation concern.

**Venue rejections and cancellations.** Venue-originated rejection, fill, or cancellation feedback arrives as **Execution Events** per the normal processing model. These are not failures in the semantic sense — they are expected Venue responses recorded in the stream, which then derive the corresponding **Order** lifecycle transitions.

---

## Failure visibility and canonical history

The rule for when a failure must enter canonical history follows directly from the invariant that **State is fully derived from Event Stream + Configuration** ([Invariants: E2](invariants.md)):

**A failure must appear as an Event if and only if:**

1. It produces or constrains a **State transition** that must be reproducible under replay; or
2. It causes the System to take a definitive action affecting derived **State** (e.g., a halt, a forced Order disposition, a Strategy disabling); or
3. It would otherwise create an irreconcilable discrepancy between the canonical **Event Stream** and the real-world state of outstanding submissions.

**A failure does not need to appear as a canonical Event if:**

1. It is transient and transparently recovered within a processing step with no State-visible effect; or
2. It is a pre-submission failure (Strategy error, Risk indeterminacy, Queue Processing transient error) that produces no State change and no definitive system action.

Diagnostic recording outside the canonical stream (e.g., operational logs, monitoring signals) is always permitted and does not affect canonical semantics.

---

## Failure containment and State integrity

The following constraints apply to all failure-handling logic. None may be relaxed on grounds of operational urgency.

**F1 — Failures must not produce out-of-band State mutations.**
A failure may not be handled by writing to a derived State projection, a cache, or any store outside the **Event Stream** + **Configuration** derivation path. Any State change resulting from failure handling must go through Event processing.

**F2 — Failures must not bypass Event-driven Order lifecycle.**
An **Order** in any lifecycle state must not have its state changed by failure-handling code that does not operate through **Execution Events**. An Order in `Submitted` with no Venue response remains in `Submitted` until an Event — including a system-generated failure-disposition Event — resolves it.

**F3 — Failures must not introduce hidden authoritative state.**
No failure-handling mechanism may introduce a private mutable store that shadows derived State and influences future processing decisions without being part of **Event Stream + Configuration**. Queue reconciliation state, inflight tracking, and exposure bookkeeping are all derived — they must remain so under failure conditions.

**F4 — Pre-submission failures are State-neutral by default.**
A failure that occurs before dispatch (Strategy, Risk, Execution Control) has no persistent State consequence unless the System takes a definitive action requiring a canonical record. The ephemeral nature of **Intents** and the derived nature of the **Queue** mean these failures leave no residual State without an explicit Event.

**F5 — Violation of these constraints places the System in an invalid State.**
As with all invariant violations ([Invariants: Violation](invariants.md#violation)), failure-handling code that breaks these constraints must not proceed silently. Propagation of inconsistent State must be prevented.

---

## Relationship to replayability and determinism

A failure is compatible with determinism if and only if its handling is transparent to replay.

**Replay-compatible failure handling:**
The failure is recorded as an **Event** (or set of Events) on the canonical stream. Replaying that stream — including the failure Events — produces the same derived **State**, including the failure-induced State transitions. The failure is part of history; replay sees it.

**Replay-incompatible failure handling:**
The failure is handled out-of-band: State is mutated through a mechanism not visible in the stream, or an Order lifecycle transition occurs without a corresponding Event. Replaying the stream does not reproduce the post-failure State. This breaks determinism ([Determinism Model: What would break determinism](determinism-model.md#what-would-break-determinism)).

The implication is that **failure handling is itself event-driven**. The System responds to failures by appending Events that represent the known outcome, and then processing continues deterministically from those Events. There is no separate failure-resolution mode that operates outside the canonical model.

---

## Relationship to other documents

| Document | What it adds |
| -------- | ------------ |
| [Invariants](invariants.md) | Non-negotiable constraints — E1, E2, and E3 are directly implicated by failure semantics |
| [Determinism Model](determinism-model.md) | What breaks determinism; hidden mutable state and out-of-band mutation as forbidden mechanisms |
| [Event Model](event-model.md) | What qualifies as a canonical Event; when intent-processing outcomes must enter the stream |
| [State Model](state-model.md) | `State = f(Event Stream, Configuration)`; no component owns mutable truth |
| [Order Lifecycle](order-lifecycle.md) | Order in `Submitted`; when and how Order state advances via Execution Events |
| [Queue Semantics](queue-semantics.md) | Queue as derived substate; re-derivability under Execution Control failures |
| [Queue Processing](queue-processing.md) | Execution Control within Event processing; no separate tick or hidden inflight state |
| [Terminology: Intent visibility](../00-guides/terminology.md#intent-visibility) | When intent-processing outcomes enter canonical history |

---

## Open boundaries intentionally left abstract

The following aspects of failure handling are intentionally not defined here because they depend on implementation decisions not yet fixed by the canonical model:

1. **Named Event types for failure records.** Which specific named Event types (e.g., a "dispatch failure" record, a "feed gap" control Event, an "Order disposition" Event) are defined for recording failures in canonical history is an implementation decision. The canonical model requires only that the relevant outcomes enter the stream as Events; it does not prescribe the taxonomy.

2. **Timeout policy for unacknowledged Orders.** When the System treats a prolonged absence of Venue feedback as grounds for a definitive Order disposition is a Configuration and policy decision. The canonical model constrains only that the disposition itself must be Event-driven when it occurs.

3. **Venue reconnection and retry behavior.** Whether and how the System attempts reconnection, re-queries Order status from a Venue, or uses alternative data paths after a connection interruption is an infrastructure and operational concern. Any resulting State changes must conform to the failure constraints in this document.

4. **Diagnostic and monitoring surface.** How failures are reported outside the canonical stream (logs, metrics, alerting) does not affect canonical semantics and is not constrained here beyond the requirement that such mechanisms must not write to canonical State directly.
