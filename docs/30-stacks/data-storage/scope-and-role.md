# Scope and Role

The Data Storage Stack provides, organizes, and exposes the persistent storage surfaces of the System — the durable substrate on which raw datasets, canonical datasets, derived artifacts, execution records, quarantined datasets, and experiment outputs are retained and made accessible to other Stacks.

---

## Purpose

The Data Storage Stack exists to give the System a stable, logically organized persistent layer. Multiple Stacks — Data Recording, Data Quality, Backtesting, Live, Analysis — need to durably persist and later retrieve datasets and artifacts. The Data Storage Stack provides the storage surfaces they write to and read from, maintains logical separation between different classes of stored data, and ensures that persisted content remains durably available for downstream use.

The Data Storage Stack is a **persistent substrate and organization layer**, not a semantic processor. It does not decide what data is valid, what becomes canonical, or what is quarantined. Those are decisions made by other Stacks — primarily the Data Quality Stack. The Data Storage Stack provides the durable surfaces on which those decisions are realized and their outcomes stored.

---

## Position in the System

The Data Storage Stack is part of the **Data Platform**. Unlike the Data Recording and Data Quality Stacks, which occupy specific positions in a sequential pipeline, the Data Storage Stack serves as a **persistent platform layer** that multiple Stacks interact with:

- The **Data Recording Stack** writes raw recorded datasets to Persistent Raw Storage.
- The **Data Quality Stack** reads from Persistent Raw Storage and writes promoted canonical datasets to Canonical Storage.
- The **Backtesting Stack** reads canonical datasets from Canonical Storage and writes experiment outputs and execution records.
- The **Live Stack** writes execution records and operational artifacts.
- The **Analysis Stack** reads canonical datasets, derived datasets, and experiment outputs.

The Data Storage Stack is **not** part of the **Core Runtime**. It does not participate in the Core Runtime Event Stream, State derivation, or any execution-path processing. It connects other Stacks through durable persistence, not through runtime causality.

### Logical storage classes

The Data Storage Stack conceptually encompasses the following primary storage classes:

| Storage class | What it holds |
| ------------- | ------------- |
| **Persistent Raw Storage** | Raw recorded datasets as produced by the Data Recording Stack |
| **Canonical Storage** | Validated, normalized datasets promoted by the Data Quality Stack |
| **Derived Storage** | Datasets and artifacts derived from canonical or execution data |
| **Quarantine Storage** | Datasets excluded from canonical promotion, held for review |
| **Experiment / Artifact Storage** | Backtesting outputs, experiment results, evaluation artifacts |
| **Execution Record Storage** | Execution records and operational artifacts from Live and Backtesting |

These storage classes are **logically distinct**, regardless of whether they share physical infrastructure. A single object store or storage system may host multiple classes, but their logical boundaries — what each class contains, who writes to it, who reads from it — must remain explicit and stable.

---

## Core Responsibilities

The Data Storage Stack is responsible for:

- Providing persistent storage surfaces for the Data Recording Stack, Data Quality Stack, Backtesting Stack, Live Stack, and Analysis Stack.
- Maintaining logical separation between storage classes so that raw, canonical, derived, quarantined, experiment, and execution-record data remain distinct.
- Enabling durable persistence — ensuring that datasets and artifacts written to storage survive infrastructure restarts, failures, and operational changes.
- Enabling durable access and retrieval — ensuring that stored datasets and artifacts are discoverable and readable by the Stacks that need them.
- Acting as the persistent integration layer across the Data Platform and its downstream consumers: Stacks exchange durable data through storage surfaces rather than through transient runtime coupling.
- Providing high-level storage organization principles — naming, partitioning, and classification conventions that keep stored content navigable and its logical class identifiable.

---

## Explicit Non-Responsibilities

The Data Storage Stack is **not** responsible for:

- **Raw Venue data capture.** Connecting to Venue feeds, recording raw messages, and annotating timing metadata are Data Recording Stack responsibilities.
- **Dataset validation or normalization.** Assessing data quality, applying consistency checks, and transforming raw formats into canonical form are Data Quality Stack responsibilities.
- **Canonical promotion decisions.** Deciding whether a dataset meets the criteria for canonical promotion is the Data Quality Stack's role. The Data Storage Stack provides Canonical Storage as a surface; it does not control what is written to it.
- **Quarantine decisions.** Deciding whether a dataset should be quarantined is the Data Quality Stack's role. The Data Storage Stack provides Quarantine Storage; it does not evaluate dataset suitability.
- **Core Runtime Event semantics.** The canonical Event Stream, Processing Order, State derivation, and all execution-path processing belong to the Core Runtime. The Data Storage Stack does not participate in or define these.
- **Strategy, Risk, Queue, or Order semantics.** These are Core Runtime concerns with no intersection to persistent storage organization.
- **Backtesting execution or Research interpretation.** The Data Storage Stack stores experiment outputs and canonical datasets; it does not execute experiments or interpret results.

---

## Why the Stack Matters

The Data Storage Stack is the persistent substrate that allows other Stacks to exchange, retain, and reuse durable datasets and artifacts without collapsing their logical boundaries. Without it, the System would not have a stable basis for:

- retaining raw recording outputs across recording sessions;
- maintaining Canonical Storage as a validated dataset layer distinct from raw data;
- preserving quarantined datasets separately from canonical ones;
- storing experiment results and execution records for later analysis;
- enabling any Stack to access persisted data produced by another Stack without transient runtime coupling.

The logical separation that the Data Storage Stack enforces — between raw and canonical, between canonical and derived, between promoted and quarantined — is what allows each storage class to carry a defined meaning. If these boundaries were not maintained, the distinction between raw data and canonical data would erode, and the authority of Canonical Storage as the validated dataset layer would be undermined.
