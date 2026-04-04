# Scope and Role

The Data Recording Stack captures raw market data from real Venues and preserves it as raw recorded datasets for downstream use by the Data Platform.

---

## Purpose

The Data Recording Stack exists to reliably obtain and durably store raw Venue market-data messages — trades, order book updates, and associated metadata — so that the System's empirical data foundation is complete, faithful to source, and available for downstream processing.

It is the System's point of contact with external Venue market-data sources. Everything downstream — validation, normalization, canonical promotion, Backtesting, Analysis — depends on the raw datasets this Stack produces. The quality, completeness, and timing fidelity of recorded data define the upper bound of realism available to all subsequent Research and operational workflows.

---

## Position in the System

The Data Recording Stack is part of the **Data Platform** — the first Stack in the Data Platform flow.

`Data Recording → Data Quality → Data Storage`

It sits between **external Venue market-data sources** and the internal Data Platform. It is upstream of the Data Quality Stack, the Data Storage Stack, the Backtesting Stack, and the Analysis Stack.

The Data Recording Stack is **not** part of the **Core Runtime**. It does not participate in the Core Runtime Event Stream, State derivation, Strategy evaluation, Risk, Execution Control, or Venue Adapter interactions. The Core Runtime and the Data Recording Stack operate in separate architectural domains: the Core Runtime processes Events and derives State; the Data Recording Stack captures raw Venue data and persists it for later use.

### Downstream relationship

The relationship between the Data Recording Stack and the Data Quality Stack is **asynchronous**. The Data Recording Stack does not actively hand off data to downstream consumers. Instead, it persists raw recorded datasets to **Persistent Raw Storage** and creates **Dataset Manifests** — metadata artifacts that signal downstream readiness and discoverability. Downstream Stacks discover and consume recorded datasets on their own schedule.

### Storage boundary

The Data Recording Stack writes to **Local Buffer** (temporary local storage on the Capture Node) and **Persistent Raw Storage** (durable storage for raw recorded datasets). It does **not** write directly to **Canonical Storage**. Even where raw and canonical storage share physical infrastructure, the logical separation between Persistent Raw Storage and Canonical Storage is strict. Promotion to Canonical Storage is the responsibility of the Data Quality and Data Storage Stacks, not the Data Recording Stack.

---

## Core Responsibilities

The Data Recording Stack is responsible for:

- Connecting to real Venue market-data sources (feeds, APIs).
- Receiving raw market-data feed messages — trades, order book updates, and related data.
- Recording those messages reliably without transformation or semantic interpretation.
- Preserving **source metadata and provenance**: which Venue, which feed, under what connection conditions.
- Preserving **timing metadata**, including **Capture Time** — the local time at which a Data Recording component observes or receives a raw Venue message.
- Buffering incoming data locally (Local Buffer) where necessary to absorb transient write latency or network interruption.
- Durably persisting raw recorded datasets to Persistent Raw Storage.
- Creating **Dataset Manifests** — metadata artifacts describing each recorded dataset and signaling its availability for downstream discovery.
- Making raw recorded datasets available for later asynchronous downstream processing by the Data Quality Stack and other consumers.

The semantic unit of recorded data is a **raw recorded dataset**, partitioned by **Venue**, **Feed**, and **Time Window**.

---

## Explicit Non-Responsibilities

The Data Recording Stack is **not** responsible for:

- **Dataset validation** — detecting gaps, anomalies, or schema violations (Data Quality Stack).
- **Schema normalization** — transforming raw Venue-specific formats into normalized representations (Data Quality Stack).
- **Canonical dataset promotion** — promoting validated datasets to Canonical Storage (Data Quality / Data Storage Stacks).
- **Canonical Storage governance** — managing the authoritative validated dataset layer (Data Storage Stack).
- **Core Runtime Event semantics** — defining, processing, or participating in the canonical Event Stream, Processing Order, or State derivation.
- **Strategy, Risk, Queue, or Order semantics** — these are Core Runtime concerns and do not intersect with data recording.
- **Lifecycle definitions** — Intent lifecycle, Order lifecycle, and related state machines are Core Runtime concepts.

This Stack captures and stores raw data. It does not interpret, validate, normalize, or promote that data. The boundary between raw recorded data and canonical validated data is enforced by the separation between the Data Recording Stack and the downstream Data Quality and Data Storage Stacks.

---

## Why the Stack Matters

The Data Recording Stack defines the empirical basis on which all downstream Research and Backtesting depend. If recorded data is incomplete, poorly timed, or missing provenance, no amount of downstream processing can recover what was not captured.

For microstructure-sensitive Research — where Strategy evaluation depends on order book dynamics, execution sequencing, and timing precision — the fidelity of recorded data is a binding constraint. This makes two design concerns particularly important:

**Environment parity.** The recording environment should approximate the conditions under which the Live Stack will later operate. This applies at two levels: external data and infrastructure parity (network path, co-location proximity, feed access) and internal pipeline and software parity (the same recording components handling data in the same way). Where residual differences between the recording environment and the Live environment exist, they should be identified and measured so that their impact on Research realism can be assessed. This is a strong design principle for the Data Recording Stack, particularly for microstructure-sensitive work, though the degree of parity achievable depends on operational and economic constraints.

**Timing fidelity.** Capture Time must be recorded accurately and consistently. Downstream processing — gap detection, latency analysis, time-alignment across feeds — depends on the timing metadata the Data Recording Stack preserves. Timing quality at the recording layer propagates directly into the realism of Backtesting inputs.
