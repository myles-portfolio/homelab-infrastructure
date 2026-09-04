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

Incremental monitoring expansion includes the personal knowledge and RAG backend with Linux host monitoring and PostgreSQL-specific coverage through the supported Checkmk agent plug-in. Application-specific API and ingestion checks remain deferred until those services enter operation.

A dedicated Nginx reverse-proxy container is now also monitored as core infrastructure through the standard Checkmk Linux agent and TLS registration workflow. The proxy is classified separately from the applications it routes so ingress health can be distinguished from backend service health.

## Secure web access

The Checkmk web interface is now accessed through the dedicated Nginx reverse proxy rather than relying on the Checkmk host as the client-facing TLS endpoint.

The current access pattern is:

```text
Client
  |
  v
Local DNS
  |
  v
Nginx reverse proxy
  |
  v
Checkmk backend
```

The reverse proxy presents a trusted wildcard certificate issued through ACME DNS challenge validation. The certificate private key remains on the proxy rather than being copied to the Checkmk host.

The migration was validated by forcing a client request to the reverse proxy before changing DNS, then updating local DNS and confirming normal Checkmk access through the production hostname.

The active DNS check was also updated to expect the proxy path. Troubleshooting identified a stale local host record that caused the resolver to return both the old backend address and the new proxy address. After removing the stale entry and reloading DNS, direct resolver tests, the underlying DNS monitoring plug-in, and the Checkmk active service all returned to the expected state.

Certificate renewal automation on the dedicated proxy and certificate-expiration monitoring remain follow-up controls.

## Monitoring behavior after proxy migration

Reverse-proxy deployment adds another dependency layer, so monitoring interpretation must distinguish:

* proxy host health
* Nginx listener and routing health
* DNS resolution
* certificate validity
* backend Checkmk application health

A healthy Checkmk backend does not prove that the client-facing proxy path is healthy. Likewise, a proxy outage does not necessarily mean the Checkmk application itself has failed.

## Other current state notes

The Proxmox API special-agent integration remains disabled because of a compatibility failure in the current integration path. The normal Linux-agent and Prometheus exporter coverage remain healthy.

Scheduled downtime is part of the standard maintenance workflow for monitored infrastructure. Planned maintenance must suppress notifications only for the affected hosts while preserving normal alerting elsewhere.

Current work is focused on observing notification quality, validating recovery and maintenance behavior, adding certificate-expiration monitoring, adding monitoring depth where it provides actionable operational evidence, and extending application-level checks as new services enter operation.
