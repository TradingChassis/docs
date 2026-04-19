# Data Storage Stack

Part of: **Data Platform**

The Data Storage Stack provides, organizes, and exposes the persistent storage surfaces of the Infrastructure — the durable substrate on which raw datasets, canonical datasets, derived artifacts, execution records, quarantined datasets, and experiment outputs are retained and made accessible to other Stacks.

---

## Purpose

The Data Storage Stack exists to give the Infrastructure a stable, logically organized persistent layer. Multiple Stacks need to durably persist and later retrieve datasets and artifacts of different kinds. The Data Storage Stack provides the storage surfaces they write to and read from, maintains logical separation between different classes of stored data, and ensures that persisted content remains durably available.

The Data Storage Stack is a **persistent substrate and organization layer**, not a semantic processor. It does not validate data, normalize formats, or decide what becomes canonical. Those decisions are made by other Stacks — primarily the Data Quality Stack. The Data Storage Stack provides the durable surfaces on which those decisions are realized and their outcomes stored.

---

## Position in the Infrastructure

The Data Storage Stack is part of the **Data Platform**. Unlike the Data Recording and Data Quality Stacks, which occupy specific sequential positions in the data pipeline, the Data Storage Stack serves as a **persistent platform layer** that multiple Stacks interact with:

- The **Data Recording Stack** writes raw recorded datasets to Persistent Raw Storage.
- The **Data Quality Stack** reads from Persistent Raw Storage and writes promoted canonical datasets to Canonical Storage or quarantined datasets to Quarantine Storage.
- The **Backtesting Stack** reads canonical datasets from Canonical Storage and writes experiment outputs.
- The **Live Stack** writes execution records.
- The **Analysis Stack** reads canonical datasets, derived datasets, and experiment outputs.

The Data Storage Stack is **not** part of the **Core Runtime**. It does not participate in the Core Runtime Event Stream, State derivation, or any execution-path processing. It connects other Stacks through durable persistence, not through runtime causality.

---

## Main Responsibilities

The Data Storage Stack is responsible for:

- Providing persistent storage surfaces for the Data Recording Stack, Data Quality Stack, Backtesting Stack, Live Stack, and Analysis Stack.
- Maintaining logical separation between storage classes so that raw, canonical, derived, quarantined, experiment, and execution-record data remain distinct.
- Ensuring durable persistence — datasets and artifacts written to storage survive infrastructure restarts, failures, and operational changes.
- Ensuring durable access and retrieval — stored content is discoverable and readable by the Stacks that need it.
- Acting as the persistent integration layer across the Data Platform and its downstream consumers.

The Data Storage Stack encompasses six primary logical storage classes: **Persistent Raw Storage**, **Canonical Storage**, **Derived Storage**, **Quarantine Storage**, **Experiment / Artifact Storage**, and **Execution Record Storage**. Each class has a defined role, defined writers, and defined readers. Logical separation between classes is mandatory, even when they share physical infrastructure.

---

## Key Boundaries

**Not a semantic processor.** The Data Storage Stack stores what producing Stacks write and makes it retrievable for consuming Stacks. It does not evaluate, validate, transform, or decide the contents of what is persisted.

**Not the promotion decision point.** The Data Storage Stack provides Canonical Storage as a surface. The Data Quality Stack decides what is written to it. Canonical promotion is a Data Quality Stack responsibility — the Data Storage Stack provides the target, not the gate.

**Not the Core Runtime.** The Data Storage Stack operates entirely within the Data Platform domain. It stores artifacts produced by Core Runtime Stacks (execution records, experiment outputs) but does not participate in the runtime that produces them.

**Logical separation is mandatory.** Different storage classes may share physical infrastructure, but their logical boundaries must remain unambiguous. A dataset's storage-class membership — raw, canonical, quarantined, derived — is determined by which surface the producing Stack writes to and is stable once written.

---

## Relationship to Other Stacks

**Data Recording Stack.** Writes raw recorded datasets and Dataset Manifests to Persistent Raw Storage. The Data Storage Stack provides the durable surface; the Data Recording Stack produces the content.

**Data Quality Stack.** Reads raw datasets from Persistent Raw Storage for assessment. Writes promoted canonical datasets to Canonical Storage and quarantined datasets to Quarantine Storage. The Data Storage Stack provides the surfaces on both sides of the quality-promotion boundary.

**Backtesting Stack.** Reads canonical datasets from Canonical Storage as historical inputs for Strategy evaluation. Writes experiment outputs and execution records to Experiment / Artifact Storage and Execution Record Storage.

**Live Stack.** Writes execution records and operational artifacts to Execution Record Storage.

**Analysis Stack.** Reads from Canonical Storage, Derived Storage, Experiment / Artifact Storage, and Execution Record Storage. May write derived artifacts to Derived Storage.

---

## Why the Stack Matters

The Data Storage Stack is the persistent substrate that allows other Stacks to exchange, retain, and reuse durable datasets and artifacts without collapsing their logical boundaries. It is what makes decoupled, schedule-independent operation possible: a producing Stack writes to a storage surface; a consuming Stack reads from it later, without requiring the producer to be running.

The logical separation the Data Storage Stack enforces — between raw and canonical, between canonical and quarantined, between promoted and derived — is what allows each storage class to carry a defined meaning. Without this separation, the distinction between raw data and canonical data would erode, and the authority of Canonical Storage as the validated dataset layer would be undermined.

Detailed treatment of scope and role, interfaces, internal structure, operational behavior, and implementation considerations is provided in the companion documents for this Stack.
