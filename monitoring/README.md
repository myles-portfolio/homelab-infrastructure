# Monitoring and Observability

## Purpose

This directory contains the sanitized monitoring and observability documentation for the homelab.

The monitoring stack uses multiple platforms with deliberately separated responsibilities:

| Capability | Primary platform |
|---|---|
| Infrastructure and service state | Checkmk |
| Linux agent monitoring | Checkmk |
| Active service checks | Checkmk |
| Network-device state | Checkmk where supported |
| Storage and hardware state | Checkmk where supported |
| Infrastructure notifications | Checkmk |
| Time-series metrics | Prometheus |
| Exporter-based telemetry | Prometheus |
| Dashboards and visualization | Grafana |
| Metrics-based alerting | Prometheus and Alertmanager, if justified |

The same operational condition should not normally generate notifications from multiple platforms.

## Directory structure

Platform-specific documentation is organized into subdirectories so each monitoring product can evolve independently.

```text
monitoring/
├── README.md
├── alerting-roadmap.md
├── checkmk/
│   ├── README.md
│   ├── checkmk-plan.md
│   ├── checkmk-configuration-standards.md
│   └── checkmk-notifications.md
└── diagrams/
```

Future Prometheus and Grafana documentation should follow the same pattern when those areas require enough dedicated material to justify their own directories.

## Checkmk

Checkmk Community is deployed on a dedicated Debian virtual machine and is the primary infrastructure and service-state monitoring platform.

Validated coverage includes:

* Linux host and service-state monitoring
* controlled service, monitoring-path, and host-failure testing
* internal and upstream DNS checks
* authenticated SMB share availability
* user-facing HTTP and HTTPS checks
* Home Assistant availability
* Prometheus and Grafana availability
* core router and switch reachability and management-interface availability
* Proxmox host, ZFS, process, interface, and SMART monitoring
* contact-group-based notification routing
* HTML email notification delivery through Postfix and a managed SMTP relay
* TLS-registered Linux agent monitoring for the dedicated Nginx reverse-proxy container
* TLS-registered Linux agent monitoring for the dedicated Tailscale remote-access gateway
* active DNS validation after migrating monitored web access through the reverse proxy

The personal knowledge and RAG backend is onboarded through the standard Linux agent and PostgreSQL plug-in workflow. Host, filesystem, database, and PostgreSQL health monitoring are active, while application-level API and ingestion checks remain future work until those services enter operation.

The dedicated reverse-proxy container is onboarded as core infrastructure. Monitoring validates host health independently from the applications routed through it. Certificate-expiration monitoring remains a planned addition so TLS lifecycle failures can be detected before they affect dependent services.

The dedicated remote-access gateway is also onboarded as production core infrastructure. Its Checkmk Linux agent is registered with TLS. Monitoring should distinguish host and systemd health from the higher-level Tailscale routing and policy behavior that is validated through network tests and operational checks.

See [`checkmk/README.md`](checkmk/README.md) for the Checkmk documentation index.

## Prometheus

Prometheus remains the primary time-series metrics platform.

Current validated telemetry includes:

* Node Exporter metrics from the Proxmox host
* Proxmox/PVE exporter metrics
* NUT exporter metrics for UPS and power telemetry

Prometheus is retained for historical analysis, rates, sustained thresholds, and capacity trends rather than duplicating Checkmk state monitoring.

A dedicated `prometheus/` directory should be introduced when Prometheus configuration, alert rules, or operational runbooks require their own documentation set.

## Grafana

Grafana remains the primary visualization layer over Prometheus metrics.

Grafana availability is monitored independently through Checkmk because a healthy dashboard application does not prove the metrics collection path is healthy.

A dedicated `grafana/` directory should be introduced when dashboard standards, provisioning, backup, or administration documentation expands beyond the current shared monitoring overview.

## Alerting architecture

Checkmk is the authoritative notification path for infrastructure and service-state conditions that it owns.

Prometheus Alertmanager remains optional and should only be introduced for Prometheus-owned conditions that require time-window evaluation, routing, grouping, or metric-specific alert handling.

See [`alerting-roadmap.md`](alerting-roadmap.md) for the cross-platform ownership model.

## Validation model

Monitoring changes should be validated at the layer they affect.

For Checkmk, validation may include:

1. host or appliance reachability
2. agent or active-check operation
3. service discovery and effective parameters
4. expected state transitions
5. notification-rule matching
6. contact selection
7. Postfix and SMTP relay delivery
8. final mailbox receipt
9. recovery behavior

For reverse-proxy changes, monitoring validation should also confirm that DNS active checks reflect the new ingress address and that stale host-level or local DNS records do not create duplicate answers.

For the remote-access gateway, validation should include host health, Checkmk agent connectivity, Tailscale daemon state, route advertisement, permitted remote paths, and at least one denied path that should remain local-only.

For database-backed application services, validation should distinguish guest health from database availability and user-facing application health.

For Prometheus and Grafana, validation should distinguish metrics collection, exporter health, Prometheus target state, query behavior, and dashboard availability.

## Operating principles

1. Monitor dependencies, not only hosts.
2. Use each monitoring platform for the model it handles best.
3. Keep one authoritative notification owner per operational condition.
4. Prefer reusable classifications and inherited configuration over per-host exceptions.
5. Validate effective configuration after creating or changing rules.
6. Tune noisy conditions narrowly rather than weakening unrelated checks.
7. Use least-privilege credentials for authenticated monitoring.
8. Match monitoring depth to supported device capabilities.
9. Validate notification delivery end to end.
10. Validate recovery behavior, not only failure detection.
11. Keep monitoring infrastructure itself backed up, maintained, and tested.
12. Keep secrets and live topology details out of the public repository.

## Related documentation

* [`checkmk/README.md`](checkmk/README.md) for Checkmk-specific documentation
* [`alerting-roadmap.md`](alerting-roadmap.md) for cross-platform alert ownership and future alerting work
* [`../networking/reverse-proxy-pattern.md`](../networking/reverse-proxy-pattern.md) for the centralized ingress and TLS pattern
* [`../networking/tailscale-remote-access.md`](../networking/tailscale-remote-access.md) for the secure remote-access pattern
* [`../proxmox/runbooks/monitoring-vm-maintenance.md`](../proxmox/runbooks/monitoring-vm-maintenance.md) for the existing Prometheus and Grafana VM maintenance workflow
