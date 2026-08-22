# Monitoring and Observability

## Overview

This section documents the sanitized monitoring architecture used by the homelab.

The current metrics stack is hosted on a dedicated Ubuntu virtual machine and uses Docker Compose to run Prometheus, Grafana, and a NUT exporter. Prometheus also scrapes host-level metrics from the Proxmox VE host through Node Exporter and a Proxmox-specific exporter.

Checkmk Community is deployed on a separate Debian virtual machine as a complementary infrastructure and service-monitoring layer. Its role is host state, service state, Linux agent monitoring, SNMP monitoring, active checks, and infrastructure-focused notifications where an operational state model is more useful than time-series analysis.

See [`checkmk-plan.md`](checkmk-plan.md) for the deployment and evaluation plan.

## Monitoring architecture

The monitoring model separates operational state monitoring from metrics collection and visualization:

```text
Infrastructure
   |
   +--> Checkmk agents / SNMP / active checks
   |          |
   |          v
   |       Checkmk
   |
   +--> exporters
              |
              v
          Prometheus
              |
              v
            Grafana
```

This separation allows each platform to focus on the monitoring model it handles best while avoiding unnecessary coupling.

## Current metric collection path

The validated Prometheus collection path currently includes:

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

Alertmanager is no longer a prerequisite for the initial infrastructure alerting rollout because Checkmk has its own notification system.

Prometheus alert rules and Alertmanager may still be introduced later for metrics-oriented conditions where Prometheus is the authoritative monitoring source. Any future implementation should avoid duplicating conditions already owned by Checkmk.

### Checkmk

Checkmk Community is deployed on a dedicated Debian virtual machine.

The current platform state includes:

* dedicated Debian guest on Proxmox
* current Community package installed
* stable internal addressing and DNS
* operational Checkmk site
* validated web interface and site services
* workload-specific Proxmox backup coverage
* successful isolated restore validation

Its intended responsibilities include:

* host and service state monitoring
* Linux agent-based monitoring
* SNMP monitoring for supported infrastructure
* active service availability checks
* infrastructure-focused notifications

Broad notification coverage will be introduced only after monitored conditions and ownership boundaries are defined.

## Monitoring VMs

The Prometheus and Grafana stack runs on a dedicated Ubuntu VM rather than directly on the Proxmox host.

Checkmk runs on a separate Debian VM. This separation provides independent maintenance, backup, recovery, testing, and retirement boundaries for the two monitoring models.

See [`../proxmox/runbooks/monitoring-vm-maintenance.md`](../proxmox/runbooks/monitoring-vm-maintenance.md) for the existing Prometheus stack maintenance workflow.

## Validation model

The Prometheus stack is validated in layers:

1. confirm the VM is healthy
2. confirm Docker is healthy
3. confirm the monitoring containers are running
4. confirm Prometheus responds
5. confirm Grafana responds
6. confirm critical Prometheus targets report healthy
7. confirm dashboards show expected data where applicable

Checkmk uses a similar layered validation model:

1. confirm the Debian VM is healthy
2. confirm the Checkmk package is installed
3. confirm the Checkmk site exists
4. confirm site services are running
5. confirm the web interface responds
6. confirm administrative authentication works
7. confirm backup coverage exists
8. confirm a backup restores successfully in an isolated temporary VM
9. validate agent connectivity and service discovery as hosts are onboarded
10. validate notification delivery after notification rules are introduced

The Phase 1 recovery test successfully restored the Checkmk VM from Proxmox backup with networking disconnected, then validated operating-system startup, Checkmk version, site presence, site data, and site service state.

This prevents a false positive where a monitoring UI is online but the underlying collection, recovery, or notification path is broken.

## Maintenance model

Application images and guest operating-system packages are treated as separate maintenance layers for the existing containerized monitoring stack.

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

Checkmk has its own maintenance workflow because it is installed natively on a dedicated Debian VM rather than deployed through the existing Docker Compose stack.

Rollback protection is selected before changes based on risk and available storage.

## Observability principles

1. **Monitor dependencies, not just hosts.** A running VM does not prove the hosted service is functional.
2. **Validate collection paths.** Prometheus target state is evidence that data is actually being scraped.
3. **Keep visualization separate from collection.** Grafana and Prometheus have different failure modes.
4. **Use exporters deliberately.** Exporters extend visibility into systems that do not expose Prometheus-native metrics.
5. **Use infrastructure checks deliberately.** Checkmk should add operational state visibility rather than duplicate every Prometheus metric.
6. **Treat monitoring as production-like infrastructure.** The monitoring systems themselves require updates, backups, validation, and rollback planning.
7. **Alert on actionable conditions.** Alert rules and service checks should identify conditions that require attention rather than every metric fluctuation.
8. **Avoid duplicate notifications.** A condition monitored by multiple platforms should normally have one authoritative notification path.
9. **Validate notification delivery.** A firing rule or critical service state is not enough if the notification path is broken.
10. **Validate recovery, not only backup creation.** A successful backup job is not sufficient evidence until restore behavior has been tested.

## Current monitoring coverage

The validated Prometheus stack currently includes Proxmox host metrics, Proxmox virtualization metrics, and UPS-related monitoring through Prometheus and Grafana.

The Checkmk platform is deployed, operational, and restore-validated. Phase 1 is complete. The next step is onboarding a low-risk Linux guest and validating the Checkmk agent and service-discovery workflow.

See [`checkmk-plan.md`](checkmk-plan.md) for the Checkmk rollout sequence and [`alerting-roadmap.md`](alerting-roadmap.md) for the Prometheus alerting backlog.
