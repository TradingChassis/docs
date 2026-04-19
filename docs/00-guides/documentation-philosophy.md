# Documentation Philosophy

---

## Purpose

This document defines the philosophy of the documentation set as an architectural system of record.

The goal is not to accumulate notes. The goal is to maintain a coherent, authoritative representation of System semantics, architecture boundaries, and implementation alignment over time.

This philosophy governs how documentation is read, written, and maintained so that the docs remain reliable under change.

---

## What this documentation is

The documentation is a structured reference model of the Infrastructure:

- **Canonical documents** define normative semantics and constraints.
- **Overview documents** summarize and orient; they do not redefine canonical meaning.
- **Architecture and Stack documents** explain responsibility boundaries and realization choices without changing canonical semantics.

A reader should be able to answer both questions from the same corpus:

- **Conceptual understanding:** "How does this System work at a model level?"
- **Precise reference:** "What is the exact authoritative definition of this concept?"

If the documentation cannot do both, it is incomplete.

---

## Governing principles

### P1 - Semantic precision over loose description

Documentation must prefer explicit, testable statements over broad narrative language. Terms with architectural meaning must be used consistently and with their canonical definition.

Ambiguous wording is a defect because it enables conflicting implementations.

### P2 - One concept, one authoritative definition

Each core concept must have exactly one canonical definition document. Other documents may reference or apply that definition, but must not redefine it.

When a concept appears in multiple places, one location is authoritative; all other locations are derivative.

### P3 - Clear document scope and boundaries

Every document must have a narrow role. It should state what it defines, what it does not define, and where adjacent concerns are defined.

Unbounded documents create overlap; overlap creates contradiction.

### P4 - Role-based architecture documentation

Architecture documents must be role-based: concepts define semantics, architecture documents define component boundaries, flow documents define sequencing, Stack documents define realization.

Mixing roles in a single document weakens authority and makes maintenance brittle.

### P5 - Overviews summarize; canonical documents define

Overview and map documents exist to aid orientation and reading order. They must summarize without adding new semantics.

If an overview introduces semantic rules not present in canonical concept documents, the documentation model is broken.

### P6 - Consistency is mandatory, not aspirational

Documentation must be non-contradictory across documents. Two documents describing the same concept must agree on meaning, boundaries, and causal interpretation.

When inconsistency exists, one definition must be made authoritative and conflicting wording removed.

### P7 - Documentation must reduce interpretation space

The docs should converge readers toward one model, not invite parallel interpretations.

When readers can derive multiple incompatible meanings from the same concept set, documentation has failed its architectural purpose.

### P8 - Evolution without semantic drift

Documentation must evolve as the Infrastructure evolves, but updates must preserve canonical semantics unless an explicit architectural decision changes them.

Edits that change wording without preserving meaning are semantic drift and must be treated as architectural regressions.

---

## How to read this documentation

Read with two modes:

- **Orientation mode:** start with overview-level documents to understand context and boundaries.
- **Reference mode:** move to canonical concept documents for exact definitions and invariants.

When two statements appear to differ, resolve by authority order:

1. Canonical concept and invariant documents
2. Architecture documents that apply those concepts
3. Overview and map documents that summarize

This order prevents summary text from silently overriding canonical semantics.

---

## How to write and maintain this documentation

### Authoring discipline

- Declare document purpose and boundaries explicitly.
- Reuse canonical terms exactly; do not introduce synonym drift for core concepts.
- Prefer normative statements where architecture constraints are being defined.
- Link to authoritative definitions instead of restating them in modified form.

### Change discipline

- Changes must preserve cross-document consistency.
- If a semantic change is intended, it must be explicit, scoped, and reflected in all affected documents.
- If no semantic change is intended, edits must not alter meaning through wording drift.
- Contradictions must be resolved immediately; they are not acceptable "temporary states."

### Maintenance standard

A document is considered healthy when:

- its role is clear;
- its statements are internally coherent;
- its terms match canonical definitions;
- it does not duplicate adjacent document roles;
- it remains aligned with the current architecture model.

---

## Boundaries of this philosophy document

This document defines documentation philosophy only. It does not:

- define System semantics;
- restate full architecture flows or lifecycle models;
- duplicate section-level structure guidance;
- prescribe prose style beyond architectural clarity requirements.
