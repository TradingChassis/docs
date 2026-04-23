---
title: hftbacktest as Simulated Venue for Backtesting
updated: 2026-04-01
adr_status: Accepted
---

# ADR-008: hftbacktest as Simulated Venue for Backtesting

Status: {{ page.meta.adr_status }}  
Updated: {{ page.meta.updated }}

---

## Context

The Infrastructure's Backtesting Runtime replays historical market data through the same Core Runtime used in Live. The Core interacts with Venues exclusively through the Venue Adapter boundary — in Live, the Adapter connects to a real Venue; in Backtesting, it must connect to a simulated Venue that produces realistic execution feedback from historical data.

The simulated Venue is not optional infrastructure. Without it, the Backtesting Runtime has no mechanism to generate Execution Events in response to dispatched outbound work. Strategy evaluation requires that the full processing loop — dispatch through the Venue Adapter, execution feedback as Events, State derivation from those Events — operates in Backtesting as it does in Live.

The simulator must satisfy specific architectural and fidelity requirements:

- **Architectural boundary compatibility.** The simulated Venue must sit behind the Venue Adapter boundary in the same position a real Venue occupies in Live. The Core must not require modifications or alternative processing paths to operate in Backtesting. This is a direct requirement of the Infrastructure's Backtesting/Live semantic parity.
- **Microstructure fidelity.** The Infrastructure targets microstructure-sensitive Strategy Research. Realistic evaluation requires that the simulated Venue model order book matching, queue priority effects, partial fills, and execution sequencing — not simplified assumptions such as immediate fill at mid-price. Strategies that depend on order book dynamics produce misleading results under simplistic simulation.
- **Deterministic behavior.** The simulated Venue must produce deterministic output given the same historical input data and the same outbound requests. Non-deterministic simulation would break the Infrastructure's replay guarantee.

Building a microstructure-level Venue simulator in-house would require:

- order book reconstruction and matching engine implementation
- queue priority modeling (or approximation depending on available data granularity)
- partial fill simulation, latency modeling, and fee calculation
- ongoing maintenance as Research requirements evolve

This represents substantial development effort in a domain (exchange microstructure simulation) that is not the Infrastructure's core competency. The question is whether an existing implementation can satisfy the architectural and fidelity requirements, avoiding the cost and risk of building this component from scratch.

---

## Decision

The Backtesting Runtime uses [hftbacktest](https://github.com/nkaz001/hftbacktest) as the simulated Venue implementation.

### Core rules

1. **hftbacktest operates behind the Venue Adapter boundary.** The Core Runtime dispatches outbound work through the Venue Adapter; the Adapter translates it into hftbacktest interactions. Execution feedback from hftbacktest is surfaced as Events that enter the Event Stream. The Core does not interact with hftbacktest directly — the Venue Adapter mediates all communication.

2. **The architectural boundary shape is preserved.** The same Venue Adapter interface that connects to a real Venue in Live connects to hftbacktest in Backtesting. The Core Runtime does not contain Backtesting-specific processing paths or conditional logic that depends on which Venue implementation is behind the Adapter.

3. **hftbacktest provides the microstructure simulation layer only.** It simulates order book matching, fill generation, and execution feedback. All surrounding infrastructure — Event processing, State derivation, Strategy evaluation, Risk, Execution Control, the Venue Adapter itself — remains implemented within the Infrastructure.

4. **Simulation behavior must be deterministic.** Given the same historical input data and the same sequence of outbound requests, hftbacktest must produce identical execution feedback. This is required for the Infrastructure's replay and reproducibility guarantees to hold in Backtesting.

### What this decision covers

This decision selects the simulated Venue implementation for Backtesting. It does not define Backtesting Runtime architecture, data flow, experiment tracking, or the Venue Adapter interface — those belong to the Backtesting Stack documentation and to the Logical Architecture.

---

## Consequences

**The full processing loop operates in Backtesting.** Because hftbacktest sits behind the Venue Adapter boundary and produces realistic execution feedback, the complete chain — Strategy emits Intents, Risk evaluates admissibility, Execution Control schedules dispatch, Venue Adapter transmits, simulated Venue responds, Execution Events derive Order state — runs in Backtesting as it does in Live. Strategy evaluation covers the entire outbound path, not a truncated approximation.

**Backtesting/Live parity is preserved at the architectural boundary.** The Core Runtime operates identically in both Runtimes. The only difference is the Venue implementation behind the Adapter. This means that architectural assumptions validated in Backtesting (component boundaries, lifecycle transitions, Execution Control behavior) hold in Live without requiring separate verification of the processing path.

**Microstructure-sensitive Research is supported.** hftbacktest models order book matching, queue priority effects, and partial fills. Strategies that depend on order book dynamics can be evaluated under conditions that approximate real Venue behavior, rather than under simplified fill assumptions that would mask microstructure-dependent effects.

**The Infrastructure depends on a third-party component for simulated execution.** hftbacktest is external software. Its simulation model, data format expectations, and behavior are not controlled by the Infrastructure. Changes to hftbacktest (API, matching semantics, supported features) require corresponding Venue Adapter updates. This dependency is scoped to the Backtesting Venue boundary — it does not affect the Core Runtime, Live execution, or any component upstream of the Venue Adapter.

---

## Trade-offs

**Third-party dependency vs in-house implementation.** Using hftbacktest introduces a dependency on external software for a critical Backtesting component. The alternative — implementing a microstructure simulator in-house — would eliminate the dependency but require substantial development in order book reconstruction, matching logic, queue priority modeling, and partial fill simulation. This effort would be ongoing as Research requirements evolve. The decision accepts the dependency in exchange for avoiding that development cost, given that hftbacktest satisfies the architectural and fidelity requirements.

**Simulation fidelity is bounded by hftbacktest's model.** The simulated Venue's realism is limited by hftbacktest's matching and fill logic. Where hftbacktest's model diverges from actual Venue behavior (e.g., specific queue priority rules, exotic order types, or Venue-specific edge cases), Backtesting results will reflect the simulation model rather than true Venue behavior. This is inherent to any simulation approach; the relevant question is whether hftbacktest's model is sufficient for the Infrastructure's current Research requirements, which it is.

---

## Summary

The Backtesting Runtime uses hftbacktest as the simulated Venue implementation, operating behind the Venue Adapter boundary in the same architectural position a real Venue occupies in Live. The Core Runtime does not contain Backtesting-specific processing paths — it interacts with the Venue Adapter identically in both Runtimes. hftbacktest provides microstructure-level simulation (order book matching, partial fills, queue priority) while all surrounding infrastructure remains within the Infrastructure. This decision accepts a third-party dependency at the Venue boundary in exchange for avoiding the cost of building a microstructure simulator in-house.
