# Milestones

This page summarizes the major project milestones.

Dates are approximate and intended to show development phases rather than exact delivery timestamps.

---

## August 2025 – November 2025

### Infrastructure Foundation

Initial work focused on building the project’s infrastructure base and deployment model.

**Main outcomes**

- Established cloud and server-side infrastructure foundation
- Built the infrastructure repository and supporting secrets/configuration setup
- Evaluated deployment and operations requirements around Kubernetes and Infrastructure as Code
- Explored exchange and market-data integration patterns, including API and transport options such as WebSocket, UDP, multicast, FIX, and lower-latency binary protocols
- Investigated provider and colocation constraints relevant to the target runtime environment

**Why this milestone mattered**

- Defined the physical and operational foundation required to run the infrastructure consistently
- Clarified which infrastructure capabilities were actually needed for the project

---

## November 2025 – January 2026

### Core, Core Runtime and Backtesting Foundation

Work shifted from cloud infrastructure into the Core model and the first usable Research and Core Runtime tooling.

**Main outcomes**

- Implemented the Core Python library and its Core Runtime
- Built the main Runtime concepts around:
  - Event Stream
  - State derivation
  - Strategy evaluation
  - Risk Engine
  - Queue / Execution Control
  - Configuration
  - Processing Order
  - Venue Adapter / Venue integration boundaries
- Built the Runtime path for Backtesting
- Added experiment and parameter-sweep support
- Added supporting runtime tooling such as scratch volumes, preload/sweep infrastructure, manifest loading, and experiment orchestration
- Integrated supporting observability and workflow tooling including MLflow, Grafana, Prometheus, Argo Workflows, and S3-based storage usage

**Why this milestone mattered**

- Produced the first coherent version of the Core
- Made the runtime usable for Backtesting and systematic experimentation/Research

---

## February 2026 – March 2026

### Formal Documentation and Project Structuring

Once the Core Runtime and supporting tooling existed, the focus moved to formalizing the architecture and making the project legible.

**Main outcomes**

- Created this MkDocs-based architecture documentation
- Documented the semantic and conceptual model of the infrastructure rather than repository-level implementation details
- Structured documentation around architecture, concepts, Stacks, operations, and evolution
- Cleaned up project structure at the GitHub organization and repository level
- Standardized repository presentation and publication setup
- Published the codebase and documentation in a more coherent public form

**Why this milestone mattered**

- Turned the project from an internal codebase into a documented infrastructure
- Established a formal architectural vocabulary and documentation surface

---

## March 2026 – April 2026

### Canonical Model Consolidation

Work focused on tightening the documentation and aligning it around a single explicit architecture model.

**Main outcomes**

- Consolidated the deterministic event-driven architecture model
- Clarified the boundaries between:
  - Event and Intent
  - Intent lifecycle and Order lifecycle
  - Risk and Execution Control
  - Queue State and canonical truth
- Strengthened the formal treatment of:
  - Processing Order
  - replayability
  - deterministic State derivation
  - Execution Control semantics
- Aligned architecture and concept documents to a single frozen canonical model
- Finalized the public project naming and documentation presentation

**Why this milestone mattered**

- Removed major semantic ambiguity from the documentation
- Established a coherent architecture model that can now guide further implementation and documentation work

---

## April 2026 – Present

This transitional implementation milestone closes Core and Core Runtime alignment:

### Core semantic milestone

- Core import root is `tradingchassis_core` and the distribution name is `tradingchassis-core`.
- The Core public canonical processing API is usable and stable for the current slice:
  - `CoreConfiguration`
  - `ProcessingPosition`
  - `EventStreamEntry`
  - `process_event_entry`
  - `fold_event_stream_entries`
- `EventStreamEntry` is a minimal envelope containing only:
  - `position`
  - `event`
- `CoreConfiguration` is explicit and versioned, with deterministic identity (fingerprinted), and is passed call-level to processing (it is not embedded in `EventStreamEntry`).
- `ProcessingPosition` defines **Processing Order**. Event Time is carried by Events as external metadata and does **not** define Processing Order.

### Core Runtime alignment

- Core Runtime import root is `core_runtime` and the distribution name is `tradingchassis-core-runtime`.
- Core Runtime consumes Core (`tradingchassis_core`) and owns runtime orchestration responsibilities around canonical processing:
  - owns a runtime `EventStreamCursor`
  - allocates `ProcessingPosition`
  - constructs `EventStreamEntry`
  - calls `process_event_entry(..., configuration=CoreConfiguration)`
- Local hftbacktest-backed runtime smoke is usable:

```
python -m core_runtime.local.backtest --config core_runtime/local/local.json
```

- Default local outputs are generated under:
  - `.runtime/local/results/`

### Packaging / deployment validation

- The Core Runtime Docker image has been validated locally and is intentionally minimal (runtime-only: no tests, no repo checkout).
- Kubernetes / Argo / GHCR execution is a **deployment validation track**, not a claim of full production readiness.

### Control-Time obligation semantics (transitional implementation track)

- Phase 16C: Runtime ordering for realized deadlines is explicit: inject/process canonical Control-Time Event before deadline-caused queue pop; failure prevents pop/marker advancement; old deadlines do not repeatedly pop.
- Phase 16F: Structured non-canonical Control Scheduling Obligations are exposed on the gate decision contract; `next_send_ts_ns_local` remains a compatibility mirror / scalar fallback.
- Phase 16H: Runtime maintains a single pending Control Scheduling Obligation (or `None`) and collapses multiple obligations deterministically; realization injects one canonical Control-Time Event; success consumes pending before transitional pop; failure preserves pending and prevents pop.
- Still deferred in this track: Core reducer remains minimal/no-op for Control-Time Events; queue pop remains Runtime-owned; strict clear-on-every-canonical-pass remains deferred when no gate decision is produced; full Core-owned Queue Processing remains deferred.

### Explicit deferred capabilities

- Runtime canonical `FillEvent` ingress
- `ExecutionFeedbackRecordSource`
- full canonical post-submission lifecycle
- `ProcessingContext`
- replay/storage / Event Stream persistence
