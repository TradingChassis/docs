# Intent Dominance

---

## Purpose and scope

This document defines **Intent dominance**: the deterministic reconciliation rule that maintains **at most one effective outbound command** per logical order key in **execution-control substate** (the Queue).

When **Strategy** generates multiple **Intents** targeting the same logical order key in succession, dominance determines which command is the **current effective** pending work. Superseded commands are collapsed before dispatch.

This document does **not**:

- define **Queue Processing** evaluation rules, eligibility, or scheduling ([Queue Processing](queue-processing.md));
- define **Queue** structure or admission rules ([Queue Semantics](queue-semantics.md));
- define **Order** lifecycle stages or transitions ([Order Lifecycle](order-lifecycle.md));
- restate formal **Event** or **State** semantics ([Event Model](event-model.md), [State Model](state-model.md));
- replace the canonical glossary ([Terminology](../00-guides/terminology.md)).

Capitalized terms are used as in [Terminology](../00-guides/terminology.md).

---

## What dominance is

**Intent dominance** is a **deterministic derivation** over current **execution-control substate** and **Configuration**. It is not a separate runtime phase, a separate Event type, or an independent source of truth.

**Normative rules:**

1. Dominance is computed as part of **Event processing**—specifically, as part of the execution-control substate update that occurs when an **allowed** Intent is admitted ([Queue Semantics](queue-semantics.md)).
2. Dominance **does not** require a separate **Event** for each supersession unless **canonical history** explicitly requires recording the outcome for replay or audit ([Terminology: Intent visibility](../00-guides/terminology.md#intent-visibility)).
3. Dominance applies **only to pre-submission pending work** in execution-control substate. It has **no** effect on **Intents** already **Dispatched** or **Inflight** ([Intent Lifecycle](../10-architecture/intent-lifecycle.md)), and it does **not** retroactively alter **Order** state after **submission** ([Order Lifecycle](order-lifecycle.md)).
4. The result of dominance is that the **Queue** holds **at most one effective command** per logical order key at any time. This is a property of derived execution-control substate, not a mutation of independent truth.

---

## Conceptual model

A **Strategy** may generate a sequence of **Intents** targeting the same logical order key:

```
NEW ➝ REPLACE ➝ REPLACE ➝ CANCEL
```

Rather than forwarding each command independently (which would produce redundant or conflicting execution requests), the Infrastructure collapses them. Dominance determines which command is the **current effective** pending action.

The result for the sequence above is:

```
CANCEL
```

Only the effective command is dispatched. This is a **deterministic function** of the incoming command and the current execution-control substate under **Configuration**—not a separate step with its own clock or state store.

---

## Logical order identity

Dominance operates at the level of a **logical order key**: the stable identifier linking a set of outbound commands to the same logical Order.

- `NEW(order_id)`, `REPLACE(order_id)`, and `CANCEL(order_id)` all target the same logical order key.
- Dominance rules are evaluated **only** among commands targeting the **same** logical order key.
- Commands targeting **different** logical order keys are **independent** and do not interact.

**Order ID** is the canonical key used throughout the Infrastructure for this purpose ([Order Lifecycle](order-lifecycle.md)).

---

## Dominance hierarchy

Intent dominance follows a **hierarchical precedence**:

```
CANCEL  >  REPLACE  >  NEW
```

This ordering reflects the **semantic strength** of each command in terms of outbound execution:

- **CANCEL** requests termination of the Order at the Venue. It supersedes any pending modification or creation command for the same key.
- **REPLACE** requests modification of an existing Order at the Venue. It supersedes a pending creation command for the same key.
- **NEW** requests creation of an Order at the Venue. It does not override a pending modification or termination command.

**Normative clarification:** A `CANCEL` Intent in execution-control substate is a **pending request to cancel**, not a completed Order termination. Whether the Order is actually cancelled depends on subsequent **Execution Events** from the **Venue** after dispatch ([Order Lifecycle](order-lifecycle.md)).

---

## Dominance interaction matrix

The following table defines the effective command resulting from each combination of existing pending command and incoming allowed command for the same logical order key.

| Existing in Queue | Incoming allowed Intent | Effective result |
| ----------------- | ----------------------- | ---------------- |
| `NEW` | `NEW` | Latest `NEW` replaces prior `NEW` |
| `NEW` | `REPLACE` | `REPLACE` replaces `NEW` |
| `NEW` | `CANCEL` | `CANCEL` replaces `NEW` |
| `REPLACE` | `NEW` | Incoming `NEW` discarded |
| `REPLACE` | `REPLACE` | Latest `REPLACE` replaces prior `REPLACE` |
| `REPLACE` | `CANCEL` | `CANCEL` replaces `REPLACE` |
| `CANCEL` | `NEW` | Incoming `NEW` discarded |
| `CANCEL` | `REPLACE` | Incoming `REPLACE` discarded |
| `CANCEL` | `CANCEL` | Latest `CANCEL` replaces prior `CANCEL` |
| *(none)* | Any | Incoming command enters as effective pending work |

The matrix is a **deterministic function** of existing substate and incoming command type under **Configuration**. No runtime decision-making beyond this table is required for dominance.

---

## Pre-submission scope

Dominance applies **only to work in execution-control substate** (pre-submission pending commands). This boundary is strict:

1. Once an Intent has been **Dispatched** to the **Venue Adapter**, it is no longer subject to dominance. The outbound request is stable from that point.
2. After **submission**, an **Order** exists in **Execution State** at **Submitted** ([Order Lifecycle](order-lifecycle.md)). Its subsequent evolution is driven by **Execution Events** from the **Venue**, not by dominance.
3. A pending `REPLACE` or `CANCEL` command in the **Queue** for an already-**Submitted** Order targets that existing Order. It does not undo or retroactively modify what has already been submitted.

---

## Queue minimality

As a consequence of dominance, the **Queue** (execution-control substate) is always **minimal**:

- **At most one** effective command per logical order key at any time.
- **No** accumulation of superseded or redundant commands.
- The Queue represents **current effective pending outbound work**, not a history of all commands generated by Strategy.

This property follows from the deterministic application of dominance rules within **Event processing**; it does not require a separate maintenance mechanism.

---

## Relationship to other documents

- [Terminology](../00-guides/terminology.md) — canonical terms, including **Intent visibility**.
- [Queue Semantics](queue-semantics.md) — Queue identity, admission, and what the Queue contains.
- [Queue Processing](queue-processing.md) — deterministic evaluation of sendability among Queue contents.
- [Intent Lifecycle](../10-architecture/intent-lifecycle.md) — Intent stage progression; dominance applies only up to **Dispatched**.
- [Intent Pipeline](../10-architecture/intent-pipeline.md) — submission boundary; dominance scope ends here.
- [Order Lifecycle](order-lifecycle.md) — **Order** evolution from **Submitted** onward; dominance does not apply after submission.
