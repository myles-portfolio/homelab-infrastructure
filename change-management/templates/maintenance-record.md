# Maintenance Record Template

## Maintenance window

**Date:** YYYY-MM-DD  
**Scope:** Host, VM, container, or service  
**Status:** Planned | In Progress | Complete | Rolled Back | Failed

## Inventory reconciliation

List every guest or service reviewed at the start of the maintenance cycle and classify it.

| System | Role | Disposition |
|---|---|---|
| Example VM | Monitoring | Maintain now |
| Example CT | Media | Intentionally stopped |

## Pre-maintenance checks

* service state confirmed
* failed-service check completed
* backup or snapshot decision recorded
* storage health checked when relevant
* application version or workload state recorded

## Changes performed

Document package updates, application updates, configuration changes, container recreation, or other actions.

## Validation

Record service-specific validation rather than only package-manager success.

Examples:

* `systemctl --failed` returns no failed units
* QEMU Guest Agent responds from Proxmox
* web endpoint responds
* database query succeeds
* Prometheus target is healthy
* DNS resolution succeeds
* file share accepts a test write
* Home Assistant automation executes as expected

## Cleanup

* temporary snapshots removed
* installation media detached
* temporary test artifacts removed
* backup artifacts retained according to policy

## Rollback readiness

Document what rollback controls existed during the maintenance window and whether they were removed after successful validation.

## Findings and follow-up

Capture newly discovered maintenance requirements, missing inventory items, documentation gaps, or future improvements.
