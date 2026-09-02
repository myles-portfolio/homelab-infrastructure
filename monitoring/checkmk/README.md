# Checkmk

This directory contains the Checkmk-specific monitoring documentation for the homelab.

Checkmk Community is the primary platform for infrastructure and service-state monitoring. Prometheus remains responsible for time-series metrics, while Grafana remains the visualization layer. Cross-platform alert ownership and observability architecture are documented one level above this directory.

## Documentation

* [`checkmk-plan.md`](checkmk-plan.md) documents the Checkmk deployment, rollout phases, monitoring coverage, recovery model, secure web access, and current implementation state.
* [`checkmk-configuration-standards.md`](checkmk-configuration-standards.md) defines naming, folder inheritance, classifications, labels, rule targeting, contact routing, notification standards, Linux onboarding, and application plug-in monitoring patterns.
* [`checkmk-notifications.md`](checkmk-notifications.md) documents the sanitized email-notification architecture, including Checkmk, Postfix, the managed SMTP relay, validation, and maintenance expectations.
* [`maintenance-downtime.md`](maintenance-downtime.md) defines scheduled downtime as the required alert-suppression control for planned maintenance on monitored hosts.
* [`../alerting-roadmap.md`](../alerting-roadmap.md) defines notification ownership across Checkmk and the Prometheus alerting path.
* [`../README.md`](../README.md) documents the overall monitoring and observability architecture.

## Current state

Checkmk rollout phases 1 through 6 are complete. Validated coverage includes Linux guests, application-level active checks, core network availability, Proxmox host and storage health, SMART monitoring, scoped interface monitoring, contact-group routing, end-to-end HTML email notification delivery, and trusted HTTPS access to the Checkmk web interface.

Incremental monitoring expansion now also includes a development Linux container running PostgreSQL. The host is onboarded through the standard semantic naming, folder, classification, TLS agent-registration, discovery, and notification workflow. PostgreSQL-specific monitoring is provided through the supported Checkmk agent plug-in and covers instance state, connection behavior, database size and statistics, locks, process counts, query duration, bloat, analyze, and vacuum state. Application-specific API and ingestion checks remain deferred until those services enter operation.

The Checkmk web interface is now served over HTTPS through the system Apache front end. Certificate issuance and renewal use ACME with DNS-based validation through the DNS provider API, avoiding any requirement to expose an HTTP validation endpoint publicly. Renewal has been validated with a dry run, and a deploy hook reloads Apache after successful certificate renewal. Credentials and live hostnames remain excluded from this repository.

The Proxmox API special-agent integration remains disabled because of a compatibility failure in the current integration path. The normal Linux-agent and Prometheus exporter coverage remain healthy.

Scheduled downtime is now part of the standard maintenance workflow for monitored infrastructure. Planned maintenance must suppress notifications only for the affected hosts while preserving normal alerting elsewhere.

Current work is focused on observing notification quality, validating recovery and maintenance behavior, adding monitoring depth where it provides actionable operational evidence, and extending application-level checks as new services enter operation.
