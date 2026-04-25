# Implementation Notes

This document captures implementation-facing rationale and design considerations for the Data Quality Stack: why specific realization choices matter, how the logical pipeline decomposes into practical concerns, and what patterns support traceability and promotion integrity.

---

## Why Quality-Promotion Realization Matters

The Data Quality Stack is the exclusive path through which raw recorded datasets become canonical. How that path is realized in practice — how validation is structured, how promotion is gated, how quarantine is handled, how decisions are recorded — determines whether the boundary between raw and canonical data is operationally trustworthy.

A weak realization (e.g., validation and promotion conflated into a single opaque step, metadata not recorded, quarantine not traceable) produces a Canonical Storage layer whose authority is nominal rather than earned. Downstream consumers — Backtesting, Analysis — would be operating on datasets whose quality status is unknown or unverifiable.

The implementation patterns described here are designed to keep the quality-promotion boundary rigorous, separable, and auditable.

---

## Validation and Promotion as Distinct Pipeline Concerns

Although validation, normalization, and promotion all reside within the Data Quality Stack, they are **separable implementation concerns** that should be realized as distinct stages rather than a monolithic process.

### Validation pipeline

Validation decomposes into independently realizable checks:

- **Schema and format validation.** Verify that raw dataset structure conforms to expected field types, message layouts, and encoding.
- **Completeness checks.** Verify coverage within the dataset's declared Time Window — detect gaps, missing segments, or unexpectedly short recordings.
- **Consistency checks.** Verify internal coherence: cross-field validity, message-sequence integrity, and absence of contradictory data within a single dataset.
- **Market-structure plausibility.** Where both order book and trade data are present, verify that they form a mutually consistent recorded market view — for example, that recorded trades are plausible given the contemporaneous order book state, and that observed spreads fall within reasonable bounds.

These checks can be implemented as independent stages or parallel tasks within a pipeline. Structuring them separately allows individual checks to be added, modified, or rerun without requiring the entire validation pipeline to execute again.

### Normalization pipeline

Normalization transforms validated raw datasets from Venue-specific formats into canonical form. It is logically separate from validation: a dataset may be structurally valid but not yet normalized, and normalization should operate only on datasets that have passed validation.

Normalization is a format transformation, not a content modification. Preserving data fidelity through normalization is an implementation invariant — the canonical output must faithfully represent the same market data present in the raw input.

### Promotion gate

Promotion is an explicit decision point, not an automatic side effect of normalization completing. The promotion gate evaluates the accumulated validation and consistency outcomes for a normalized dataset and determines whether it meets the criteria for canonical promotion.

Realizing promotion as a distinct gate — rather than embedding it at the end of the normalization step — makes the promotion decision inspectable, configurable, and separable from the transformation logic.

---

## Dataset Progression and Metadata Tracking

Datasets progress through the Data Quality Stack's processing stages, and that progression should be tracked through persistent metadata rather than transient runtime state.

A practical metadata model may track states such as:

| State | Meaning |
| ----- | ------- |
| **Discovered** | Dataset Manifest located; raw dataset registered for processing |
| **In Validation** | Validation pipeline is actively assessing the dataset |
| **Validated** | All validation checks passed |
| **Normalized** | Successfully transformed into canonical form |
| **Promotable** | Passed promotion-gate evaluation; eligible for canonical promotion |
| **Promoted** | Written to Canonical Storage with promotion metadata |
| **Quarantined** | Held for review — not yet promoted, not yet rejected |
| **Rejected** | Definitively excluded from canonical promotion |
| **Incomplete** | Not yet processable (e.g., raw data not fully available in Persistent Raw Storage) |

Each state transition should be recorded with a timestamp and, where relevant, the reason for the transition (especially for quarantine and rejection). This metadata is the mechanism that makes dataset progression traceable after the fact.

The metadata model is an implementation realization, not a canonical infrastructure semantic. Its specific states and transitions may evolve as the stack's processing logic matures, without affecting the broader infrastructure's semantic model.

---

## Workflow and Orchestration Patterns

The Data Quality Stack's processing — discovery, validation, normalization, promotion, quarantine — can be realized as a **workflow-orchestrated pipeline** in which each stage is a discrete task with defined inputs, outputs, and completion conditions.

Workflow orchestration infrastructures such as Argo Workflows are a natural fit for this pattern:

- **Discovery as a trigger.** A scheduled or event-driven task scans for new Dataset Manifests. Each discovered manifest triggers a pipeline instance for the corresponding raw dataset.
- **Validation as a task graph.** Individual validation checks (schema, completeness, consistency, market-plausibility) can be expressed as parallel or sequential tasks within a single workflow. Each task writes its outcome to validation metadata.
- **Normalization as a downstream task.** Normalization executes only after the validation task graph completes successfully. It reads from Persistent Raw Storage and writes the canonical output to a staging location.
- **Promotion as a gated step.** The promotion gate evaluates accumulated metadata from prior stages. If the dataset meets promotion criteria, the canonical output is written to Canonical Storage and promotion metadata is recorded. If not, the dataset is routed to quarantine or rejection.
- **Quarantine and rejection as terminal tasks.** Non-promotion outcomes produce their own metadata records and may trigger review workflows or operational alerts.

Argo Workflows is presented here as an example implementation approach, not as a required technology. Any orchestration infrastructure that supports task-graph execution, conditional routing, and metadata persistence can realize this pattern.

The key implementation property is that each stage is **independently observable and retriable**: if normalization fails, it can be rerun without re-executing validation; if a promotion gate decision needs re-evaluation, it can be re-executed against updated criteria without reprocessing the entire pipeline.

---

## Quarantine and Non-Promotion Handling

Not every dataset that enters the Data Quality Stack's processing scope will be promoted. The implementation must handle non-promotion as a first-class outcome, not as an error condition that requires ad hoc intervention.

**Quarantine** should be realized as a durable, inspectable state — not as a silent discard. A quarantined dataset should retain its validation metadata, its raw dataset reference, and the reason for quarantine. This allows quarantined datasets to be re-evaluated when validation rules change, when raw data is re-recorded, or when an operator investigates a quality issue.

**Rejection** should be equally traceable. A rejected dataset should carry metadata documenting why it was rejected and by which validation or consistency check. Rejection metadata is the evidence that the Data Quality Stack assessed the dataset and made an explicit exclusion decision rather than silently dropping it.

**Incomplete** datasets — those whose raw data is not yet fully available — should be tracked so they can be revisited on subsequent processing cycles without requiring manual re-triggering.

The implementation pattern is: every dataset that enters the Data Quality Stack's scope must exit with a recorded outcome (Promoted, Quarantined, Rejected, or Incomplete). No dataset should be silently lost or left in an unresolved state.

---

## Traceability and Auditability of Quality Decisions

The Data Quality Stack makes consequential decisions: which datasets become canonical inputs for Research and Backtesting, and which are excluded. Those decisions must be **traceable** — reconstructible from metadata after the fact — so that questions like "why was this dataset promoted?" or "why was this dataset quarantined?" have definitive answers.

Traceability in practice requires:

- **Validation metadata per check.** Each validation and consistency check records its outcome independently. The aggregate validation result is derivable from individual check results.
- **Promotion-gate decision record.** The promotion decision records which criteria were evaluated, what the accumulated validation status was, and whether the result was promote, quarantine, or reject.
- **Immutability of promotion metadata.** Once a dataset is promoted and its promotion metadata is written, that metadata should not be silently modified. If a promoted dataset is later found to be defective, the appropriate action is to record a new quality event (e.g., a post-promotion review outcome), not to retroactively alter the original promotion record.
- **Stable dataset identity.** Each raw dataset and each canonical dataset should carry stable identifiers that link the two through the validation-normalization-promotion chain. A canonical dataset should be traceable back to the specific raw dataset and the specific validation/promotion cycle that produced it.

These properties are implementation concerns — they describe how the Data Quality Stack's realization preserves auditability. They do not introduce new canonical infrastructure semantics.

---

## Implementation Boundaries

**Implementation choices, not canonical semantics.** The metadata model, workflow patterns, orchestration tooling, and pipeline decomposition described here are realization concerns. They do not define or modify the infrastructure's canonical Event, State, or lifecycle semantics.

**Raw and canonical remain logically distinct.** Regardless of how the pipeline is implemented, the logical separation between raw recorded datasets (in Persistent Raw Storage) and canonical datasets (in Canonical Storage) must be preserved. The promotion gate is the boundary between the two. No implementation shortcut should blur this distinction.

**No Core Runtime interaction.** The Data Quality Stack's implementation does not intersect with the Core Runtime Event Stream, State derivation, or execution-path processing. Quality processing operates entirely within the Data Platform domain.

**No deployment specification.** The patterns above describe logical realization approaches. Physical infrastructure, cluster topology, and deployment procedures are not specified by this document.
