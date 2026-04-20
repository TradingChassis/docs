# Interfaces

This document defines the inputs, outputs, and interface boundaries of the Analysis Stack — what it consumes, what it produces, and how it relates to the persistent storage surfaces and persisted infrastructure outputs it depends on.

---

## Inputs

The Analysis Stack consumes persisted outputs and artifacts that have been durably stored by other Stacks. All inputs are read from persistent storage surfaces — the Analysis Stack does not receive data through runtime coupling, synchronous handoffs, or transient message passing.

### Experiment results and run artifacts

The primary analytical input. Experiment results (Strategy performance metrics, execution statistics, per-run summaries) and run artifacts (execution traces, Order lifecycle records, dispatch histories) produced by the Backtesting Stack and persisted in **Experiment / Artifact Storage**.

### Execution records from Live

Persisted records of Live execution activity — order history, fill records, position data, execution metadata — stored in **Execution Record Storage**. The Analysis Stack consumes these retrospectively for post-hoc evaluation, execution-quality assessment, and performance review. It does not observe Live execution in real time.

### Canonical datasets

Validated, normalized datasets from **Canonical Storage**, consumed where direct analysis of the underlying market data is required — for example, when evaluating Strategy behavior against the specific market conditions present in the canonical input.

### Derived datasets

Previously produced analytical or computed artifacts from **Derived Storage**, consumed when analysis builds on earlier analytical work or when cross-analysis comparison requires access to previously derived outputs.

### Analysis configurations

Definitions that specify what an analysis should evaluate — comparison parameters, evaluation criteria, dataset selections, and other structured inputs that configure a particular analysis run. These may be provided as files, configuration objects, or structured definitions.

---

## Outputs

The Analysis Stack produces derived analytical artifacts that are the result of evaluating, comparing, and interpreting persisted inputs.

### Derived analytical artifacts

Comparative evaluations, summary statistics, performance assessments, and structured analytical outputs that distill persisted inputs into evaluative results.

### Comparison artifacts

Cross-experiment comparisons, parameter-sweep analyses, Strategy rankings, and other outputs that relate multiple runs, configurations, or time periods to each other.

### Analysis datasets

Computed datasets derived from persisted inputs — aggregations, transformations, or projections that serve as inputs for further analysis or as standalone analytical products.

### Versioned analytical results

Analysis outputs that are traceable to their inputs and analysis definitions, and that can be reproduced by re-executing the same analysis against the same persisted inputs.

### Retrospective evaluation outputs

Outputs from Research–Live discrepancy analysis — comparisons between Research-oriented expectations and persisted Live outcomes that make discrepancies measurable and modelable.

All outputs may be persisted to the Data Storage Stack's persistent surfaces — primarily **Derived Storage** and **Experiment / Artifact Storage** — so that they remain durably available for future reference, further analysis, or downstream consumption.

---

## Relationship to Persistent Storage Surfaces

The Analysis Stack depends on the Data Storage Stack's persistent surfaces for both its inputs and its outputs. It does not manage these surfaces — it reads from and writes to them.

| Storage surface | Relationship |
| --------------- | ------------ |
| **Experiment / Artifact Storage** | Read: experiment results, run artifacts. Write: derived analytical artifacts. |
| **Execution Record Storage** | Read: persisted Live execution records for retrospective analysis. |
| **Canonical Storage** | Read: canonical datasets where direct analysis requires them. |
| **Derived Storage** | Read: previously produced derived datasets. Write: new derived analytical outputs. |

The Analysis Stack does not define storage classes, manage storage organization, or govern retention. Those are Data Storage Stack responsibilities. The Analysis Stack uses the storage surfaces as they are provided.

---

## Relationship to Persisted Infrastructure Outputs

The Analysis Stack operates **downstream of persisted outputs** produced by other Stacks. It does not interact with those Stacks during their execution — it consumes their stored results after the fact.

**Backtesting outputs.** The Backtesting Stack produces experiment results, run artifacts, and execution records and persists them to Experiment / Artifact Storage and Execution Record Storage. The Analysis Stack reads these outputs for Strategy evaluation, cross-experiment comparison, and parameter-sweep analysis. The two Stacks do not interact during Backtesting execution.

**Live outputs.** The Live Stack produces execution records, order history, fill data, and position records and persists them to Execution Record Storage. The Analysis Stack reads these outputs for retrospective execution-quality review, performance assessment, and Research–Live discrepancy analysis. The Analysis Stack does not observe or interact with Live execution in real time — it works on persisted records after execution has occurred.

**Canonical and derived data.** The Data Platform produces canonical datasets (through the Data Quality Stack) and may hold derived datasets from prior analytical work. The Analysis Stack reads from these surfaces where its analytical scope requires access to the underlying data.

---

## Interface Boundaries

**Persisted artifacts are the input basis.** The Analysis Stack consumes data that is already durably stored in the Data Storage Stack's persistent surfaces. It does not consume transient runtime state, live event streams, or real-time operational signals.

**Analysis is asynchronous and retrospective.** The Analysis Stack operates on its own schedule, independently of the Stacks that produced its inputs. There is no synchronous coupling between the Analysis Stack and the Backtesting or Live Stacks.

**Outputs are derived, comparative, and persistable.** The Analysis Stack produces new analytical artifacts from persisted inputs. These outputs are themselves persistable and traceable to their inputs and analysis definitions.

**The Analysis Stack does not participate in runtime execution.** It does not run Strategies, process Events, evaluate Risk, manage Execution Control, or interact with Venues. It operates on the products of runtime execution, not within it.

**The Analysis Stack is not operational monitoring.** It does not track runtime health, emit alerts, or provide real-time operational visibility. Its scope is retrospective analysis of persisted outputs, not ongoing observation of running infrastructures.

**Storage surfaces are used, not governed.** The Analysis Stack reads from and writes to the Data Storage Stack's persistent surfaces but does not manage their organization, retention, or access policies.
