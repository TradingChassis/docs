# Data Recording Stack Overview

Part of: **Data Platform**

The Data Recording Stack captures raw market data from real Venues and preserves it as raw recorded datasets for downstream use by the Data Platform.

---

## Purpose

The Data Recording Stack exists to reliably obtain and durably store the Infrastructure's raw empirical market-data record. It connects to external Venue market-data feeds, receives raw messages — trades, order book updates, and associated metadata — and persists them as raw recorded datasets with source and timing metadata preserved.

This Stack focuses exclusively on data capture and raw persistence. It does not validate, normalize, or promote datasets. Those responsibilities belong to downstream Stacks in the Data Platform.

---

## Position in the Infrastructure

The Data Recording Stack is the first Stack in the Data Platform flow:

`Venue Market-Data Sources → Data Recording → Data Quality → Data Storage`

It sits between external Venue market-data sources and the internal Data Platform. It is upstream of the Data Quality Stack, the Data Storage Stack, the Backtesting Stack, and the Analysis Stack.

The Data Recording Stack is **not** part of the **Core Runtime**. It does not participate in the Core Runtime Event Stream, State derivation, Strategy evaluation, Risk, Execution Control, or Venue Adapter interactions. The Core Runtime and the Data Recording Stack operate in separate architectural domains.

---

## Main Responsibilities

The Data Recording Stack is responsible for:

- Connecting to real Venue market-data sources.
- Receiving raw market-data feed messages (trades, order book updates, associated data).
- Recording those messages reliably without transformation or semantic interpretation.
- Preserving source metadata and provenance (Venue identity, feed identity, connection conditions).
- Preserving timing metadata, including **Capture Time** — the local time at which a recording component observed or received a raw Venue message.
- Buffering incoming data locally (**Local Buffer**) where necessary to absorb transient write latency.
- Durably persisting raw recorded datasets to **Persistent Raw Storage**.
- Creating **Dataset Manifests** — metadata artifacts that describe recorded datasets and signal their availability for downstream discovery.

The semantic unit of recorded data is a **raw recorded dataset**, partitioned by Venue, Feed, and Time Window.

---

## Key Boundaries

**Not the Core Runtime Event Stream.** The Data Recording Stack captures raw Venue market data. It does not process, produce, or participate in the canonical Event Stream that drives the Core Runtime. Recording metadata such as Capture Time is recording-domain context, not a Core Runtime causal concept.

**Not Canonical Storage.** The Data Recording Stack writes to Local Buffer and Persistent Raw Storage. It does not write to Canonical Storage. Even where raw and canonical layers share physical storage infrastructure, the logical separation is strict. Promotion from raw recorded datasets to canonical validated datasets is a downstream responsibility of the Data Quality and Data Storage Stacks.

**No validation or normalization.** The Data Recording Stack records raw data faithfully. It does not detect gaps, validate schemas, or normalize Venue-specific formats. Those are Data Quality Stack responsibilities.

---

## Relationship to Other Stacks

**Data Quality Stack.** The primary downstream consumer. The relationship is **asynchronous and discovery-based**: the Data Recording Stack persists raw datasets and publishes Dataset Manifests; the Data Quality Stack discovers and consumes them on its own schedule. There is no synchronous handoff or active push.

**Data Storage Stack.** No direct interaction. Raw recorded datasets reach Canonical Storage only after passing through the Data Quality Stack's validation and normalization pipeline and the Data Storage Stack's canonical promotion process.

**Backtesting and Analysis Stacks.** Indirect downstream dependents. Both consume canonical datasets that originate from raw recorded data captured by this Stack. The quality, completeness, and timing fidelity of the raw record constrain the realism available to all downstream Research.

---

## Why the Stack Matters

The Data Recording Stack defines the empirical basis on which all downstream Research and Backtesting depend. If recorded data is incomplete, poorly timed, or missing provenance, no amount of downstream processing can recover what was not captured.

For microstructure-sensitive Research, the conditions under which data is recorded are part of the data's meaning. The design principle of **environment parity** — aligning external recording conditions (feed path, locality, latency profile) and internal pipeline behavior (recording software, processing path) with the eventual Live environment — strengthens the realism of recorded data as a Research foundation. Where residual differences between recording and Live environments exist, they should be identified, modeled, and measured so their impact can be assessed.

Detailed treatment of scope and role, interfaces, internal structure, operational behavior, and implementation considerations is provided in the companion documents for this Stack.
