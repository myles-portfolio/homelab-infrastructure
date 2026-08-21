# Checkmk Deployment Plan

## Purpose

This document defines the introduction of Checkmk as an infrastructure and service-monitoring layer for the homelab.

Checkmk is not intended to replace the existing Prometheus and Grafana stack. Prometheus remains the primary time-series metrics platform, while Grafana remains the primary visualization layer. Checkmk is being introduced for host state, service state, agent-based monitoring, SNMP monitoring, and operational alerting where that model provides clearer infrastructure visibility.

## Why add Checkmk

The existing Prometheus stack is effective for metrics collection, historical analysis, dashboards, and exporter-based telemetry. Checkmk addresses a different operational question: whether a host, service, interface, or infrastructure component is currently healthy and requires attention.

The intended division of responsibilities is:

| Capability | Primary platform |
|---|---|
| Time-series metrics | Prometheus |
| Dashboards and visualization | Grafana |
| Exporter-based telemetry | Prometheus |
| Host and service state | Checkmk |
| Linux agent monitoring | Checkmk |
| SNMP infrastructure monitoring | Checkmk |
| Metrics-based alert rules | Prometheus and Alertmanager, if retained |
| Infrastructure state notifications | Checkmk, subject to final alerting design |

This boundary may be adjusted after operational testing to avoid duplicate alerts and unnecessary collection overlap.

## Deployment

Checkmk Community is deployed on a dedicated Debian virtual machine hosted by Proxmox VE.

Current platform sizing:

| Resource | Allocation |
|---|---:|
| vCPU | 2 |
| Memory | 4 GB |
| Storage | 64 GB |
| Network adapter | VirtIO |
| Operating system | Debian 13 |
| Checkmk edition | Community |

The dedicated VM provides workload isolation, an independent maintenance boundary, straightforward backup and recovery, and a deployment model that closely matches Checkmk's standard Linux installation workflow.

Checkmk is not installed directly on the Proxmox host.

## Deployment sequence

### Phase 1: Platform deployment

Status: **In progress, restore validation pending**

Completed work:

1. Created a dedicated Debian VM in Proxmox.
2. Applied operating-system updates.
3. Configured hostname, internal DNS, and stable addressing.
4. Installed the supported Checkmk Community package.
5. Created the initial Checkmk site.
6. Validated the Checkmk web interface and site services.
7. Added the VM to centralized Proxmox backup coverage.

Remaining validation:

1. complete a fresh backup run
2. restore the Checkmk VM backup as a temporary isolated test VM
3. boot the restored guest with networking disconnected to avoid duplicate addressing
4. validate Debian startup, Checkmk installation, site presence, and site service state
5. remove the temporary restore-test VM after successful validation

The restore test is required before Phase 1 is considered complete. Successful backup creation alone is not treated as proof of recoverability.

### Phase 2: Low-risk onboarding

Begin with a non-hypervisor Linux guest so the Checkmk agent workflow can be learned and validated without modifying the Proxmox host.

Validation should include:

* successful agent registration or connectivity
* service discovery
* CPU, memory, filesystem, and network checks
* expected host and service state transitions
* clean removal or rollback of the agent if required

### Phase 3: Core guest coverage

After the initial guest is stable, expand monitoring to critical workloads such as:

* DNS services
* Vaultwarden
* file services
* Home Assistant availability
* monitoring platform endpoints

Application checks should validate meaningful service availability rather than only process state where practical.

### Phase 4: Network infrastructure

Evaluate SNMP monitoring for supported network devices.

Initial goals:

* device availability
* interface state and errors
* resource health
* hardware or environmental status where exposed

SNMP credentials and community strings must not be committed to the public repository.

### Phase 5: Proxmox host onboarding

Add the Proxmox host only after the Checkmk agent and service-discovery model is understood and documented.

The goal is to complement, not duplicate, the existing Node Exporter and Proxmox exporter metrics already collected by Prometheus.

## Relationship to Prometheus

Prometheus and Checkmk currently coexist.

The working model is:

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

Prometheus remains appropriate for questions such as utilization trends, historical metrics, rates, and capacity analysis.

Checkmk is intended for questions such as whether a host, filesystem, service, interface, or dependency is currently in an operational state.

## Alerting design

Checkmk includes its own notification system, so Prometheus Alertmanager is no longer considered a prerequisite for infrastructure alerting.

Before enabling broad notifications:

1. classify which conditions belong to Checkmk
2. determine whether Prometheus alert rules and Alertmanager are still needed for metrics-oriented conditions
3. identify overlapping checks
4. suppress or remove duplicate notification paths
5. test warning, critical, recovery, acknowledgement, and maintenance behavior

The preferred result is one authoritative notification path per operational condition.

## Backup and recovery

Checkmk should be protected at two layers where practical:

* Proxmox VM backup for full guest recovery
* Checkmk-native site backup for application-level recovery and migration

A scheduled Proxmox backup job now covers the Checkmk VM. A controlled restore test is the current Phase 1 acceptance requirement.

Checkmk-native site backup remains a planned second recovery layer after the initial platform restore path is validated.

Recovery documentation should include site restoration, version compatibility, credentials handling, and post-restore validation.

## Security and publication requirements

Public documentation must not include:

* internal IP addresses
* live hostnames where they expose unnecessary topology details
* SNMP credentials
* API tokens
* Checkmk automation secrets
* notification addresses or credentials
* authentication material

Sanitized examples should preserve architecture and operating concepts without exposing live configuration.

## Success criteria

The Checkmk evaluation will be considered successful when:

* the dedicated monitoring VM is stable and documented
* the VM backup has been successfully restored and validated
* at least one Linux guest is monitored successfully
* service discovery produces useful, actionable checks
* critical service availability can be represented clearly
* network monitoring can be added without excessive custom work
* backup and recovery paths are documented
* Checkmk and Prometheus responsibilities are clearly separated
* notification design avoids duplicate alerts

If Checkmk adds significant operational complexity without enough additional value, the deployment can be retired while retaining Prometheus and Grafana as the primary observability stack.
