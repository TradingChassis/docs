# Implementation Notes

This document captures implementation-facing rationale and design considerations for the Data Recording Stack: why specific recording design choices matter, what implementation patterns support data fidelity, and where realization decisions influence downstream Research quality.

---

## Why Recording Design Matters

The Data Recording Stack produces the raw empirical record from which all downstream datasets, Backtesting inputs, and Research results are ultimately derived. Implementation choices at the recording layer — where Capture Nodes are placed, how feeds are connected, how timing metadata is preserved, how raw data is persisted — directly constrain the realism and reliability of everything built on top of that record.

This is not a generic "data collection" concern. For microstructure-sensitive Research, the conditions under which data was recorded are part of the data's meaning. A trade tape recorded through a high-latency indirect feed path carries different timing characteristics than one recorded at a co-located Capture Node. Both are "correct" records of trades, but they represent different observation conditions, and those conditions affect the validity of Strategy evaluation that depends on timing precision.

Implementation decisions in the Data Recording Stack therefore carry Research consequences that extend well beyond the Stack's own operational boundary.

---

## Capture Time and Recording Context

**Capture Time** — the local time at which a Data Recording component observes or receives a raw Venue message — is implementation-relevant recording metadata. It records when the infrastructure first saw a message, under the network and infrastructure conditions that prevailed at the Capture Node.

Capture Time matters because it anchors recorded data to the observation conditions of the recording environment. Downstream processing — gap detection, latency analysis, time-alignment across feeds, Backtesting input preparation — depends on Capture Time to reconstruct the temporal structure of what the recording infrastructure actually observed. If Capture Time is inaccurate, inconsistent, or missing, the temporal fidelity of all downstream uses is degraded.

Capture Time is recording metadata, not a Core Runtime causal concept. It does not define Processing Order, Event Time, or any canonical runtime sequencing. Its significance is entirely within the recording and data-platform domain: it describes when and under what conditions raw data entered the infrastructure.

---

## External Environment Parity

The realism of recorded data depends in part on how closely the recording environment matches the environment in which the Live Stack will eventually operate.

This applies to external, infrastructure-level factors:

- **Feed path and locality.** A Capture Node co-located with a Venue receives data under different latency and ordering conditions than one connected through a remote network path. The feed path shapes message arrival timing, burst structure, and packet-loss characteristics — all of which become part of the recorded data's temporal profile.
- **Feed source and access method.** Different Venue feed tiers (direct market data vs. consolidated feeds, full-depth vs. top-of-book) produce different data. The feed source used during recording should reflect what the Live Stack will consume, or differences should be understood.
- **Network and transport conditions.** Batching behavior, TCP vs. UDP delivery, reconnect semantics, and message ordering under load are properties of the recording environment that influence what the raw record contains.

Raw data may be self-recorded by the infrastructure's own infrastructure or obtained from external providers. The architectural concern is not the source per se, but the degree to which the recording conditions — whether self-operated or externally provided — approximate the conditions the Live Stack will face. Self-recording from infrastructure that mirrors the Live environment provides strong parity by construction; externally sourced data may or may not carry equivalent observation conditions, depending on how and where it was recorded.

---

## Internal Pipeline Parity

External environment parity addresses how data reaches the recording infrastructure. Internal pipeline parity addresses how the recording infrastructure handles data after receipt.

If the Data Recording Stack uses different software, different buffering strategies, or different persistence paths than the Live Stack uses for its data intake, the recorded raw datasets may exhibit processing artifacts that differ from what Live operation would produce — even when the external feed source is identical. Examples include differences in message batching at the application layer, timestamp resolution, buffer flush cadence, and write ordering under load.

The design principle is that the internal recording pipeline should, where practical, align with the software and processing behavior that the Live Stack will use for the same data. This reduces the set of differences between recorded data and Live-observed data to those that are genuinely unavoidable (historical vs. real-time timing, simulated vs. real Venue execution).

---

## Residual-Difference Modeling

Full parity between the recording environment and the eventual Live environment is not always achievable. Infrastructure constraints, cost, data-provider limitations, and the inherent difference between historical recording and real-time operation mean that some residual differences will exist.

The design principle is not to eliminate all differences — that is impractical — but to **identify, model, and measure** the differences that remain:

- **Identify** which aspects of the recording environment differ from the intended Live environment (feed path, latency profile, capture-node placement, internal pipeline version, timestamp source).
- **Model** the expected impact of each difference on downstream data characteristics (e.g., a remote feed path adds N milliseconds of latency variance relative to co-located capture; an external data provider uses a different timestamp source than the infrastructure's own Capture Time).
- **Measure** actual residual differences where possible, so that their magnitude is known rather than assumed.

This is especially important for microstructure-sensitive Research, where Strategy evaluation may be sensitive to timing differences on the order of milliseconds or less. For less timing-sensitive work, residual differences may be operationally acceptable without detailed modeling — but they should still be acknowledged.

---

## Example Realization Patterns

The following are implementation patterns relevant to the Data Recording Stack. They illustrate how the design principles above translate into concrete realization decisions. They are not prescriptive deployment specifications.

**Capture Node placement.** Placing Capture Nodes in the same data center or co-location facility as the Venue reduces feed-path latency and improves external environment parity with a co-located Live Stack. Where co-location is not feasible, VPS instances in the same cloud region as the Venue data center provide a practical intermediate option. The choice depends on the required degree of timing fidelity and the operational constraints of the deployment.

**Raw storage layout.** Persistent Raw Storage can be organized to reflect the semantic recorded-data unit: partitioned by Venue, Feed, and Time Window. This layout makes individual datasets independently addressable and supports efficient downstream discovery via Dataset Manifests.

**Dataset Manifest publication.** The Dataset Manifest Writer creates manifest artifacts in a known, stable location after each raw recorded dataset is durably written. Downstream consumers (primarily the Data Quality Stack) poll or watch this location to discover new datasets. The manifest-based pattern keeps the downstream relationship asynchronous and decouples the recording pipeline from downstream processing cadence.

**Logical separation on shared physical storage.** Where Persistent Raw Storage and Canonical Storage share physical infrastructure (e.g., the same object store or filesystem), the logical separation must be maintained through distinct namespace, path, or bucket boundaries. Raw recorded datasets and canonical validated datasets must not be co-mingled in a way that blurs which datasets have been promoted and which have not. The Data Recording Stack writes only to the raw namespace; canonical promotion is a downstream responsibility.

---

## Implementation Boundaries

This document describes implementation rationale and design patterns for the Data Recording Stack. It does not define:

- **Core Runtime semantics.** Capture Time is recording metadata; it is not Processing Order, Event Time, or a canonical runtime concept.
- **Canonical Storage governance.** The Data Recording Stack writes to Persistent Raw Storage. Canonical promotion, validation, and dataset governance are downstream concerns.
- **Deployment procedures.** The realization patterns above are design considerations, not operational runbooks or step-by-step deployment instructions.
- **Monitoring specifications.** Observability of recording quality is important but is defined in operational and monitoring documentation, not here.
