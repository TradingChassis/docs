# Operational Behavior

This document describes how the Live Stack behaves during normal operation: how real-time market data drives runtime execution, how the Core Runtime interacts with real Venues, how execution outcomes become persistent records, and how operational control and observability function during live trading.

---

## Real-Time Market Input and Runtime Execution

During normal operation, the Live Stack continuously receives real-time market data from live Venue feeds — trade messages, order book updates, and associated data — and feeds them into the Core Runtime's processing loop as Events.

The Core Runtime processes these Events deterministically in the same manner as in any execution context: each Event is applied in Processing Order, deriving updated State (Market State, Execution State, Control State). Strategy evaluates derived State projections and may emit Intents. Intents pass through Risk (policy admissibility) and Execution Control (Queue Processing, dominance, inflight gating, rate-limited dispatch). Outbound work selected for dispatch is transmitted to real Venues through the Venue Adapter.

This processing loop runs continuously for the duration of a live session. The Live Stack does not process Events in batch or on a schedule — it processes them as they arrive from the market, maintaining the Core Runtime's real-time responsiveness to market conditions.

The Live Stack executes the Core Runtime's processing semantics faithfully. It does not modify Event processing, State derivation, Risk evaluation, or Execution Control behavior. The Core Runtime model applies identically during live operation.

---

## Venue Interaction During Live Operation

Real Venue interaction is a continuous, bidirectional operational concern during live execution.

**Outbound.** When the Core Runtime's processing chain produces dispatch decisions, the Venue Adapter translates them into Venue-specific API requests and transmits them to real Venues. These are real market actions — order submissions, modifications, and cancellations — with real financial consequences.

**Inbound.** Venues respond with execution feedback: acknowledgements, fills, partial fills, rejections, and cancellations. This feedback is received by the Venue Adapter and surfaced as Execution Events that re-enter the Core Runtime's processing loop. Execution Events drive Order lifecycle transitions (`Submitted → Accepted → Partially Filled → Filled, or Submitted → Rejected`, etc.) and update Execution State.

The outbound-inbound cycle runs continuously during live operation. The Live Stack maintains active connections to real Venues throughout the session, handling the full spectrum of Venue responses as they arrive. Venue interaction is not a periodic batch operation — it is an ongoing, latency-sensitive part of the real-time processing loop.

The Venue Adapter isolates Venue-specific protocol behavior from the Core Runtime. The Core Runtime processes canonical Execution Events regardless of which Venue produced them or what API protocol was used.

---

## Execution Outcomes and Persistent Records

Live execution continuously produces outcomes that must be durably persisted:

- **Order history** — the full lifecycle of each Order from submission through terminal state, as driven by Execution Events.
- **Fill records** — individual fill details as reported by Venues.
- **Position data** — derived position state resulting from accumulated execution activity.
- **Execution metadata** — dispatch decisions, timing information, and operational context associated with each execution action.

These records are written to **Execution Record Storage** during and after live execution. Persistence is not deferred to session end — execution records are written as outcomes are produced, so that a durable record exists even if the session is interrupted.

Persisted execution records are the Live Stack's durable contribution to the System. After live execution completes, these records are available for downstream consumption by the Analysis Stack and operational tooling — enabling post-hoc execution-quality review, performance evaluation, and audit.

---

## Operational Control and Observability

Live execution operates under operational control and with observability support throughout.

### Operational control

The Live Stack accepts operational control inputs during execution:

- **Session lifecycle** — start and stop of live trading sessions.
- **Trading controls** — enable/disable trading, kill-switch activation, and other safety controls that govern whether the Core Runtime's dispatch decisions result in outbound Venue actions.
- **Configuration updates** — where the operational model supports controlled updates to Configuration or parameters during a live session.

Operational control inputs are applied within the bounds of the Core Runtime's processing model. They govern when and under what conditions live execution proceeds — they do not alter the Core Runtime's internal processing semantics.

### Observability

The Live Stack emits runtime telemetry throughout execution — execution throughput, order status, processing latency, error conditions, and system health indicators. The Monitoring Stack consumes this telemetry for real-time dashboards, alerting, and operational visibility.

Observability is operationally critical for live trading. The Live Stack must remain observable at all times during execution so that operators can assess system health, detect anomalies, and make informed operational decisions. However, the Monitoring Stack owns the monitoring platform — the Live Stack's responsibility is to emit the telemetry that makes observability possible.

---

## Operational Boundaries

**The Core Runtime is executed, not modified.** The Live Stack's operational behavior consists of feeding, executing, and capturing the outputs of the Core Runtime in a real-time production context. It does not alter Event processing, State derivation, Risk evaluation, or Execution Control behavior during operation.

**Real-time market data is the operational input basis.** The Live Stack operates on live Venue feeds, not on historical datasets. The execution context is real-time and continuous.

**Venue interaction is real.** Outbound execution requests go to real Venues and produce real market effects. This distinguishes live operational behavior from simulated execution and carries the corresponding operational and financial stakes.

**Persistence is mediated by the Data Storage Stack.** The Live Stack writes execution records to Execution Record Storage. It does not manage the storage surface, its organization, or its retention policies.

**Monitoring is a supporting concern.** The Live Stack emits telemetry and supports observable execution. It does not own the monitoring platform, dashboards, or alerting infrastructure.

**This document describes behavior, not procedures.** Operational runbooks, incident response, and recovery procedures are defined elsewhere. This document describes the architectural behavior of the Live Stack during normal operation.

---

## Why This Behavior Matters

The Live Stack's operational behavior is where the System's architecture produces real market outcomes. Every architectural decision upstream — the deterministic processing model, the event-derived state model, the layered runtime architecture, the separation of Risk from Execution Control, the explicit lifecycle state machines — converges into real-time execution that submits real orders and manages real positions.

If the Live Stack's operational behavior does not faithfully execute the Core Runtime's processing model, the system's architectural guarantees do not hold in production. If execution outcomes are not persistently recorded, the system has no durable history of what it did. If the live environment is not observable and controllable, it cannot be operated safely.

The operational behavior described here — continuous real-time execution of the Core Runtime, bidirectional Venue interaction, persistent execution-record production, and operator-visible, controllable runtime behavior — is what makes the Live Stack a production-grade trading infrastructure.
