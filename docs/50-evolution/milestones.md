# Milestones

This page summarizes the major project milestones.

Dates are approximate and intended to show development phases rather than exact delivery timestamps.

---

## August 2025 – November 2025

### Infrastructure Foundation

Initial work focused on building the project’s infrastructure base and deployment model.

**Main outcomes**

- Established cloud and server-side infrastructure foundation
- Built the Infrastructure repository and supporting secrets/configuration setup
- Evaluated deployment and operations requirements around Kubernetes and Infrastructure as Code
- Explored exchange and market-data integration patterns, including API and transport options such as WebSocket, UDP, multicast, FIX, and lower-latency binary protocols
- Investigated provider and colocation constraints relevant to the target runtime environment

**Why this milestone mattered**

- Defined the physical and operational foundation required to run the infrastructure consistently
- Clarified which infrastructure capabilities were actually needed for the project

---

## November 2025 – January 2026

### Core, Core Runtime and Backtesting Foundation

Work shifted from cloud infrastructure into the Core model and the first usable Research and Core Runtime tooling.

**Main outcomes**

- Implemented the Core Python library and its Core Runtime
- Built the main Runtime concepts around:
  - Event Stream
  - State derivation
  - Strategy evaluation
  - Risk Engine
  - Queue / Execution Control
  - Configuration
  - Processing Order
  - Venue Adapter / Venue integration boundaries
- Built the Runtime path for Backtesting
- Added experiment and parameter-sweep support
- Added supporting runtime tooling such as scratch volumes, preload/sweep infrastructure, manifest loading, and experiment orchestration
- Integrated supporting observability and workflow tooling including MLflow, Grafana, Prometheus, Argo Workflows, and S3-based storage usage

**Why this milestone mattered**

- Produced the first coherent version of the Core
- Made the runtime usable for Backtesting and systematic experimentation/Research

---

## February 2026 – March 2026

### Formal Documentation and Project Structuring

Once the Core Runtime and supporting tooling existed, the focus moved to formalizing the architecture and making the project legible.

**Main outcomes**

- Created the MkDocs-based architecture documentation
- Documented the semantic and conceptual model of the Infrastructure rather than repository-level implementation details
- Structured documentation around architecture, concepts, Stacks, operations, and evolution
- Cleaned up project structure at the GitHub organization and repository level
- Standardized repository presentation and publication setup
- Published the codebase and documentation in a more coherent public form

**Why this milestone mattered**

- Turned the project from an internal codebase into a documented infrastructure
- Established a formal architectural vocabulary and documentation surface

---

## March 2026 – April 2026

### Canonical Model Consolidation

Work focused on tightening the documentation and aligning it around a single explicit architecture model.

**Main outcomes**

- Consolidated the deterministic event-driven architecture model
- Clarified the boundaries between:
  - Event and Intent
  - Intent lifecycle and Order lifecycle
  - Risk and Execution Control
  - Queue State and canonical truth
- Strengthened the formal treatment of:
  - Processing Order
  - replayability
  - deterministic State derivation
  - Execution Control semantics
- Aligned architecture and concept documents to a single frozen canonical model
- Finalized the public project naming and documentation presentation

**Why this milestone mattered**

- Removed major semantic ambiguity from the documentation
- Established a coherent architecture model that can now guide further implementation and documentation work

---

## Current Position

At this stage, the project has:

- a defined infrastructure foundation
- a working Core Runtime and Backtesting foundation
- a documented deterministic event-driven architecture model
- a public documentation and repository structure that reflects the Infrastructure coherently

The next milestones are expected to focus on downstream documentation alignment, additional Runtime refinement, and further maturation.
