# Operational Behavior

This document describes how the Backtesting Stack behaves during normal operation: how runs are initialized and executed, how simulated execution conditions function at runtime, how experiments and sweeps are coordinated, and how results and artifacts emerge from execution.

---

## Run Initialization and Historical Input Use

A Backtesting run begins with a **run definition**: a specific combination of canonical dataset, Strategy, Configuration, and parameters that fully specifies what the Core Runtime should execute and under what conditions.

During initialization:

1. The run definition is resolved — the canonical dataset is identified in Canonical Storage, the Strategy code is loaded, and Configuration and parameters are prepared.
2. The canonical dataset is read from Canonical Storage and transformed into the historical Event stream that the Core Runtime will process. This stream supplies the historical market data (order book updates, trades, associated metadata) in the Processing Order required for deterministic replay.
3. The Core Runtime is instantiated with the run's Strategy and Configuration, and the Simulated Venue is prepared behind the Venue Adapter boundary.

After initialization, the run is ready for execution. The canonical dataset is the **sole historical input basis** — the Backtesting Stack does not consume raw recorded datasets during operation. All historical data has passed through the Data Quality Stack's validation and promotion process before the Backtesting Stack encounters it.

---

## Simulated Execution During Runs

Once a run is initialized, the Core Runtime processes the historical Event stream deterministically. The processing chain operates in full:

- **Event intake and State derivation** — each historical Event is applied in Processing Order, deriving Market State and Execution State.
- **Strategy evaluation** — the Strategy reads derived State projections and emits Intents.
- **Risk and Execution Control** — Intents pass through Risk (policy admissibility) and Execution Control (Queue Processing, dominance, inflight gating, rate-limited dispatch).
- **Simulated Venue interaction** — outbound work selected for dispatch is transmitted through the Venue Adapter to the Simulated Venue, which generates realistic execution feedback (fills, acknowledgements, rejections) based on historical order book and trade data.
- **Feedback re-entry** — Execution Events from the Simulated Venue re-enter the Event stream, driving Order lifecycle transitions and further State derivation.

The Simulated Venue is integral to operational behavior. Without it, the processing loop has no execution feedback, and the run cannot produce Order lifecycle outcomes, fill statistics, or execution-quality metrics. The Simulated Venue operates behind the same Venue Adapter boundary used in Live — the Core Runtime does not contain Backtesting-specific processing paths.

The entire processing loop runs within deterministic Event processing. There is no separate runtime tick, no background scheduler, and no non-deterministic state mutation. Given the same canonical dataset, the same Strategy, and the same Configuration, the run produces identical State, dispatch decisions, and Order lifecycle outcomes at every Processing Order position.

---

## Experiment Execution and Sweep Behavior

The Backtesting Stack supports execution at three levels of granularity:

### Single runs

A single run executes one complete Backtesting pass with one run definition. This is the atomic unit of execution — every experiment, batch, or sweep ultimately decomposes into single runs.

### Batch runs

A batch groups multiple independent single runs for coordinated execution. Batch runs share an experiment context but may use different canonical datasets, time periods, or Strategies. The Backtesting Stack coordinates their execution (sequentially or concurrently, depending on resource availability) and tracks completion across the group.

### Parameter sweeps

A parameter sweep systematically varies one or more parameters across a set of runs while holding other dimensions constant. The sweep produces a structured set of results that can be compared along the varied dimensions. Parameter sweeps are a primary mechanism for Strategy evaluation — they allow Research to assess how Strategy behavior changes across parameter ranges.

In all cases, each individual run is self-contained and deterministic. The Backtesting Stack's orchestration layer coordinates execution order, resource allocation, and completion tracking across runs, but does not influence the Core Runtime's processing within any individual run.

---

## Result and Artifact Production

Each completed run produces structured outputs:

- **Experiment results** — Strategy performance metrics, execution statistics, and per-run summaries that capture the quantitative outcome of the run.
- **Execution records** — detailed traces of dispatch decisions, Order lifecycle progression, and Execution Events generated during processing.
- **Metrics and evaluation outputs** — aggregated or derived indicators suitable for cross-run comparison, parameter-sweep analysis, and performance evaluation.

These outputs are persisted to the Data Storage Stack's persistent surfaces — Experiment / Artifact Storage for experiment results and run artifacts, Execution Record Storage for execution traces. Persistence occurs after run completion; outputs are durably stored and available for later retrieval by the Analysis Stack.

For parameter sweeps and batch runs, result production follows the same pattern per run. The Backtesting Stack additionally tracks which results belong to which experiment and sweep, preserving the structured relationship between runs so that the Analysis Stack can perform cross-run comparison.

---

## Operational Boundaries

**The Core Runtime is executed, not modified.** The Backtesting Stack's operational behavior consists of preparing, executing, and capturing the outputs of Core Runtime runs. It does not alter the Core Runtime's processing semantics during execution. Event processing, State derivation, Risk, Execution Control, and Order lifecycle behavior operate identically to their canonical definitions.

**Canonical datasets are the historical basis.** The Backtesting Stack reads from Canonical Storage during operation. It does not perform validation, normalization, or promotion. Those have already occurred before the Backtesting Stack encounters the data.

**Determinism is an operational invariant.** Every run must produce identical results given identical inputs and Configuration. The Backtesting Stack's operational behavior must not introduce non-determinism through infrastructure variation, scheduling order, or resource contention. Determinism is not aspirational — it is a requirement that the operational behavior of the stack must preserve.

**Persistence is mediated by the Data Storage Stack.** The Backtesting Stack writes outputs to the Data Storage Stack's persistent surfaces. It does not manage those surfaces or govern their retention, organization, or access policies.

**Observability.** Run status, execution throughput, completion rates, and error conditions should remain observable at a high level during operation. The Monitoring Stack provides these capabilities; the Backtesting Stack emits the telemetry that supports them.

---

## Why This Behavior Matters

The Backtesting Stack's operational behavior determines whether the System's deterministic Core Runtime model is faithfully applied to historical data for Research. If runs are not deterministic, results are not reproducible and Strategy evaluation loses credibility. If simulated execution is absent or unrealistic, the processing loop is incomplete and execution-quality metrics are meaningless. If results are not durably persisted, experiments cannot be compared or revisited.

The operational behavior described here — deterministic execution of the full Core Runtime processing chain against canonical historical data, with realistic simulated execution, structured result production, and durable persistence — is what makes the Backtesting Stack a reliable basis for Research.
