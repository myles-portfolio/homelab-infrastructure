# Checkmk Deployment Plan

## Purpose

This document records the sanitized deployment and operating model for Checkmk as the homelab's primary infrastructure and service-state monitoring platform.

Checkmk complements rather than replaces Prometheus and Grafana. Prometheus remains focused on time-series metrics, while Grafana remains the visualization layer.

## Monitoring responsibilities

| Capability | Primary platform |
|---|---|
| Time-series metrics | Prometheus |
| Dashboards and visualization | Grafana |
| Exporter-based telemetry | Prometheus |
| Host and service state | Checkmk |
| Linux agent monitoring | Checkmk |
| Network-device state | Checkmk where supported |
| Storage and hardware state | Checkmk where supported |
| Infrastructure notifications | Checkmk |
| Metrics-based alerting | Prometheus and Alertmanager only where justified |

The same operational condition should not normally generate duplicate notifications from multiple platforms.

## Deployment model

Checkmk Community runs on a dedicated Linux virtual machine hosted by Proxmox VE.

The public repository intentionally omits exact VM sizing, operating-system patch level, site identifier, internal addresses, hostnames, and software version fingerprints unless they are necessary to explain a compatibility issue.

The dedicated VM provides workload isolation, a clear maintenance boundary, and independent recovery controls.

Checkmk is not installed directly on the hypervisor.

## Web access security

The Checkmk web interface is presented through the dedicated Nginx reverse proxy rather than using the monitoring VM as the client-facing TLS endpoint.

```text
Client
  |
  v
Internal name resolution
  |
  v
Nginx reverse proxy
  |
  v
Checkmk backend
```

The reverse proxy presents trusted TLS while certificate private keys remain isolated from the Checkmk host.

The migration was staged and validated before DNS cutover. Public documentation records the method rather than the live hostname, address changes, or resolver records.

Certificate renewal automation and certificate-expiration monitoring remain explicit lifecycle controls.

## Configuration structure

The Checkmk folder hierarchy is used as a configuration-inheritance model rather than only a visual filing system.

The operating model is:

* **Folders** define broad inheritance boundaries.
* **Host tags** define controlled classifications that can be referenced consistently in rules.
* **Labels** carry flexible metadata that does not justify a rigid taxonomy.

The current classification model includes dimensions such as environment, service criticality, platform, virtualization type, and service class.

Semantic host naming keeps monitoring object identity independent of IP addressing and implementation details.

See [`checkmk-configuration-standards.md`](checkmk-configuration-standards.md) for the naming, taxonomy, and rule-targeting standards.

## Deployment sequence

### Phase 1: Platform deployment

Status: **Complete**

Validated outcomes:

* dedicated monitoring VM deployed and patched
* Checkmk Community installed and site services validated
* web access verified
* workload-specific backup coverage assigned
* isolated guest restore completed successfully
* restored monitoring application state validated before temporary recovery resources were removed

### Phase 2: Low-risk onboarding

Status: **Complete**

A low-risk Linux workload was used to validate the standard onboarding workflow before expanding to core infrastructure.

Validated workflow:

1. classify the host
2. install and register the Linux agent
3. restrict agent reachability appropriately
4. validate connectivity
5. run service discovery
6. review discovered labels and services
7. activate configuration
8. confirm healthy final state

Controlled testing demonstrated expected problem and recovery state transitions.

### Phase 3: Core guest coverage

Status: **Complete**

Coverage was expanded to production workloads using both agents and active checks where appropriate.

Representative monitoring includes:

* DNS health
* private web application availability
* authenticated file-service checks
* Home Assistant availability
* monitoring-stack host health
* Prometheus and Grafana availability
* filesystem and systemd state

Configuration tuning remains narrowly targeted so one noisy check does not weaken unrelated monitoring.

### Phase 4: Network infrastructure monitoring

Status: **Complete**

Current network devices are monitored to the level supported by their management capabilities.

The design uses reachability and management-interface checks where deeper telemetry is unavailable, while retaining SNMP as the preferred deeper-monitoring method for future supported devices.

Live management addresses, model-specific limitations, community strings, and credentials are not published.

### Phase 5: Hypervisor onboarding

Status: **Complete**

The Proxmox host is monitored as a deliberately scoped infrastructure target.

Coverage includes Linux health, storage state, selected interfaces, processes, disk health, and other host-level conditions that complement existing Prometheus telemetry.

A deeper API-based integration was evaluated but deferred because of a current compatibility problem. The public repository intentionally avoids documenting unnecessary parser details, live API identities, or exact platform versions that could improve fingerprinting.

### Phase 6: Notification delivery

Status: **Complete**

A complete outbound notification path is operational using a local mail transport and managed SMTP relay.

Validation included:

* monitoring-rule matching
* contact selection
* local mail handoff
* authenticated relay delivery
* final mailbox receipt

Provider-specific endpoints, sender-domain details, credentials, and live recipient identities are omitted.

See [`checkmk-notifications.md`](checkmk-notifications.md) for the sanitized delivery model.

### Incremental infrastructure expansion

Status: **Ongoing**

New infrastructure workloads are onboarded using the same classification, agent-registration, discovery, activation, and validation pattern.

Recent examples include shared ingress and secure remote-access infrastructure. Public documentation describes the monitoring pattern rather than publishing live object names, addresses, or policy details.

## Relationship to Prometheus

```text
Infrastructure
   |
   +--> Checkmk agents / active checks / supported device monitoring
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

Prometheus is used for historical metrics, rates, and capacity trends.

Checkmk is used for current host, service, storage, interface, and dependency state.

## Alerting design

Checkmk is the authoritative notification path for infrastructure and service-state conditions that it owns.

Alertmanager remains optional and should only be introduced for Prometheus-owned metric conditions that justify a separate routing component.

Alert tuning is evidence-based and may include transient-state delays, reminders for persistent conditions, classification-based routing, acknowledgement behavior, scheduled-downtime suppression, and certificate-expiration visibility.

The preferred result is one authoritative notification path per operational condition.

## Backup and recovery

The monitoring platform is protected by workload-appropriate guest backup coverage and has been validated through an isolated restore test.

Application-native backup remains a planned additional recovery layer where it provides useful portability or granularity.

The public repository intentionally omits exact backup placement, storage topology, schedules, retention values, and failure combinations that could reveal the live recovery boundary.

See [`../../backup-recovery/README.md`](../../backup-recovery/README.md) for the sanitized recovery model.

## Security and publication requirements

Public documentation must not include:

* internal IP addresses or live hostnames where they add no technical value
* monitoring credentials, API tokens, or automation secrets
* notification addresses or SMTP authentication material
* password-manager contents
* DNS API credentials
* certificate private keys or account secrets
* exact monitoring object inventories that reconstruct the live topology
* exact platform sizing and patch levels unless central to the lesson being documented

Sanitized examples should preserve architecture, operating principles, and validation methods without exposing live configuration.

## Success criteria

The Checkmk deployment is considered operational because:

* the monitoring platform is stable and documented
* guest-level recovery has been demonstrated
* Linux workloads are monitored successfully
* service discovery and active checks provide actionable state
* classification and rule targeting are reusable
* network infrastructure is monitored to supported capability
* the hypervisor has meaningful host and storage visibility
* notification delivery is validated end to end
* Checkmk and Prometheus responsibilities are clearly separated
* the web interface is protected by trusted TLS through centralized ingress
* monitoring infrastructure is itself monitored and backed up

Remaining work is incremental coverage expansion, certificate-expiration monitoring, alert-quality tuning, and deferred compatibility improvements.
