# Roadmap

This roadmap captures planned homelab improvements derived from the internal operational backlog. Items are intentionally generalized for public documentation.

## In progress

### Secure remote access

Deploy a mesh VPN for encrypted remote access to internal homelab services without exposing inbound WAN ports.

Success criteria:

* remote access works without router port forwarding
* internal DNS behavior remains predictable
* administrative access paths are documented
* rollback to local-only access is straightforward

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

### Backup verification and restore testing

Implement a repeatable process for validating guest backups and periodically testing restoration.

Goals:

* verify backups are not only present but recoverable
* document restore steps for critical guests
* distinguish VM-level, container-level, application-level, and database-level recovery

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

Expand centralized backup coverage for selected large or rebuild-intensive datasets.

Goals:

* prioritize data that is expensive to recreate
* document storage-capacity impact
* avoid treating replaceable application binaries as equivalent to unique data

## Continuous improvement

The following practices are ongoing rather than one-time roadmap items:

* reconcile the live guest inventory before maintenance
* maintain application-specific runbooks
* validate services after package and application updates
* remove obsolete snapshots after successful maintenance
* periodically review storage utilization and thin-provisioning risk
* sanitize configuration examples before publication
* update public documentation when the architecture materially changes
