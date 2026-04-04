# Contributing

Thank you for your interest in contributing to the Trading Engineering Architecture Knowledge Base.

This repository documents the architecture, concepts, and evolution of the Trading Engineering System.

Contributions should improve clarity, correctness, and long-term maintainability of the documentation.

The goal is to keep the architecture explicit, deterministic, and understandable for future contributors.

---

## Design Principles

All contributions should follow the core architectural philosophy:

- Determinism over convenience
- Explicit system modeling
- Clear architectural boundaries
- Traceable design decisions
- Long-term system evolution

Avoid introducing ambiguous descriptions, undocumented assumptions, or unclear system behavior.

Architecture documentation should explain why the System exists in its current form, not only how it is implemented.

---

## What Can Be Contributed

Typical contributions include:

- clarifying architectural concepts
- improving system explanations
- adding missing documentation
- documenting architectural decisions
- correcting inconsistencies
- improving structure or readability
- expanding operational knowledge

Large conceptual changes should ideally be introduced as architecture decision documents.

---

## Workflow

1. Fork the repository
2. Create a feature branch
3. Make small, well-scoped commits
4. Open a Pull Request with a clear explanation of the change

Pull requests should explain:

- what was changed
- why the change improves the documentation
- which part of the architecture it affects

---

## Commit Style

Use clear and descriptive commit messages:

docs: clarify deterministic model  
docs: add overview section  
fix: correct architecture diagram reference  
chore: restructure documentation navigation

---

## Documentation Guidelines

When writing documentation:

- Prefer clear and simple language
- Avoid unnecessary jargon
- Keep explanations structured and hierarchical
- Use diagrams or examples where helpful
- Keep sections focused on a single concept

Documentation should prioritize understandability.

---

## Local Preview

To preview the documentation locally:

```bash
mike serve
```

Then open:

```
http://127.0.0.1:8000
```

Ensure the documentation builds correctly before submitting a pull request.

---

## Questions

If something in the architecture is unclear, feel free to open an issue for discussion before submitting large changes.
