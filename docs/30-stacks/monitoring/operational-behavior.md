# Operational Behavior

This document describes how the Monitoring Stack behaves during normal operation: how observability signals are collected from running system parts, how health and status become visible, how alert-oriented monitoring arises from ongoing conditions, and how operational visibility is maintained while the System is running.

---

## Runtime Signal Observation

The Monitoring Stack operates concurrently with the system parts it observes. During normal operation, it continuously receives observability-relevant signals — telemetry, metrics, status indicators, health signals, error and failure events — from running Stacks and components.

Signal observation is **ongoing and runtime-concurrent**. The Monitoring Stack does not wait for execution to complete before it begins. It observes execution as it happens, maintaining a current picture of what is running, what state it is in, and whether conditions are nominal.

The strongest signal relationships exist with the **Live Stack** and the **Backtesting Stack**, where runtime observability is most operationally critical. Other running system parts may also emit signals toward the Monitoring Stack where observability support is needed.

Signal collection is passive with respect to the observed systems — the Monitoring Stack receives and integrates signals without altering the execution logic, processing decisions, or runtime behavior of the Stacks it observes. Running system parts retain full ownership of their execution semantics. The Monitoring Stack provides visibility into that execution, not influence over it.

---

## Health, Status, and Telemetry During Operation

During active system execution, the Monitoring Stack maintains structured representations of health, status, and telemetry for monitored components.

**Health visibility.** Health signals — liveness checks, readiness indicators, health-check responses — are evaluated as they arrive to produce current health assessments. The Monitoring Stack maintains an up-to-date view of whether monitored components are functioning within expected parameters, partially degraded, or in a failure state. Health visibility is continuous: health assessments reflect the most recent signals, not a historical snapshot.

**Status tracking.** Status signals indicate the operational state of individual components or subsystems — running, idle, initializing, errored, stopped, or similar runtime states. The Monitoring Stack maintains current-status representations that reflect the latest reported state of each monitored part. Status transitions — a component moving from running to degraded, or from errored to recovering — are captured as they occur.

**Telemetry structuring.** Metrics, counters, gauges, histograms, and other telemetry are aggregated into structured representations as they arrive. Telemetry is organized by source, component, and structuring dimensions so that it is queryable and meaningful. Time-series metrics accumulate during operation, providing both instantaneous readings and trend information.

Health, status, and telemetry are maintained as a **current operational picture** — a structured, continuously updated representation of how monitored system parts are behaving right now. This is the operational core of the Monitoring Stack's runtime behavior.

---

## Alert-Oriented Monitoring Behavior

Alert-oriented monitoring arises from ongoing evaluation of the signals the Monitoring Stack collects. It is not a separate operational phase — it is a continuous evaluation that runs alongside signal collection and health tracking.

During operation, the Monitoring Stack evaluates collected metrics, health assessments, and status information against defined monitoring conditions. When a noteworthy condition is detected — a threshold breach, an error-rate spike, a health degradation, an anomalous pattern — the Monitoring Stack produces an alert-oriented output.

Alert-oriented behavior is **condition-driven**. Alerts emerge from the runtime data as conditions arise; they are not scheduled or batch-processed. The time between a condition occurring and an alert being produced is determined by signal collection frequency, evaluation cadence, and the nature of the condition — but the intent is timely detection, not eventual discovery.

Alert outputs may be consumed by operators directly, routed to external notification systems, or recorded as alert artifacts. The Monitoring Stack is responsible for detecting conditions and producing alerts; operational judgment about how to respond remains a human responsibility. The Monitoring Stack surfaces the condition — it does not prescribe the response.

---

## Operational Visibility in Active Execution

The Monitoring Stack exposes operational visibility surfaces that make running system behavior accessible during active execution.

During operation, telemetry, health, status, and alert information are available through visibility surfaces — queryable views, metric displays, status summaries, and diagnostic views that operators and operational tooling can access. These surfaces reflect the current state of the Monitoring Stack's collected and structured data, updated as new signals arrive.

Operational visibility serves several runtime needs:

- **Routine awareness.** During normal operation, visibility surfaces confirm that systems are running nominally — metrics are within expected ranges, health is positive, no alert conditions are active. This is the baseline operational behavior: the Monitoring Stack provides assurance that running system parts are behaving as expected.
- **Degradation detection.** When conditions begin to deviate from normal, visibility surfaces make the deviation inspectable — which component is affected, what metrics are changing, how health assessments have shifted. The Monitoring Stack makes it possible to see degradation as it develops, not only after it has produced visible downstream consequences.
- **Incident-relevant observation.** When failures or significant anomalies occur, visibility surfaces provide the observability needed to identify, locate, and characterize the issue — correlated views, error aggregations, and diagnostic data that support understanding what is happening and where.

Operational visibility is **runtime-continuous**. It does not require a triggering event to become active — it is the normal operating state of the Monitoring Stack. Visibility surfaces are available throughout system execution, whether conditions are nominal or anomalous.

---

## Operational Boundaries

**Runtime-concurrent, not retrospective.** The Monitoring Stack's operational behavior is defined by what is happening while the System is running. It does not evaluate persisted experiment results, compare Strategy performance across historical runs, or produce derived analytical artifacts. Retrospective evaluation of stored outputs is an Analysis Stack responsibility.

**Observability, not execution.** The Monitoring Stack observes running system parts without participating in their execution logic. It does not run Strategies, process Events, evaluate Risk, manage Execution Control, or interact with Venues. Its operational activity is signal collection, health evaluation, alert detection, and visibility exposure — not runtime business logic.

**Detection, not response.** The Monitoring Stack detects conditions and makes them visible. It does not execute incident-response procedures, initiate recovery actions, or make operational decisions. Those are human responsibilities supported by the visibility the Monitoring Stack provides.

**Tool overlap does not change operational behavior.** When a single product or platform combines monitoring, orchestration, and execution-status concerns, the Monitoring Stack's operational behavior remains scoped to observability and operational visibility. The behavior described here is defined by architectural role, not by product packaging.

---

## Why This Behavior Matters

The Monitoring Stack's operational behavior determines whether running system parts are observable during execution. Without ongoing signal collection, health evaluation, and operational visibility, runtime Stacks execute without structured awareness of their operational state — degradations proceed without detection, failures become visible only through their downstream consequences, and the operational picture remains incomplete until after the fact.

The operational behavior described here — continuous signal observation, structured health and status tracking, condition-driven alert evaluation, and runtime-concurrent visibility exposure — is what makes the Monitoring Stack an effective observability layer. It ensures that while the System is running, its operational state is accessible, its health is assessable, and conditions that require attention are surfaced in time to act on them.
