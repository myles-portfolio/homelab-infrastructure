# Monitoring and Observability

## Overview

This section documents the sanitized monitoring architecture used by the homelab.

The current stack is hosted on a dedicated Ubuntu virtual machine and uses Docker Compose to run Prometheus, Grafana, and a NUT exporter. The goal is to separate monitoring responsibilities from the workloads being observed and to validate monitoring health at both the platform and target levels.

## Architecture

```text
Infrastructure and service targets
            |
            v
        Prometheus
            |
            +--> NUT exporter
            |
            v
      Time-series data
            |
            v
          Grafana
            |
            v
      Dashboards / review
```

## Components

### Prometheus

Prometheus collects metrics from configured targets and exposes target health, query results, and scrape status.

Operational validation includes:

* Prometheus HTTP response
* active target status
* individual exporter health
* successful query execution when needed

### Grafana

Grafana provides dashboards and visualization over the collected metrics.

A healthy Grafana container does not prove that Prometheus or its targets are healthy, so dashboard availability and data-source health are treated as separate checks.

### NUT exporter

The NUT exporter exposes UPS-related metrics to Prometheus.

Because the exporter port is not necessarily published to the VM host, validation is performed through Prometheus target health rather than assuming direct localhost access should work.

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

This prevents a false positive where the monitoring UI is online but the underlying collection path is broken.

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
6. **Avoid alert noise.** Planned future alerting should focus on actionable conditions rather than every metric fluctuation.

## Current monitoring coverage

The validated stack includes infrastructure and UPS-related monitoring. Additional service monitoring and alerting remain active areas for expansion.

See [`alerting-roadmap.md`](alerting-roadmap.md) for planned improvements.
