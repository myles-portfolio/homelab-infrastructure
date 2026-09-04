# Proxmox VE Operations

## Overview

This section documents sanitized maintenance and service-validation practices for a small Proxmox VE homelab environment.

The operating model treats the hypervisor and each guest as separate maintenance domains. A successful package upgrade is not considered sufficient by itself. Each guest must also pass application-level validation appropriate to its role.

Planned maintenance on monitored infrastructure also requires Checkmk scheduled downtime before intentional disruption. This keeps maintenance-generated state changes from producing actionable alerts while preserving normal monitoring for systems outside the maintenance scope.

## Guest inventory model

Every maintenance cycle begins by reconciling the live Proxmox guest inventory against the runbook. Each guest is explicitly classified as one of:

* maintain now
* intentionally stopped
* intentionally excluded
* application-specific maintenance required

This prevents guests from being omitted simply because they were missing from an older checklist.

### Current sanitized inventory

| Guest type | Role | Maintenance focus |
|---|---|---|
| LXC | File services | OS packages, file-sharing services, client validation |
| LXC | Media server | Preserve intentional stopped state when applicable |
| VM | Metrics and visualization | Ubuntu, Docker, Prometheus, Grafana, NUT exporter, QEMU Guest Agent |
| LXC | Password manager | OS packages plus Vaultwarden image refresh, recreation, and client validation |
| LXC | DNS / filtering | OS packages plus Pi-hole application and DNS validation |
| VM | Development workload | Ubuntu, PostgreSQL, application database backup and query validation |
| VM | Home automation | Home Assistant-specific maintenance, backup, and integration validation |
| VM | Infrastructure monitoring | Debian, Checkmk site health, notification transport, monitoring recovery |
| LXC | Personal knowledge / RAG backend | Debian, PostgreSQL, pgvector, ingestion service, API health, database connectivity |
| LXC | Reverse proxy / TLS ingress | Debian, Nginx, certificate lifecycle, backend routing, SSH hardening, Checkmk agent health |

The public inventory intentionally omits live Proxmox guest IDs, hostnames, and addresses. Operational procedures use placeholders such as `<vmid>` and `<ctid>` where an identifier is required.

### Personal knowledge / RAG backend

A dedicated unprivileged Debian 13 container now provides the infrastructure foundation for a private-first personal knowledge and retrieval system.

Sanitized starting allocation:

```text
vCPU: 2
RAM: 2 GB
Swap: 512 MB
Root disk: 32 GB
Autostart: enabled
```

Current infrastructure components:

* PostgreSQL 17
* pgvector 0.8.0
* dedicated application database and non-superuser database identity
* LAN-restricted database access for development
* Checkmk Linux agent with TLS registration
* Checkmk PostgreSQL agent plug-in for database health and performance checks
* future Python ingestion and FastAPI application services

The source knowledge vault is stored separately on ZFS-backed storage and synchronized through the file-services workload. The RAG database is derived state and should remain rebuildable from protected source Markdown. Exact addresses, database credentials, vault paths, and guest identifiers are intentionally excluded from this repository.

Checkmk onboarding is complete for host-level and PostgreSQL monitoring. Current coverage includes availability, CPU, memory, filesystem, systemd state, PostgreSQL instance health, connection metrics, database size and statistics, locks, process counts, query duration, bloat, analyze, and vacuum state. Notification routing and delivery have also been validated. Application-specific RAG API and ingestion checks remain future work until those services enter operation. Backup coverage remains a separate follow-up requirement before this workload is treated as production-ready.

### Reverse proxy / TLS ingress

A dedicated unprivileged Debian 13 container now hosts Nginx as the central reverse proxy and TLS termination layer for selected internal services.

Sanitized starting allocation:

```text
vCPU: 1
RAM: 1 GB
Swap: 512 MB
Root disk: 8 GB
Autostart: enabled
```

Current implementation characteristics:

* Nginx reverse proxy with hostname based routing
* wildcard certificate issued through ACME DNS challenge validation
* non-root administrative account with Ed25519 key based SSH authentication
* direct root password SSH disabled
* password based SSH disabled after validating a second key-authenticated session
* unnecessary local mail service removed from the workload
* Checkmk Linux agent with TLS registration
* service migration performed incrementally, with the monitoring web interface used as the first production validation target

The reverse proxy is treated as core infrastructure because multiple service access paths will depend on it as migrations continue. Scheduled guest backup coverage and certificate lifecycle monitoring remain follow-up requirements.

## Maintenance principles

1. Schedule Checkmk downtime for every monitored host that will be intentionally disrupted.
2. Reconcile the actual guest inventory before maintenance.
3. Record intentionally stopped or excluded guests rather than silently skipping them.
4. Take rollback protection when the change risk justifies it.
5. Check storage health before generating substantial snapshot or upgrade writes.
6. Update the guest operating system separately from application containers when practical.
7. Validate systemd health after package changes.
8. Validate application services after operating-system changes.
9. Confirm Proxmox guest communication where QEMU Guest Agent is expected.
10. Validate hypervisor system-mail delivery when Postfix or notification configuration changes.
11. Maintain trusted TLS on the Proxmox management interface and verify ACME renewal remains functional.
12. Validate reverse-proxy routing, certificate trust, and dependent service access after Nginx maintenance.
13. Remove temporary snapshots and installation media after validation.
14. Confirm Checkmk returns to the expected final state before maintenance downtime is removed.
15. Document deviations discovered during the maintenance window.

## Application-level validation

Examples of service-specific validation include:

* file shares are reachable from a client
* DNS resolves expected records
* password manager web and extension access succeeds
* monitoring targets are healthy and dashboards load
* PostgreSQL accepts application database queries after maintenance
* Home Assistant integrations, automations, dashboards, and backup locations remain operational
* Checkmk host and service state, site health, and notification delivery remain operational
* Proxmox system mail successfully leaves the host through the authenticated relay and the deferred queue remains empty
* the Proxmox management interface presents a trusted certificate for its configured management FQDN
* the knowledge-platform database accepts authenticated application connections and the vector extension remains available
* the knowledge-platform PostgreSQL monitoring remains healthy after maintenance
* the knowledge-platform API and ingestion workflow are validated independently once those application components enter service
* the reverse proxy listens on expected ports, presents the expected trusted certificate, routes each configured hostname to the correct backend, and remains green in Checkmk

## Maintenance orchestration

* [Full Proxmox environment maintenance playbook](full-maintenance-playbook.md)
* [Checkmk maintenance downtime standard](../monitoring/checkmk/maintenance-downtime.md)

## Runbooks

* [Proxmox VE hypervisor maintenance](runbooks/hypervisor-maintenance.md)
* [Proxmox ACME TLS certificate](runbooks/acme-tls-certificate.md)
* [Proxmox system mail relay](runbooks/system-mail-relay.md)
* [File services container maintenance](runbooks/fileshare-container-maintenance.md)
* [Monitoring VM maintenance](runbooks/monitoring-vm-maintenance.md)
* [Vaultwarden container maintenance](runbooks/vaultwarden-container-maintenance.md)
* [Pi-hole container maintenance](runbooks/pihole-container-maintenance.md)
* [Development VM maintenance](runbooks/development-vm-maintenance.md)
* [Home Assistant VM maintenance](runbooks/home-assistant-vm-maintenance.md)
* [Checkmk VM maintenance](runbooks/checkmk-vm-maintenance.md)
