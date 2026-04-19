---
title: Canonical Storage as Authoritative Dataset Layer
updated: 2026-04-01
adr_status: Accepted
---

# ADR-002: Canonical Storage as Authoritative Dataset Layer

Status: {{ page.meta.adr_status }}  
Updated: {{ page.meta.updated }}

---

## Context

The Infrastructure operates across multiple Stacks — Data Recording, Data Quality, Data Storage, Backtesting, Live, Analysis, Monitoring — that share persistent datasets. Market data is collected from external sources, normalized, validated, and then consumed by both Research workflows (Backtesting, Analysis) and Live operations.

Without a single authoritative dataset layer, the following problems arise:

- **Uncontrolled duplication.** Each Stack acquires, transforms, or copies datasets independently. Multiple versions of the same logical dataset accumulate across Stacks, with no mechanism to determine which version is authoritative.
- **Inconsistent results.** Backtesting, Analysis, and Live operate on different copies of nominally the same data. Results diverge not because of logic differences but because of data differences — a class of inconsistency that is difficult to diagnose.
- **Irreproducible backtests.** If the dataset used for a backtest is not immutable and identifiable, re-running the same experiment does not guarantee the same inputs. Reproducibility depends on being able to identify and retrieve the exact dataset that was used.
- **Weak provenance.** Derived datasets (e.g., normalized market data, validated order book histories) cannot be traced to a specific validated source if multiple storage locations hold overlapping data at different stages of processing.
- **Implicit cross-stack dependencies.** Stacks that consume data from other Stacks without a shared authoritative layer create hidden dependencies on storage locations, file formats, and promotion status that are not documented or enforceable.

The architecture requires an explicit authoritative dataset layer that all Stacks reference as the single validated source of persistent data.

---

## Decision

The Infrastructure maintains **Canonical Storage** as the authoritative validated dataset layer for cross-stack use.

### Core rules

1. **Canonical Storage is the authoritative source of validated datasets.** All Stacks that consume persistent datasets — Backtesting, Live, Analysis — read from Canonical Storage. There is no alternative dataset path that bypasses it for validated data.

2. **Dataset promotion is staged and explicit.** External data follows a defined promotion path before reaching Canonical Storage:

    `raw → normalized → validated → canonical`

    Data is first collected by the Data Recording Stack, then normalized and validated by the Data Quality Stack. Only datasets that pass validation are promoted to Canonical Storage. Unpromoted data does not enter the canonical layer.

3. **Canonical datasets are immutable after promotion.** Once a dataset is promoted to Canonical Storage, it is not modified. Corrections, re-normalizations, or re-validations produce new dataset versions rather than mutating existing ones.

4. **Canonical datasets are versioned and identifiable.** Each canonical dataset carries metadata sufficient to identify its exact version, provenance, and promotion history. Any consumer can determine which version of a dataset it is using.

5. **Stacks produce into and consume from Canonical Storage through defined roles.** The Data Platform (Recording, Quality, Storage) produces validated datasets into Canonical Storage. The Core Runtime (Backtesting, Live) and Analysis consume from it. Stacks do not exchange datasets through ad hoc storage locations outside this layer.

### What Canonical Storage is not

**Canonical Storage is not the Runtime Event Stream.** The Event Stream is the canonical record of system history for runtime semantics — deterministic replay, State derivation (`State = f(Event Stream, Configuration)`), and Processing Order are defined from Events, not from storage datasets. Canonical Storage provides validated persistent datasets (market data, experiment results, Research artifacts) that the Core Runtime and Analysis consume. These are complementary but distinct: the Event Stream governs runtime causality; Canonical Storage governs dataset authority.

**Canonical Storage is not generic persistent storage.** It is specifically the layer for validated, promoted, immutable datasets with defined provenance. Scratch storage, intermediate processing artifacts, and unpromoted raw data are not part of Canonical Storage.

---

## Consequences

**Backtesting inputs are reproducible.** Because canonical datasets are immutable and versioned, a backtest can be re-run against the exact same input data. Reproducibility depends on dataset identity, which Canonical Storage provides by construction.

**Cross-stack data consistency is structural.** All Stacks that need validated data reference the same canonical datasets. There is no divergence caused by Stacks holding different copies of the same logical data — the canonical layer is the single reference point.

**Dataset provenance is traceable.** Every canonical dataset has a defined promotion path: raw collection, normalization, validation, promotion. Derived results can be traced to the specific canonical datasets they consumed, and those datasets can be traced to their raw sources.

**Stack boundaries are explicit with respect to data.** Stacks produce into or consume from Canonical Storage through defined roles, not through ad hoc file sharing or implicit storage dependencies. The data contract between Stacks is mediated by the canonical layer.

**Immutability constrains correction workflows.** Because canonical datasets are immutable after promotion, correcting a dataset means producing a new version through the full promotion pipeline. This is more operationally involved than in-place correction — but in-place mutation is precisely the mechanism that destroys reproducibility and provenance.

---

## Trade-offs

**Canonical Storage requires promotion infrastructure.** The staged promotion model (raw → normalized → validated → canonical) requires data pipelines, validation rules, and metadata management. This is infrastructure that must be built and maintained. The alternative — allowing Stacks to access unvalidated or ad hoc data directly — eliminates this infrastructure but reintroduces the duplication, inconsistency, and provenance problems the decision is designed to prevent.

**Immutability increases storage volume.** Dataset corrections and re-validations produce new versions rather than replacing existing ones. Over time, multiple versions of the same logical dataset accumulate. This is a storage cost, but it is also what makes it possible to determine exactly which dataset version was used for any past experiment or operation.

---

## Summary

The Infrastructure maintains Canonical Storage as the authoritative validated dataset layer. Datasets reach this layer only through staged promotion (`raw → normalized → validated → canonical`) and are immutable after promotion. All Stacks that consume validated data reference Canonical Storage as the single authoritative source. Canonical Storage is distinct from the Runtime Event Stream: the Event Stream governs runtime causality and State derivation; Canonical Storage governs persistent dataset authority across Stacks. This decision ensures reproducible inputs, consistent cross-stack data, traceable provenance, and explicit data boundaries between Stacks.
