# Checkmk Deployment Plan

## Purpose

This document defines the introduction of Checkmk as an infrastructure and service-monitoring layer for the homelab.

Checkmk is not intended to replace the existing Prometheus and Grafana stack. Prometheus remains the primary time-series metrics platform, while Grafana remains the primary visualization layer. Checkmk is being introduced for host state, service state, agent-based monitoring, supported network-device monitoring, active checks, and operational alerting where that model provides clearer infrastructure visibility.

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
| Network-device state | Checkmk where supported |
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

## Configuration structure

The Checkmk folder hierarchy is designed as a configuration-inheritance model rather than only a visual filing system.

The initial hierarchy separates production and development environments, then introduces technology or platform boundaries only where they provide useful inheritance behavior.

The operating model is:

* **Folders** define configuration inheritance boundaries and broad organizational structure.
* **Host tags** define controlled, mutually exclusive classifications that can be referenced consistently in rules.
* **Labels** carry flexible metadata that does not justify a rigid folder or tag dimension.

The initial custom classification taxonomy is defined and validated:

* Environment
* Service Criticality
* Platform
* Virtualization
* Service Class

Flexible labels are used for metadata such as service role, backup policy, and hypervisor platform. Built-in Checkmk classifications remain responsible for agent, SNMP, address-family, and discovered operating-system or device metadata.

A semantic host-naming standard is also established so Checkmk object identity remains independent of IP addressing and implementation details. See [`checkmk-configuration-standards.md`](checkmk-configuration-standards.md) for the naming convention, taxonomy, label conventions, and rule-targeting standard.

## Deployment sequence

### Phase 1: Platform deployment

Status: **Complete**

Completed work:

1. Created a dedicated Debian VM in Proxmox.
2. Applied operating-system updates.
3. Configured hostname, internal DNS, and stable addressing.
4. Installed the supported Checkmk Community package.
5. Created the initial Checkmk site.
6. Validated the Checkmk web interface and site services.
7. Added the VM to workload-specific Proxmox backup coverage.
8. Created a fresh backup and restored it as a temporary isolated VM.
9. Validated restored Debian startup, Checkmk installation, site presence, site service state, and site data.
10. Removed the temporary restore-test VM after successful validation.

Phase 1 acceptance is satisfied because both platform operation and VM-level recoverability have been demonstrated.

### Phase 2: Low-risk onboarding

Status: **Complete**

A non-hypervisor development Linux guest was used as the first managed host so the agent, service-discovery, classification, rule-targeting, and state-transition workflows could be validated without introducing risk to core infrastructure.

Validated onboarding workflow:

1. place the host in the appropriate Checkmk folder
2. install the Checkmk Linux agent package
3. register the agent with the Checkmk site
4. restrict the agent listener so only the monitoring server can reach it
5. validate agent connectivity
6. run service discovery
7. review discovered host labels and services before acceptance
8. activate the resulting monitoring configuration
9. confirm the host and accepted services report healthy states

Controlled state-transition testing was also completed for service failure, agent communication failure, and full host outage. In each case, Checkmk detected the expected state transition and returned automatically to a healthy state after recovery.

Phase 2 acceptance is satisfied because onboarding, inheritance, classification, rule targeting, problem detection, and recovery behavior have all been demonstrated on a low-risk host.

### Phase 3: Core guest coverage

Status: **Complete**

Phase 3 expanded the validated Checkmk model to core production workloads and confirmed that active checks can represent meaningful service behavior instead of only host reachability or process state.

Completed coverage includes:

* DNS service health, including internal-record and upstream-resolution checks
* password-management application health through a user-facing HTTPS check
* file-service health, mounted storage visibility, and authenticated SMB share access
* Home Assistant availability through an active web check without installing an unsupported guest agent
* monitoring-stack host health through a registered Linux agent
* Prometheus application availability through an active HTTP check
* Grafana application availability through an independent active HTTP check

Additional Phase 3 improvements included:

* semantic host naming with network addresses stored separately
* least-privilege credentials for authenticated active checks
* container-specific page-table memory thresholds targeted only to Linux containers
* validation that the revised page-table thresholds cleared non-actionable warnings while preserving the rest of the Linux memory check
* cleanup of unused container images on the monitoring VM after Checkmk highlighted elevated root-filesystem utilization
* narrow firewall access for the Checkmk agent listener on the monitoring VM

Phase 3 acceptance is satisfied because every planned core guest target is now represented through a healthy host or appliance object and at least one meaningful service-level validation where practical.

### Phase 4: Network infrastructure monitoring

Status: **Complete**

Phase 4 evaluated the actual management capabilities of the current core network devices and adjusted the monitoring design to match supported interfaces rather than forcing SNMP where it is unavailable.

Completed work:

* onboarded the core router as a production network device
* onboarded the core switch as a separate production network device
* classified both devices through the existing Environment, Service Criticality, Platform, Virtualization, and Service Class model
* confirmed both devices report healthy ICMP reachability
* verified that the current router firmware does not expose an SNMP service through its supported management interface
* verified that the current switch model does not support SNMP
* retained SNMP as the preferred deeper-monitoring method for future network devices that support it
* added independent HTTP management-interface availability checks for the router and switch
* confirmed both management-interface checks report healthy state

Phase 4 acceptance is satisfied because both current core network devices are monitored to the maximum practical level supported by their management capabilities: host reachability plus management-interface availability.

SNMP credentials, community strings, management addresses, and live hostnames must not be committed to the public repository.

### Phase 5: Proxmox host onboarding

Status: **Planned**

Add the Proxmox host only after the Checkmk agent and service-discovery model is understood and documented.

The goal is to complement, not duplicate, the existing Node Exporter and Proxmox exporter metrics already collected by Prometheus.

Hypervisor onboarding should include:

* general Linux host health
* storage and ZFS health where practical
* virtualization-platform state where Checkmk provides a supported integration path
* hardware health where available
* deliberate separation from historical metrics already owned by Prometheus

## Relationship to Prometheus

Prometheus and Checkmk currently coexist.

The working model is:

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

Prometheus remains appropriate for questions such as utilization trends, historical metrics, rates, and capacity analysis.

Checkmk is intended for questions such as whether a host, filesystem, service, interface, management endpoint, or dependency is currently in an operational state.

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

Checkmk is currently protected by a workload-specific Proxmox VM backup policy for full guest recovery.

VM-level recovery has been validated through an isolated restore test. The restored guest booted successfully and retained the Checkmk installation, monitoring site, site data, and service state.

A Checkmk-native site backup remains a planned second recovery layer for application-level recovery and migration.

The current Proxmox backup destination resides on the redundant primary ZFS pool. This protects against guest-level failures and a single mirrored-disk failure, but it is not an independent copy against complete pool or host loss. An independent backup destination remains a future resilience improvement.

## Security and publication requirements

Public documentation must not include:

* internal IP addresses
* live internal or public hostnames where they expose unnecessary topology details
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
* Linux guests are monitored successfully
* service discovery produces useful, actionable checks
* reusable classification and rule targeting are validated
* critical service availability can be represented clearly
* current network infrastructure is monitored to the level supported by the hardware
* backup and recovery paths are documented
* Checkmk and Prometheus responsibilities are clearly separated
* notification design avoids duplicate alerts

Phases 1 through 4 demonstrate that the platform, guest, service, active-check, and network-reachability models are operational. The remaining evaluation work is Proxmox host onboarding and notification design.
