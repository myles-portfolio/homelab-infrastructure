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

Phase 1 platform deployment, Phase 2 low-risk Linux onboarding, and Phase 3 core guest coverage are complete. Phase 3 extended the validated monitoring model across foundational infrastructure, application services, an appliance-style VM, file services, and the existing monitoring stack. Application-level checks now validate meaningful service behavior in addition to host state.

Goals:

* provide host and service state monitoring for critical infrastructure
* add straightforward Linux agent and SNMP-based monitoring
* monitor service availability independently of metrics dashboards
* retain Prometheus for time-series metrics and Grafana visualization
* avoid unnecessary duplication between Checkmk and Prometheus
* define alert ownership and notification behavior before broad rollout

Next rollout sequence:

1. add network-device monitoring where supported
2. onboard the Proxmox host after the monitoring model has been validated across core guests
3. define notification ownership between Checkmk and the Prometheus alerting path
4. continue adding application-level checks where they provide meaningful operational evidence

See [`monitoring/checkmk-plan.md`](monitoring/checkmk-plan.md) for the detailed deployment plan and [`monitoring/checkmk-configuration-standards.md`](monitoring/checkmk-configuration-standards.md) for the current taxonomy and rule-targeting model.

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
* sanitize configuration examples before publication
* update public documentation when the architecture materially changes
