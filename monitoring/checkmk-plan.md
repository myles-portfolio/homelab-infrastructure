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

## Configuration structure

The Checkmk folder hierarchy is designed as a configuration-inheritance model rather than only a visual filing system.

The initial hierarchy separates production and development environments, then introduces technology or platform boundaries only where they provide useful inheritance behavior.

The operating model is:

* **Folders** define configuration inheritance boundaries and broad organizational structure.
* **Host tags** define controlled, mutually exclusive classifications that can be referenced consistently in rules.
* **Labels** carry flexible metadata that does not justify a rigid folder or tag dimension.

The initial folder structure includes production and development roots, with Linux, infrastructure, monitoring, appliance, home-automation, and network branches introduced only where currently useful. Additional branches will be added as monitoring coverage expands rather than creating speculative empty structure.

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

The first Linux discovery produced a useful baseline covering agent health, CPU, memory, filesystem usage, interfaces, kernel performance, systemd summaries, time synchronization, TCP connections, and uptime.

The first host was classified using the custom taxonomy and flexible labels. A filesystem monitoring rule was targeted through folder scope and multiple custom host tags, then its effective service parameters were inspected to confirm that the intended warning and critical thresholds were applied.

Controlled state-transition testing was also completed for service failure, agent communication failure, and full host outage. In each case, Checkmk detected the expected state transition and returned automatically to a healthy state after recovery.

Phase 2 acceptance is satisfied because onboarding, inheritance, classification, rule targeting, problem detection, and recovery behavior have all been demonstrated on a low-risk host.

### Phase 3: Core guest coverage

Status: **In progress**

Expand monitoring to critical workloads using the standards validated in Phase 2.

Initial targets include:

* DNS services
* Vaultwarden
* file services
* Home Assistant availability
* monitoring platform endpoints

Application checks should validate meaningful service availability rather than only process state where practical.

#### DNS onboarding

The first production-class Phase 3 target is the DNS service.

Completed work:

* corrected the Checkmk host naming approach so semantic host IDs are used and network addresses are stored separately
* recreated the development and DNS hosts using the new naming convention
* onboarded the DNS container under the production infrastructure hierarchy
* installed and registered the Checkmk Linux agent
* resolved LXC namespace failures by enabling the required nesting capability rather than weakening systemd service hardening
* identified and removed an unused DHCPv6 configuration that delayed network readiness and DNS service startup after reboot
* validated agent connectivity and Linux service discovery
* added an active DNS check targeted through Production, Core Infrastructure, and `role:dns` metadata
* validated a known internal DNS record against its expected address

This active check proves that the DNS service is answering application-level queries correctly, rather than merely confirming that the host and DNS process are running.

The remaining DNS onboarding work is to add an upstream-resolution check using a stable public record. This will distinguish local-record availability from recursive or forwarded Internet DNS resolution.

A page-table memory warning on the small LXC remains under observation. Overall memory availability is healthy, and broad memory alert suppression was deliberately avoided because no sufficiently specific page-table threshold rule was identified.

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

Checkmk is currently protected by a workload-specific Proxmox VM backup policy for full guest recovery.

VM-level recovery has been validated through an isolated restore test. The restored guest booted successfully and retained the Checkmk installation, monitoring site, site data, and service state.

A Checkmk-native site backup remains a planned second recovery layer for application-level recovery and migration.

The current Proxmox backup destination resides on the redundant primary ZFS pool. This protects against guest-level failures and a single mirrored-disk failure, but it is not an independent copy against complete pool or host loss. An independent backup destination remains a future resilience improvement.

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
* reusable classification and rule targeting are validated
* critical service availability can be represented clearly
* network monitoring can be added without excessive custom work
* backup and recovery paths are documented
* Checkmk and Prometheus responsibilities are clearly separated
* notification design avoids duplicate alerts

If Checkmk adds significant operational complexity without enough additional value, the deployment can be retired while retaining Prometheus and Grafana as the primary observability stack.
