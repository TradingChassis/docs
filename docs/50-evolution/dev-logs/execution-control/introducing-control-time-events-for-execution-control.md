---
title: Introducing Control-Time Events for Execution Control
updated: 2026-04-20
adr_status: In progress
---

# Introducing Control-Time Events for Execution Control

Status: {{ page.meta.adr_status }}  
Updated: {{ page.meta.updated }}

---

## Context

The Core Runtime is modeled as a deterministic event-driven system.

Queue Processing currently runs only as part of Event processing. This keeps the model deterministic and avoids hidden runtime ticks, but it also means that pending Execution Control work may remain unevaluated when no new external Events arrive.

This becomes problematic when the Queue contains allowed but not yet dispatchable work that should be retried once Execution Control constraints (such as rate capacity) allow it again.

The goal is to preserve deterministic replay and Backtesting/Live semantic parity while removing unnecessary dependence on external Market or Execution Events for Queue re-evaluation.

---

## Iteration 1 – Initial Observation

### Problem

Execution Control currently depends too strongly on external Event arrival.
If the Queue contains pending outbound work and no new Market or Execution Events arrive, the Queue may not be re-evaluated even when dispatch would become possible again.

### Observations

- Queue Processing is part of Execution Control and does not run on an independent runtime tick.
- This avoids hidden scheduling semantics and preserves deterministic replay.
- However, pending dispatch work can remain blocked in quiet periods if re-evaluation depends only on external Events.

### Initial Hypotheses

- The current model lacks an explicit deterministic mechanism for time-based re-evaluation of the Queue's outbound work.
- A hidden timer or autonomous Queue loop would solve the symptom but would violate the architecture principles.
- A better approach may be to model time-based re-evaluation explicitly as canonical Control-Time Events.

---

## Iteration 2 – Investigation

### Actions Taken

- Re-examined the boundary between Risk and Execution Control.
- Reconfirmed that Risk should decide admissibility only, while Queue / Queue Processing should decide dispatch timing and ordering.
- Reframed the problem as one of deterministic re-evaluation rather than Queue autonomy.

### Results

- Moving time-based dispatchability into Risk would blur policy and Execution Control responsibilities.
- Adding a hidden Queue loop or wall-clock-based runtime tick would weaken determinism and replayability.
- The most promising direction is to introduce explicit Control-Time Events as part of canonical processing input.

### Updated Hypotheses

- Time should be modeled explicitly, not implicitly.
- Time-based re-evaluation should be triggered by sparse, deterministic Control-Time Events rather than periodic polling.
- These Control-Time Events should be injected into the Event Stream, just like Market and Execution Events.

---

## Iteration 3 – Current Understanding

### Root Cause

The current model is semantically clean but overly dependent on external Event arrival for Execution Control progress.
As a result, pending Queue work may remain unevaluated in quiet periods even though it would become dispatchable under deterministic rate / inflight rules.

### Changes Proposed

- Introduce explicit Control-Time Events as part of Control Events.
- Use them only when Execution Control can derive a meaningful next processing deadline.
- Keep Queue Processing inside canonical Event processing.
- Do not introduce a separate runtime tick or autonomous Queue thread.

### Outcome

This would preserve:

- deterministic replay
- explicit causal history
- Backtesting / Live semantic parity
- strict separation of Risk from Execution Control

while allowing pending Queue work to be re-evaluated without depending solely on the occurrence of Market or Execution Events.

---

## Iteration 4 – Refined Control-Time Model

### Refined Understanding

The model is now refined further:

- Control-Time Events are explicit Control Events.
- They are canonical Events only once injected into the Event Stream.
- Their purpose is to create deterministic future re-evaluation opportunities for Execution Control when pending outbound work exists and a future relevant processing point is implied by current State and Configuration.
- They are not periodic ticks.
- They are not Venue-originated Events.
- They do not introduce an autonomous Queue loop.
- They do not move scheduling responsibility into the Risk Engine.

### Boundary Clarification

A further distinction is now necessary between two artifacts:

- A Control Scheduling Obligation is a non-canonical runtime-facing obligation derived by the Core when current State and Configuration imply a future relevant control-time re-evaluation point.
- A Control-Time Event is the later canonical Event injected into the Event Stream when that obligation is realized by the Runtime.

This keeps the Core purely event-driven while avoiding hidden wall-clock mutation or self-triggered Queue activity.

### Architectural Consequence

The Core does not autonomously wake itself up.

Instead:

1. the Core derives a future Control Scheduling Obligation;
2. the Runtime realizes that obligation externally to the Core;
3. the Runtime later injects a canonical Control-Time Event into the Event Stream;
4. the Core processes that Event like any other Event.

This means that the full processing model still applies:

- State is derived
- Strategy may evaluate the new State and emit Intents
- Risk evaluates admissibility
- Queue / Queue Processing perform Execution Control

### Outcome

This refinement preserves:

- deterministic replay
- explicit canonical history
- no hidden runtime tick
- no Queue self-autonomy
- no collapse of Risk into scheduling
- semantic parity between Backtesting and Live

while also removing unnecessary dependence on external Market or Execution Events for future Queue re-evaluation.

---

## Open Questions

- What is the exact canonical shape of a Control-Time Event?
- What is the exact shape of the non-canonical Control Scheduling Obligation derived by the Core?
- How should Control-Time Events be ordered relative to Market Events and Execution Events at identical timestamps?
- Should there be at most one outstanding Control Scheduling Obligation or Control-Time Event per Execution Control scope?
- How explicitly should the Runtime-side realization mechanism be named and documented?
