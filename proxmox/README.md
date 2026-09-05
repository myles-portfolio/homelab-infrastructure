# Proxmox VE Operations

## Overview

This section documents sanitized maintenance and service-validation practices for a small Proxmox VE homelab environment.

The operating model treats the hypervisor and each guest as separate maintenance domains. A successful package upgrade is not considered sufficient by itself. Each guest must also pass application-level validation appropriate to its role.

Planned maintenance on monitored infrastructure also uses Checkmk scheduled downtime before intentional disruption so maintenance-generated state changes do not create actionable alerts.

## Guest inventory model

Every maintenance cycle begins by reconciling the live Proxmox guest inventory against the runbook. Each guest is classified as one of:

* maintain now
* intentionally stopped
* intentionally excluded
* application-specific maintenance required

This prevents workloads from being omitted simply because an older checklist did not include them.

### Sanitized workload inventory

The public repository documents workload categories rather than live guest identifiers, hostnames, addresses, exact resource allocations, or software patch levels.

| Guest type | Role | Maintenance focus |
|---|---|---|
| Linux container | File services | OS packages, file-sharing services, client validation |
| Virtual machine | Metrics and visualization | Linux, container runtime, metrics stack, visualization, guest integration |
| Linux container | Password management | OS packages, application refresh, recreation, client validation |
| Linux container | DNS and filtering | OS packages, application health, DNS validation |
| Virtual machine | Development application | Linux, database, application backup, query and workflow validation |
| Virtual machine | Home automation | Appliance-specific maintenance, backup, integration validation |
| Virtual machine | Infrastructure monitoring | Monitoring-site health, notifications, monitoring recovery |
| Linux container | Personal knowledge and retrieval | Database, vector extension, ingestion, API, monitoring |
| Linux container | Reverse proxy and TLS ingress | Nginx, certificate lifecycle, routing, SSH hardening, monitoring |
| Linux container | Secure remote access | Overlay networking, route advertisement, access policy, monitoring |

Media hosting is currently deferred until dedicated storage infrastructure is available.

## Workload-specific operating patterns

### Personal knowledge and retrieval backend

A dedicated unprivileged Linux container provides the infrastructure foundation for a private-first personal knowledge and retrieval system.

The public documentation intentionally omits exact guest sizing, operating-system patch level, database version, vector-extension version, addresses, credentials, and storage paths.

Operational concerns include:

* database availability and authenticated application access
* vector-extension availability
* filesystem and systemd health
* rebuildability of derived retrieval data from protected source content
* independent monitoring of host and database health
* application-specific API and ingestion checks as those services enter operation
* workload-appropriate backup coverage and later restore validation

### Reverse proxy and TLS ingress

A dedicated unprivileged Linux container hosts Nginx as the central reverse proxy and TLS termination layer for selected internal services.

Public documentation preserves the security and operating pattern without publishing the live host identity, exact sizing, listener inventory, certificate names, backend addresses, or SSH configuration values.

Operational characteristics include:

* hostname-based routing
* wildcard certificate lifecycle through ACME DNS validation
* non-root administration and key-based SSH
* removal or disabling of unnecessary services
* TLS-registered Checkmk monitoring
* incremental application migration and rollback validation
* scheduled guest backup coverage

### Secure remote access gateway

A dedicated unprivileged Linux container provides Tailscale subnet routing for authenticated encrypted remote access.

The public repository documents the role without exposing live network ranges, route advertisements, exact policy rules, guest sizing, device identities, or detailed shared failure domains.

Operational characteristics include:

* dedicated subnet-router role with no application hosting
* least-privilege support for overlay networking
* persistent forwarding required for the selected private address family
* no general-purpose exit-node role
* deny-by-default remote-access policy with explicit allowed paths
* both positive and negative access validation
* TLS-registered Checkmk monitoring
* scheduled guest backup coverage

Proxmox management is intentionally not published through Nginx. Remote administrative reachability uses the secure overlay network instead.

## Maintenance principles

1. Schedule Checkmk downtime for monitored hosts that will be intentionally disrupted.
2. Reconcile the actual guest inventory before maintenance.
3. Record intentionally stopped or excluded guests rather than silently skipping them.
4. Take rollback protection when change risk justifies it.
5. Check storage health before generating substantial maintenance writes.
6. Update guest operating systems separately from application containers when practical.
7. Validate systemd health after package changes.
8. Validate application services after operating-system changes.
9. Confirm guest integration where expected.
10. Validate system-mail delivery when mail configuration changes.
11. Maintain trusted TLS on the hypervisor management interface.
12. Validate reverse-proxy routing and certificate trust after Nginx maintenance.
13. Validate remote-access service state and intended policy behavior after gateway maintenance.
14. Remove temporary snapshots and installation media after validation.
15. Confirm Checkmk returns to the expected final state before maintenance downtime is removed.
16. Document deviations discovered during the maintenance window.

## Application-level validation

Examples of service-specific validation include:

* file shares are reachable from a client
* DNS resolves expected records
* password-management clients can authenticate and operate normally
* monitoring targets are healthy and dashboards load
* application databases accept expected queries
* development application workflows remain functional
* Home Assistant integrations, automations, dashboards, and backup targets remain operational
* Checkmk site health and notification delivery remain operational
* system mail leaves the hypervisor through the configured relay path
* the hypervisor management interface presents trusted TLS
* knowledge-platform database and retrieval dependencies remain available
* the reverse proxy presents expected TLS and routes configured names correctly
* the remote-access gateway permits intended paths and rejects restricted paths

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
* [Reverse proxy container maintenance](runbooks/reverse-proxy-container-maintenance.md)
