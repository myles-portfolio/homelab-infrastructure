# Checkmk

This directory contains the Checkmk-specific monitoring documentation for the homelab.

Checkmk Community is the primary platform for infrastructure and service-state monitoring. Prometheus remains responsible for time-series metrics, while Grafana remains the visualization layer. Cross-platform alert ownership and observability architecture are documented one level above this directory.

## Documentation

* [`deployment-plan.md`](deployment-plan.md) documents the Checkmk deployment, rollout phases, monitoring coverage, recovery model, and current implementation state.
* [`configuration-standards.md`](configuration-standards.md) defines naming, folder inheritance, classifications, labels, rule targeting, contact routing, and notification standards.
* [`notifications.md`](notifications.md) documents the sanitized email-notification architecture, including Checkmk, Postfix, the managed SMTP relay, validation, and maintenance expectations.
* [`../alerting-roadmap.md`](../alerting-roadmap.md) defines notification ownership across Checkmk and the Prometheus alerting path.
* [`../README.md`](../README.md) documents the overall monitoring and observability architecture.

## Current state

Checkmk rollout phases 1 through 6 are complete. Validated coverage includes Linux guests, application-level active checks, core network availability, Proxmox host and storage health, SMART monitoring, scoped interface monitoring, contact-group routing, and end-to-end HTML email notification delivery.

The Proxmox API special-agent integration remains disabled because of a compatibility failure in the current integration path. The normal Linux-agent and Prometheus exporter coverage remain healthy.

Current work is focused on observing notification quality, validating recovery and maintenance behavior, and adding monitoring depth only where it provides actionable operational evidence.
