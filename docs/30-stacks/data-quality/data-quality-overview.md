# Data Quality Stack

Part of: **Data Platform**

The Data Quality Stack discovers recorded raw datasets, assesses and validates their quality, transforms them into canonical form, and promotes only eligible datasets into Canonical Storage. It is the gatekeeper that controls the boundary between raw recorded data and canonical data.

---

## Purpose

The Data Quality Stack exists to ensure that only datasets meeting defined integrity, completeness, and consistency standards become canonical Research inputs. It takes raw recorded datasets — produced by the Data Recording Stack and persisted in Persistent Raw Storage — and determines whether each dataset is suitable for canonical promotion.

The Stack performs four roles:

- **Validator.** Assesses raw datasets against structural, completeness, and consistency constraints — including market-structure plausibility checks where order book and trade data are evaluated together for coherence.
- **Transformer.** Normalizes raw Venue-specific formats into the canonical form required by Canonical Storage.
- **Promoter.** Writes validated, normalized datasets to Canonical Storage with promotion metadata.
- **Gatekeeper.** Prevents datasets that fail quality assessment from entering Canonical Storage. Unsuitable datasets are quarantined or rejected — they do not become canonical.

Canonical datasets arise **only** through the Data Quality Stack. There is no alternative path into Canonical Storage.

---

## Position in the Infrastructure

The Data Quality Stack is part of the Data Platform, positioned between the Data Recording Stack and Canonical Storage:

`Data Recording ➝ Data Quality ➝ Data Storage (Canonical Storage)`

It sits downstream of the Data Recording Stack and upstream of the Backtesting Stack and the Analysis Stack. It controls the promotion boundary: raw recorded datasets on one side, canonical validated datasets on the other.

The Data Quality Stack is **not** part of the **Core Runtime**. It does not participate in the Core Runtime Event Stream, State derivation, Strategy evaluation, Risk, Execution Control, or any execution-path processing.

---

## Main Responsibilities

The Data Quality Stack is responsible for:

- Discovering recorded raw datasets via **Dataset Manifests** published by the Data Recording Stack.
- Loading raw datasets from **Persistent Raw Storage** for assessment.
- Validating dataset integrity, completeness, and consistency — including market-structure plausibility.
- Normalizing eligible datasets from Venue-specific formats into canonical form.
- Promoting eligible datasets into **Canonical Storage** with promotion metadata.
- Preventing unsuitable datasets from entering Canonical Storage — routing them to quarantine or rejection.
- Producing validation metadata and promotion metadata that document assessment and promotion outcomes.

Datasets progress through states such as Discovered, In Validation, Validated, Normalized, Promotable, Promoted, Quarantined, Rejected, and Incomplete. Not every discovered dataset reaches promotion — the quality-assessment pipeline determines the outcome.

---

## Key Boundaries

**Not raw data capture.** The Data Quality Stack does not connect to Venue feeds, record raw market data, or manage recording infrastructure. Those are Data Recording Stack responsibilities. The Data Quality Stack consumes finished raw datasets from Persistent Raw Storage.

**Not Canonical Storage governance.** The Data Quality Stack writes promoted datasets to Canonical Storage but does not manage the storage layer itself. Organization, retention, and access management of Canonical Storage are Data Storage Stack responsibilities.

**Not the Core Runtime.** The Data Quality Stack processes raw market datasets. It does not participate in or define the Core Runtime Event Stream, State derivation, Processing Order, or any execution-path semantics.

**Controlled promotion, not automatic storage.** A dataset becomes canonical only when the Data Quality Stack explicitly promotes it after successful validation, consistency assessment, and normalization. Dataset existence in Persistent Raw Storage does not imply canonical status.

---

## Relationship to Other Stacks

**Data Recording Stack.** The sole upstream source. The relationship is **asynchronous and discovery-based**: the Data Recording Stack persists raw datasets and publishes Dataset Manifests; the Data Quality Stack discovers and consumes them on its own schedule. There is no synchronous handoff or active push.

**Data Storage Stack.** Receives promoted canonical datasets written by the Data Quality Stack. Manages Canonical Storage as a layer — its organization, retention, and downstream access. The Data Quality Stack writes to Canonical Storage; the Data Storage Stack governs it.

**Backtesting and Analysis Stacks.** Indirect downstream dependents. Both consume canonical datasets from Canonical Storage. Every canonical dataset they use has passed through the Data Quality Stack's validation and promotion logic. The rigor of quality assessment here constrains the reliability of all downstream Research.

---

## Why the Stack Matters

The Data Quality Stack is the point at which the Infrastructure decides which recorded datasets are reliable and suitable enough to serve as canonical Research inputs. Every canonical dataset — every Backtesting input, every Analysis baseline — passes through this Stack's assessment and promotion logic before it becomes authoritative.

If the Data Quality Stack is weak or permissive, Research and Backtesting may operate on incomplete, inconsistent, or structurally defective data without detection. The gatekeeper role is not auxiliary infrastructure — it is the mechanism that gives Canonical Storage its authority as the validated dataset layer for the Infrastructure.

Detailed treatment of scope and role, interfaces, internal structure, operational behavior, and implementation considerations is provided in the companion documents for this Stack.
