# Operational Behavior

This document describes how the Data Recording Stack behaves during normal operation: how capture proceeds, how buffered data becomes durably persisted, and how recorded datasets become discoverable by downstream consumers.

---

## Normal Recording Behavior

During normal operation, the Data Recording Stack continuously receives raw market-data messages from Venue feeds — trades, order book updates, and associated data — and records them with timing and source metadata attached.

Recording proceeds as a steady-state activity for the duration of each configured capture session. The Stack connects to Venue feeds, begins receiving messages, and records them as they arrive. Each raw message is annotated with **Capture Time** (the local time at which the recording component observed or received the message) and source metadata (Venue identity, feed identity, connection provenance) at the point of recording.

Recording is **continuous within a capture session**. The Stack does not batch-request historical data from Venues; it records what the feed delivers in real time. The recorded output represents the raw feed traffic as observed by the recording infrastructure, including whatever timing characteristics, message ordering, and delivery patterns the feed exhibited.

The Data Recording Stack records raw Venue market data only. It does not participate in the Core Runtime Event Stream, State derivation, or any execution-path processing.

---

## Buffering and Durable Persistence

Capture-rate intake and durable persistence operate at different cadences, decoupled by **Local Buffer**.

### Local Buffer

Incoming annotated raw messages are first written to Local Buffer — temporary local storage on the Capture Node. Local Buffer absorbs feed traffic at capture rate, preventing durable-write latency or transient storage delays from causing message loss at the intake layer.

Data in Local Buffer is **not yet durably persisted** and is **not available to downstream consumers**. It is transient working state internal to the recording process.

### Persistent Raw Storage

The Raw Writer reads from Local Buffer and writes completed raw recorded datasets to **Persistent Raw Storage**. Once data reaches Persistent Raw Storage, it is durably stored and survives Capture Node failure or restart.

Each raw recorded dataset in Persistent Raw Storage is partitioned by **Venue**, **Feed**, and **Time Window** — the semantic recorded-data unit. The dataset includes the raw message payload, Capture Time, and source metadata, written without transformation or normalization.

Persistent Raw Storage is **not Canonical Storage**. The Data Recording Stack does not write to Canonical Storage, and raw recorded datasets are not canonical datasets. The path from raw persistence to canonical promotion passes through the Data Quality and Data Storage Stacks.

### The transition from buffer to durable storage

The operational significance of this transition is durability and availability. Before the Raw Writer completes its write, data exists only in transient local form. After completion, the data is durably stored and can survive infrastructure disruption. This transition is the point at which raw data becomes a persistent artifact of the System rather than ephemeral capture-time state.

---

## Discoverability for Downstream Processing

Durable persistence alone does not make a recorded dataset available to downstream consumers. Discoverability requires a separate step: **Dataset Manifest publication**.

After a raw recorded dataset has been durably written to Persistent Raw Storage, the Dataset Manifest Writer creates a **Dataset Manifest** — a metadata artifact that describes the dataset (Venue, feed, time window, storage location, completeness status) and signals that it is ready for downstream consumption.

The Dataset Manifest is the mechanism by which the Data Recording Stack's operational output becomes visible to the rest of the Data Platform. Downstream consumers — primarily the Data Quality Stack — discover datasets by locating Dataset Manifests, not by monitoring the Data Recording Stack's internal state or runtime activity.

This interaction is **asynchronous**. The Data Recording Stack does not invoke, notify, or synchronously hand off to the Data Quality Stack. It persists data and publishes a manifest; the Data Quality Stack discovers and consumes on its own schedule. The Data Recording Stack's operational responsibility ends at manifest publication.

---

## Operational Boundaries

**No validation or quality assessment.** The Data Recording Stack records faithfully and persists durably, but does not assess whether the recorded data is complete, well-formed, or free of gaps. Quality assessment is the Data Quality Stack's operational concern.

**No normalization or promotion.** Recorded data is raw. Schema normalization and canonical promotion are downstream operational concerns that occur after the Data Quality Stack has processed the raw datasets.

**No Core Runtime interaction.** The Data Recording Stack operates independently of the Core Runtime. Its operational cycle — feed intake, buffering, persistence, manifest publication — does not intersect with Event processing, State derivation, Strategy evaluation, or any execution-path activity.

**Observability.** The operational health of recording — feed connectivity status, capture throughput, buffer utilization, persistence completion, manifest publication — should remain observable at a high level. The specifics of monitoring instrumentation and alerting thresholds are operational concerns defined elsewhere, not part of this document's scope.

---

## Why This Behavior Matters

The Data Recording Stack's operational behavior determines the completeness and fidelity of the raw empirical record on which all downstream Research and Backtesting depend. Gaps, timing inaccuracies, or metadata loss introduced during recording propagate into every dataset derived from the raw record.

For microstructure-sensitive Research, the conditions under which data is recorded are particularly significant. The design principle of **environment parity** — aligning the recording infrastructure with the conditions under which the Live Stack will later operate, both in external data access (network path, co-location, feed source) and internal pipeline behavior (same recording software, same processing path) — strengthens the realism of recorded data as a basis for Strategy evaluation. Where residual differences between recording and Live environments exist, they should be identified and measured so their impact on downstream analysis can be assessed. This is a strong design principle, not an absolute guarantee; the achievable degree of parity depends on operational and economic constraints.
