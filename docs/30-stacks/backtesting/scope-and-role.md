# Scope and Role

The Backtesting Stack provides the runtime environment, orchestration, and supporting infrastructure required to execute the Core Runtime in Backtesting mode against historical canonical datasets for Research.

---

## Purpose

The Backtesting Stack exists to make the Core Runtime executable for Research. It operationalizes deterministic, reproducible Strategy evaluation by running the Core Runtime against historical canonical datasets with simulated Venue execution, producing experiment results, artifacts, and metrics for later analysis.

The Backtesting Stack **uses** the Core Runtime — it does not **define** it. The Core Runtime's semantic model (Events, State, Intents, Orders, Risk, Execution Control, Queue Processing, Determinism) is established in architecture and concept documents. The Backtesting Stack realizes that model in a Backtesting execution context: it supplies historical inputs, provides a simulated Venue, orchestrates experiment runs, and persists the outputs.

---

## Position in the Infrastructure

The Backtesting Stack belongs to the **Core Runtime** group:

`Data Platform ➝ [Canonical Storage] ➝ Backtesting Stack ➝ [Experiment Outputs] ➝ Analysis Stack`

It sits downstream of **Canonical Storage** (consuming validated historical datasets) and upstream of the **Analysis Stack** (producing experiment results and artifacts for evaluation). It depends on:

- **Data Storage Stack** — for canonical dataset inputs and for persisting experiment outputs, execution records, and artifacts.
- **Monitoring Stack** — for observability of Backtesting runs where relevant.

The Backtesting Stack is **not** part of the Data Platform. It does not capture, validate, normalize, or promote datasets. It is a consumer of canonical datasets and a producer of Research artifacts.

### Relationship to the Core Runtime

The Backtesting Stack executes the same Core Runtime that the Live Stack executes — the same Event-driven processing model, the same Strategy ➝ Risk ➝ Execution Control ➝ Venue Adapter chain, the same deterministic State derivation. The difference is the execution context: historical canonical datasets replace live Venue feeds, and a simulated Venue replaces a real one. The Core Runtime semantics do not change between Backtesting and Live; the Backtesting Stack provides the environment in which those semantics are applied to historical data.

### Relationship to Research

The Backtesting Stack is the technical runtime and orchestration basis on which Research-oriented backtests are executed. It belongs functionally to Research, but it is not identical with Research as a practice. Research encompasses Strategy design, hypothesis formation, result interpretation, and iterative refinement — activities that extend beyond what the Backtesting Stack provides. The Backtesting Stack provides the execution infrastructure; Research is the broader discipline that uses it.

---

## Core Responsibilities

The Backtesting Stack is responsible for:

- Making the Core Runtime executable in Backtesting mode — providing the runtime environment in which the Core processes historical Events and produces deterministic outputs.
- Providing a **simulated execution environment** — a simulated Venue behind the Venue Adapter boundary that generates realistic execution feedback from historical data, completing the processing loop that the Core Runtime requires.
- Using **canonical datasets** from Canonical Storage as the historical input basis for Strategy evaluation.
- Orchestrating experiment runs — managing the execution of individual backtests, batch runs, and parameter sweeps with defined configurations.
- Supporting **reproducible run configurations** — ensuring that a backtest can be re-executed against the same canonical dataset, the same Strategy, and the same Configuration to produce identical results.
- Realizing **determinism** as an execution property — ensuring that the deterministic guarantees of the Core Runtime are preserved through the Backtesting Stack's orchestration and environment, not undermined by infrastructure non-determinism.
- Producing experiment results, run artifacts, metrics, and evaluation outputs for later consumption by the Analysis Stack.
- Persisting Backtesting outputs — experiment results, execution records, and artifacts — to the Data Storage Stack for durable retention and downstream analysis.

---

## Explicit Non-Responsibilities

The Backtesting Stack is **not** responsible for:

- **Core Runtime semantic definitions.** The Event model, State model, Determinism model, Intent lifecycle, Order lifecycle, Risk semantics, Queue semantics, and all canonical processing rules are defined in architecture and concept documents. The Backtesting Stack realizes these semantics in execution; it does not define or modify them.
- **Raw Venue data capture.** Connecting to Venue feeds, recording raw market data, and managing recording infrastructure are Data Recording Stack responsibilities.
- **Dataset validation, normalization, or canonical promotion.** Assessing data quality and promoting datasets to Canonical Storage are Data Quality Stack responsibilities. The Backtesting Stack consumes canonical datasets; it does not produce them.
- **Canonical Storage governance.** Managing the storage layer, its organization, and its retention policies are Data Storage Stack responsibilities.
- **Live execution.** Real-time trading against real Venues is the Live Stack's domain. The Backtesting Stack operates on historical data with simulated execution.
- **Research interpretation.** Interpreting experiment results, forming conclusions about Strategy viability, and making deployment decisions are Research activities performed by humans using the Analysis Stack. The Backtesting Stack produces the raw material for these activities; it does not perform them.

---

## Why the Stack Matters

The Backtesting Stack is the mechanism through which the Infrastructure's deterministic, event-driven Core Runtime is applied to historical data for Strategy evaluation. Without it, the Core Runtime would have no Research execution context — canonical datasets would exist but could not be used for systematic, reproducible Strategy evaluation.

The value of the Backtesting Stack depends directly on the properties it preserves from the Core Runtime: determinism, reproducibility, and semantic parity with Live. A backtest that does not reproduce the same processing semantics as Live execution provides misleading results. The Backtesting Stack's role is to ensure that the execution environment it provides is faithful to the Core Runtime model, so that conclusions drawn from Backtesting results are architecturally grounded.
