# Monitoring and Observability

## Overview

This section documents the sanitized monitoring architecture used by the homelab.

The current stack is hosted on a dedicated Ubuntu virtual machine and uses Docker Compose to run Prometheus, Grafana, and a NUT exporter. Prometheus also scrapes host-level metrics from the Proxmox VE host through Node Exporter and a Proxmox-specific exporter. Alertmanager and Prometheus alert rules are the next implementation step and are represented in the target-state architecture below.

## Monitoring and alerting architecture

The following diagram shows the sanitized monitoring flow from infrastructure and UPS metric sources through exporters into Prometheus, then into Grafana for visualization and Alertmanager for notification routing.

![Monitoring and alerting flow](diagrams/monitoring-diagram.png)

The diagram is intentionally sanitized and does not expose the live environment's real domains, IP addresses, ports, credentials, or other sensitive details. The alerting portion represents the near-term target state and will be implemented after the current documentation refresh.

## Current metric collection path

The validated collection path currently includes:

```text
Proxmox VE Host
   |          |
   v          v
Node       PVE
Exporter   Exporter
   \          /
    \        /
     v      v
    Prometheus
        |
        v
      Grafana

UPS / NUT source
       |
       v
  NUT Exporter
       |
       v
   Prometheus
```

This distinction matters because Grafana is the visualization layer, while Prometheus is the collection and time-series storage layer.

## Components

### Prometheus

Prometheus collects metrics from configured targets and exposes target health, query results, and scrape status.

The current validated scrape targets include:

* Node Exporter on the Proxmox host for host operating-system metrics
* Proxmox/PVE exporter for virtualization platform metrics
* NUT exporter for UPS and power telemetry

Operational validation includes:

* Prometheus HTTP response
* active target status
* individual exporter health
* successful query execution when needed

Prometheus also stores the collected time-series data locally.

### Grafana

Grafana provides dashboards and visualization over the metrics collected by Prometheus.

A healthy Grafana container does not prove that Prometheus or its targets are healthy, so dashboard availability and data-source health are treated as separate checks.

### Node Exporter

Node Exporter runs on the Proxmox host and exposes host-level system metrics to Prometheus.

This provides visibility into the underlying Linux host independently of the virtualization-specific metrics exposed by the PVE exporter.

### Proxmox/PVE exporter

The PVE exporter exposes Proxmox-specific metrics to Prometheus, providing visibility into the virtualization platform beyond generic host operating-system metrics.

### NUT exporter

The NUT exporter translates Network UPS Tools data into Prometheus-compatible metrics.

Because the exporter port is not published to the Monitoring VM host, validation is performed through Prometheus target health rather than assuming direct localhost access should work.

### Alertmanager

Alertmanager is the planned notification-routing layer for Prometheus alerts.

The intended flow is:

```text
Prometheus alert rule
        |
        v
    Alertmanager
        |
        v
Notification channel
```

Alertmanager will handle grouping, deduplication, silencing, and routing after alert rules are implemented.

## Monitoring VM

The monitoring stack runs on a dedicated Ubuntu VM rather than directly on the Proxmox host.

This provides:

* workload isolation
* independent operating-system maintenance
* application-level container management
* a clear rollback boundary
* easier separation between monitoring platform health and hypervisor health

See [`../proxmox/runbooks/monitoring-vm-maintenance.md`](../proxmox/runbooks/monitoring-vm-maintenance.md) for the verified maintenance workflow.

## Validation model

Monitoring is validated in layers:

1. confirm the VM is healthy
2. confirm Docker is healthy
3. confirm the monitoring containers are running
4. confirm Prometheus responds
5. confirm Grafana responds
6. confirm critical Prometheus targets report healthy
7. confirm dashboards show expected data where applicable
8. after alerting is implemented, confirm test alerts reach Alertmanager and the configured notification channel

This prevents a false positive where the monitoring UI is online but the underlying collection or notification path is broken.

## Maintenance model

Application images and guest operating-system packages are treated as separate maintenance layers.

Typical sequence:

```text
Baseline validation
    |
    v
Guest OS package maintenance
    |
    v
System health validation
    |
    v
Container image refresh
    |
    v
Compose recreation
    |
    v
Target and UI validation
```

Rollback protection is selected before the change based on risk and available storage.

## Observability principles

1. **Monitor dependencies, not just hosts.** A running VM does not prove the hosted service is functional.
2. **Validate collection paths.** Prometheus target state is evidence that data is actually being scraped.
3. **Keep visualization separate from collection.** Grafana and Prometheus have different failure modes.
4. **Use exporters deliberately.** Exporters extend visibility into systems that do not expose Prometheus-native metrics.
5. **Treat monitoring as production-like infrastructure.** The monitoring system itself requires updates, backups, validation, and rollback planning.
6. **Alert on actionable conditions.** Alert rules should identify conditions that require attention rather than every metric fluctuation.
7. **Validate notification delivery.** A firing rule is not enough if the notification path is broken.

## Current monitoring coverage

The validated stack currently includes Proxmox host metrics, Proxmox virtualization metrics, and UPS-related monitoring. Alertmanager, notification routing, and alert-rule implementation are the next planned improvements.

See [`alerting-roadmap.md`](alerting-roadmap.md) for the alerting implementation backlog.
