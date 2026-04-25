# Snapshot-Driven Inputs

---

## Purpose

This document defines how **snapshot-driven inputs** fit into a deterministic, event-driven infrastructure.

It establishes:

- what a snapshot-driven input is in this infrastructure's semantic context;
- how snapshots relate to canonical history and the Event Stream;
- what constraints must hold for snapshot use to remain compatible with determinism and replayability;
- what snapshot-driven inputs are not permitted to do.

Capitalized terms are used as in [Terminology](../00-guides/terminology.md).

---

## Scope

**In scope:**

- Inputs that arrive in snapshot form (full-state representations at a bounded point in time, as opposed to incremental change streams)
- Snapshot use as a starting anchor for replay or State bootstrapping
- Bounded derived views of State maintained for efficiency and read access
- Constraints on how all of the above must behave within the canonical model

**Out of scope:**

- Specific storage formats, schemas, or technologies used to persist snapshots
- Concrete data feed protocols or Venue-specific snapshot delivery formats
- Implementation of snapshot serialization, deserialization, or indexing

---

## Core definition

A **snapshot-driven input** is an input to the infrastructure that represents the **condition of a source at a bounded point in time**, rather than an ordered sequence of incremental change records.

Instead of receiving every change that led to the current condition, the infrastructure receives the current condition itself — the full state of the source as of a specific moment.

Common scenarios where snapshot-driven inputs arise:

- A market data feed that delivers a full order book state on connection or reconnect, followed by incremental updates
- An initial position or balance report provided at infrastructure start before incremental execution events begin flowing
- A periodic full-state delivery from a Venue or data source that does not expose incremental history

In each case, the snapshot represents a **bounded, point-in-time view** of some external reality. It is not a sequence of what happened; it is a summary of what is known to be true at a specific moment.

---

## Relationship between snapshots and canonical history

The canonical model requires that `State = f(Event Stream, Configuration)` and that **Events are the only source of State transitions**. Snapshot-driven inputs must be integrated in a way that does not violate this.

There are two canonical-compatible ways to integrate a snapshot into the infrastructure:

### Integration path A: snapshot as Event

The snapshot is **canonicalized as an Event** (specifically, a **Market Event** or **Control Event** as appropriate — see [Event Model](event-model.md)) and appended to the **Event Stream** at the relevant **Processing Order** position.

From that Event Stream position onward, derived State reflects the contents of the snapshot. Subsequent incremental Events (deltas, updates) are applied on top in **Processing Order**, exactly as they would be if no snapshot had been used.

This path makes the snapshot a full part of canonical history. A replay of the Event Stream reproduces the snapshot Event and derives the same State from it. The snapshot does not exist as a hidden input — it is visible in the Event Stream.

**Event Time on the snapshot Event** records when the snapshot was taken externally. Processing Order, not Event Time, determines when it is applied to derived State — as with all Events in the infrastructure ([Time Model](../20-concepts/time-model.md)).

### Integration path B: snapshot as bounded Configuration anchor

The snapshot is provided as a **stable, versioned Configuration anchor** — a known starting State for a bounded processing scope. It is not appended as an Event, but it is an explicit, immutable input to the infrastructure for that processing context.

Subsequent Events are applied on top of this anchor under the same derivation rules as usual.

This path is compatible with determinism and replayability only if the snapshot anchor is treated as a **versioned, fixed input** — the same anchor must be used for every replay of the same processing scope. A snapshot anchor that is read as live, current, mutable state and therefore differs between runs breaks determinism ([Determinism Model](determinism-model.md)).

### What is not permitted

A snapshot that **silently mutates derived State** outside either of the above paths violates the canonical model. Specifically:

- Maintaining a mutable "current state" store updated by incoming snapshots, where that store influences infrastructure behavior without entering the Event Stream, is **hidden mutable truth** and is forbidden ([Invariants: E1, E2, D3](invariants.md)).
- Treating a live, non-versioned snapshot read as equivalent to a stable canonical input breaks replayability: the same Event Stream replayed at a different time may read a different snapshot and produce a different result.

---

## Snapshots as derived input representations

Not all snapshot-related structures are external inputs. The infrastructure also uses snapshot representations on the **read side**: bounded views of derived State maintained for efficiency and component access.

A **derived State projection** — a materialized view of some portion of current derived State, maintained incrementally as Events are processed — is semantically valid as a read-side efficiency mechanism. Strategy, for example, reads **projections** of derived Market and Execution State rather than re-deriving full State from the Event Stream on each processing step ([State Model](state-model.md)).

Such derived views are valid under the following conditions:

1. They are **derived from the Event Stream**, not authoritative sources that differ from it.
2. They are **updated through Event processing**, not through direct writes that bypass the derivation path.
3. They do **not feed back** into canonical processing as independent sources of truth. Components read them as projections; those reads do not bypass `State = f(Event Stream, Configuration)`.
4. They are **recomputable**: given the same Event Stream and Configuration, the same projection can be rebuilt by replay.

A derived view that violates any of these conditions is no longer a projection — it is hidden mutable truth, which is forbidden.

---

## Compatibility with replayability and determinism

The central requirement for any snapshot use is **replay-stability**: the same Event Stream and Configuration — including any snapshot anchors that are part of the canonical input — must always produce the same derived State.

This places the following requirements on snapshot-driven inputs:

**S1 — Snapshot inputs that affect derived State must be stable across replays.**
A snapshot used as an anchor or as a canonical Event must be the same in every replay of the same scope. If the snapshot is read from a live, changing source, it must be **fixed at recording time** as a versioned canonical record before it influences any processing.

**S2 — Snapshot inputs must enter through canonical processing paths.**
A snapshot that advances derived State does so either as an Event in the Event Stream or as an explicit Configuration anchor. There is no third path. Snapshots that influence processing through out-of-band writes to derived State projections are not compatible with determinism.

**S3 — Processing Order, not snapshot Event Time, determines causality.**
A snapshot Event carries an **Event Time** that reflects when the snapshot was externally produced. This Event Time does not override the snapshot Event's position in **Processing Order**. Derived State at the snapshot's Event Stream position reflects the full history up to and including that position, in **Processing Order** sequence.

**S4 — Derived views must be recomputable from the Event Stream.**
A bounded State projection maintained as a snapshot for efficiency must produce the same result as re-deriving from the Event Stream and Configuration. If a derived view cannot be reconstructed by replay, it is not a valid projection.

---

## Constraints and non-goals

### Constraints

- A snapshot must not be used as a **primary source of State** that independently determines infrastructure behavior. It is either part of the canonical input (Event or Configuration anchor) or a derived view — never an authoritative truth that runs in parallel.

- A snapshot anchor used to bootstrap State must be **explicitly versioned and fixed** for its processing scope. Using a live read of current state as an anchor introduces non-determinism.

- The **Queue**, Order projections, and Execution Control substate are themselves derived views ([Queue Semantics](queue-semantics.md)). They must obey the same constraints as any derived projection: recomputable, Event-driven, not independently authoritative.

- Snapshot recording that would require bypassing **Event processing** to update derived State is architecturally invalid regardless of operational convenience.

### Non-goals

- This document does not define **which** snapshot Event types are recognized in the infrastructure. The Event Model establishes that Market Events may include order book snapshots ([Event Model: Market Events](event-model.md#market-events)); specific named types are defined at implementation time.

- This document does not prescribe **how often** snapshots are consumed or at what granularity derived views are materialized. Frequency is an implementation and performance decision subject to the semantic constraints above.

- This document does not define **recovery procedures** for corrupt or missing snapshot anchors. Failure semantics are addressed in [Failure Semantics](failure-semantics.md).

---

## Boundaries to other documents

| Document | Relationship |
| -------- | ------------ |
| [Event Model](event-model.md) | Defines Market Events including order book snapshots as a canonical Event category; defines what qualifies as an Event |
| [State Model](state-model.md) | Defines `State = f(Event Stream, Configuration)` and the projection model that snapshot-driven views must conform to |
| [Time Model](time-model.md) | Defines Event Time vs Processing Order; snapshot Event Time carries external metadata but does not override Processing Order |
| [Determinism Model](determinism-model.md) | Defines what breaks determinism; snapshot anchors that are not replay-stable fall under hidden mutable state violations |
| [Invariants](invariants.md) | E1, E2, D3: Events as sole State transition source; no hidden mutable inputs; no out-of-band State mutation |
| [Failure Semantics](failure-semantics.md) | Defines how failures in snapshot delivery or processing must be handled within the canonical model |

---

## Open boundaries intentionally left abstract

1. **Named Event types for snapshot inputs.** The Event Model acknowledges that Market Events may include snapshots (e.g., order book snapshots), but the specific named Event types and their schema are implementation decisions not fixed by the canonical model.

2. **Configuration anchor format and versioning scheme.** The mechanism by which a snapshot anchor is fixed as a versioned Configuration input — how it is stored, identified, and associated with a processing scope — is an implementation detail.

3. **Frequency and scope of derived State projections.** Which projections of derived State are maintained as snapshots, at what granularity, and how they are kept consistent with the Event Stream during processing are implementation decisions governed by the semantic constraints above but not prescribed by them.

4. **Handling of snapshot sequence gaps.** How the infrastructure responds when an expected incremental update sequence is disrupted and a new snapshot must be consumed (e.g., feed reconnect with sequence gap) is an operational and implementation concern. The canonical constraint — that any resulting State transition must enter through Event processing — applies, but the specific recovery protocol is not defined here.
