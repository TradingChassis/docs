# Terminology

This document is the **canonical semantic source** for core Infrastructure terms.

It defines what concepts mean and how they relate. It does **not** specify stack layout, operational procedures, or full end-to-end runtime walkthroughs; those belong in architecture and operations documentation.

Capitalized terms denote formally defined concepts. Other words are descriptive only.

All definitions below are **normative** for interpretation of the rest of the documentation.

---

## Reading Guide

Read sections **in order**. Each section assumes the ones before it.

Dependency chain:

**Event and ordering ➝ State and derivation ➝ Intent ➝ Risk ➝ Queue and Execution Control ➝ Order**

Supporting terms at the end assume the core definitions.

---

## Event

An **Event** is an immutable record of an occurrence that the Infrastructure is required to treat as input to processing.

**Normative rules:**

1. **Events are the only source of State transitions.** No State change occurs except as the result of processing an Event under Configuration.
2. Events are applied strictly in **Processing Order** (see [Processing Order](#processing-order)).
3. After creation, an Event must not be altered.

The Infrastructure is fully event-driven: all evolution of derived State is accounted for by the Event Stream and Configuration.

---

## Event Stream

The **Event Stream** is the canonical, totally ordered sequence of Events the Runtime applies.

**Normative rules:**

1. The Event Stream determines **Processing Order** for State derivation.
2. **State is fully derived from Event Stream + Configuration.** No other input is authoritative for State.
3. Under identical Event Stream and identical Configuration, derived State must be identical (**determinism**; see [Determinism](#determinism)).

---

## Event Time

**Event Time** is the time at which an occurrence took place in an external or market context (e.g. Venue timestamp, feed timestamp).

Event Time preserves external semantics. It **does not** define internal causality or the order in which Events affect State.

---

## Capture Time

**Capture Time** is the time at which a Data Recording Stack Component observes or receives a raw Venue message.

Capture Time preserves internal semantics. It **does not** define internal causality or the order in which Events affect State.

---

## Processing Order

[Processing Order](../20-concepts/time-model.md#processing-order) is the strict internal order in which Events are processed and affect State.

It determines:

- causal ordering of State Transitions;
- deterministic replay;
- equivalence of Backtesting and Live **with respect to the same Event Stream and Configuration**.

Processing Order is independent of Event Time and Capture Time.

---

## Configuration

**Configuration** is the explicit, versioned set of rules, parameters, and ordering assumptions under which the Infrastructure processes the **Event Stream** and derives **State**.

**Normative rules:**

1. **Configuration** is part of the canonical input to the Infrastructure.
2. Derived **State** depends on **Event Stream + Configuration**.
3. **Configuration** defines processing rules and parameters, but is not itself **Processing Order**.
4. **Configuration** must not change silently during processing.
5. Any **Configuration** change that affects derived **State** or dispatch decisions must be represented as an explicit, versioned input consistent with canonical history.
6. **Configuration** is not hidden mutable runtime state and must not bypass **Event processing** semantics.

---

## State

**State** is the complete derived condition of the Infrastructure.

**Normative rules:**

1. **State is fully derived from Event Stream + Configuration:**

   `State = f(Event Stream, Configuration)`

2. State is **not** an independent source of truth. The Event Stream (with Configuration) is canonical for history and reconstruction.
3. State is **reconstructible** by replaying the Event Stream under the same Configuration.

---

## State Domains

Derived State is grouped into **three** conceptual domains:

| Domain | Role |
| ------ | ---- |
| **Market State** | Observed market conditions (e.g. order book, trades) derived from market-related Events. |
| **Execution State** | Trading and execution projection: Orders, fills, positions, balances, and related status derived from execution-related Events. |
| **Control State** | Runtime configuration, control, and operational flags derived from infrastructure and control Events. |

These three domains together constitute to the full State.

**Normative rule:** There is **no** separate top-level State domain for the Queue. Queue-related data is **derived Execution Control substate** and is part of the derivation of Execution State (or of the single derived State that includes Execution Control), not a fourth peer domain.

---

## State Transition

A **State Transition** is a deterministic change to derived State caused by processing **one** Event.

**Normative rules:**

1. State Transitions occur strictly in Processing Order.
2. No State Transition occurs without a corresponding processed Event.
3. There are no spontaneous, wall-clock-driven, or hidden State changes outside Event processing.

---

## Determinism

The Infrastructure is **deterministic** if and only if: for identical Event Stream and identical Configuration, all derived State (including Execution Control substate) is identical at every Processing Order position.

Derived behavior must not depend on:

- wall-clock time;
- OS scheduler timing;
- concurrency or thread interleaving;
- mutable state outside what is defined by Event Stream + Configuration.

---

## Intent

An **Intent** is a **command**: a Strategy’s desired trading action (e.g. create, replace, cancel), expressed in internal form, for a point in processing.

**Normative rules:**

1. An Intent is **not** an Event.
2. An Intent is **not** persistent. It exists only as transient input within the processing step in which Strategy evaluation occurs.
3. An Intent does not, by itself, change State. Effects appear only through subsequent **State derivation** after Events that the Infrastructure records when canonical history requires them (see [Intent visibility](#intent-visibility)).

Strategies produce Intents; they do not send orders to Venues or perform Execution Control.

---

## Intent visibility

**Intent processing becomes visible through Events** when the canonical history must record an outcome that affects replay or audit (e.g. risk decision, dispatch to a Venue, terminal failure that must not be re-inferred).

**Normative rules:**

1. **Internal deterministic derivations**—including dominance reconciliation, eligibility, and scheduling order—are **not** separate Events **unless** explicitly required for canonical history. They are computed during Event processing as pure functions of current State and Configuration.
2. **Rate limits and wakeups** must be modeled **without introducing unnecessary extra Event types.** Where a delay or limit is relevant to determinism, it is expressed through rules that tie the next permissible action to Processing Order and to State derived from Events already in the Stream (e.g. token refills, next-send indices), not through ad hoc parallel event taxonomies.

---

## Risk Engine

The **Risk Engine** is the **policy layer** only.

**Normative rules:**

1. It answers whether a given Intent is **allowed** or **denied** under policy (limits, kill-switch, parameter validity, etc.).
2. It does **not** schedule transmission, order Queue contents, apply dominance, enforce rate limits for send timing, or manage inflight gating. That is **Execution Control** (see [Execution Control](#execution-control)).
3. Allowed/denied outcomes that must appear in canonical history are reflected via Events and thence in derived State; the Risk Engine does not hold parallel authoritative state.

---

## Queue

The **Queue** is **derived state**: the data structure (or equivalent projection) that holds **allowed** outbound Intents that are not yet fully completed from an Execution Control perspective, after reconciliation rules (e.g. dominance) are applied.

**Normative rules:**

1. The Queue is **not** a second source of truth. It is recomputable from Event Stream + Configuration together with the deterministic Execution Control rules.
2. The Queue is **Execution Control substate**, not a fourth top-level State domain (see [State domains](#state-domains)).
3. The Queue stores **effective** pending outbound work (e.g. at most one reconciled command per logical order key), not a redundant copy of every raw Strategy emission.

---

## Queue Processing

**Queue Processing** is the deterministic computation that, upon processing an Event (and any Strategy/Risk steps defined for that processing step), decides **whether** and **in what order** reconciled allowed Intents may be presented to the Venue Adapter, subject to inflight rules, rate rules, and ordering rules.

**Normative rules:**

1. Queue Processing implements **Execution Control** only, not policy. It assumes Intents are already risk-allowed unless the model explicitly re-validates against derived limits that are themselves State.
2. There is **no separate runtime tick.** Queue Processing is **part of deterministic Event processing**—the same sequential application of Events that advances Market, Execution, and Infrastructure domains also advances Execution Control substate and dispatch decisions.
3. Dominance, eligibility, and scheduling are **internal deterministic derivations** within that processing unless canonical history explicitly requires additional Events ([Intent visibility](#intent-visibility)).

---

## Execution Control

**Execution Control** is the collective responsibility of **Queue** and **Queue Processing**: reconciliation (e.g. dominance), eligibility, ordering, inflight gating, and rate-compliant timing of outbound requests, **without** re-deciding policy.

Execution Control is strictly separated from the **Risk Engine** (policy). Strategy expresses desire; Risk permits or forbids; Execution Control schedules and sends among **permitted** work.

---

## Order

An **Order** is a **derived entity** in **Execution State**.

**Normative rules:**

1. Orders exist only as projections maintained while processing the Event Stream; they are not a separate source of truth.
2. The **Order lifecycle begins at submission** with state **Submitted**—the stage at which the Infrastructure represents an outbound request as submitted and awaiting Venue acknowledgement or further execution Events. Prior stages are Intent and Execution Control derivation, not a persisted Order entity.
3. Orders evolve only through Events (e.g. acknowledgements, fills, cancellations, rejections).

---

## Order lifecycle

The **Order lifecycle** is the defined set of states and transitions an Order may occupy in Execution State.

**Normative rules:**

1. It is fully driven by Events and Processing Order.
2. It is deterministically reconstructible from the Event Stream and Configuration.
3. The same conceptual lifecycle applies across Backtesting and Live when Event Streams are comparable.

---

## Infrastructure principle

**All** derived State—including Market, Execution (including Orders and Execution Control substate), and Infrastructure domains—is produced **only** by processing the Event Stream under Configuration.

No Component may mutate State outside this mechanism.

---

> The following terms are supporting terminology and rely on the core definitions above. They do not redefine Event, State, Intent, Order, Queue, Risk, or Execution Control.

## Infrastructure

The **Infrastructure** is the trading infrastructure described in this documentation: Core Runtime, data platform, Backtesting and Live Runtimes, and analysis and monitoring, as documented elsewhere.

It **excludes** proprietary Strategy implementations (except as interfaces), portfolio allocation, and external regulatory frameworks unless explicitly documented.

## Core

The **Core** is the deterministic engine that applies the Event Stream, derives State, invokes Strategy, applies Risk, and runs Execution Control (Queue and Queue Processing) as part of Event processing.

The Core behaves the same in concept across Backtesting and Live; surrounding Stacks and Adapters differ.

## Strategy

A **Strategy** is a Component that reads derived State and emits **Intents**.

Strategies do not interact directly with Venues, Queues, or Risk outcomes except through the defined processing pipeline. All outbound trading effects pass through Risk and Execution Control.

## Runtime

A **Runtime** is an environment in which the Core processes Events (e.g. Backtesting Runtime, Live Runtime). It supplies canonical input to the Core and realizes external-to-Core obligations required to preserve semantic parity across Backtesting and Live.

This includes market/Execution input delivery and, where required, realization of control-time scheduling obligations as explicit Control Events.

Runtimes share the same **semantic** model; they differ in data sources, Venue implementation, and infrastructure.

## Control Scheduling Obligation

A **Control Scheduling Obligation** is a non-canonical, runtime-facing obligation derived by the Core when current **State** and **Configuration** together imply a future relevant control-time re-evaluation point—specifically, when pending allowed outbound work exists and a future moment is implied at which **Execution Control** conditions may change.

**Normative rules:**

1. A Control Scheduling Obligation is **not** a canonical **Event** and does not enter the **Event Stream**. It produces no **State Transition**.
2. It is derived as a pure function of current **State** and **Configuration**. The Core does not perform wall-clock-driven mutations to derive it.
3. It is **not** a periodic tick, a background loop, or a polling signal. It arises only when specific conditions in **State + Configuration** imply a future deadline-style re-evaluation point.
4. The obligation is a signal to the **Runtime**. The canonical record appears only when the **Runtime** injects the corresponding **Control-Time Event** into the **Event Stream**.

---

## Control-Time Event

A **Control-Time Event** is the canonical **Event** injected into the **Event Stream** by the **Runtime** when a **Control Scheduling Obligation** is realized.

**Normative rules:**

1. A Control-Time Event **is** a canonical **Event**. Once injected, it is processed identically to any other **Event**: **State** is derived, **Strategy** may read the new **State** and emit **Intents**, **Risk** evaluates admissibility of any resulting **Intents**, and **Queue Processing** performs **Execution Control**.
2. A Control-Time Event does not originate from a **Venue** or **Venue Adapter**. It is not a **Venue** Event and is not part of **Venue** semantics.
3. Control-Time Events are **sparse and deadline-style**: each is injected only when a **Control Scheduling Obligation** has been derived from specific **State + Configuration** conditions, not periodically or on an independent timer.
4. A Control-Time Event belongs to **Control State** semantics. It does not alter the role of the **Risk Engine** (which remains policy only) and does not make **Queue Processing** autonomous.
5. Determinism is preserved: identical **Event Stream** (including all injected Control-Time Events) combined with identical **Configuration** produces identical derived **State** at every **Processing Order** position.

---

## Backtesting

**Backtesting** is deterministic Strategy evaluation using historical inputs and a simulated Venue, producing an Event Stream and derived State under the same rules as Live when inputs are equivalent.

## Live

**Live** is real-time Strategy Execution against a real Venue under the same Core semantics as Backtesting.

## Execution

**Execution** is the activity of turning permitted, scheduled outbound actions into Venue requests and ingesting Venue responses as Events.

**Execution** in this sense spans Venue Adapter I/O; **Execution Control** spans Queue and Queue Processing.

## Venue

A **Venue** is an external or simulated execution environment that receives requests and emits market and execution-related Events.

## Venue Adapter

A **Venue Adapter** maps internal outbound actions to Venue protocols and maps Venue messages to Events. It does not define policy or replace Queue Processing.

## Stack

A **Stack** is a documented subinfrastructure grouping (e.g. Data Recording, Data Storage, Live). Stacks are organizational; core semantics in this document are independent of Stack boundaries.

## Component

A **Component** is a deployable or logical unit within a Stack.

## Research

**Research** is the use of the Infrastructure for Strategy development and evaluation (typically Backtesting, Analysis, Canonical Storage). It is a usage context, not a Core semantic primitive.

## Analysis

**Analysis** is read-oriented inspection of datasets and results produced by the Infrastructure.

## Canonical Storage

**Canonical Storage** is the specialized, persistent, and authoritative storage layer for datasets (validated, promoted data) used across Stacks. It is not the Runtime Event Stream; replay and State reconstruction are defined from Events, not from storage alone.

## Flow

A **Flow** is a narrative or diagram of how information moves between parts of the Infrastructure. Flows are explanatory; **Processing Order** and the Event Stream are normative for causality.

## Pipeline

A **Pipeline** is a staged description of processing. For example, the **intent pipeline** is a documented staging of

**Strategy ➝ Risk ➝ Queue ➝ Adapter**.

Stages map to the Core semantics defined in this document, not to separate sources of truth.
