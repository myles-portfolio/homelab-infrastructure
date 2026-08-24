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

| Guest | Role | Maintenance focus |
|---|---|---|
| CT 101 | File services | OS packages, file-sharing services, client validation |
| CT 102 | Media server | Preserve intentional stopped state when applicable |
| VM 103 | Metrics and visualization | Ubuntu, Docker, Prometheus, Grafana, NUT exporter, QEMU Guest Agent |
| CT 104 | Password manager | OS packages plus Vaultwarden image refresh, recreation, and client validation |
| CT 105 | DNS / filtering | OS packages plus Pi-hole application and DNS validation |
| VM 106 | Development workload | Ubuntu, PostgreSQL, application database backup and query validation |
| VM 107 | Home automation | Home Assistant-specific maintenance, backup, and integration validation |
| VM 108 | Infrastructure monitoring | Debian, Checkmk site health, notification transport, monitoring recovery |

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
10. Remove temporary snapshots and installation media after validation.
11. Confirm Checkmk returns to the expected final state before maintenance downtime is removed.
12. Document deviations discovered during the maintenance window.

## Application-level validation

Examples of service-specific validation include:

* file shares are reachable from a client
* DNS resolves expected records
* password manager web and extension access succeeds
* monitoring targets are healthy and dashboards load
* PostgreSQL accepts application database queries after maintenance
* Home Assistant integrations, automations, dashboards, and backup locations remain operational
* Checkmk host and service state, site health, and notification delivery remain operational

## Maintenance orchestration

* [Full Proxmox environment maintenance playbook](full-maintenance-playbook.md)
* [Checkmk maintenance downtime standard](../monitoring/checkmk/maintenance-downtime.md)

## Runbooks

* [Proxmox VE hypervisor maintenance](runbooks/hypervisor-maintenance.md)
* [File services container maintenance](runbooks/fileshare-container-maintenance.md)
* [Monitoring VM maintenance](runbooks/monitoring-vm-maintenance.md)
* [Vaultwarden container maintenance](runbooks/vaultwarden-container-maintenance.md)
* [Pi-hole container maintenance](runbooks/pihole-container-maintenance.md)
* [Development VM maintenance](runbooks/development-vm-maintenance.md)
* [Home Assistant VM maintenance](runbooks/home-assistant-vm-maintenance.md)
* [Checkmk VM maintenance](runbooks/checkmk-vm-maintenance.md)
