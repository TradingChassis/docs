---
title: Layered Runtime Architecture of the Core
updated: 2026-04-01
adr_status: Accepted
---

# ADR-007: Layered Runtime Architecture of the Core

Status: {{ page.meta.adr_status }}  
Updated: {{ page.meta.updated }}

---

## Context

The Infrastructure must process market data, evaluate trading decisions, enforce safety policy, schedule outbound work, and interact with external Venues — all within a deterministic, event-driven processing model. These are fundamentally different responsibilities, and the architectural question is how they are organized within the Core Runtime.

Without explicit layering, responsibilities become entangled:

- **Strategy absorbs control logic.** If no architectural boundary separates decision-making from execution scheduling, Strategy implementations begin encoding rate-limit awareness, inflight tracking, or dispatch timing — concerns that belong to Execution Control. Strategy becomes coupled to execution mechanics rather than expressing desired actions against derived State.
- **Risk drifts into execution-control logic.** Without a clear mandate that Risk decides admissibility only, the policy layer accumulates scheduling, timing, and dispatch-ordering responsibilities. The result is a component that conflates "is this allowed?" with "when should this be sent?" — two questions that must be answered by different logic with different inputs.
- **Execution Control drifts into policy logic.** Without a boundary, Queue Processing may re-evaluate admissibility rather than only scheduling allowed work. Policy enforcement becomes duplicated and potentially inconsistent between Risk and Execution Control.
- **Venue-specific concerns leak into the Core.** If protocol translation is not isolated, Venue-specific message formats, API semantics, and connection handling spread into Strategy, Risk, or Execution Control. Core logic becomes coupled to a specific Venue's interface, breaking portability across Venues and across Runtimes (Backtesting vs Live).
- **Feedback bypasses the canonical path.** Without explicit layering, Venue responses may update component state directly rather than re-entering the Infrastructure through the Event Stream. This introduces out-of-band mutation paths that break deterministic replay.

These problems are not speculative. Trading infrastructures that grow without explicit layering routinely produce monolithic processing paths where decision logic, safety policy, scheduling, and Venue protocol handling are interleaved in ways that cannot be tested, audited, or evolved independently.

The architecture requires explicit runtime layers with defined responsibilities and hard boundaries, so that each concern is handled by exactly one layer and no layer assumes responsibilities that belong to another.

---

## Decision

The Core Runtime is organized into four layered responsibilities with strict boundaries. Each layer has a single defined role in the processing chain and does not absorb the responsibilities of adjacent layers.

### Processing chain

The canonical outbound processing sequence is:

`Event → derived State → Strategy → Intent → Risk → Queue / Queue Processing → Venue Adapter → Venue`

Venue feedback re-enters the Infrastructure only as Events appended to the Event Stream. There is no path by which Venue responses update derived State without passing through Event processing.

### Layer definitions

**Strategy** reads projections of derived State and emits Intents — ephemeral commands expressing desired trading actions. Strategy decides **what** actions should occur.

- Strategy does not interact with Venues, Venue Adapters, or execution-control substate.
- Strategy does not enforce policy, schedule outbound transmission, or manage dispatch timing.
- Strategy must not assume an Intent has been executed or that an Order exists until Execution State reflects it through Events. Orders are derived entities; they begin at submission, not at Intent generation.

**Risk Engine** evaluates each Intent against policy and decides **whether** the action is permitted: allowed or denied.

- Risk decides admissibility only. The outcome is binary: allowed or denied.
- Risk does not schedule transmission, choose send timing, apply rate limits, enforce inflight gating, or determine dispatch ordering. Those are Execution Control responsibilities.
- Risk does not produce a "queued" or "send later" outcome. Delay is an execution-control decision applied after an Intent is allowed.

**Queue + Queue Processing (Execution Control)** schedules and controls outbound dispatch of allowed work. Execution Control decides **when and in what order** allowed work is dispatched.

- The Queue holds derived execution-control substate: effective reconciled allowed pending outbound work. It is not a fourth top-level State domain and not a second source of truth — it is recomputable from Event Stream + Configuration.
- Queue Processing computes eligibility, ordering, inflight gating, rate-compliant timing, and dominance as deterministic derivations within Event processing. There is no separate runtime tick, background timer, or autonomous scheduler.
- Execution Control does not re-evaluate policy admissibility. That responsibility belongs exclusively to Risk.

**Venue Adapter** performs protocol translation and external I/O at the Infrastructure boundary. The Venue Adapter handles **how** outbound work is transmitted and how Venue feedback is surfaced.

- The Venue Adapter translates outbound work selected by Execution Control into Venue-specific requests and transmits them.
- Venue responses are surfaced so they can enter the Event Stream as Execution Events. The Adapter does not update derived State directly.
- The Venue Adapter does not decide policy (Risk) or dispatch scheduling (Execution Control). It performs protocol translation only.

### Feedback path

Venue feedback — acknowledgements, fills, rejections, cancellations — re-enters the Infrastructure exclusively as Events appended to the Event Stream. On subsequent processing steps, these Events derive updated Execution State (Order lifecycle transitions, position changes). Strategy may then emit new Intents based on the updated projections.

There is no parallel path by which Venue responses update State without flowing through Event processing. This is what closes the processing loop while preserving determinism.

### Layer separation is a four-way partition

| Layer | Control question | What it must not do |
| ----- | ---------------- | ------------------- |
| **Strategy** | What should occur? | Enforce policy, schedule dispatch, interact with Venues |
| **Risk** | Is this allowed? | Schedule dispatch, manage timing, gate inflight requests |
| **Execution Control** | When and in what order? | Re-evaluate policy, interact with Venues directly |
| **Venue Adapter** | How is it transmitted? | Decide policy, decide scheduling, own derived State |

Each layer answers exactly one control question. No layer answers a question that belongs to another.

---

## Consequences

**Each layer is testable against its own contract.** Strategy can be tested against derived State projections without instantiating Risk, Execution Control, or a Venue. Risk can be tested against policy rules without scheduling logic. Execution Control can be tested against dispatch rules without policy evaluation. The Venue Adapter can be tested against protocol translation without upstream logic. This is possible because each layer has a defined input, a defined output, and no dependency on the internals of adjacent layers.

**Strategy is isolated from execution mechanics.** Strategy reads derived State and emits Intents. It does not know whether an Intent will be rate-limited, delayed by inflight gating, or collapsed by dominance. Execution stability does not depend on Strategy implementation discipline — it is enforced by the layers downstream.

**Policy enforcement is centralized and consistent.** Every Intent passes through Risk before entering Execution Control. There is no alternative path by which Strategy output reaches a Venue without a prior policy decision. This holds across all Strategies and all Runtimes.

**Execution Control is deterministic and replayable.** Because Queue Processing runs within Event processing (not as a separate tick), and because the Queue is derived from Event Stream + Configuration, replay of the same stream under the same Configuration produces identical dispatch decisions at every Processing Order position. Backtesting and Live evaluate the same execution-control logic.

**Venue-specific concerns do not propagate into the Core.** The Venue Adapter boundary ensures that Venue protocol differences, API specifics, and connection semantics are confined to the Adapter. Replacing or adding a Venue Adapter does not require changes to Strategy, Risk, or Execution Control. The same Core logic applies in Backtesting (with a simulated Venue) and Live (with a real Venue).

**Feedback is canonically mediated.** Because Venue responses re-enter only through Events on the Event Stream, no component holds Venue-derived state that the Event Stream cannot reproduce. This preserves the derivation guarantee: `State = f(Event Stream, Configuration)`.

---

## Trade-offs

**Every outbound action traverses four layers.** The layered architecture means no shortcut path exists from Strategy to Venue. This is more processing per Intent than a direct pass-through design — but direct pass-through cannot provide the policy enforcement, execution-control stability, and deterministic replay guarantees the Infrastructure requires.

**Layer boundaries must be enforced by implementation discipline.** The architecture defines what each layer must and must not do, but enforcement depends on implementation respecting those boundaries. A component that quietly absorbs an adjacent layer's responsibility (e.g., Strategy encoding rate-limit logic, or Risk applying dispatch timing) violates the layered model without necessarily producing an immediate failure. Architectural review and testing must verify that boundaries hold.

---

## Summary

The Core Runtime is organized into four layered responsibilities — Strategy, Risk, Execution Control (Queue + Queue Processing), and Venue Adapter — with strict boundaries. Strategy decides what actions should occur. Risk decides whether they are allowed. Execution Control decides when and in what order allowed work is dispatched. The Venue Adapter handles protocol translation and external I/O. Venue feedback re-enters only through the Event Stream. Each layer answers exactly one control question, does not absorb adjacent responsibilities, and can be tested, audited, and evolved independently. All processing runs within deterministic Event processing; there is no separate runtime tick.
