# Backtesting Stack

Part of: **Core Runtime**

The Backtesting Stack provides the runtime environment, orchestration, and supporting infrastructure required to execute the Core Runtime in Backtesting mode against historical canonical datasets for Research.

---

## Purpose

The Backtesting Stack exists to make the Core Runtime executable for Research — to operationalize deterministic, reproducible Strategy evaluation by running the same event-driven processing model used in Live against historical data with simulated Venue execution.

The Backtesting Stack **uses** the Core Runtime; it does not **define** it. The Core Runtime's semantic model — Events, State derivation, Intents, Risk, Execution Control, Order lifecycle, Determinism — is established in architecture and concept documents. The Backtesting Stack provides the execution environment, historical input feeding, simulated Venue, experiment orchestration, and result persistence that make that model usable for Research.

---

## Position in the Infrastructure

The Backtesting Stack belongs to the **Core Runtime** group:

`Data Platform → [Canonical Storage] → Backtesting Stack → [Experiment Outputs] → Analysis Stack`

It sits downstream of **Canonical Storage** (consuming validated historical datasets as input) and upstream of the **Analysis Stack** (producing experiment results and artifacts for evaluation). It depends on:

- **Data Storage Stack** — for canonical dataset inputs and for persisting experiment outputs, execution records, and artifacts.
- **Monitoring Stack** — for observability of Backtesting runs.

The Backtesting Stack is not part of the Data Platform. It does not capture, validate, normalize, or promote datasets. It is a consumer of canonical data and a producer of Research artifacts.

---

## Main Responsibilities

The Backtesting Stack is responsible for:

- Executing the Core Runtime in Backtesting mode — running the full processing chain (Event intake, State derivation, Strategy, Risk, Execution Control, Venue Adapter) against historical canonical data.
- Providing a **simulated execution environment** — a Simulated Venue behind the Venue Adapter boundary that generates realistic execution feedback from historical data, completing the processing loop.
- Consuming **canonical datasets** from Canonical Storage as the historical input basis for all Backtesting runs.
- Orchestrating experiment execution — single runs, batch runs, and parameter sweeps with defined configurations.
- Supporting **reproducible run definitions** — ensuring that a backtest can be re-executed with the same inputs and Configuration to produce identical results.
- Realizing **determinism** as an execution property — preserving the Core Runtime's deterministic guarantees through the Backtesting Stack's infrastructure.
- Producing and persisting experiment results, run artifacts, execution records, and metrics for downstream consumption by the Analysis Stack.

---

## Key Boundaries

**Executes the Core Runtime, does not define it.** The Core Runtime's processing semantics apply identically in Backtesting and Live. The Backtesting Stack provides the execution context (historical inputs, simulated Venue); it does not modify the processing model.

**Canonical datasets are the historical basis.** The Backtesting Stack reads from Canonical Storage. It does not consume raw recorded datasets, perform validation, or participate in canonical promotion. All historical data has been validated and promoted by the Data Platform before the Backtesting Stack encounters it.

**Research infrastructure, not Research itself.** The Backtesting Stack is the technical runtime and orchestration basis on which Research backtests are executed. Research as a practice — Strategy design, hypothesis formation, result interpretation, deployment decisions — extends beyond what the stack provides.

**Not Live execution.** The Backtesting Stack operates on historical data with simulated execution. Real-time trading against real Venues is the Live Stack's domain.

---

## Relationship to Other Stacks

**Live Stack.** Shares the same Core Runtime — the same processing model, the same component chain, the same deterministic semantics. The difference is the execution context: the Backtesting Stack uses historical inputs and a Simulated Venue; the Live Stack uses live inputs and real Venues. This parity is what makes Backtesting results architecturally meaningful for Strategy evaluation.

**Data Storage Stack.** Provides the persistent surfaces for both inputs (Canonical Storage) and outputs (Experiment / Artifact Storage, Execution Record Storage). The Backtesting Stack reads from and writes to these surfaces; the Data Storage Stack provides and governs them.

**Analysis Stack.** The primary downstream consumer of Backtesting outputs. Reads experiment results, execution records, and artifacts from the Data Storage Stack's persistent surfaces for Strategy evaluation, performance analysis, and cross-experiment comparison.

**Monitoring Stack.** Provides observability capabilities — run status, execution throughput, error visibility. The Backtesting Stack emits telemetry; the Monitoring Stack consumes and presents it.

---

## Why the Stack Matters

The Backtesting Stack is the mechanism through which the Infrastructure's deterministic Core Runtime is applied to historical data for Strategy evaluation. Without it, the Core Runtime would have no Research execution context — canonical datasets would exist but could not be used for systematic, reproducible Strategy evaluation.

The value of Backtesting depends directly on two properties the stack must preserve: **determinism** (the same inputs produce the same results) and **semantic parity with Live** (the processing model is the same in both Runtimes). A backtest that does not reproduce the Core Runtime's processing semantics provides misleading results. The Backtesting Stack's role is to ensure that the execution environment it provides is faithful to the Core Runtime model, so that conclusions drawn from Research are architecturally grounded.

Detailed treatment of scope and role, interfaces, internal structure, operational behavior, and implementation considerations is provided in the companion documents for this Stack.
