# Interfaces

This document defines the persistent storage surfaces exposed by the Data Storage Stack, how other Stacks interact with those surfaces, and what the interface boundaries mean for a storage substrate.

---

## Inputs and Outputs

The Data Storage Stack is a persistent substrate, not an active semantic processor. Its interface model differs from Stacks that transform or evaluate data in transit.

### Inputs

Other Stacks write to the Data Storage Stack's persistent surfaces:

- **Persistable datasets** — raw recorded datasets, canonical datasets, derived datasets.
- **Persistable records** — execution records, operational artifacts.
- **Persistable artifacts** — experiment outputs, evaluation results, metadata.

The Data Storage Stack accepts these writes and ensures they are durably stored in the appropriate logical storage class. It does not evaluate, validate, or transform what is written.

### Outputs

Other Stacks read from the Data Storage Stack's persistent surfaces:

- **Durable stored datasets** — retrievable by the consuming Stack from the relevant storage class.
- **Durable stored records and artifacts** — retrievable in the same manner.
- **Logically separated storage classes** — each class exposes a distinct retrieval surface so that consumers can access raw, canonical, derived, quarantine, experiment, or execution-record data without ambiguity.

The Data Storage Stack does not push data to consumers. All retrieval is initiated by the consuming Stack against the relevant storage surface.

---

## Storage Surfaces

The Data Storage Stack exposes the following primary storage classes as persistent surfaces:

| Storage class | What it holds | Primary writers | Primary readers |
| ------------- | ------------- | --------------- | --------------- |
| **Persistent Raw Storage** | Raw recorded datasets as produced by the Data Recording Stack | Data Recording Stack | Data Quality Stack |
| **Canonical Storage** | Validated, normalized datasets promoted by the Data Quality Stack | Data Quality Stack | Backtesting Stack, Analysis Stack |
| **Derived Storage** | Datasets and artifacts derived from canonical or execution data | Backtesting Stack, Analysis Stack | Analysis Stack |
| **Quarantine Storage** | Datasets excluded from canonical promotion, held for review | Data Quality Stack | Data Quality Stack, operational tooling |
| **Experiment / Artifact Storage** | Backtesting outputs, experiment results, evaluation artifacts | Backtesting Stack | Analysis Stack |
| **Execution Record Storage** | Execution records and operational artifacts from Live and Backtesting | Live Stack, Backtesting Stack | Analysis Stack, operational tooling |

Each storage class is a logically distinct surface. The Data Storage Stack provides these surfaces; it does not decide what is written to them. Promotion decisions (what enters Canonical Storage), quarantine decisions (what enters Quarantine Storage), and recording decisions (what enters Persistent Raw Storage) are made by the respective producing Stacks.

---

## Relationship to the Data Recording Stack

The Data Recording Stack writes raw recorded datasets and Dataset Manifests to **Persistent Raw Storage**. The Data Storage Stack provides this surface — durable, logically distinct from Canonical Storage, accessible to the Data Quality Stack for downstream consumption.

The Data Storage Stack does not participate in recording. It does not connect to Venue feeds, manage Local Buffers, or annotate timing metadata. It provides the persistent surface to which the Data Recording Stack writes its finished raw datasets.

---

## Relationship to the Data Quality Stack

The Data Quality Stack interacts with two storage surfaces:

- **Persistent Raw Storage** — read surface. The Data Quality Stack reads raw recorded datasets from this surface for validation and assessment.
- **Canonical Storage** — write surface. The Data Quality Stack writes validated, normalized datasets to this surface upon successful promotion.
- **Quarantine Storage** — write surface. The Data Quality Stack writes datasets excluded from canonical promotion to this surface.

The Data Storage Stack provides these surfaces but does not participate in the validation, normalization, or promotion logic. The decision of whether a dataset becomes canonical is made entirely by the Data Quality Stack. The Data Storage Stack's role is to ensure that Canonical Storage and Quarantine Storage exist as logically separate, durable surfaces — not to evaluate what is written to them.

---

## Relationship to Backtesting, Live, and Analysis

### Backtesting Stack

- Reads canonical datasets from **Canonical Storage** as historical input for Strategy evaluation.
- Writes experiment outputs, execution records, and evaluation artifacts to **Experiment / Artifact Storage** and **Execution Record Storage**.

### Live Stack

- Writes execution records and operational artifacts to **Execution Record Storage**.

### Analysis Stack

- Reads from **Canonical Storage**, **Derived Storage**, **Experiment / Artifact Storage**, and **Execution Record Storage** as needed for dataset inspection, experiment analysis, and performance evaluation.
- May write derived datasets or analysis artifacts to **Derived Storage**.

In all cases, the Data Storage Stack provides the persistent surface. It does not participate in the semantic processing that Backtesting, Live, or Analysis perform — it stores their inputs and outputs durably and makes them retrievable.

---

## Logical Separation Boundary

The Data Storage Stack enforces **logical separation** between storage classes. Each class has a defined role, defined writers, and defined readers. This separation is mandatory even when multiple storage classes share physical infrastructure.

A single object store, filesystem, or storage cluster may host Persistent Raw Storage, Canonical Storage, Quarantine Storage, and other classes simultaneously. Logical separation is maintained through distinct namespaces, paths, or organizational boundaries — not through physical isolation alone.

The consequence of this principle is that every persisted dataset or artifact has an unambiguous storage-class membership. A dataset in Persistent Raw Storage is raw. A dataset in Canonical Storage is canonical. A dataset in Quarantine Storage is quarantined. These classifications are determined by which surface the producing Stack writes to, and they remain stable once written. The Data Storage Stack preserves these boundaries; other Stacks rely on them to determine what they are reading.
