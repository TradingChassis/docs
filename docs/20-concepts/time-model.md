# Time Model

---

## Purpose and scope

The **Time Model** defines how temporal ordering and causality are represented within the System.

It specifies:

- which **temporal domains** exist and what they mean;
- which domain determines **internal causal order**;
- how **deterministic replay** is guaranteed;
- how **time-dependent behavior** is expressed without introducing hidden time-driven State.

This document does **not** redefine **Event** semantics, **State** derivation, or **Queue** semantics; it provides the temporal foundation that those models rely on. Capitalized terms are used as in [Terminology](../00-guides/terminology.md).

---

## Temporal domains

The System distinguishes two temporal domains. They represent different aspects of reality and must not be conflated.

| Domain | Meaning | Role in causality |
| ------ | ------- | ----------------- |
| **Event Time** | When an occurrence happened externally (in a market, feed, or Venue) | External semantics only; does **not** determine internal State ordering |
| **Processing Order** | The strict internal sequence in which Events are applied | **Canonical causal order** for all State derivation and replay |

---

## Event Time

**Event Time** is a property of an **Event** that records **when** the corresponding occurrence happened in an external or market context.

Typical sources:

- Venue-assigned timestamp;
- market feed timestamp;
- Venue-generated execution time.

**Normative rules:**

1. **Event Time is metadata carried by an Event.** It describes external semantics; it does not cause State changes by itself.
2. **Event Time does not determine Processing Order.** Two Events with the same Event Time are still ordered by their **stream position** ([Event Model](event-model.md)).
3. **Venue timestamps never override internal ordering.** Processing Order is always determined by stream position under the System's processing rules.

---

## Processing Order

**Processing Order** is the strict, total internal ordering of Events within the **Event Stream**. It is the **sole** determinant of how **State Transitions** are applied.

**Normative rules:**

1. **State Transitions occur strictly in Processing Order.** No transition occurs before, after, or outside the sequential application of Events in stream order.
2. Processing Order is **independent of Event Time.** A later-arriving Event that carries an earlier Event Time is still processed in its stream position; it does not retroactively mutate prior derived State.
3. Processing Order is **not** determined by wall-clock time, thread scheduling, or OS timers. It is determined by the logical ordering rules applied to the **Event Stream** under **Configuration**.

**Typical realizations of Processing Order:**

- monotonic sequence identifiers assigned at stream recording;
- deterministic replay indices from historical datasets;
- causal ordering rules defined in **Configuration**.

---

## Temporal separation

**Event Time** and **Processing Order** are not guaranteed to align.

Network latency, buffering, feed jitter, replay from historical data, or ordering rules may introduce divergence between when an Event occurred externally and when it is processed internally.

This separation is **intentional and explicitly modeled**. The System preserves external market semantics through **Event Time** (carried in Events) while guaranteeing deterministic internal behavior through **Processing Order**.

---

## Causality rule

Every **State Transition** must be caused by a **single processed Event** in **Processing Order**.

**Normative prohibitions:**

- No spontaneous State changes.
- No implicit time-based mutations.
- No background updates driven by wall-clock time, sleep intervals, or scheduler events outside the **Event Stream**.
- No hidden state that evolves independently of **Event processing**.

All internal evolution is **event-driven**. Time observations may be carried inside **Events** (as **Event Time** or as data fields), but they do not bypass **Processing Order** as the causal mechanism.

---

## Time-dependent behavior and deterministic replay

Some System behaviors are **conceptually time-dependent**—for example, rate-limit capacity recovery or the re-eligibility of pending outbound work after a delay.

Such behavior must be expressed in a way that remains **compatible with deterministic replay** under identical **Event Stream** and **Configuration**.

**Normative rules:**

1. **Time observations that affect State must enter as Events or as Configuration.** If a timestamp or elapsed-time computation is needed for a State transition, it must be carried in an **Event** already on the stream (e.g. as a field of an existing **Execution** or **Control Event**) or encoded in **Configuration**-level rules—not derived from a private wall-clock read at runtime.
2. **Rate-limit rules and capacity recovery must be expressed as deterministic functions of prior stream history and Configuration.** For example, a token-bucket rule whose replenishment rate is defined in **Configuration** and whose consumption is tracked through prior **Events** is fully deterministic and replayable. An independent timer that fires outside **Event processing** is not.
3. **Wakeups and delayed reevaluations must not introduce unnecessary extra Event types.** Where a future processing obligation exists (e.g. re-evaluate outbound Queue after rate-limit window expires), it must be tied to an **Event** that naturally occurs at that time (e.g. a market update, an execution report, a control signal)—or to a minimal scheduled record already defined in the stream semantics—not to ad hoc timer Events invented solely for scheduling.

---

## Time and Execution Control

**Queue Processing**—the deterministic reevaluation of pending outbound work—runs **within Event processing**, not on an independent clock or timer loop.

**Normative rules:**

1. There is **no** separate runtime tick that advances execution-control state independently of **Event processing**.
2. Queue Processing is reevaluated whenever an **Event** advances the **Processing Order** position in a way that is relevant to Execution Control (e.g. an **Execution Event** clearing an inflight slot, a market update, a control signal).
3. Time-dependent execution-control constraints (e.g. rate limits) are enforced through deterministic rules applied during **Event processing**, not through background timers that mutate state outside the stream.

See [Queue Processing](queue-processing.md) for execution-control evaluation rules.

---

## Ordering principles (summary)

| Rule | Statement |
| ---- | --------- |
| **Market semantics** | Preserved by **Event Time** carried in Events |
| **Internal causality** | Defined by **Processing Order** (stream position) |
| **State evolution** | Follows **Processing Order** strictly; no retroactive mutation |
| **Determinism** | Same **Event Stream** + **Configuration** → same derived State at every position |
| **Wall-clock independence** | Runtime timing must not define canonical System behavior |

---

## Deterministic replay

Given:

- an identical **Event Stream**;
- identical **Configuration**;
- the same **Processing Order**;

the System must produce **identical State Transitions** at every stream position, including all execution-control derivations.

The **Event Stream** and **Configuration** together constitute the canonical input. No private runtime state, wall-clock read, or scheduler timing may influence replay outcomes.

---

## Live and Backtesting contexts

The same temporal model applies in both Runtimes.

| Runtime | How Processing Order is established |
| ------- | ----------------------------------- |
| **Live** | Emerges from real-time Event delivery in arrival order under System recording rules |
| **Backtesting** | Reconstructed deterministically from historical datasets under the same ordering rules |

In both contexts, **State Transitions** depend solely on **Processing Order**, and time-dependent behavior is expressed through **Events** and **Configuration** rather than through runtime timing.

---

## Relationship to other documents

- [Terminology](../00-guides/terminology.md) — canonical terms including **Event Time**, **Processing Order**, and **State Transition**.
- [Event Model](event-model.md) — how Events carry **Event Time** as metadata; **Processing Order** as the canonical sequence.
- [State Model](state-model.md) — `State = f(Event Stream, Configuration)`; State Transitions in Processing Order.
- [Queue Processing](queue-processing.md) — execution-control reevaluation as part of **Event processing**, not a separate tick.
