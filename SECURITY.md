# Security Policy

This document describes how to report security issues related to this
documentation repository and its supporting infrastructure.

This repository contains architecture documentation only and does
not include production trading systems.

---

## Supported Versions

Only the latest state of the `main` branch is actively maintained.

Historical commits and older documentation snapshots may not receive
security updates.

---

## Reporting a Vulnerability

If you discover a security issue related to:

- the documentation website
- the MkDocs build process
- GitHub Actions workflows
- deployment or hosting configuration
- third-party services used for documentation delivery

please do not open a public GitHub issue.

Instead, report it through:

- GitHub Security Advisories  
- private communication with the repository maintainers

When submitting a report, please include:

- a clear description of the issue  
- affected components  
- potential impact  
- steps to reproduce (if applicable)

All valid reports will be handled through responsible disclosure.

---

## Repository Scope

This repository contains:

- system architecture documentation
- conceptual system models
- design decisions and trade-offs
- operational and evolution documentation

It does not contain:

- production runtime systems
- executable trading infrastructure
- Strategy implementations
- proprietary datasets
- private research code

Security concerns should therefore focus only on documentation
infrastructure and delivery mechanisms.

---

## Components Outside This Repository

The broader trading system may include components such as:

- runtime execution engines
- live trading infrastructure
- market data recording pipelines
- cloud deployment environments

These components are implemented and secured in their respective
repositories and are outside the scope of this security policy.

---

## Dependency and Build Security

The documentation stack uses:

- MkDocs for site generation
- Material for MkDocs theme
- pinned Python dependencies

Security practices include:

- explicit dependency versioning
- minimal build surface
- automated CI builds
- no secrets stored in the repository

Security-related dependency updates are prioritized.

---

## Responsible Disclosure

Please allow reasonable time for investigation and remediation before
public disclosure.

Coordinated disclosure helps protect users and infrastructure.