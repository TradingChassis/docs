# Operational Behavior

This document describes how the Data Storage Stack behaves during normal operation: how its persistent surfaces are used by other Stacks, how logical separation is maintained in practice, and how durability and availability function as architectural concerns.

---

## Persistent Availability and Use

The Data Storage Stack operates as a **continuously available persistent substrate**. It does not run processing jobs, evaluate data quality, or transform datasets. Its operational behavior is defined by the persistent surfaces it makes available and the durability guarantees those surfaces provide.

During normal operation:

- Storage classes are **durably available for writes** from their designated producing Stacks. The Data Recording Stack writes to Persistent Raw Storage; the Data Quality Stack writes to Canonical Storage and Quarantine Storage; the Backtesting and Live Stacks write to Experiment / Artifact Storage and Execution Record Storage.
- Storage classes are **durably available for reads** by their designated consuming Stacks. The Data Quality Stack reads from Persistent Raw Storage; the Backtesting and Analysis Stacks read from Canonical Storage; the Analysis Stack reads from Derived Storage, Experiment / Artifact Storage, and Execution Record Storage.
- **Persistence is durable.** Once a dataset, record, or artifact is successfully written to a storage surface, it survives infrastructure restarts, process failures, and operational changes. Consuming Stacks can rely on the continued availability of persisted content without depending on the producing Stack's runtime state.

The Data Storage Stack does not push data to consumers or notify them of new content. All access is **initiated by the consuming Stack** against the relevant storage surface. The Stack's operational role is to ensure that surfaces are available and that persisted content remains retrievable.

---

## Behavioral Role of Storage Classes

Although the storage classes share the Data Storage Stack as their persistent substrate, they exhibit different behavioral characteristics in operation:

**Persistent Raw Storage** receives a continuous flow of raw recorded datasets during active recording sessions. Its content grows as the Data Recording Stack captures data from Venue feeds. Raw datasets are read by the Data Quality Stack for assessment and are retained as the Infrastructure's raw empirical record.

**Canonical Storage** receives datasets only through the Data Quality Stack's promotion process. Writes to Canonical Storage are less frequent than writes to Persistent Raw Storage and occur only after successful validation, normalization, and promotion. Canonical datasets are **immutable after promotion** — corrections produce new dataset versions rather than modifying existing ones. This immutability is an operational property of the storage class, not merely a policy preference.

**Quarantine Storage** receives datasets that the Data Quality Stack has excluded from canonical promotion. Quarantined content may be re-evaluated or investigated, but it is operationally inert with respect to downstream consumers — Backtesting and Analysis do not read from Quarantine Storage during normal operation.

**Derived Storage** receives computed or analytical artifacts produced from canonical or execution data. Its content grows as the Analysis Stack or other processing produces derived outputs. Derived datasets are secondary to canonical datasets — they depend on canonical inputs but do not feed back into Canonical Storage.

**Experiment / Artifact Storage** receives Backtesting outputs, experiment results, and evaluation artifacts. Its content grows with each Backtesting run. Experiment artifacts are retained for later analysis and comparison.

**Execution Record Storage** receives execution records and operational artifacts from the Live Stack and the Backtesting Stack. Its content represents the persisted trace of execution activity and is consumed by the Analysis Stack and operational tooling for post-hoc review.

---

## Cross-Stack Operational Dependence

Other Stacks depend on the Data Storage Stack through **durable persistence**, not through runtime coupling or transient message passing.

This means:

- The Data Recording Stack does not need the Data Quality Stack to be running in order to write raw datasets. It writes to Persistent Raw Storage; the Data Quality Stack reads from that surface later, on its own schedule.
- The Data Quality Stack does not need the Backtesting Stack to be running in order to promote canonical datasets. It writes to Canonical Storage; the Backtesting Stack reads from that surface when it needs historical inputs.
- The Analysis Stack does not need the Live Stack or Backtesting Stack to be running in order to inspect execution records or experiment artifacts. It reads from the relevant storage surfaces independently.

The Data Storage Stack is what makes this decoupling possible. By providing durable, continuously available storage surfaces, it allows producing and consuming Stacks to operate on independent schedules without requiring synchronous coordination.

---

## Operational Boundaries

**No semantic processing.** The Data Storage Stack does not evaluate, validate, transform, or decide the contents of what is written to its surfaces. It persists what producing Stacks write and makes it retrievable for consuming Stacks. Semantic decisions — what is canonical, what is quarantined, what is derived — are made by other Stacks before they write.

**No Core Runtime participation.** The Data Storage Stack's operational behavior does not intersect with the Core Runtime Event Stream, State derivation, or execution-path processing. It stores artifacts produced by Core Runtime Stacks (execution records, experiment outputs), but it does not participate in the runtime that produces them.

**Logical separation under shared infrastructure.** During normal operation, multiple storage classes may reside on the same physical storage system. Logical separation is maintained through organizational conventions (namespaces, paths, prefixes, or equivalent boundaries). The operational consequence is that a consuming Stack reading from Canonical Storage never inadvertently reads from Persistent Raw Storage or Quarantine Storage — the logical boundaries are stable and enforced regardless of physical co-location.

**Observability.** The operational health of the Data Storage Stack — surface availability, write/read success rates, storage utilization per class, durability status — should remain observable at a high level. The specifics of monitoring instrumentation and alerting are defined elsewhere.

---

## Why This Behavior Matters

The Data Storage Stack's operational behavior — durable availability, logical separation, and passive persistence — is what enables the rest of the Infrastructure to operate as decoupled Stacks with independent processing schedules.

If storage surfaces were unavailable or unreliable, producing Stacks could not persist their outputs and consuming Stacks could not retrieve their inputs. If logical separation collapsed — if raw and canonical data co-mingled in an undifferentiated storage pool — the distinction between unassessed raw data and validated canonical data would be operationally meaningless, regardless of what the Data Quality Stack had decided.

The Data Storage Stack's behavior is simple by design: persist durably, separate logically, remain available. That simplicity is what makes it a reliable foundation for the Stacks that depend on it.
