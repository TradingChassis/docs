# Internal Structure

This document defines the logical internal structure of the Data Storage Stack: its primary storage classes, their roles, and the structural relationships between them.

---

## Structural Overview

The Data Storage Stack is not a processing pipeline. Its internal structure is a **logical storage organization** — a set of persistent storage classes, each with a defined role, that together provide the Infrastructure's durable substrate.

The key structural principle is **logical separation**. Each storage class holds a distinct category of persisted data, has defined writers and readers, and carries a stable meaning within the Infrastructure. These classes are logically distinct regardless of whether they share physical infrastructure. The Data Storage Stack's internal structure is the arrangement of these classes and the boundaries between them.

---

## Primary Storage Classes

### Persistent Raw Storage

Holds raw recorded datasets as produced by the Data Recording Stack — Venue market-data messages with source and timing metadata, partitioned by Venue, Feed, and Time Window, written without transformation or normalization.

**Role:** Durable retention of the Infrastructure's raw empirical record before any quality assessment or canonical promotion has occurred.

### Canonical Storage

Holds validated, normalized datasets promoted by the Data Quality Stack. Canonical datasets have passed integrity, completeness, and consistency assessment and have been transformed into canonical form.

**Role:** The authoritative validated dataset layer for the Infrastructure. Canonical Storage is the persistent surface from which Backtesting, Analysis, and other downstream consumers read validated Research inputs.

### Derived Storage

Holds datasets and artifacts derived from canonical data, execution records, or other persisted sources — for example, aggregated statistics, computed features, or analytical outputs produced by the Analysis Stack or other processing.

**Role:** Durable retention of computed or derived artifacts that are distinct from both raw recordings and canonical datasets.

### Quarantine Storage

Holds datasets that the Data Quality Stack has excluded from canonical promotion — datasets that failed validation, consistency assessment, or normalization, and that have been routed to quarantine for review or potential reprocessing.

**Role:** Durable retention of non-promoted datasets in a logically separate location, so that quarantined data does not co-mingle with canonical data and remains available for diagnostic or recovery purposes.

### Experiment / Artifact Storage

Holds Backtesting outputs, experiment results, evaluation artifacts, and related Research outputs.

**Role:** Durable retention of experiment-specific artifacts so that Backtesting runs and their outputs are preserved, retrievable, and available for Analysis.

### Execution Record Storage

Holds execution records and operational artifacts produced by the Live Stack and the Backtesting Stack — persisted traces of execution activity, dispatch records, and related operational data.

**Role:** Durable retention of execution history for post-hoc analysis, audit, and operational review.

### Note on normalized storage

Implementations may include intermediate storage for normalized datasets (e.g., datasets that have been normalized but not yet promoted). This is a valid implementation-level concern, but `normalized` is not modeled as a primary top-level storage class in this specification. Normalized data in transit between validation and promotion is a Data Quality Stack processing concern; if it is persisted, it resides in an implementation-specific staging area rather than in a named primary storage class of the Data Storage Stack.

---

## Structural Relationships Between Storage Classes

The storage classes are not stages in a sequential pipeline. They are **parallel persistent domains**, each serving a different category of data, with different writers and different readers.

However, some structural relationships exist:

**Raw ➝ Canonical progression.** Persistent Raw Storage holds the input material from which canonical datasets are eventually derived. A dataset moves from raw to canonical only through the Data Quality Stack's validation and promotion process. The Data Storage Stack provides both surfaces; it does not control or participate in the transition.

**Canonical ➝ Derived dependency.** Derived Storage may hold artifacts computed from canonical datasets. The relationship is downstream: derived artifacts depend on canonical inputs, but Canonical Storage does not depend on Derived Storage.

**Quarantine as a parallel outcome.** Quarantine Storage holds datasets that were candidates for Canonical Storage but did not pass quality assessment. It exists alongside Canonical Storage, not in sequence with it. A quarantined dataset is not "on its way" to Canonical Storage — it has been explicitly excluded.

**Experiment and Execution Record as output domains.** Experiment / Artifact Storage and Execution Record Storage hold outputs of the Core Runtime Stacks (Backtesting, Live). They do not feed back into Canonical Storage or Persistent Raw Storage. Their relationship to the other storage classes is consumption-side: they store what downstream Stacks produce, and the Analysis Stack may later read from them.

---

## Internal Structural Boundaries

**Logical separation is mandatory.** Each storage class must remain logically distinct, even when multiple classes share the same physical storage infrastructure. A dataset's storage-class membership must be unambiguous: a dataset in Persistent Raw Storage is raw; a dataset in Canonical Storage is canonical; a dataset in Quarantine Storage is quarantined. These classifications must not be blurred by shared infrastructure.

**No semantic processing.** The Data Storage Stack's internal structure organizes and separates persistent data. It does not validate, normalize, promote, quarantine, or transform data. Those are responsibilities of other Stacks. The internal structure provides the durable surfaces on which those Stacks' decisions are stored.

**No Core Runtime semantics.** The storage classes described here hold persistent datasets and artifacts. They do not participate in or define the Core Runtime Event Stream, State derivation, Processing Order, or any execution-path processing.

**Logical structure, not deployment specification.** The storage classes described here are logical roles. They may be realized as separate object-store buckets, distinct prefixes within a shared bucket, separate filesystem directories, or distinct database schemas. The physical realization is not specified by this document; the logical separation is.
