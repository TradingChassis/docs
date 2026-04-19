# Operational Behavior

This document describes how the Analysis Stack behaves during normal operation: how persisted inputs are consumed, how asynchronous analysis is executed, how derived outputs are produced, and how reproducibility and versioning function as operational concerns.

---

## Persisted Input Consumption

The Analysis Stack operates on persisted artifacts — experiment results, execution records, canonical datasets, and derived datasets that are already durably stored in the Data Storage Stack's persistent surfaces.

During normal operation, the Analysis Stack reads inputs from the relevant storage surfaces as needed for the current analytical task:

- **Experiment results and run artifacts** from Experiment / Artifact Storage — the primary input for evaluating Backtesting outcomes.
- **Execution records** from Execution Record Storage — consumed retrospectively for post-hoc analysis of Live execution outcomes.
- **Canonical datasets** from Canonical Storage — consumed where the analysis requires direct access to the underlying market data.
- **Derived datasets** from Derived Storage — consumed where the analysis builds on earlier analytical work.

Input consumption is **on-demand and asynchronous**. The Analysis Stack reads from storage surfaces on its own schedule, independently of the Stacks that produced the stored artifacts. There is no synchronous coupling — the Analysis Stack does not wait for other Stacks to complete, nor do other Stacks wait for the Analysis Stack to consume their outputs.

---

## Asynchronous Evaluation and Comparison

Once inputs are loaded, the Analysis Stack applies analytical logic — evaluation functions, statistical assessments, metrics computations, comparative analyses — to the persisted artifacts.

### Single-result evaluation

A single set of experiment results or execution records is evaluated against defined criteria: performance metrics, risk measures, execution-quality indicators, or other evaluative measures. The analysis produces structured evaluative outputs that characterize the quality or significance of the results.

### Cross-result comparison

Multiple result sets are compared along defined dimensions: parameter-sweep analysis across Backtesting runs, cross-experiment comparison of different Strategies or configurations, or time-period comparison of execution outcomes. The analysis produces comparative outputs that make relationships between result sets explicit and measurable.

### Research–Live retrospective analysis

Persisted Backtesting outcomes are compared with persisted Live execution outcomes to identify, measure, and model discrepancies between Research-oriented expectations and actual production behavior. This analysis is retrospective — it operates on persisted outputs from both contexts after the fact, not on running systems. Its purpose is to make the gap between Research and Live outcomes understandable and quantifiable.

All analytical work is asynchronous. The Analysis Stack does not observe running systems in real time. It analyzes what has already been persisted — the results of experiments that have completed, the records of Live execution that has already occurred.

---

## Derived Output and Result Production

The Analysis Stack produces derived analytical artifacts as the outcomes of its evaluation and comparison work:

- **Derived analytical datasets** — aggregations, transformations, or projections computed from persisted inputs.
- **Comparative evaluation outputs** — cross-experiment rankings, parameter-sensitivity analyses, Strategy comparisons.
- **Evaluation results** — structured assessments of Strategy performance, execution quality, or discrepancy magnitude.
- **Reports and summaries** — structured outputs that distill analytical findings into communicable form.

These outputs are written to the Data Storage Stack's persistent surfaces — primarily Derived Storage and Experiment / Artifact Storage — where they become part of the Infrastructure's persistent record. Persisted analytical outputs are available for future reference, further analysis, or consumption by other analytical work.

---

## Reproducibility and Versioned Analysis Behavior

Reproducibility and versioning are not aspirational properties of the Analysis Stack — they are operational requirements that influence how analysis behaves in practice.

**Reproducible execution.** The same analysis definition applied to the same persisted inputs must produce the same outputs. This requires that analysis logic is deterministic with respect to its inputs — it does not depend on wall-clock time, execution order, or transient system state.

**Versioned analysis definitions.** Changes to analysis logic produce new versions. An analytical output is associated with the specific version of the analysis definition that produced it. This makes it possible to determine whether two outputs were produced by the same or different analytical logic.

**Input traceability.** Every analytical output records which persisted inputs it consumed — which experiment results, which execution records, which canonical datasets, which analysis configuration. This linkage is what makes outputs meaningful: an evaluation result is interpretable only if its inputs are known.

**Output traceability.** Analytical outputs are themselves identifiable and versionable. A derived dataset or evaluation result can be referenced by subsequent analysis, and that reference is stable.

In practice, this means that the Analysis Stack's operational behavior includes provenance tracking as an integral part of analysis execution, not as an optional post-processing step.

---

## Operational Boundaries

**Asynchronous and retrospective.** The Analysis Stack operates on persisted artifacts, on its own schedule, independently of the Stacks that produced them. It does not observe or interact with running systems in real time.

**Not operational monitoring.** The Analysis Stack does not track runtime health, emit alerts, or provide real-time operational visibility. Its scope is retrospective analysis of persisted outputs — a fundamentally different operational mode from ongoing observation of running systems.

**Not runtime execution.** The Analysis Stack does not run Strategies, process Events, evaluate Risk, manage Execution Control, or interact with Venues. It analyzes the products of runtime execution, not the execution itself.

**Storage surfaces are used, not governed.** The Analysis Stack reads from and writes to the Data Storage Stack's persistent surfaces but does not manage their organization, retention, or access policies.

**Analysis is the operational activity.** The Analysis Stack's operational behavior is analytical work — loading artifacts, applying evaluative logic, producing derived outputs, preserving traceability. This is the stack's normal operating mode. It does not have a "runtime" in the same sense as the Backtesting or Live Stacks; its operational activity is the analysis itself.

---

## Why This Behavior Matters

The Analysis Stack's operational behavior determines whether the Infrastructure's persisted outputs are turned into actionable, trustworthy analytical knowledge. If analysis is not reproducible, evaluative conclusions are fragile — they may change unpredictably when re-executed. If outputs are not traceable to their inputs, conclusions are unverifiable. If comparison between Research and Live outcomes is not supported, the gap between expectations and production reality remains unmeasured.

The operational behavior described here — asynchronous consumption of persisted artifacts, structured evaluation and comparison, derived output production, and reproducible versioned execution with full traceability — is what makes the Analysis Stack a reliable basis for Research evaluation and production assessment.
