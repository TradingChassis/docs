---
title: Derived Execution-Control Queue
updated: 2026-04-01
adr_status: Accepted
---

# ADR-004: Derived Execution-Control Queue

Status: {{ page.meta.adr_status }}  
Updated: {{ page.meta.updated }}

---

## Context

Strategy produces Intents — ephemeral commands expressing desired trading actions — during Event processing. A Strategy may generate multiple Intents targeting the same logical order key in rapid succession:

`NEW → REPLACE → REPLACE → CANCEL`

If every Intent were forwarded directly to the Venue Adapter as an outbound request, the System would produce unstable execution behavior:

- **Duplicate requests.** Multiple `NEW` commands for the same logical order key would produce duplicate submissions.
- **Replace and cancel storms.** Rapid successive modifications or cancellations would generate bursts of conflicting requests to the Venue.
- **Concurrent conflicts.** Overlapping inflight requests for the same order key would produce ambiguous Venue-side state — the System could not determine which request the Venue is processing.
- **Strategy-dependent stability.** Outbound execution stability would depend on how frequently and in what order Strategy generates commands, coupling execution behavior to Strategy implementation detail.

These problems require a mechanism between Risk evaluation and Venue dispatch that reconciles outbound work before transmission. The mechanism must:

1. Collapse redundant or superseded commands so that at most one effective outbound action exists per logical order key.
2. Prevent concurrent inflight requests for the same order key.
3. Regulate outbound transmission rate against Venue limits.
4. Operate deterministically within Event processing — not as an independent scheduler, background timer, or autonomous loop — so that all execution-control decisions are replayable.

The mechanism must also respect the separation between policy and execution control: Risk decides whether an Intent is **allowed**; the outbound mechanism decides **when and in what order** allowed work is dispatched. These are distinct responsibilities.

---

## Decision

The System implements an outbound Queue as **derived execution-control substate** within Execution State.

The Queue holds **effective reconciled allowed pending outbound work** — at most one command per logical order key — and is maintained deterministically from the Event Stream and Configuration. It is not a raw event history of Strategy output, not a second source of truth, and not a fourth top-level State domain.

### Core rules

1. **The Queue is derived.** Its contents are fully recomputable by replaying the Event Stream under the same Configuration and execution-control rules. The Queue does not hold authoritative state that is unavailable in the Event Stream.

2. **The Queue holds only allowed work.** An Intent enters execution-control substate only after Risk has classified it as **allowed**. Denied Intents do not enter the Queue. The Queue does not re-evaluate policy.

3. **At most one effective command per logical order key.** When a new allowed Intent targets the same logical order key as existing pending work, dominance rules determine which command is effective. Superseded commands are collapsed:

   `CANCEL > REPLACE > NEW`

   The result is that the Queue always holds the minimal, conflict-free set of pending outbound work.

4. **Inflight gating.** At most one outbound request per logical order key may be inflight at any time. A subsequent command for the same key is not dispatched until the prior request reaches a terminal outcome.

5. **Rate-limited dispatch.** Outbound transmission is constrained by rate rules from Configuration. Pending work that exceeds available outbound capacity remains in the Queue and is re-evaluated on the next processing step.

6. **Execution control runs within Event processing.** Queue updates, dominance, eligibility, inflight gating, scheduling, and dispatch selection are deterministic computations within the canonical Event-processing step. There is no separate runtime tick, background timer, or autonomous scheduler.

7. **Dominance, eligibility, and scheduling are internal derivations.** These computations do not produce separate Events in the canonical stream unless canonical history explicitly requires recording the outcome for replay or audit.

### Boundaries

**Risk → Queue boundary.** Risk evaluates each Intent for policy admissibility (allowed / denied). Risk does not schedule, order, gate, or rate-limit outbound work. Those are exclusively execution-control responsibilities implemented by Queue and Queue Processing.

**Queue → Order lifecycle boundary.** Queue residency is pre-submission. No Order exists in Execution State during Queue residency. An Order comes into existence at submission — when Queue Processing selects work for dispatch and the Venue Adapter transmits the outbound request. After submission, Order state evolves through Execution Events; it is not governed by Queue mechanics.

---

## Consequences

**Strategy is decoupled from outbound transmission.** Strategy produces Intents at whatever frequency its logic requires. The Queue absorbs rapid successive commands and collapses them to the minimal effective outbound work. Execution stability does not depend on Strategy command frequency or ordering.

**Outbound traffic is structurally stable.** Dominance prevents duplicate and redundant requests. Inflight gating prevents concurrent conflicts for the same order key. Rate limiting prevents bursts that exceed Venue constraints. These properties hold by construction, not by Strategy discipline.

**All execution-control decisions are deterministic and replayable.** Because the Queue is derived from Event Stream + Configuration and all execution-control computations run within Event processing, replay of the same stream under the same Configuration produces identical dispatch decisions at every Processing Order position. Backtesting and Live evaluate the same execution-control logic.

**The Queue is not an independent truth layer.** It does not accumulate a history of Strategy emissions. It does not hold state that the Event Stream cannot reproduce. Components that need execution-control information read it as a projection of derived State, not as an authoritative parallel store.

**Superseded Intents are collapsed, not silently lost.** Dominance collapses intermediate commands before dispatch — this is the intended behavior, not data loss. Where an intent-processing outcome must be part of canonical history (for replay or audit), it is recorded as an Intent-related Event per the intent-visibility rules, not by retaining raw Strategy output in the Queue.

---

## Trade-offs

**The Queue does not preserve intermediate Strategy intent history.** By design, only the effective command per logical order key is retained. A sequence `NEW → REPLACE → REPLACE → CANCEL` produces `CANCEL` in execution-control substate; the intermediate replaces are not individually visible in the Queue. Where visibility of superseded commands is needed for replay or audit, the canonical mechanism is Intent-related Events recorded on the stream when required — not Queue-level retention.

**Execution-control rules add derivation complexity.** Dominance, inflight gating, rate-limit evaluation, and eligibility are deterministic functions applied at every relevant processing step. This is more complex than direct pass-through, but direct pass-through cannot provide the stability and determinism guarantees the System requires.

---

## Summary

The System uses a derived execution-control Queue that reconciles allowed pending outbound work to at most one effective command per logical order key. The Queue is execution-control substate within Execution State — derived, not independently authoritative. Dominance, inflight gating, and rate-limited dispatch ensure stable, deterministic outbound behavior. All execution-control computation runs within Event processing; there is no separate tick. Queue residency is pre-submission; Orders begin at dispatch.
