# Checkmk Deployment Plan

## Purpose

This document defines the introduction of Checkmk as an infrastructure and service-monitoring layer for the homelab.

Checkmk is not intended to replace the existing Prometheus and Grafana stack. Prometheus remains the primary time-series metrics platform, while Grafana remains the primary visualization layer. Checkmk is being introduced for host state, service state, agent-based monitoring, supported network-device monitoring, active checks, and operational alerting where that model provides clearer infrastructure visibility.

## Why add Checkmk

The existing Prometheus stack is effective for metrics collection, historical analysis, dashboards, and exporter-based telemetry. Checkmk addresses a different operational question: whether a host, service, interface, storage layer, or infrastructure component is currently healthy and requires attention.

The intended division of responsibilities is:

| Capability | Primary platform |
|---|---|
| Time-series metrics | Prometheus |
| Dashboards and visualization | Grafana |
| Exporter-based telemetry | Prometheus |
| Host and service state | Checkmk |
| Linux agent monitoring | Checkmk |
| Network-device state | Checkmk where supported |
| Storage and hardware state | Checkmk where supported |
| Metrics-based alert rules | Prometheus and Alertmanager, if retained |
| Infrastructure state notifications | Checkmk |

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

### Web access security

The Checkmk web interface is served through the system Apache front end and is protected with a publicly trusted ACME certificate.

The certificate workflow uses DNS-based validation through the DNS provider API rather than HTTP-based validation. This allows certificate issuance and renewal without exposing a public HTTP challenge endpoint.

The certificate client and DNS provider plug-in are isolated from the system Python environment. Automated renewal is scheduled through systemd using the certificate client environment that contains the required DNS plug-in. Renewal was validated with a full dry run, and a deploy hook reloads Apache after successful renewal so the new certificate is presented without manual intervention.

Private hostnames, DNS API credentials, certificate account details, and other live topology values are intentionally omitted from this repository.

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

Status: **Complete**

Phase 5 added the Proxmox VE hypervisor as a deliberately scoped production infrastructure host while preserving the separation between Checkmk state monitoring and Prometheus historical metrics.

Completed work:

* created a semantic hypervisor host object under the production Linux infrastructure hierarchy
* installed and registered the normal Checkmk Linux agent directly on the Proxmox node
* completed Linux service discovery and validated CPU, memory, disk I/O, systemd, time synchronization, filesystems, networking, and Proxmox process checks
* confirmed ZFS pool health through the native `zpool status` service and storage-pool capacity checks
* reduced interface noise by retaining only the physical host interface and intentional Proxmox bridge while disabling guest firewall and virtual interface services
* disabled per-guest ZFS backing-filesystem services so guest storage capacity remains owned by guest monitoring rather than duplicated at the hypervisor layer
* installed the current `smart_posix` Checkmk plug-in and validated SMART data for all physical drives
* confirmed the physical drives report healthy SMART state
* added targeted temperature thresholds for the installed high-capacity HDDs after validating their normal operating range, while leaving unrelated drive checks at their defaults
* verified all retained hypervisor services report healthy state

The Proxmox VE special-agent integration was also evaluated using a dedicated read-only Proxmox API account with the built-in auditor role. Network connectivity, API reachability, and authentication setup were validated, but the current special agent crashes while parsing the node API response because the expected timezone mapping is absent. The special-agent rule is therefore disabled until a compatible Checkmk or Proxmox update resolves the condition.

This blocker does not prevent Phase 5 acceptance because the normal Linux agent, ZFS checks, SMART monitoring, Proxmox process checks, and existing Prometheus exporters already provide strong complementary visibility into the hypervisor. The API integration remains a deferred enhancement rather than a requirement for baseline host monitoring.

### Phase 6: Notification delivery

Status: **Complete**

Phase 6 established a complete outbound notification path for Checkmk Community.

Completed work:

* verified a dedicated sender subdomain through the managed SMTP provider using public DNS authentication records
* created a dedicated SMTP identity for homelab monitoring and stored the credential outside the repository
* installed Postfix on the Checkmk VM as the local outbound mail transport
* configured Postfix to relay through the managed SMTP provider on TCP 587 using authenticated STARTTLS
* validated DNS resolution, TCP connectivity, TLS negotiation, Postfix relay authentication, provider acceptance, and mailbox delivery
* retained the default Checkmk HTML email method and refined the baseline notification rule to include host DOWN and UP transitions plus service WARN, CRIT, UNKNOWN, and OK transitions
* created an administrative contact group and assigned it to monitored hosts through a contact-group assignment rule
* configured the Checkmk contact email destination and a fallback email address
* validated Checkmk notification-rule matching, contact selection, notification plug-in execution, Postfix handoff, SMTP relay activity, and final mailbox delivery through an actual test notification

The initial notification rule intentionally remains broad while real alert volume is observed. Warning notifications will remain enabled initially so tuning decisions can be based on operational evidence rather than assumptions.

See [`checkmk-notifications.md`](checkmk-notifications.md) for the sanitized delivery architecture and configuration model.

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

Checkmk is intended for questions such as whether a host, filesystem, ZFS pool, physical disk, service, interface, management endpoint, or dependency is currently in an operational state.

## Alerting design

Checkmk is the authoritative notification path for infrastructure and service-state conditions that it owns. Prometheus Alertmanager is no longer considered a prerequisite for those alerts and should only be introduced for Prometheus-owned metric conditions that justify a separate routing component.

The baseline Checkmk notification design now includes:

1. contact-group-based recipient ownership
2. HTML email notifications for host DOWN and UP state changes
3. HTML email notifications for service WARN, CRIT, UNKNOWN, and OK state changes
4. a local Postfix relay using authenticated STARTTLS to a managed SMTP provider
5. a fallback notification destination for unmatched events
6. end-to-end delivery validation from Checkmk through the recipient mailbox

The preferred result remains one authoritative notification path per operational condition.

Next alerting work should focus on observed behavior rather than adding rules immediately. Candidate tuning includes transient-state delays, periodic reminders for persistent critical conditions, routing by classification, acknowledgement behavior, and scheduled-downtime suppression.

See [`alerting-roadmap.md`](alerting-roadmap.md) for the cross-platform ownership model and [`checkmk-notifications.md`](checkmk-notifications.md) for the notification-delivery implementation.

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
* SMTP authentication material
* password-manager contents
* DNS API credentials
* certificate private keys or account secrets

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
* the Proxmox hypervisor has meaningful host, storage, and physical-disk health coverage
* backup and recovery paths are documented
* Checkmk and Prometheus responsibilities are clearly separated
* notification design avoids duplicate alerts
* notification delivery is validated end to end
* the Checkmk web interface is protected by trusted HTTPS with validated automated certificate renewal

Phases 1 through 6 demonstrate that the platform, guest, service, active-check, network, hypervisor, storage, hardware-monitoring, and notification-delivery models are operational. Trusted HTTPS is also in place for the Checkmk web interface. Remaining work is incremental monitoring expansion, alert-quality tuning, and deferred compatibility improvements.
