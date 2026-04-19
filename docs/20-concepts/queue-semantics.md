# Queue Semantics

---

## Purpose and scope

This document defines the **semantic role** of the **Queue**: what it is, what it holds, how identity works within it, and what constraints govern its contents.

This document does **not**:

- define **Risk** policy decisions ([Logical Architecture](../10-architecture/logical-architecture.md));
- define **Queue Processing** evaluation rules, eligibility, or scheduling ([Queue Processing](queue-processing.md));
- define **Order** lifecycle stages ([Order Lifecycle](order-lifecycle.md));
- restate formal **Event** or **State** semantics ([Event Model](event-model.md), [State Model](state-model.md));
- replace the canonical glossary ([Terminology](../00-guides/terminology.md)).

Capitalized terms are used as in [Terminology](../00-guides/terminology.md).

---

## What the Queue is

The **Queue** is **derived execution-control substate**: a projection within **Execution State** that holds **allowed** pending outbound work, maintained deterministically from the **Event Stream** and **Configuration**.

**Normative rules:**

1. The **Queue** is **not** a source of truth. Its contents are fully **recomputable** by replaying the **Event Stream** under the same **Configuration** and execution-control rules.
2. The **Queue** is **not** a fourth top-level **State domain**. It is **execution-control substate** within **Execution State** ([State Model](state-model.md)).
3. The **Queue** is **not** a Strategy buffer, a desired-state store, or an independent decision center. It holds **effective** reconciled pending outbound work only.
4. **Queue** state does **not** exist outside **Event processing**. No Queue update occurs except as a deterministic consequence of processing an **Event** under **Configuration**.

---

## What the Queue contains

The **Queue** holds **effective reconciled allowed pending outbound work**: commands that **Risk** has **allowed** and that have not yet been dispatched to the **Venue Adapter**.

Specifically:

- The **Queue** contains **at most one effective command** per logical order key at any time ([Intent Dominance](intent-dominance.md) governs which command is effective when multiple target the same key).
- The **Queue** does **not** accumulate raw **Strategy** output or superseded commands. Reconciliation (e.g. dominance) ensures only the **current effective** command per key is retained.
- The **Queue** does **not** contain **denied** Intents. **Risk** denial is final; denied commands do not enter the Queue.

The **Queue** therefore remains **minimal**, **conflict-free**, and **deterministic** at all times.

---

## Queue identity

**Queue identity** is based on a **logical order key**: the stable identifier linking an outbound action to the specific Order it concerns (e.g. a `NEW`, `REPLACE`, or `CANCEL` command all reference the same logical order key for a given Order).

**Normative rules:**

1. At most **one** effective command may be held per logical order key. A new command targeting an existing key **replaces** the prior effective command according to dominance rules ([Intent Dominance](intent-dominance.md)).
2. **Order ID** is the canonical key used for lifecycle tracking and identity throughout the Infrastructure ([Order Lifecycle](order-lifecycle.md)).
3. A **Venue-side identifier** (e.g. exchange order ID) may be associated with an existing Order after **Venue** acknowledgement; it does not affect Queue identity rules.

---

## Admission

An **allowed** Intent—one that **Risk** has classified as **allowed**—enters execution-control substate (the **Queue**) for scheduling by **Queue Processing**.

**Normative boundaries:**

- **Queue** admission requires **Risk** to have **allowed** the Intent. **Risk** is the policy layer; the **Queue** does not re-evaluate policy.
- An Intent that **Risk** has **denied** does **not** enter the **Queue** under any condition.
- Execution constraints such as rate limits, inflight restrictions, and sequencing requirements affect **when** work in the **Queue** is dispatched; they are evaluated by **Queue Processing** ([Queue Processing](queue-processing.md)), not at Queue admission.

The distinction between "cannot send yet" and "must not send" is fundamental: the Queue holds work that **may** be sent; **Queue Processing** determines **when**.

---

## Queue residency is pre-submission

**Queue** residency is **pre-submission only**. Work held in the **Queue** has not yet been dispatched to the **Venue Adapter**.

**Normative rules:**

1. **Queue** residency alone does **not** create an **Order** in **Execution State**.
2. An **Order** comes into existence at **submission**—when the **Venue Adapter** transmits the outbound request ([Order Lifecycle](order-lifecycle.md), [Intent Pipeline](../10-architecture/intent-pipeline.md)).
3. Once an Intent is dispatched, it leaves Queue residency. Its Intent lifecycle arc moves to **Submitted** ([Intent Lifecycle](../10-architecture/intent-lifecycle.md)).

---

## Relationship to other documents

- [Terminology](../00-guides/terminology.md) — canonical terms.
- [Logical Architecture](../10-architecture/logical-architecture.md) — Queue as a logical component; Risk vs. execution-control separation.
- [State Model](state-model.md) — Queue as execution-control substate within **Execution State**.
- [Intent Dominance](intent-dominance.md) — rules governing which command is effective per order key.
- [Queue Processing](queue-processing.md) — deterministic evaluation of sendability among Queue contents.
- [Intent Lifecycle](../10-architecture/intent-lifecycle.md) — Intent stage progression through **Pending submission** → **Submitted** → **Inflight** → **Closed**.
- [Intent Pipeline](../10-architecture/intent-pipeline.md) — submission as the boundary between Queue residency and Order existence.
- [Order Lifecycle](order-lifecycle.md) — **Order** evolution from **Submitted** onward.
