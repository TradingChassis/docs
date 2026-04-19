# Interfaces

This document defines the inputs, outputs, and interface boundaries of the Data Recording Stack — what enters it, what leaves it, and how it relates to upstream sources and downstream consumers.

---

## Inputs

The Data Recording Stack receives **raw market-data feed messages** from real Venues. These are external data sources outside the Infrastructure boundary.

Typical input data includes:

- **Trade / tape messages** — individual executed trades reported by the Venue.
- **Order book updates** — incremental or snapshot updates to the Venue's order book state.
- **Source metadata** — Venue identity, feed identity, connection parameters, and other provenance information associated with each feed.
- **Timing metadata** — timestamps carried by the Venue message itself, supplemented by **Capture Time**: the local time at which the Data Recording component observes or receives the message.

The Data Recording Stack receives **raw Venue data only**. It does not receive or process the Core Runtime Event Stream. The canonical Event Stream, Processing Order, and State derivation belong to the Core Runtime and do not intersect with data recording inputs.

---

## Outputs

The Data Recording Stack produces two categories of output:

### Raw recorded datasets

Durably persisted collections of raw Venue messages, written to **Persistent Raw Storage**. Each dataset preserves the original message content, source metadata, and timing metadata (including Capture Time) without transformation, normalization, or semantic interpretation.

### Dataset Manifests

Metadata artifacts that describe a recorded dataset — its Venue, feed, time window, storage location, and completeness status — and signal that the dataset is available for downstream discovery. The Dataset Manifest is the mechanism by which the Data Recording Stack communicates readiness to downstream consumers.

The Data Recording Stack writes to **Local Buffer** (temporary storage on the Capture Node during recording) and to **Persistent Raw Storage** (durable storage for completed raw datasets). It does **not** write to **Canonical Storage**. Even where Persistent Raw Storage and Canonical Storage share physical infrastructure, the logical separation is strict. Raw recorded datasets are not canonical datasets; promotion to Canonical Storage is a downstream responsibility.

---

## Relationship to Venues

Venues are external market-data sources outside the Infrastructure boundary. The Data Recording Stack connects to Venue market-data feeds to receive raw messages.

This relationship is **inbound only** at the data-recording level: the Stack receives data from Venues but does not send execution requests to them. Outbound Venue interaction (order submission, modification, cancellation) belongs to the Core Runtime's Venue Adapter, which is architecturally separate from the Data Recording Stack. The term "Venue Adapter" is not used for Data Recording components; feed connectivity here is handled by Feed Connectors within the Data Recording Stack.

---

## Relationship to Data Quality Stack

The Data Quality Stack is the primary downstream consumer of the Data Recording Stack's output.

The relationship is **asynchronous and discovery-based**. The Data Recording Stack does not actively push data to the Data Quality Stack, invoke it, or notify it through a synchronous handoff. Instead:

1. The Data Recording Stack persists a raw recorded dataset to Persistent Raw Storage.
2. The Data Recording Stack creates a Dataset Manifest describing the dataset and its availability.
3. The Data Quality Stack discovers the Dataset Manifest and consumes the raw recorded dataset on its own schedule.

This boundary ensures that the Data Recording Stack's responsibilities end at durable persistence and readiness signaling. Validation, gap detection, schema normalization, and all other quality processing are the Data Quality Stack's concern.

---

## Relationship to Data Storage Stack

The Data Recording Stack does not interact with the Data Storage Stack directly. Raw recorded datasets reside in Persistent Raw Storage, which is logically distinct from Canonical Storage. The path from raw data to Canonical Storage passes through the Data Quality Stack (validation, normalization) and the Data Storage Stack (canonical promotion and governance). The Data Recording Stack has no role in that promotion path.

---

## Relationship to Downstream Consumers

Beyond the Data Quality Stack, other Stacks (Backtesting, Analysis) may consume raw recorded datasets from Persistent Raw Storage for purposes such as diagnostic analysis or raw-data inspection. These interactions follow the same pattern: the consumer discovers datasets via Dataset Manifests and reads from Persistent Raw Storage. The Data Recording Stack does not mediate or coordinate these reads.

No downstream consumer receives data from the Data Recording Stack through active delivery. All downstream access is asynchronous, discovery-based, and read-oriented against persisted artifacts.

---

## Recorded Data Unit and Discovery Boundary

The semantic unit of recorded data is a **raw recorded dataset**, partitioned by:

| Dimension | Meaning |
| --------- | ------- |
| **Venue** | The external source from which data was captured |
| **Feed** | The specific market-data feed within that Venue |
| **Time Window** | The temporal extent of the recorded data |

Each raw recorded dataset is a self-contained unit: it includes the raw messages, source metadata, and timing metadata for one Venue, one feed, and one time window. The **Dataset Manifest** describes this unit and marks it as available for downstream discovery.

The Dataset Manifest is the **discovery boundary** between the Data Recording Stack and all downstream consumers. Downstream Stacks do not depend on the Data Recording Stack's internal state, runtime status, or active cooperation. They depend on the presence of a Dataset Manifest in a known location and the corresponding raw recorded dataset in Persistent Raw Storage.
