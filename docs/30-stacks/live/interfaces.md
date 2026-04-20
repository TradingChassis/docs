# Interfaces

This document defines the inputs, outputs, and interface boundaries of the Live Stack — what enters it, what leaves it, and how it relates to external Venues and other Stacks.

---

## Inputs

The Live Stack consumes real-time inputs that drive the Core Runtime's processing in a production trading context.

### Real-time market data

The primary runtime input. Live Venue market-data feeds deliver trade messages, order book updates, and associated data in real time. These enter the Core Runtime's Event processing loop as the live equivalent of historical Events — driving State derivation, Strategy evaluation, and the full processing chain.

Real-time market data is the Live Stack's operational basis. The stack does not primarily consume historical canonical datasets or raw recorded datasets during live execution.

### Configuration and Strategy definitions

The Strategy code to be executed and the Configuration under which the Core Runtime operates — execution-control rules, Risk policy parameters, and other settings. These are supplied at startup or through controlled operational updates and define the processing behavior for the current live session.

### Operational control inputs

Control signals relevant to live operation — session start and stop, trading enable/disable, kill-switch activation, and other operational commands that influence the Live Stack's runtime state within the bounds of the Core Runtime's processing model.

---

## Outputs

The Live Stack produces outputs in two directions: outbound toward real Venues, and inbound toward the Infrastructure's persistent and observability layers.

### Outbound execution

The primary operational output. The Core Runtime's processing chain — `Strategy ➝ Risk ➝ Execution Control ➝ Venue Adapter` — produces outbound execution requests (order submissions, modifications, cancellations) that are transmitted to real Venues through the Venue Adapter. These are real market actions with real financial consequences.

### Execution feedback

Venue responses — acknowledgements, fills, rejections, cancellations — re-enter the Live Stack as Execution Events on the Event Stream. This feedback drives Order lifecycle transitions and further State derivation within the Core Runtime's processing loop.

### Execution records and related outputs

Persisted records of live execution activity:

- **Order history** — the full lifecycle of each Order from submission through terminal state.
- **Fill records** — individual fill Events and their details.
- **Position data** — derived position state resulting from execution activity.
- **Execution metadata** — dispatch decisions, timing information, and other execution-related data.

These records are written to the Data Storage Stack's persistent surfaces — primarily **Execution Record Storage** — for durable retention and downstream consumption.

### Observability outputs

Runtime telemetry, execution metrics, and operational signals emitted during live execution. These are consumed by the Monitoring Stack for real-time visibility into trading activity and infrastructure health. Observability outputs are a supporting interface of the Live Stack, not its primary output.

---

## Relationship to Real Venues

Real Venues are the external execution counterpart of the Live Stack. The interface is bidirectional:

- **Outbound.** The Venue Adapter translates outbound execution requests from the Core Runtime into Venue-specific API calls and transmits them. The Venue Adapter handles protocol translation and external I/O; it does not decide policy or scheduling.
- **Inbound.** Venue responses (execution reports, acknowledgements, rejections) are received by the Venue Adapter and surfaced as Execution Events that enter the Event Stream for processing by the Core Runtime.

The Live Stack operates at the Infrastructure boundary with the real market. The Venue Adapter isolates Venue-specific protocol behavior from the Core Runtime — the Core processes canonical Execution Events regardless of which Venue produced them.

---

## Relationship to the Core Runtime

The Live Stack **uses** the Core Runtime as its execution kernel. The Core Runtime provides the deterministic, event-driven processing model — Event intake, State derivation, Strategy evaluation, Risk, Execution Control, Venue Adapter interaction, and feedback re-entry — that the Live Stack executes against real-time inputs.

The interface between the Live Stack and the Core Runtime is:

- The Live Stack supplies **real-time Events** (from live Venue market-data feeds and Execution Events from Venue feedback) and **Configuration** to the Core Runtime.
- The Core Runtime processes those Events deterministically and produces State, dispatch decisions, and execution-control outcomes.
- Outbound work selected for dispatch is transmitted through the Venue Adapter to real Venues.
- Venue feedback returns as Execution Events, closing the processing loop.

The Live Stack does not redefine Event, State, Intent, Order, Queue, Risk, or Determinism semantics. Those are established in architecture and concept documents and apply identically in Live execution.

---

## Relationship to Data Storage, Monitoring, and Analysis

### Data Storage Stack

Provides the persistent surfaces for the Live Stack's durable outputs. The Live Stack writes execution records, order history, fill records, position data, and execution metadata to **Execution Record Storage**. The Data Storage Stack provides and governs these surfaces; the Live Stack writes to them.

### Monitoring Stack

Provides the observability and operational visibility platform. The Live Stack emits runtime telemetry — execution throughput, order status, error conditions, infrastructure health indicators — that the Monitoring Stack consumes, processes, and presents. The Monitoring Stack is a separate Stack; the Live Stack supports observable execution without owning the monitoring infrastructure.

### Analysis Stack

Consumes persisted Live outputs for post-hoc analysis — execution-quality review, performance evaluation, and operational assessment. The Analysis Stack reads from Execution Record Storage and other relevant persistent surfaces after live execution has occurred. There is no synchronous interface between the Live Stack and the Analysis Stack during live operation; the relationship is mediated by durable persistence.

---

## Interface Boundaries

**Real-time market data is the primary input basis.** The Live Stack operates on live Venue feeds, not on historical datasets or raw recorded data. The execution context is real-time.

**Real Venues are the execution counterpart.** Outbound execution requests go to real Venues; inbound execution feedback comes from real Venues. This is the defining characteristic that distinguishes the Live Stack from other execution contexts.

**Execution records are the primary persistable output.** The Live Stack's durable contribution to the Infrastructure is the execution record — the persisted history of what was dispatched, what the Venue reported, and what Order lifecycle outcomes resulted. These records are written to the Data Storage Stack and are available for downstream analysis.

**The Core Runtime is used, not defined.** The Live Stack supplies inputs to and captures outputs from the Core Runtime. It does not modify the Core Runtime's processing semantics.

**Monitoring is a supporting interface.** The Live Stack emits telemetry for observability, but monitoring is not its primary interface or output. The Monitoring Stack consumes what the Live Stack emits; the Live Stack does not own the monitoring layer.

**Persistence is mediated by the Data Storage Stack.** Execution records and related outputs are written to the Data Storage Stack's persistent surfaces. The Live Stack does not manage storage infrastructure.
