# Monitoring and Observability

## Overview

This section documents the sanitized monitoring architecture used by the homelab.

The current metrics stack is hosted on a dedicated Ubuntu virtual machine and uses Docker Compose to run Prometheus, Grafana, and a NUT exporter. Prometheus also scrapes host-level metrics from the Proxmox VE host through Node Exporter and a Proxmox-specific exporter.

Checkmk Community is deployed on a separate Debian virtual machine as a complementary infrastructure and service-monitoring layer. Its role is host state, service state, Linux agent monitoring, supported network-device monitoring, active checks, and infrastructure-focused notifications where an operational state model is more useful than time-series analysis.

See [`checkmk-plan.md`](checkmk-plan.md) for the deployment and evaluation plan and [`checkmk-configuration-standards.md`](checkmk-configuration-standards.md) for the current folder, classification, label, and rule-targeting standards.

## Monitoring architecture

```text
Infrastructure
   |
   +--> Checkmk agents / supported network monitoring / active checks
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

## Components

### Prometheus

Prometheus collects metrics from configured targets and exposes target health, query results, and scrape status.

The current validated scrape targets include:

* Node Exporter on the Proxmox host for host operating-system metrics
* Proxmox/PVE exporter for virtualization platform metrics
* NUT exporter for UPS and power telemetry

Prometheus itself is also monitored from Checkmk through an active HTTP availability check so application reachability is validated independently of the Prometheus metrics path.

### Grafana

Grafana provides dashboards and visualization over the metrics collected by Prometheus.

A healthy Grafana container does not prove that Prometheus or its targets are healthy, so dashboard availability and data-source health are treated as separate checks. Grafana has its own independent active HTTP availability check in Checkmk.

### Node Exporter

Node Exporter runs on the Proxmox host and exposes host-level system metrics to Prometheus.

### Proxmox/PVE exporter

The PVE exporter exposes Proxmox-specific metrics to Prometheus, providing visibility into the virtualization platform beyond generic host operating-system metrics.

### NUT exporter

The NUT exporter translates Network UPS Tools data into Prometheus-compatible metrics.

Because the exporter port is not published to the Monitoring VM host, validation is performed through Prometheus target health rather than assuming direct localhost access should work.

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
* scalable folder hierarchy for configuration inheritance
* custom classification taxonomy and flexible host labels
* validated classification-based rule targeting
* controlled service, agent, and host state-transition testing
* core guest coverage completed across DNS, password management, file services, Home Assistant, and the monitoring stack
* network infrastructure coverage completed for the current core router and switch
* active checks validating DNS behavior, authenticated SMB access, user-facing web availability, Prometheus availability, Grafana availability, and network management-interface availability

## Checkmk configuration model

Checkmk folders are treated as configuration-inheritance boundaries rather than only visual organization.

The classification model uses:

* folders for inherited configuration and broad organizational boundaries
* host tags for controlled cross-cutting classifications used by rules
* labels for flexible metadata that may evolve over time

The validated custom host-tag dimensions are Environment, Service Criticality, Platform, Virtualization, and Service Class.

Rules are targeted through reusable classifications wherever practical. Validated examples include development filesystem thresholds, DNS active checks, authenticated SMB checks, application HTTP checks, network management-interface checks, and Linux-container-specific memory thresholds.

## Linux agent onboarding

The validated workflow is:

1. place the host in the correct Checkmk folder
2. install the Checkmk Linux agent package
3. register the agent with the Checkmk site
4. restrict the agent listener to the monitoring server where host firewalling is enabled
5. verify agent connectivity
6. run service discovery
7. review discovered labels and services before acceptance
8. activate the configuration
9. confirm the host and accepted services report healthy states
10. verify effective parameters when new rules or classifications are introduced
11. add meaningful application-level checks where practical

## Active service validation

Phase 3 established several application-level patterns:

* DNS checks verify both internal-record resolution and upstream public resolution.
* Password-management monitoring checks the user-facing HTTPS endpoint, including status, latency, and certificate validity.
* File-services monitoring verifies an authenticated SMB share through a dedicated read-only monitoring identity.
* Home Assistant is monitored as an appliance-style VM through its web endpoint without forcing an unsupported general-purpose Linux agent onto the operating system.
* Prometheus and Grafana are checked independently through separate HTTP availability checks.

Phase 4 extended the same principle to current network infrastructure:

* the core router is monitored for ICMP reachability and management-interface availability
* the core switch is monitored independently for ICMP reachability and management-interface availability
* SNMP was evaluated but is not available on the current router firmware or switch model
* SNMP remains the preferred deeper-monitoring method for future network hardware that supports it

These checks answer whether the service or management path is actually usable, rather than only whether a device exists on the network.

## Container-specific monitoring

Small Linux containers produced page-table memory warnings even when overall memory availability was healthy. Effective service parameters identified the default page-table thresholds as the source.

A targeted Linux-container rule now adjusts only the page-table warning and critical thresholds while leaving RAM, swap, committed-memory, and other Linux memory checks unchanged. The rule is scoped through Platform and Virtualization classifications rather than explicit host lists.

A privileged file-services container also exposed an expected guest AppArmor-loader failure under host-level Proxmox confinement. The unnecessary guest loader was disabled after confirming that host-level confinement remained intact.

## State-transition validation

Controlled failure testing demonstrated distinct handling of:

1. service failure
2. Checkmk agent communication failure
3. complete host outage

Each scenario produced the expected Checkmk state and recovered automatically when the underlying condition was corrected.

## Monitoring VMs

The Prometheus and Grafana stack runs on a dedicated Ubuntu VM rather than directly on the Proxmox host.

Checkmk runs on a separate Debian VM. This separation provides independent maintenance, backup, recovery, testing, and retirement boundaries for the two monitoring models.

The Prometheus/Grafana VM is also monitored through the Checkmk Linux agent. During onboarding, Checkmk highlighted elevated root-filesystem utilization. Inspection identified reclaimable container image data, and safe image pruning restored additional free space before application checks were added.

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

1. confirm the Checkmk VM is healthy
2. confirm site services and the web interface are operational
3. confirm backup coverage and restore behavior
4. confirm host agent, appliance, or network-device reachability
5. confirm service discovery returns expected data where supported
6. confirm effective classification-based rules
7. confirm meaningful application or management-interface checks
8. validate distinct service, agent, and host failure behavior
9. validate notification delivery after notification rules are introduced

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

## Observability principles

1. **Monitor dependencies, not just hosts.** A running VM does not prove the hosted service is functional.
2. **Validate collection paths.** Prometheus target state or Checkmk agent connectivity is evidence that data is actually being collected.
3. **Keep visualization separate from collection.** Grafana and Prometheus have different failure modes.
4. **Use exporters deliberately.** Exporters extend visibility into systems that do not expose Prometheus-native metrics.
5. **Use infrastructure checks deliberately.** Checkmk should add operational state visibility rather than duplicate every Prometheus metric.
6. **Treat monitoring as production-like infrastructure.** The monitoring systems themselves require updates, backups, validation, and rollback planning.
7. **Alert on actionable conditions.** Alert rules and service checks should identify conditions that require attention rather than every metric fluctuation.
8. **Avoid duplicate notifications.** A condition monitored by multiple platforms should normally have one authoritative notification path.
9. **Validate notification delivery.** A firing rule or critical service state is not enough if the notification path is broken.
10. **Validate recovery, not only backup creation.** A successful backup job is not sufficient evidence until restore behavior has been tested.
11. **Design for inheritance before scale.** Folder, tag, label, and rule structure should be established before onboarding large numbers of hosts.
12. **Verify effective configuration.** Creating a rule is not sufficient until the intended service shows the resulting effective parameters.
13. **Test failure modes deliberately.** A monitoring design is not validated until expected service, agent, and host failures produce distinct and recoverable states.
14. **Use least privilege for authenticated checks.** Monitoring identities should receive only the access required to validate the service.
15. **Match monitoring depth to device capability.** Prefer supported management protocols and avoid weakening or replacing stable device firmware solely to gain telemetry.

## Current monitoring coverage

Checkmk Phases 1 through 4 are complete. Platform deployment and restore behavior are validated, the low-risk onboarding model has been failure-tested, core guest coverage includes DNS, password management, file services, Home Assistant, Prometheus, and Grafana, and the current core router and switch are monitored for reachability and management-interface availability.

The next rollout work is deliberate Proxmox hypervisor onboarding, followed by notification ownership and delivery once an outbound mail path is available.

See [`checkmk-plan.md`](checkmk-plan.md) for the Checkmk rollout sequence and [`alerting-roadmap.md`](alerting-roadmap.md) for notification architecture and ownership planning.
