# Interfaces

This document defines the inputs, outputs, and interface boundaries of the Backtesting Stack — what enters it, what leaves it, and how it relates to upstream and downstream Stacks.

---

## Inputs

The Backtesting Stack consumes two categories of input: historical data and execution configuration.

### Canonical datasets

The primary historical input basis. Canonical datasets are validated, normalized datasets read from **Canonical Storage** — the authoritative dataset layer maintained by the Data Platform. These datasets provide the historical market data (order book updates, trades, associated metadata) against which Strategies are evaluated.

The Backtesting Stack does **not** primarily consume raw recorded datasets. Raw data resides in Persistent Raw Storage and must pass through the Data Quality Stack's validation and promotion process before it enters Canonical Storage. The Backtesting Stack operates on the canonical output of that process.

### Experiment and run configuration

Definitions that specify what a Backtesting run should execute:

- **Strategy definitions / Strategy code** — the Strategy logic to be evaluated.
- **Experiment configuration** — which canonical datasets to use, which time periods to cover, which instruments to include.
- **Run configuration** — runtime parameters, Configuration settings, and execution-control rules applied to the Core Runtime for this run.
- **Parameter sets** — parameter variations for batch runs and parameter sweeps.
- **Scenario definitions** — where applicable, structured scenario specifications that define particular experimental conditions.

---

## Outputs

The Backtesting Stack produces experiment-facing and analysis-facing artifacts:

### Experiment results

The primary output: the quantitative outcomes of each Backtesting run — Strategy performance metrics, execution statistics, and per-run result summaries.

### Run artifacts

Detailed artifacts from individual runs — execution traces, Order lifecycle records, dispatch histories, and other per-run data that supports detailed post-hoc analysis.

### Execution records

Persisted records of execution activity produced during the Backtesting run — dispatch decisions, Venue Adapter interactions with the simulated Venue, and Execution Events generated during processing. These are written to **Execution Record Storage** for durable retention.

### Metrics and evaluation outputs

Aggregated or derived metrics suitable for comparison across runs, parameter sweeps, or experimental conditions. These may include performance summaries, risk metrics, and execution-quality indicators.

All outputs are persisted through the Data Storage Stack's storage surfaces — primarily **Experiment / Artifact Storage** and **Execution Record Storage** — so that they remain durably available for downstream analysis.

---

## Relationship to the Core Runtime

The Backtesting Stack **uses** the Core Runtime as its execution kernel. The Core Runtime provides the deterministic, event-driven processing model — Event intake, State derivation, Strategy evaluation, Risk, Execution Control, Venue Adapter, and the feedback loop through Execution Events — that the Backtesting Stack executes against historical data.

The interface between the Backtesting Stack and the Core Runtime is:

- The Backtesting Stack supplies **historical Events** (derived from canonical datasets) and **Configuration** to the Core Runtime.
- The Core Runtime processes those Events deterministically and produces State, dispatch decisions, and execution-control outcomes.
- The simulated Venue (behind the Venue Adapter boundary) generates execution feedback that re-enters the Core Runtime as Execution Events, completing the processing loop.
- The Backtesting Stack captures the Core Runtime's outputs — derived State, Order lifecycle progression, dispatch history, and all processing outcomes — as the basis for experiment results and artifacts.

The Backtesting Stack does not redefine Event, State, Intent, Order, Queue, Risk, or Determinism semantics. Those are established in architecture and concept documents and apply identically in Backtesting and Live.

---

## Relationship to the Data Storage Stack

The Data Storage Stack mediates both the durable inputs and the durable outputs of the Backtesting Stack:

**Inputs.** The Backtesting Stack reads canonical datasets from **Canonical Storage**. This is the primary data interface — the historical basis for all Backtesting runs.

**Outputs.** The Backtesting Stack writes experiment results and run artifacts to **Experiment / Artifact Storage**, and execution records to **Execution Record Storage**. These storage surfaces are provided by the Data Storage Stack and ensure that Backtesting outputs survive the run and remain available for downstream analysis.

The Backtesting Stack does not manage these storage surfaces. It writes to and reads from them; the Data Storage Stack provides and governs them.

---

## Relationship to Monitoring and Analysis

### Monitoring Stack

The Monitoring Stack provides **observability capabilities** that the Backtesting Stack may use during execution — run status, processing throughput, error rates, and runtime health indicators. Monitoring is a supporting interface: the Backtesting Stack emits telemetry and metrics that the Monitoring Stack consumes, but monitoring is not the Backtesting Stack's primary output or owned concern.

### Analysis Stack

The Analysis Stack is the primary downstream consumer of the Backtesting Stack's outputs. It reads experiment results, run artifacts, and execution records from the Data Storage Stack's persistent surfaces to perform Strategy evaluation, performance analysis, and cross-experiment comparison.

The interface is mediated by storage: the Backtesting Stack writes outputs to Experiment / Artifact Storage and Execution Record Storage; the Analysis Stack reads from those surfaces on its own schedule. There is no synchronous handoff or runtime coupling between the two Stacks.

---

## Interface Boundaries

**Canonical datasets are the historical input basis.** The Backtesting Stack consumes validated, promoted datasets from Canonical Storage. It does not consume raw recorded datasets directly and does not participate in validation, normalization, or promotion.

**Outputs are experiment-oriented artifacts.** The Backtesting Stack produces experiment results, run artifacts, execution records, and metrics. It does not produce canonical datasets, validated data, or Data Platform outputs.

**The Core Runtime is used, not defined.** The Backtesting Stack supplies inputs to and captures outputs from the Core Runtime. It does not modify the Core Runtime's processing semantics. The same Core Runtime model applies in Backtesting and Live.

**Persistence is mediated by the Data Storage Stack.** Both inputs (canonical datasets) and outputs (experiment results, execution records) are accessed through the Data Storage Stack's persistent surfaces. The Backtesting Stack does not manage storage infrastructure.

**Monitoring is a supporting capability.** The Backtesting Stack emits telemetry for observability, but monitoring is not its primary interface or output. The Monitoring Stack consumes what the Backtesting Stack emits; the Backtesting Stack does not own the monitoring layer.
