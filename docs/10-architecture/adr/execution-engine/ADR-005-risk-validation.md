---
title: Mandatory Risk Validation Before Execution Control
updated: 2026-04-01
adr_status: Accepted
---

# ADR-005: Mandatory Risk Validation Before Execution Control

Status: {{ page.meta.adr_status }}  
Updated: {{ page.meta.updated }}

---

## Context

Strategy produces Intents — ephemeral commands expressing desired trading actions — during Event processing. These commands, if eventually dispatched, become outbound requests that directly affect the infrastructure's market exposure.

Without a centralized policy gate, the following problems arise:

- **Uncontrolled exposure.** Strategy logic could produce commands that exceed position limits, violate risk parameters, or create dangerous market exposure. If no mandatory validation layer exists, the path from Strategy decision to Venue submission has no safety constraint.
- **Inconsistent enforcement.** If policy validation is distributed across components — some checks in Strategy, some in Queue Processing, some in the Venue Adapter — the enforcement surface is fragmented. Rules may be applied inconsistently, duplicated with divergent logic, or silently omitted.
- **Strategy-Venue coupling.** If Strategy can interact with Venues or Venue Adapters directly, bypassing policy evaluation, then safety depends entirely on Strategy implementation discipline. This is not an acceptable architectural guarantee.
- **Policy and Execution Control conflation.** Policy admissibility ("is this Intent allowed?") and Execution Control scheduling ("when should this allowed work be dispatched?") are fundamentally different questions. Mixing them in a single component makes both harder to reason about, audit, and evolve independently.

The infrastructure requires an architectural boundary that separates **policy admissibility** from **Execution Control scheduling**, enforces policy on every Intent before it can proceed toward Venue interaction, and prevents any component from bypassing that enforcement.

---

## Decision

**All Intents must pass through the Risk Engine before entering Execution Control substate (the Queue). The Risk Engine decides admissibility only: allowed or denied.**

### Core rules

1. **Risk is mandatory and cannot be bypassed.** No path exists from Strategy to Queue, Venue Adapter, or Venue that does not pass through the Risk Engine. No component may cause an outbound execution action without a prior **allowed** decision from Risk for the corresponding Intent.

2. **Risk decides admissibility only.** The Risk Engine evaluates each Intent against policy — position and exposure limits, kill-switch state, parameter validity, and other admissibility rules defined in Configuration — and produces a binary outcome:

   - **Allowed** — the Intent may proceed to Execution Control substate (Queue) for scheduling and dispatch.
   - **Denied** — the Intent does not proceed. Risk denial is final within the current processing step.

3. **Risk does not decide execution timing.** The Risk Engine does not schedule outbound transmission, choose send timing, apply rate limits, enforce inflight gating, manage queue ordering, or determine dispatch priority. Those responsibilities belong exclusively to Queue Processing (execution control). "Send later" is not a Risk outcome; delay is an Execution Control decision applied after an Intent is allowed.

4. **The pipeline sequence is fixed.** The outbound processing chain follows a strict order:

   `Strategy ➝ Risk Engine ➝ Queue ➝ Queue Processing ➝ Venue Adapter ➝ Venue`

   Strategy produces Intents. Risk evaluates admissibility. The Queue holds allowed pending work. Queue Processing evaluates sendability and selects work for dispatch. The Venue Adapter translates and transmits. The Venue responds with Execution Events that re-enter the Event Stream.

5. **Risk outcomes that must appear in canonical history are recorded as Events.** Where a policy decision must be part of the replayable record (for deterministic replay or audit), the outcome is reflected through an Intent-related Event on the canonical Event Stream. Risk does not maintain parallel authoritative state.

### What Risk evaluates

Risk evaluates **policy admissibility** — whether the infrastructure's rules permit this Intent to proceed. Typical policy checks include:

- Position and exposure limits
- Trading enable/disable controls (kill-switch)
- Order parameter validity (price, quantity, instrument constraints)
- Strategy-level or account-level policy constraints

### What Risk does not evaluate

Risk does not evaluate any of the following — these are Execution Control concerns handled by Queue Processing:

- Rate-limit compliance
- Inflight request restrictions
- Dispatch ordering or priority
- Send timing relative to Venue constraints
- Dominance or reconciliation among pending commands

---

## Consequences

**Strategy cannot reach Venues without passing through policy validation.** The pipeline sequence enforces this structurally. Strategy emits Intents; it does not interact with Venues, Venue Adapters, or Execution Control substate directly. Every outbound action is evaluated by Risk before it can enter the Execution Control path.

**Policy and execution control are independently auditable and evolvable.** Because Risk decides only admissibility and Queue Processing decides only scheduling and dispatch, the two concerns can be reviewed, tested, and modified independently. A change to rate-limit rules does not affect Risk policy logic; a change to exposure limits does not affect dispatch scheduling.

**Risk enforcement is consistent across all Strategies and all Runtimes.** The same Risk Engine evaluates Intents in Backtesting and Live under the same Configuration. This consistency is a direct consequence of the mandatory gate: there is no alternative path that some Strategies or some Runtimes might use to bypass policy.

**Risk evaluation is deterministic and replayable.** Risk decisions are functions of the current Intent, derived State, and Configuration. Because Risk runs within Event processing and its outcomes (when required for canonical history) are recorded as Events, replay of the same Event Stream under the same Configuration reproduces the same policy decisions at every Processing Order position.

**Denied Intents do not enter Execution Control substate.** A denied Intent produces no Queue entry, no pending outbound work, and no Order. The denial is terminal for that Intent's processing arc. This boundary is clean: the Queue holds only work that Risk has allowed.

---

## Trade-offs

**Every Intent incurs a Risk evaluation.** Policy evaluation runs on every Intent, including those that will ultimately be superseded by dominance in Execution Control substate before dispatch. This is the cost of mandatory enforcement: the infrastructure evaluates admissibility before it knows whether a command will be collapsed. The alternative — evaluating policy only at dispatch time — would allow inadmissible commands to reside in Execution Control substate, blurring the boundary between policy and execution control.

**Risk cannot express "send later."** The binary allowed/denied model means Risk has no mechanism to delay an Intent. If a condition is temporary (e.g., a rate-limit window), Risk must either allow the Intent (letting execution control handle timing) or deny it. This is intentional: mixing delay semantics into Risk would conflate policy with execution control and make both harder to reason about.

---

## Summary

The Risk Engine is a mandatory policy gate: every Intent passes through it before entering Execution Control substate. Risk decides admissibility only — allowed or denied — and does not manage execution timing, dispatch scheduling, rate limits, or inflight gating. Those responsibilities belong exclusively to Queue Processing. This separation ensures that policy enforcement is centralized, consistent across Runtimes, and independently auditable from Execution Control logic.
