# Implementation Notes

This document captures implementation-facing rationale and design considerations for the Monitoring Stack: how runtime observability is instrumented and collected in practice, how telemetry and health integration are structured, how alert-oriented monitoring is realized, and how conceptual separation between monitoring and execution is preserved at the implementation level.

---

## Why Monitoring Realization Matters

The Monitoring Stack makes running infrastructure behavior visible. How that visibility is realized — how signals are instrumented, how telemetry is collected, how health is evaluated, how alerts are produced, how operational views are exposed — determines whether runtime Stacks are genuinely observable or merely nominally monitored.

A weak realization (e.g., unstructured log inspection, ad hoc health checks, alert rules that do not correspond to meaningful operational conditions) produces operational visibility that is unreliable, noisy, or incomplete. The implementation patterns described here are designed to make runtime observability structured, integrable, and operationally useful.

---

## Realizing Runtime Observability

Runtime observability begins with instrumentation: running infrastructure parts must emit signals that the Monitoring Stack can collect. The quality and structure of those signals determine the quality of the operational visibility the Monitoring Stack can provide.

**Instrumentation at the source.** Running Stacks — primarily the Live Stack and the Backtesting Stack — instrument their components to emit telemetry, metrics, status indicators, health responses, and error signals. Instrumentation is a responsibility of the emitting Stack; the Monitoring Stack provides the receiving infrastructure and integration surfaces, not the instrumentation logic itself.

**Collection paths.** Observability signals travel from running components to the Monitoring Stack through collection paths. These may be realized as:

- **Push-based emission** — running components emit metrics, events, or status signals to a collection endpoint provided by the Monitoring Stack.
- **Pull-based polling** — the Monitoring Stack queries health endpoints, status APIs, or metric surfaces exposed by running components on a defined cadence.
- **Log and trace ingestion** — structured log output or distributed trace spans are collected from running components and integrated into the Monitoring Stack's telemetry pipeline.

No specific collection mechanism is architecturally required. The key implementation properties are:

- Collection paths are **non-intrusive** — they do not alter the execution logic or runtime behavior of the observed infrastructures.
- Collection is **continuous during execution** — signals are gathered while the Infrastructure is running, not retroactively after execution completes.
- Collection supports **multiple signal types** — metrics, status, health, errors, and traces can coexist within the same integration infrastructure.

---

## Telemetry, Health, and Status Integration Patterns

Once signals reach the Monitoring Stack, they must be structured and organized for downstream consumption — health evaluation, alert evaluation, and operational visibility.

**Metrics aggregation.** Raw metric signals are aggregated into time-series representations — counters, gauges, histograms, and rates organized by component, dimension, and time window. Aggregation transforms high-frequency signal streams into queryable metric surfaces suitable for dashboards, trend analysis, and threshold evaluation.

**Health integration.** Health signals — liveness checks, readiness responses, health-check results — are integrated into a current-health representation for each monitored component. Health integration may combine multiple signal types (e.g., a component is considered healthy only if its liveness check passes, its error rate is below threshold, and its latency is within bounds). The implementation should support composite health definitions that reflect operational reality rather than single-signal checks.

**Status normalization.** Status indicators from different running components may use different representations. The Monitoring Stack normalizes these into a consistent status model so that operational visibility surfaces present a coherent picture. A component reporting "active" and another reporting "running" should be representable in a unified status view.

**Signal retention.** Collected signals have a monitoring-relevant retention window — recent metrics, current health, recent status transitions. This is operational retention for monitoring purposes, not long-term analytical storage. Where monitoring records are persisted beyond the immediate operational window, they serve operational continuity and post-incident reference, not retrospective analytical evaluation.

---

## Alert-Oriented Monitoring Patterns

Alert-oriented monitoring transforms ongoing signal evaluation into actionable notifications when noteworthy conditions are detected.

**Condition-based alert evaluation.** Alert logic evaluates collected metrics, health assessments, and status data against defined conditions — threshold breaches, error-rate spikes, sustained degradations, anomalous patterns. Conditions are defined as monitoring configuration, not as architectural commitments — they are expected to evolve as operational understanding matures.

**Alert severity and classification.** Alert outputs benefit from severity classification (e.g., critical, warning, informational) and condition classification (e.g., health degradation, threshold breach, error spike). Structured classification supports routing and prioritization without requiring human operators to triage raw signals.

**Alert routing.** Alert-oriented outputs may be consumed through multiple channels — direct display on visibility surfaces, delivery to external notification infrastructures, recording as alert artifacts. The implementation should support configurable routing so that alert delivery can be adapted to operational needs without changing alert evaluation logic.

**Alert lifecycle awareness.** Conditions that trigger alerts may resolve. The implementation should support tracking whether an alert condition is active or resolved, so that operational visibility surfaces reflect current conditions rather than accumulating stale alerts. Firing, acknowledgment, and resolution form a natural alert lifecycle that the implementation should respect.

---

## Operational Visibility Surfaces

Operational visibility surfaces are the consumption point where monitoring data becomes operationally useful.

**Dashboards and metric views.** Metric time series, health summaries, and status overviews are presented through structured views that operators can inspect during normal operation and during incidents. These surfaces should be organized to support both routine awareness (everything is nominal) and directed investigation (something is wrong — where and what).

**Status and health summaries.** Aggregated views that present the operational state of monitored components at a glance — which components are healthy, which are degraded, which are in error states. These summaries provide an entry point for operational reasoning without requiring operators to inspect individual metric streams.

**Diagnostic views.** When an issue is detected, operators need correlated views that bring together the relevant signals — error rates, health transitions, metric anomalies, and status changes for the affected components. Diagnostic views support incident identification and characterization without requiring operators to manually correlate across disconnected signal sources.

**Queryable telemetry.** Beyond pre-structured views, the implementation should support ad hoc querying of collected telemetry — allowing operators to inspect specific metrics, filter by time range or component, and explore signal data in response to questions that pre-built dashboards may not anticipate.

---

## Conceptual Separation Despite Tool Overlap

Real products and platforms frequently combine monitoring, orchestration, execution management, and operational visibility in a single interface. This practical convergence creates an implementation-level challenge: maintaining the Monitoring Stack's conceptual boundary when the tooling does not enforce it.

**Architectural role over product packaging.** When a single tool provides both execution management and monitoring-like views, the Monitoring Stack's implementation responsibility remains scoped to observability and operational visibility. Execution management features of the same tool belong architecturally to the relevant execution Stack, not to the Monitoring Stack.

**Configuration separation where practical.** Where tools support it, monitoring configuration (alert rules, metric collection definitions, dashboard layouts) should be managed separately from execution configuration (run parameters, scheduling, resource allocation). This separation reinforces the conceptual boundary at the implementation level.

**Documentation clarity.** When a single tool serves multiple architectural roles, implementation documentation should identify which capabilities are considered Monitoring Stack responsibilities and which belong to other Stacks. This prevents the Monitoring Stack's scope from expanding implicitly to encompass everything a multi-purpose tool can do.

---

## Implementation Boundaries

**Implementation choices, not canonical semantics.** The collection patterns, aggregation strategies, alert structures, and visibility approaches described here are implementation-level concerns. They do not define or modify the Infrastructure's canonical Event, State, or lifecycle semantics.

**Monitoring operates on runtime signals.** The implementation collects and processes signals from running infrastructure parts. It does not consume persisted analytical outputs, historical experiment results, or retrospective evaluation datasets.

**Not retrospective analysis.** The Monitoring Stack's implementation is oriented around runtime-concurrent observation. It does not implement asynchronous evaluation of stored experiment results, cross-run comparison, or derived analytical artifact production. Those are Analysis Stack concerns.

**No storage governance.** Where the Monitoring Stack persists monitoring records, it writes to available storage surfaces but does not manage their organization, retention policies, or access governance.

**No deployment specification.** The patterns above describe design considerations and realization approaches. They do not prescribe specific metrics platforms, alerting infrastructures, dashboard tools, or infrastructure configurations. Those belong in deployment documentation.
