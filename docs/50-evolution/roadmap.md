# Roadmap

This page outlines the current development direction.

It is not a fixed delivery plan.  
It describes the intended sequence of major work streams based on the current architecture, repository structure, and priorities.

The roadmap is expected to evolve as the Core and surrounding Runtimes become more mature.

---

## Planning assumptions

The main architecture documentation is now established and will continue to be refined over time.  
It is treated as an ongoing reference layer rather than a one-off milestone.

The current roadmap is therefore centered on hardening, Runtime separation, and Stack-by-Stack maturation.

Two assumptions shape the current plan:

- The **Core** must become stable before the broader platform can mature around it.
- **Backtesting and Research infrastructure** should be solid before expanding further into Live capabilities.

---

## Current focus

The immediate focus is to strengthen the **Core** and the **Runtime** around it.

This includes:

- hardening the Core as an independent unit
- separating the Core and Runtime more clearly into distinct repositories
- ensuring a Runtime can consume the Core cleanly in Backtesting and later Live contexts
- improving monitoring, persistence, and Runtime reliability
- introducing the remaining architectural changes required to complete the deterministic event-driven model, including explicit Control-Time Events

At this stage, the main priority is **not** broad feature expansion.  
The main priority is to make the Core and its Backtesting Runtime structurally solid.

---

## Status framing

This roadmap is directional. The sections below capture what is **completed**, what remains **transitional**, and what is explicitly **deferred**.

### Completed

- The Core canonical processing boundary is stable for the current transitional implementation milestone.
- Core Runtime canonical integration is validated for current canonical paths.
- Local runtime smoke is usable:
  - `python -m core_runtime.local.backtest --config core_runtime/local/local.json`
- Local Docker runtime image validation works and the runtime image is intentionally minimal (runtime-only).

### Transitional

- Post-submission lifecycle remains compatibility/snapshot-driven.
- Snapshot-derived fill progression remains compatibility-only and must not be described as canonical runtime `FillEvent` ingress.
- `ControlTimeEvent` exists and is integrated as a canonical runtime input **for realized scheduled deadlines** in the current transitional slice. This does not imply a finalized or complete execution-control scheduling implementation.
  - Implemented ordering semantics (Phase 16C): on realized deadline, Runtime processes canonical Control-Time Event before any deadline-caused queue pop; failure prevents pop/marker advancement; old deadlines do not repeatedly pop.
  - Implemented structured obligation boundary (Phase 16F): non-canonical structured Control Scheduling Obligations are exposed on the gate decision contract; `next_send_ts_ns_local` remains a compatibility mirror / scalar fallback.
  - Implemented single pending runtime semantics (Phase 16H): Runtime maintains one pending obligation (or `None`) and collapses multiple obligations deterministically; realization injects one Control-Time Event; success consumes pending before transitional pop; failure preserves pending and prevents pop.
  - Still deferred in this track: Core reducer remains minimal/no-op for Control-Time Events; queue pop remains Runtime-owned; strict clear-on-every-canonical-pass remains deferred when no gate decision is produced; full Core-owned Queue Processing remains deferred.

### Deferred explicit

- Runtime canonical `FillEvent` ingress
- `ExecutionFeedbackRecordSource`
- full canonical post-submission lifecycle
- `ProcessingContext`
- replay/storage / Event Stream persistence

---

## Near-term roadmap

### 1. Harden the Core

The first priority is to make the Core stable, explicit, and self-contained.

Planned direction:

- continue refactoring and renaming where necessary
- reduce ambiguity in internal structure and boundaries
- keep the semantic model stable while improving code organization
- introduce explicit **Control-Time Events** into the model
- ensure the Core stands cleanly on its own as a dedicated repository

Desired outcome:

- a more robust and clearly bounded Core implementation
- cleaner architecture/code alignment
- improved readiness for reuse across multiple Runtimes

---

### 2. Separate and strengthen the Runtime

Once the Core is more stable, the next priority is the Runtime layer that uses it.

Planned direction:

- separate Runtime concerns from Core concerns
- make the Runtime usable as a distinct repository that depends on the Core
- strengthen the Backtesting Runtime first
- improve persistent storage integration, monitoring, and Runtime observability

Desired outcome:

- a solid Backtesting Runtime built around the Core
- a clearer separation between reusable logic and environment-specific logic

Near-term next steps:

- Risk vs ExecutionControl boundary cleanup
- State vs snapshot compatibility split
- deferred execution feedback capabilities and runtime `FillEvent` ingress work

---

### 3. Stabilize Research-facing Stacks

After the Core and Runtime are stabilized, the focus moves to the parts required for systematic Research.

This primarily includes:

- Backtesting
- Monitoring / Observability
- Storage required to support repeatable experimentation

Desired outcome:

- a coherent Research infrastructure that supports experimentation, replay, monitoring, and persistent result handling

---

## Mid-term roadmap

### 4. Build out the internal data platform

Once the Research-facing infrastructure is solid, the next step is to extend the internal data platform.

Planned direction:

- develop the data recording layer
- strengthen data quality processes
- expand Canonical Storage capabilities
- make data capture and storage more usable for Research workflows

Relevant Stack direction:

- Data Recording
- Data Quality
- Data Storage

Desired outcome:

- a stronger internal platform for collecting, validating, and storing Research-relevant data

---

### 5. Develop the Analysis layer

After Research Runtime and data platform capabilities are in place, the next step is to improve data Analysis capabilities.

Planned direction:

- make it easier to inspect and evaluate collected data
- improve the analytical workflow around Research results and stored Runtime artifacts

Desired outcome:

- a more complete path from data collection to Research Analysis

---

## Longer-term roadmap

### 6. Complete the Live Stack

The longer-term goal is to mature the infrastructure toward Live environments.

Planned direction:

- extend the Runtime and surrounding infrastructure toward real Venue interaction
- build out the remaining Live-specific capabilities
- keep the same conceptual Runtime semantics that already govern Backtesting

Desired outcome:

- a Live-capable infrastructure built on the same deterministic event-driven Core model
- continuity between Backtesting semantics and real Venue execution

---

## Documentation track

Alongside implementation work, a second documentation track will continue in parallel in each repository individually.

The main documentation will remain the architectural and semantic reference layer, while codebase-facing documentation will be expanded using a different structure focused on fast understanding of implementation repositories.

The intended documentation order for those codebase-oriented docs is:

`Concept ➝ Flow ➝ Code ➝ API`

This track is meant to help readers understand concrete repositories and Runtime code more quickly, without turning the architecture documentation into implementation-level repository docs.

---

## Notes on change

This roadmap is directional rather than fixed.

The overall sequence is expected to remain stable:

1. strengthen the Core  
2. strengthen the Runtime  
3. stabilize Research-facing Stacks  
4. build out the data platform  
5. improve Analysis capabilities  
6. complete the Live Stack  

However, the exact shape of intermediate work may change as implementation and architectural refinement continue.
