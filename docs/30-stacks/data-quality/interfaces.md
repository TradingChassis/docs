# Interfaces

This document defines the inputs, outputs, and interface boundaries of the Data Quality Stack — what enters it, what leaves it, and how it relates to upstream and downstream Stacks.

---

## Inputs

The Data Quality Stack consumes two primary inputs from the Data Recording Stack's outputs:

### Dataset Manifests

Metadata artifacts published by the Data Recording Stack that describe recorded raw datasets and signal their availability. The Data Quality Stack discovers new work by locating Dataset Manifests — it does not receive active notifications, synchronous calls, or push deliveries from the Data Recording Stack.

### Raw recorded datasets

The raw data itself, loaded from **Persistent Raw Storage** based on information in the corresponding Dataset Manifest. Each raw recorded dataset is partitioned by Venue, Feed, and Time Window and contains the original Venue messages with source and timing metadata preserved.

The Data Quality Stack consumes **raw Venue market data only** — trades, order book updates, and associated metadata as recorded by the Data Recording Stack. It does not consume or process the Core Runtime Event Stream.

---

## Outputs

The Data Quality Stack produces four categories of output:

### Canonical datasets

Validated, normalized datasets promoted to **Canonical Storage**. These are the authoritative validated datasets available to downstream consumers. A dataset becomes canonical only after it has passed through the Data Quality Stack's validation, normalization, and promotion logic. There is no alternative path into Canonical Storage.

### Validation metadata

Metadata documenting the assessment outcome for each processed dataset — what checks were applied, what passed, what failed, and the resulting quality classification. Validation metadata is available regardless of whether the dataset was ultimately promoted or not.

### Promotion metadata

Metadata documenting the promotion outcome — whether the dataset was promoted to Canonical Storage, when promotion occurred, and what canonical location it was written to. Promotion metadata accompanies each successfully promoted dataset.

### Non-promotion outcomes

Datasets that fail validation are not promoted. These produce non-promotion outcomes: **quarantine** (the dataset is held for review or reprocessing) or **rejection** (the dataset is definitively excluded from canonical promotion). Non-promotion outcomes and their associated validation metadata remain available for diagnostic and operational purposes.

---

## Relationship to the Data Recording Stack

The Data Recording Stack is the sole upstream source of raw data for the Data Quality Stack. The relationship is **asynchronous and discovery-based**:

1. The Data Recording Stack persists raw recorded datasets to Persistent Raw Storage and publishes Dataset Manifests.
2. The Data Quality Stack discovers Dataset Manifests and loads the corresponding raw datasets from Persistent Raw Storage on its own schedule.

There is no synchronous handoff, active push, or callback between the two Stacks. The Data Quality Stack depends on the presence of Dataset Manifests in a known location and the availability of the corresponding raw datasets in Persistent Raw Storage. It does not depend on the Data Recording Stack's internal state or runtime activity.

The Data Quality Stack does not manage Persistent Raw Storage, Local Buffer, or any recording-side infrastructure. Those are Data Recording Stack concerns.

---

## Relationship to Canonical Storage

Canonical Storage is the downstream destination for datasets that pass the Data Quality Stack's validation and promotion process.

The Data Quality Stack is the **exclusive promotion path** into Canonical Storage. No dataset enters Canonical Storage without passing through this Stack's validation, normalization, and eligibility logic. This is the mechanism that ensures Canonical Storage contains only assessed, validated, normalized datasets.

The Data Quality Stack writes promoted datasets to Canonical Storage but does not govern the storage layer itself — organization, retention, and access management of Canonical Storage are Data Storage Stack responsibilities.

Persistent Raw Storage and Canonical Storage are logically distinct, even where they share physical infrastructure. The Data Quality Stack enforces this boundary: it reads from Persistent Raw Storage (raw) and writes to Canonical Storage (canonical). The two are never co-mingled from the Data Quality Stack's perspective.

---

## Relationship to Downstream Consumers

The primary downstream consumers of the Data Quality Stack's outputs are the **Backtesting Stack** and the **Analysis Stack**. These Stacks consume canonical datasets from Canonical Storage — they do not interact with the Data Quality Stack directly during normal operation.

The interface is mediated by Canonical Storage: the Data Quality Stack promotes datasets; downstream consumers read promoted datasets from Canonical Storage. The Data Quality Stack does not push data to Backtesting or Analysis, does not notify them of promotions, and does not coordinate their consumption schedules.

Validation metadata and promotion metadata may also be consumed by operational and diagnostic tooling to assess data-platform health, track promotion throughput, and investigate non-promotion outcomes. These are secondary interface consumers.

---

## Dataset Discovery and Promotion Boundary

The Data Quality Stack operates across two interface boundaries:

### Discovery boundary (upstream)

The **Dataset Manifest** is the discovery boundary between the Data Recording Stack and the Data Quality Stack. The Data Quality Stack's processing begins when it discovers a Dataset Manifest and loads the corresponding raw dataset. Before discovery, the dataset is invisible to the Data Quality Stack. After discovery, the dataset enters the Data Quality Stack's processing scope.

### Promotion boundary (downstream)

**Canonical Storage** is the promotion boundary between the Data Quality Stack and all downstream consumers. A dataset crosses this boundary only when the Data Quality Stack writes a validated, normalized dataset to Canonical Storage with accompanying promotion metadata. Before promotion, the dataset is not canonical and is not available to downstream consumers through Canonical Storage.

Datasets that do not cross the promotion boundary — those that are quarantined or rejected — remain in a non-promoted state. They do not enter Canonical Storage and are not visible to downstream consumers as canonical data. Their validation metadata and non-promotion status remain available for diagnostic purposes.

The progression of a dataset through the Data Quality Stack — from Discovered through validation, normalization, and eligibility assessment to Promoted, Quarantined, or Rejected — determines which boundary it ultimately crosses. The detailed state model is defined in companion documents; at the interface level, the relevant distinction is between datasets that have been promoted (canonical, available downstream) and those that have not (non-canonical, excluded from Canonical Storage).
