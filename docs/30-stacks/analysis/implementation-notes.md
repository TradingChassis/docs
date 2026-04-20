# Implementation Notes

This document captures implementation-facing rationale and design considerations for the Analysis Stack: how asynchronous analytical workflows are realized in practice, how reproducibility and versioning are supported, and what patterns enable structured evaluation, comparison, and derived-artifact production.

---

## Why Analytical Realization Matters

The Analysis Stack turns persisted infrastructure outputs into evaluative knowledge. How that transformation is realized — how artifacts are loaded, how analysis logic is structured, how outputs are versioned and persisted — determines whether analytical conclusions are trustworthy, reproducible, and traceable.

A weak realization (e.g., ad hoc scripts with untracked inputs, unversioned analysis logic, results that cannot be traced to specific persisted artifacts) produces analytical outputs that cannot be verified, reproduced, or meaningfully compared. The implementation patterns described here are designed to make analysis systematic, reproducible, and durable.

---

## Realizing Asynchronous Analysis Workflows

The Analysis Stack's work is inherently asynchronous — it runs independently of the Stacks that produced its inputs, on its own schedule, against artifacts that are already durably stored.

Analysis workflows can be realized through several patterns:

- **Notebooks.** Interactive or parameterized notebooks (e.g., Jupyter) that load persisted artifacts, apply analysis logic, and produce evaluative outputs. Notebooks are effective for exploratory analysis and for analyses where visual inspection and iterative refinement are important.
- **Batch analysis jobs.** Scheduled or manually triggered batch jobs that apply a defined analysis pipeline to a set of persisted artifacts and produce structured outputs. Batch jobs are effective for systematic evaluation — parameter-sweep analysis, cross-experiment comparison, periodic performance reviews.
- **Analysis pipelines.** Multi-stage pipelines where artifact loading, evaluation, comparison, and derived-output generation are decomposed into discrete steps. Pipelines are effective for complex analyses where intermediate results feed subsequent stages.

No specific tool or framework is architecturally required. The key implementation properties are:

- Analysis runs are **self-contained** — each run specifies which persisted artifacts it consumes and which analysis definition it applies.
- Analysis runs are **independent of the producing Stacks** — the Analysis Stack reads from storage surfaces; it does not depend on the Backtesting or Live Stacks being active.
- Analysis logic is **separable from artifact loading and result persistence** — the analytical computation is distinct from the I/O that surrounds it.

---

## Comparison and Evaluation Workflow Patterns

Comparison and evaluation are among the most common analytical tasks. Their realization requires structured access to multiple result sets and defined evaluative criteria.

**Cross-experiment comparison.** An analysis workflow loads results from multiple Backtesting runs — different parameter values, different Strategies, different time periods — and applies comparative logic: ranking by performance metric, computing parameter sensitivity, or identifying systematic patterns across runs. The implementation must support structured iteration over result sets and consistent application of comparative criteria.

**Research–Live discrepancy analysis.** An analysis workflow loads persisted Backtesting outcomes and persisted Live execution records for the same Strategy and time period, and compares them along defined dimensions: fill quality, execution timing, slippage, performance divergence. This is a retrospective analytical task — it operates on persisted outputs from both contexts, not on running infrastructures. The implementation must support cross-context artifact loading (from both Experiment / Artifact Storage and Execution Record Storage) and structured comparison across contexts.

**Threshold and criterion evaluation.** Analysis logic applies defined criteria — minimum performance thresholds, maximum drawdown limits, execution-quality standards — to individual or comparative results. The implementation should support configurable criteria so that evaluative standards can evolve without rewriting analysis logic.

---

## Reproducible and Versioned Analysis

Reproducibility and versioning are implementation-level requirements that influence how analysis workflows are structured and how outputs are managed.

**Reproducible analysis runs.** The same analysis definition applied to the same persisted inputs must produce the same outputs. In practice, this requires:

- Analysis logic that is deterministic with respect to its inputs — no dependence on wall-clock time, random seeds without explicit seeding, or transient infrastructure state.
- Explicit specification of which persisted artifacts are consumed — not implicit discovery that may resolve to different artifacts on different runs.
- Frozen dependencies where the analysis environment includes libraries or frameworks whose behavior may change across versions.

**Versioned analysis definitions.** Analysis logic should be versioned so that outputs are associated with the specific version of the analysis that produced them. When analysis logic changes, the new version is distinct from the old — analytical outputs from different versions are not silently co-mingled.

**Provenance tracking.** Each analytical output should record:

- Which persisted inputs were consumed (artifact identifiers, dataset versions).
- Which analysis definition and version were applied.
- When the analysis was executed.

This provenance record is what makes outputs traceable and what enables an analysis to be re-executed against the same inputs for verification or extension.

In practice, provenance can be realized through metadata written alongside analytical outputs — a manifest, a metadata file, or structured annotations within the output artifact itself.

---

## Derived Artifact and Result Persistence

The Analysis Stack produces derived artifacts that must be durably persisted so they are available for future reference, further analysis, or downstream consumption.

**Persistence targets.** Derived analytical artifacts and evaluation results are written to the Data Storage Stack's persistent surfaces — primarily **Derived Storage** for computed datasets and analytical outputs, and **Experiment / Artifact Storage** for evaluation results associated with specific experiments.

**Structured outputs.** Analytical outputs should be structured for downstream usability — not opaque blobs but datasets, tables, or structured records that can be loaded by subsequent analysis workflows or inspection tools.

**Idempotent persistence.** Where practical, writing an analytical output should be idempotent — re-executing the same analysis against the same inputs produces the same output in the same location without creating duplicates or conflicting artifacts.

The Analysis Stack writes to storage surfaces but does not manage them. Storage organization, retention, and access governance are Data Storage Stack responsibilities.

---

## Retrospective Live-Outcome Analysis

The Analysis Stack may analyze persisted Live execution outputs — order history, fill records, position data, execution metadata — for retrospective evaluation. This is a central use case, not an edge case, because understanding how Live execution compares to Research expectations is fundamental to Strategy development.

Implementation considerations:

- **Cross-context artifact loading.** Retrospective Live analysis requires loading artifacts from Execution Record Storage (Live outputs) and potentially from Experiment / Artifact Storage (corresponding Backtesting results). The implementation must support loading from multiple storage surfaces within a single analysis workflow.
- **Temporal alignment.** Comparing Research and Live outcomes for the same Strategy and time period requires aligning artifacts by time range, instrument, and Strategy configuration. The implementation should support structured alignment rather than requiring manual matching.
- **Discrepancy quantification.** The analysis should produce structured outputs that quantify the discrepancy — not just "different" but "different by how much, along which dimensions, and with what pattern." This requires that discrepancy analysis produce measurable, comparable outputs, not just narrative observations.

Retrospective Live analysis is analytical and asynchronous. It operates on persisted outputs after execution has occurred. It is not real-time operational monitoring — it does not observe running infrastructures or emit alerts.

---

## Implementation Boundaries

**Implementation choices, not canonical semantics.** The workflow patterns, versioning strategies, persistence approaches, and comparison structures described here are implementation-level concerns. They do not define or modify the Infrastructure's canonical Event, State, or lifecycle semantics.

**Analysis operates on persisted artifacts.** The implementation reads from and writes to the Data Storage Stack's persistent surfaces. It does not interact with running infrastructures, consume transient runtime state, or participate in real-time processing.

**Not operational monitoring.** The Analysis Stack's implementation is oriented around asynchronous, retrospective analytical work. It does not implement real-time observation, alerting, or runtime health tracking. That is a separate concern.

**No storage governance.** The Analysis Stack reads from and writes to storage surfaces but does not manage their organization, retention, or access policies.

**No deployment specification.** The patterns above describe design considerations and realization approaches. They do not prescribe specific notebook platforms, pipeline frameworks, scheduling infrastructures, or infrastructure configurations. Those belong in deployment documentation.
