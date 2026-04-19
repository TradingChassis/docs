# Implementation Notes

This document captures implementation-facing rationale and design considerations for the Data Storage Stack: how the logical storage-class model may be realized in practice, why specific realization choices matter, and what patterns support the logical separation that the stack requires.

---

## Why Storage Realization Matters

The Data Storage Stack defines six primary storage classes, each with a distinct role. How those classes are realized in actual storage infrastructure determines whether the logical separation that the architecture requires is preserved in practice.

If storage classes are mapped carelessly — for example, raw and canonical datasets co-mingled under a single undifferentiated prefix, or quarantined datasets placed alongside promoted ones without clear boundaries — the logical model collapses. Consuming Stacks cannot reliably determine what they are reading, and the authority of Canonical Storage as the validated dataset layer is undermined.

Storage realization choices are therefore architectural choices, even though they operate at the implementation layer. They must be made deliberately and maintained consistently.

---

## Logical vs Physical Separation

The primary storage classes — Persistent Raw Storage, Canonical Storage, Derived Storage, Quarantine Storage, Experiment / Artifact Storage, Execution Record Storage — are **logically distinct**. This distinction is mandatory.

Physical separation is **not** mandatory. Multiple storage classes may reside on the same physical storage system — the same object store, the same filesystem, the same cluster. What is mandatory is that the logical boundaries between classes remain unambiguous in the realization:

- A consuming Stack reading from Canonical Storage must not inadvertently read raw or quarantined data.
- A producing Stack writing to Quarantine Storage must not inadvertently write to Canonical Storage.
- An operator inspecting storage contents must be able to determine which class a given dataset belongs to from its location alone.

Logical separation may be realized through any mechanism that provides stable, unambiguous boundaries: separate buckets, distinct top-level prefixes within a shared bucket, separate filesystem directories, distinct database schemas, or equivalent organizational tools. The choice depends on operational and infrastructure constraints. The requirement is that the chosen mechanism preserves the logical model under all normal operating conditions.

---

## Example Storage-Class Realization Patterns

A common realization pattern uses an object store (e.g., S3-compatible storage) with top-level prefixes or buckets that map directly to logical storage classes:

`raw/`  
`canonical/`  
`derived/`  
`quarantine/`  
`experiments/`  
`execution-records/`

Within each top-level class, further partitioning by Venue, Feed, Time Window, experiment identifier, or execution session provides navigability and supports the access patterns of consuming Stacks.

For example, Persistent Raw Storage might be organized as:

`raw/{venue}/{feed}/{time-window}/`

Canonical Storage might follow the same partitioning:

`canonical/{venue}/{feed}/{time-window}/`

This parallel structure makes it straightforward to trace a dataset from its raw form through to its canonical promotion — the partitioning keys are the same, and the top-level class prefix distinguishes the storage-class membership.

**Separate buckets vs shared bucket with prefixes.** Both are valid realization strategies. Separate buckets provide stronger physical isolation and may simplify access-policy management. A shared bucket with distinct prefixes is operationally simpler and may reduce infrastructure overhead. The choice is an implementation trade-off; the logical separation requirement is satisfied by either approach as long as boundaries are stable and unambiguous.

---

## Retention, Immutability, and Access Characteristics

Different storage classes serve different purposes and may therefore exhibit different implementation-level characteristics:

**Persistent Raw Storage.** Receives a continuous flow of raw datasets during active recording. Write volume is high relative to other classes. Retention is long-term — the raw empirical record is the foundation of the data platform. Access is primarily by the Data Quality Stack for assessment. Raw datasets are not modified after writing, but they are not subject to the same immutability guarantee as canonical datasets.

**Canonical Storage.** Receives datasets only through the Data Quality Stack's promotion process. Write frequency is lower than raw. **Immutability after promotion is an implementation-level invariant** — corrections produce new dataset versions, not in-place modifications. Canonical datasets are read by multiple downstream Stacks (Backtesting, Analysis) and must remain stable and unmodified once promoted. Retention is indefinite under normal conditions.

**Quarantine Storage.** Receives datasets excluded from canonical promotion. Write frequency is variable. Quarantined datasets may eventually be re-evaluated, deleted, or archived. Retention and lifecycle handling may be more flexible than for canonical data — quarantined content is not authoritative and may be subject to periodic review and cleanup.

**Derived Storage.** Receives computed artifacts. Write and read patterns depend on Analysis workflows. Retention may be shorter than canonical — derived artifacts can typically be recomputed from their canonical inputs.

**Experiment / Artifact Storage.** Receives Backtesting outputs and experiment results. Write volume depends on experiment frequency. Retention may be governed by experiment-lifecycle needs — some artifacts are kept long-term for comparison, others may be archived or removed after evaluation.

**Execution Record Storage.** Receives execution records from Live and Backtesting. Retention may be governed by audit and operational-review requirements.

These characteristics are implementation-level considerations, not canonical system semantics. They describe how storage classes are expected to behave in practice and inform realization decisions such as storage-tier selection, lifecycle policies, and backup strategies.

---

## Intermediate and Non-Primary Storage Classes

Implementations may introduce storage areas that do not correspond to the six primary storage classes. The most common example is **normalized storage** — a staging area for datasets that have been normalized by the Data Quality Stack but not yet promoted to Canonical Storage.

Normalized storage is a valid implementation-level concern: the Data Quality Stack may need a durable location to write normalized output before the promotion gate evaluates it. However, `normalized` is not modeled as a primary top-level storage class because it represents a transient processing state within the Data Quality Stack's pipeline, not a persistent storage domain with its own consuming Stacks and long-term retention semantics.

In implementation terms, normalized storage may be realized as:

- A staging prefix within the Data Quality Stack's working area (e.g., `staging/normalized/`).
- A temporary location that is cleared after successful promotion or quarantine.
- A prefix adjacent to but logically distinct from Canonical Storage.

The key requirement is that normalized staging does not blur the boundary between Canonical Storage and non-canonical data. A dataset in a normalized staging area is **not yet canonical** — it becomes canonical only when the Data Quality Stack writes it to Canonical Storage through the promotion gate.

Other intermediate areas — validation scratch space, processing temporary files, workflow-engine working directories — may also exist in implementations. These are transient infrastructure artifacts and do not need to be modeled as storage classes.

---

## Implementation Boundaries

**Implementation choices, not canonical semantics.** The realization patterns, retention characteristics, and organizational examples described here are implementation-level concerns. They do not define or modify the Infrastructure's canonical Event, State, or lifecycle semantics.

**Logical separation is the invariant.** Regardless of how storage is physically organized, the logical distinction between storage classes must be preserved. This is the single non-negotiable implementation requirement that this document reinforces.

**No deployment specification.** The patterns above describe design considerations and example layouts. They do not prescribe specific cloud vendors, object-store configurations, bucket naming conventions, or infrastructure-as-code templates. Those belong in deployment documentation.

**No Core Runtime interaction.** The Data Storage Stack's implementation does not intersect with the Core Runtime Event Stream, State derivation, or execution-path processing. Storage realization operates entirely within the Data Platform domain.
