# Roadmap

This roadmap captures planned homelab improvements derived from the internal operational backlog. Items are intentionally generalized for public documentation.

Day-to-day implementation work should be tracked in GitHub Projects and linked issues where appropriate. This document remains the high-level architectural roadmap and records major planned capabilities rather than acting as a task board.

## In progress

### Secure remote access

Deploy a mesh VPN for encrypted remote access to internal homelab services without exposing inbound WAN ports.

Success criteria:

* remote access works without router port forwarding
* internal DNS behavior remains predictable
* administrative access paths are documented
* rollback to local-only access is straightforward

### Checkmk infrastructure monitoring

Checkmk Community is deployed as an infrastructure and service-monitoring layer alongside the existing Prometheus and Grafana observability stack.

Phases 1 through 5 are complete. The rollout now covers platform deployment, low-risk validation, core guest services, current network infrastructure, and the Proxmox hypervisor. Hypervisor coverage includes Linux host state, Proxmox processes, ZFS pool health, physical-disk SMART health, and intentionally scoped host interfaces. The Proxmox API special agent remains disabled because of a compatibility crash in the current integration path, while the normal Linux-agent and existing Prometheus exporter coverage remain healthy.

Outbound Checkmk notification delivery is also operational. Checkmk Community hands HTML notifications to a local Postfix relay, which submits mail through a managed SMTP service using authenticated STARTTLS. Contact-group routing, a fallback destination, and end-to-end test delivery are validated. Notification volume will now be observed before introducing additional throttling, delays, or escalation behavior.

Goals:

* provide host and service state monitoring for critical infrastructure
* add straightforward Linux agent and supported network-device monitoring
* monitor service availability independently of metrics dashboards
* retain Prometheus for time-series metrics and Grafana visualization
* avoid unnecessary duplication between Checkmk and Prometheus
* maintain one authoritative notification path per operational condition

Next rollout sequence:

1. observe real notification volume and tune noisy or transient conditions only where operational evidence justifies it
2. validate recovery, acknowledgement, and scheduled-downtime notification behavior through controlled tests
3. revisit the Proxmox special-agent integration after a compatible Checkmk or Proxmox update
4. continue adding application-level checks where they provide meaningful operational evidence
5. use SNMP for future network devices where the hardware supports it
6. evaluate Alertmanager only for Prometheus-owned metric conditions that justify a separate routing path

See [`monitoring/checkmk-plan.md`](monitoring/checkmk-plan.md) for the detailed deployment plan, [`monitoring/checkmk-configuration-standards.md`](monitoring/checkmk-configuration-standards.md) for the current taxonomy and rule-targeting model, and [`monitoring/checkmk-notifications.md`](monitoring/checkmk-notifications.md) for the sanitized notification-delivery design.

### Backup verification and restore testing

Scheduled Proxmox backup coverage is organized into workload-specific jobs rather than one all-guests job. This isolates failures, supports workload-specific backup modes, and allows restore requirements to evolve independently.

A controlled Checkmk restore test has validated the recovery process for one critical VM. Restore testing should now become a repeatable operational practice rather than a one-time deployment task.

Goals:

* periodically verify that backups remain recoverable
* document restore steps for critical guests
* distinguish VM-level, container-level, application-level, and database-level recovery
* review workload-specific backup mode and retention as services change
* identify externally mounted datasets that require separate protection

## Planned

### Dedicated reverse-proxy service

Move reverse-proxy and TLS termination responsibilities into a dedicated guest rather than co-locating them with an application workload.

Goals:

* improve service isolation
* centralize certificate handling
* simplify multi-service routing
* reduce coupling between application lifecycle and ingress lifecycle

### Certificate-expiration monitoring

Add monitoring and alerting for certificate expiration.

Goals:

* detect approaching expiration before service impact
* surface status through the existing monitoring stack
* document alert thresholds and escalation behavior

### SSH hardening

Harden administrative SSH access across supported Linux guests.

Goals:

* use key-based authentication
* disable unnecessary password-based administrative access
* reduce or eliminate direct root login where practical
* preserve a documented recovery path before enforcing stricter settings

### UPS and power-event notifications

Add notification delivery for power and UPS state changes.

Goals:

* notify on power-loss events
* notify on degraded UPS state
* notify when automated shutdown behavior is triggered
* validate delivery without exposing mail credentials in public configuration

### Automatic recovery after power restoration

Evaluate and configure unattended infrastructure recovery after a complete power outage.

Goals:

* confirm host firmware power-restore behavior
* define service startup order
* ensure DNS and other dependencies become available before dependent services
* validate behavior through a controlled test

### Expanded backup coverage

Expand centralized backup coverage for selected large or rebuild-intensive datasets where guest-level backups do not include externally mounted or otherwise excluded data.

Goals:

* prioritize data that is expensive to recreate
* document storage-capacity impact
* identify bind mounts and external datasets excluded from guest backups
* add an independent backup destination when practical so recovery copies do not depend solely on the primary ZFS pool
* avoid treating replaceable application binaries as equivalent to unique data

## Continuous improvement

The following practices are ongoing rather than one-time roadmap items:

* reconcile the live guest inventory before maintenance
* maintain application-specific runbooks
* validate services after package and application updates
* remove obsolete snapshots after successful maintenance
* periodically review storage utilization and thin-provisioning risk
* periodically test backup restoration for critical workloads
* review notification quality and remove noisy or redundant alert paths
* sanitize configuration examples before publication
* update public documentation when the architecture materially changes
