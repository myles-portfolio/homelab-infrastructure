# Changelog

This directory contains the curated, sanitized history of material homelab changes.

The changelog is kept in the repository so operational history remains versioned alongside the architecture, runbooks, standards, and configuration examples it describes. Entries intentionally omit exact private IP addresses, public-facing domains, credentials, personal identifiers, and other sensitive implementation details.

## Organization

Change history is organized by year and month so the archive remains navigable as the environment grows.

```text
changelog/
├── README.md
└── 2026/
    ├── README.md
    ├── 2026-08.md
    ├── 2026-06.md
    ├── 2026-05.md
    └── 2026-02.md
```

Within each monthly file, entries are ordered newest first.

## Years

* [2026](2026/README.md)

## Current month

* [August 2026](2026/2026-08.md)

## Scope

The changelog records material changes such as:

* infrastructure deployments and retirements
* monitoring and backup architecture changes
* maintenance that materially changes system state
* security and networking changes
* recovery validation
* automation and configuration milestones
* operating standards that change how the environment is managed

Routine observations and highly detailed implementation notes remain in internal operational records or workload-specific documentation rather than being duplicated here.
