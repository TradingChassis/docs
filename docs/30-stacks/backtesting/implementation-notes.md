# Implementation Notes

This document captures implementation-facing rationale and design considerations for the Backtesting Stack: how deterministic Research execution is realized in practice, how experiments and sweeps are orchestrated, and what patterns support reproducibility and artifact persistence.

---

## Why Backtesting Realization Matters

The Backtesting Stack operationalizes the Core Runtime for Research. The Core Runtime defines the processing semantics — Events, State derivation, Strategy evaluation, Risk, Execution Control, Order lifecycle — but those semantics are usable for Research only if a practical execution environment exists around them.

How that environment is realized determines whether the stack delivers on its central requirements: deterministic execution, reproducible experiments, efficient sweep exploration, and durable result persistence. A weak realization — non-reproducible run definitions, lost artifacts, non-deterministic infrastructure effects — undermines the value of the Core Runtime's determinism guarantee, because the guarantee holds at the semantic level but is not preserved through to Research output.

---

## Realizing Historical Runtime Execution

The Core Runtime processes Events deterministically in Processing Order. In Backtesting, the Event stream is constructed from canonical datasets rather than arriving from live Venue feeds. The implementation must bridge canonical persistent data and the Core Runtime's Event-processing model without introducing non-determinism.

**Canonical dataset materialization.** Canonical datasets are stored in Canonical Storage as validated, normalized dataset files partitioned by Venue, Feed, and Time Window. The Historical Input Feeder reads these files and produces the historical Event stream in the Processing Order the Core Runtime expects. This materialization step must be deterministic: the same canonical dataset must always produce the same Event stream in the same order.

**Simulated Venue realization.** The Simulated Venue sits behind the Venue Adapter boundary and generates execution feedback from historical data. Its behavior must also be deterministic: given the same historical input data and the same sequence of outbound requests, it must produce identical execution feedback. The current realization uses hftbacktest for this purpose, as documented in ADR-008. The surrounding infrastructure — Event processing, State derivation, Strategy evaluation, Risk, Execution Control, the Venue Adapter itself — remains implemented within the Infrastructure.

**Determinism preservation through infrastructure.** The Core Runtime is deterministic by design, but the execution environment can introduce non-determinism if not carefully controlled. Implementation must ensure that:

- Thread scheduling, process ordering, and resource contention do not affect Event processing order.
- Timestamp sources used within the processing loop are derived from the Event stream (Processing Order), not from wall-clock time.
- No hidden mutable state outside the Event processing path influences run outcomes.

These are implementation-level constraints that preserve the Core Runtime's determinism guarantee through to Research output.

---

## Experiment and Sweep Orchestration Patterns

The Backtesting Stack must support structured experiment execution beyond single ad hoc runs. In practice, this means orchestrating:

- **Single runs** — one Strategy, one canonical dataset, one Configuration, one set of parameters.
- **Batch runs** — a set of independent single runs grouped for coordinated execution (e.g., evaluating the same Strategy across multiple time periods or instruments).
- **Parameter sweeps** — systematic variation of one or more parameters across a run set, producing structured result grids for cross-parameter comparison.

Orchestration can be realized through several patterns:

- **Workflow engines** (e.g., Argo Workflows, Prefect, or similar) that express experiment plans as directed acyclic graphs of run tasks — each task being a self-contained single run with defined inputs and outputs.
- **Batch schedulers** that manage a queue of run definitions and dispatch them to available compute resources.
- **Sweep controllers** that generate the Cartesian product of parameter variations, submit the resulting runs, and track completion.

No specific orchestration tool is architecturally required. The key implementation properties are:

- Each run is **self-contained** — it carries its full run definition (dataset reference, Strategy, Configuration, parameters) and does not depend on the runtime state of other runs.
- Orchestration coordinates execution **around** runs but does not interfere **within** them — the Core Runtime's processing within a run is not influenced by the orchestration layer.
- Sweep and batch completion is tracked so that downstream consumers (the Analysis Stack, the researcher) can determine when a full experiment is ready for analysis.

---

## Reproducible Run Definitions

Reproducibility requires that a run can be re-executed at a later time and produce identical results. This is achievable only if the run definition fully specifies all inputs that influence the outcome.

A reproducible run definition must capture:

- **Canonical dataset reference** — a stable identifier for the exact dataset version used (Venue, Feed, Time Window, dataset version or promotion identifier).
- **Strategy definition** — the Strategy code or a versioned reference to it.
- **Configuration** — all Configuration parameters that the Core Runtime uses during processing, including execution-control rules, Risk policy parameters, and any other settings.
- **Run parameters** — any additional parameters that vary across sweep dimensions.

The run definition should be **serializable and storable** — it can be persisted alongside the run's results so that the experiment can be audited, reproduced, or extended later. A stored run definition and the corresponding canonical dataset should be sufficient to reproduce the run's outputs exactly.

In practice, this means:

- Strategy code must be versioned or snapshotted at run time.
- Configuration must be explicit and complete — no implicit defaults that are not captured in the run definition.
- Canonical dataset references must be stable — a dataset identifier must always resolve to the same data (guaranteed by Canonical Storage's immutability-after-promotion property).

---

## Result, Artifact, and Evaluation Persistence

Every completed run produces outputs that must be durably persisted for downstream use. The implementation must ensure that result persistence is reliable, structured, and traceable.

**Result structure.** Experiment results should carry the run definition that produced them, so that any result can be traced back to its exact inputs. This linkage is what makes cross-run comparison and parameter-sweep analysis possible — the Analysis Stack can determine which parameter variation or dataset produced each result.

**Storage surfaces.** Results and run artifacts are written to the Data Storage Stack's persistent surfaces:

- **Experiment / Artifact Storage** for experiment results, performance summaries, and evaluation outputs.
- **Execution Record Storage** for detailed execution traces, Order lifecycle records, and dispatch histories.

The Backtesting Stack writes to these surfaces but does not manage them. Storage organization, retention, and access governance are Data Storage Stack responsibilities.

**Sweep-level structure.** For parameter sweeps and batch experiments, result persistence should preserve the structured relationship between runs — which runs belong to which experiment, which parameter values were used, and how runs relate to each other in the sweep grid. This structure is what enables the Analysis Stack to perform meaningful cross-run comparison rather than treating each run as an isolated result.

---

## Monitoring and Observability Integration

The Backtesting Stack may integrate with the Monitoring Stack for operational observability during execution. Typical concerns include:

- **Run status and progress** — which runs are pending, executing, completed, or failed within a batch or sweep.
- **Execution throughput** — how fast the Core Runtime is processing the historical Event stream (events per second, time-to-completion estimates).
- **Resource utilization** — compute, memory, and I/O consumption during execution.
- **Error and failure visibility** — runs that fail or produce unexpected outcomes should be visible promptly.

Monitoring is a supporting capability, not a primary output of the Backtesting Stack. The Backtesting Stack emits telemetry; the Monitoring Stack consumes and presents it. The specifics of monitoring instrumentation, dashboards, and alerting thresholds are defined in monitoring documentation, not here.

---

## Implementation Boundaries

**Implementation choices, not canonical semantics.** The orchestration patterns, run-definition formats, persistence structures, and monitoring integrations described here are implementation-level concerns. They do not define or modify the Core Runtime's Event, State, or lifecycle semantics.

**The Core Runtime is executed, not redefined.** The Backtesting Stack's implementation wraps the Core Runtime in an experiment execution environment. It does not alter the Core Runtime's processing model. Determinism, Event processing, State derivation, and all canonical processing rules are as defined in architecture and concept documents.

**Canonical datasets are the historical basis.** The implementation reads from Canonical Storage. It does not include raw data loading, validation, normalization, or promotion. Those are Data Platform responsibilities that have completed before the Backtesting Stack encounters the data.

**No deployment specification.** The patterns above describe design considerations and realization approaches. They do not prescribe specific compute platforms, container runtimes, cluster configurations, or infrastructure-as-code templates. Those belong in deployment documentation.
