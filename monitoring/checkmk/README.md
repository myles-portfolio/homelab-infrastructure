# Checkmk

This directory contains the Checkmk-specific monitoring documentation for the homelab.

Checkmk Community is the primary platform for infrastructure and service-state monitoring. Prometheus remains responsible for time-series metrics, while Grafana remains the visualization layer. Cross-platform alert ownership and observability architecture are documented one level above this directory.

## Documentation

* [`checkmk-plan.md`](checkmk-plan.md) documents the Checkmk deployment, rollout phases, monitoring coverage, recovery model, and current implementation state.
* [`checkmk-configuration-standards.md`](checkmk-configuration-standards.md) defines naming, folder inheritance, classifications, labels, rule targeting, contact routing, and notification standards.
* [`checkmk-notifications.md`](checkmk-notifications.md) documents the sanitized email-notification architecture, including Checkmk, Postfix, the managed SMTP relay, validation, and maintenance expectations.
* [`maintenance-downtime.md`](maintenance-downtime.md) defines scheduled downtime as the required alert-suppression control for planned maintenance on monitored hosts.
* [`../alerting-roadmap.md`](../alerting-roadmap.md) defines notification ownership across Checkmk and the Prometheus alerting path.
* [`../README.md`](../README.md) documents the overall monitoring and observability architecture.

## Current state

Checkmk rollout phases 1 through 6 are complete. Validated coverage includes Linux guests, application-level active checks, core network availability, Proxmox host and storage health, SMART monitoring, scoped interface monitoring, contact-group routing, and end-to-end HTML email notification delivery.

The Proxmox API special-agent integration remains disabled because of a compatibility failure in the current integration path. The normal Linux-agent and Prometheus exporter coverage remain healthy.

Scheduled downtime is now part of the standard maintenance workflow for monitored infrastructure. Planned maintenance must suppress notifications only for the affected hosts while preserving normal alerting elsewhere.

Current work is focused on observing notification quality, validating recovery and maintenance behavior, and adding monitoring depth only where it provides actionable operational evidence.
