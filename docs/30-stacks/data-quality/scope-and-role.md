# Scope and Role

The Data Quality Stack discovers recorded raw datasets, assesses and validates their quality, transforms them into canonical form, and promotes only eligible datasets into Canonical Storage. It is the gatekeeper that controls the boundary between raw recorded data and canonical data.

---

## Purpose

The Data Quality Stack exists to ensure that only datasets meeting defined integrity, completeness, and consistency standards enter Canonical Storage. It takes raw recorded datasets — produced by the Data Recording Stack and persisted in Persistent Raw Storage — and determines whether each dataset is suitable for canonical promotion.

This Stack performs four distinct roles:

- **Validator.** Assesses raw datasets against structural integrity, completeness, and consistency constraints.
- **Transformer.** Normalizes raw Venue-specific data formats into the canonical form required by Canonical Storage.
- **Promoter.** Writes validated, normalized datasets to Canonical Storage with appropriate promotion metadata.
- **Gatekeeper.** Prevents datasets that fail validation from entering Canonical Storage. Unsuitable datasets are quarantined or rejected — they do not become canonical.

Canonical datasets arise **only** through the Data Quality Stack. There is no alternative path by which raw recorded data enters Canonical Storage.

---

## Position in the infrastructure

The Data Quality Stack is part of the **Data Platform**, positioned between the Data Recording Stack and Canonical Storage:

`Data Recording ➝ Data Quality ➝ Data Storage (Canonical Storage)`

It sits downstream of the Data Recording Stack and upstream of Canonical Storage, the Backtesting Stack, and the Analysis Stack. It controls the promotion boundary: raw recorded datasets on one side, canonical validated datasets on the other.

The Data Quality Stack is **not** part of the **Core Runtime**. It does not participate in the Core Runtime Event Stream, State derivation, Strategy evaluation, Risk, Execution Control, or any execution-path processing. The Data Quality Stack and the Core Runtime operate in separate architectural domains.

### Upstream relationship

The Data Quality Stack discovers recorded raw datasets **asynchronously** from the outputs of the Data Recording Stack. It does not receive an active handoff, synchronous notification, or push delivery. Instead, it locates **Dataset Manifests** — metadata artifacts published by the Data Recording Stack — and loads the corresponding raw datasets from **Persistent Raw Storage** for assessment.

### Downstream relationship

Datasets that pass validation and normalization are promoted to **Canonical Storage**, where they become available to downstream consumers — primarily the Backtesting Stack and the Analysis Stack. The Data Quality Stack's promotion decision is what makes a dataset canonical; before promotion, a dataset is raw and not authoritative for Research or operational use.

---

## Core Responsibilities

The Data Quality Stack is responsible for:

- Discovering recorded raw datasets via Dataset Manifests published by the Data Recording Stack.
- Loading raw datasets from Persistent Raw Storage for assessment.
- Validating dataset integrity — verifying structural correctness, schema conformance, and data completeness.
- Validating dataset consistency — assessing whether recorded data is internally coherent and plausible. This includes market-structure consistency checks such as whether order book data and trade data are mutually consistent, whether observed spreads are plausible, and whether recorded trades make sense in the context of the accompanying market structure.
- Normalizing raw datasets from Venue-specific formats into the canonical form required by Canonical Storage.
- Promoting eligible datasets — those that have passed validation and normalization — into Canonical Storage with appropriate promotion metadata.
- Preventing unsuitable datasets from entering Canonical Storage. Datasets that fail validation are quarantined or rejected; they do not become canonical.
- Producing validation metadata and promotion metadata that document the assessment outcome and promotion status of each dataset.
- Enforcing the semantic boundary between raw and canonical data. This boundary is not incidental — it is the mechanism that ensures Canonical Storage contains only assessed, validated, normalized datasets.

The Data Quality Stack tracks dataset progression through states such as Discovered, In Validation, Validated, Normalized, Promotable, Promoted, Quarantined, Rejected, and Incomplete. The detailed state model is defined in companion documents.

---

## Explicit Non-Responsibilities

The Data Quality Stack is **not** responsible for:

- **Raw Venue data capture.** Venue feed connectivity, message recording, Capture Time annotation, Local Buffer management, and raw persistence are Data Recording Stack responsibilities.
- **Canonical Storage governance.** The Data Storage Stack manages the Canonical Storage layer itself — its organization, retention, and access patterns. The Data Quality Stack writes promoted datasets to Canonical Storage but does not govern the storage layer.
- **Core Runtime Event semantics.** The canonical Event Stream, Processing Order, State derivation, and all execution-path processing belong to the Core Runtime. The Data Quality Stack does not participate in or define these.
- **Strategy, Risk, Queue, or Order semantics.** These are Core Runtime concerns with no intersection to data quality processing.
- **Backtesting execution or Research interpretation.** The Data Quality Stack produces canonical datasets; it does not consume them for Strategy evaluation or analysis. Those are downstream concerns.

---

## Why the Stack Matters

The Data Quality Stack is the point at which the infrastructure decides which recorded datasets are reliable and suitable enough to serve as canonical Research inputs. Every canonical dataset — every Backtesting input, every Analysis baseline — passes through this Stack's validation and promotion logic before it becomes authoritative.

If the Data Quality Stack is weak, ambiguous, or permissive, Research and Backtesting may operate on incomplete, inconsistent, or structurally defective data without detection. The downstream consequence is that Strategy evaluation results lose credibility: conclusions drawn from low-quality canonical data are unreliable regardless of how rigorous the downstream processing is.

The gatekeeper role is therefore not auxiliary infrastructure — it is the mechanism that gives Canonical Storage its authority as the validated dataset layer for the infrastructure.
