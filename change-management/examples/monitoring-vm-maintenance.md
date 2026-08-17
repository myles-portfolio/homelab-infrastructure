# Change Example: Monitoring VM Maintenance and Recovery

## Summary

Performed full maintenance on the dedicated monitoring VM, including access recovery, operating-system package updates, Docker application refresh, QEMU Guest Agent remediation, and application-level validation.

## Change type

Routine maintenance with an emergency access-recovery component

## Risk

Medium

## Problem statement

The monitoring VM had fallen outside the established maintenance checklist and needed to be brought back into regular maintenance coverage. During the maintenance window, administrative access was also unavailable because the expected credentials were not documented and QEMU Guest Agent communication was not functional.

The goal was to restore manageable access, update the platform and monitoring applications, validate service health, and incorporate the VM into the recurring maintenance model.

## Scope

Affected components included:

* Ubuntu guest operating system
* QEMU Guest Agent
* Docker Engine and Compose components
* Prometheus
* Grafana
* NUT exporter
* Proxmox snapshot and guest-management workflows

Exact addresses, hostnames, credentials, and storage identifiers are intentionally excluded.

## Implementation

### Access recovery

1. Confirmed the VM was running.
2. Confirmed QEMU Guest Agent was configured in Proxmox but not available inside the guest.
3. Identified the guest network address from Proxmox network information.
4. Used temporary Ubuntu installation media as a rescue environment.
5. Mounted the installed root filesystem and identified the local administrative account.
6. Reset the account password from a chroot environment.
7. Restored the VM boot order and removed the temporary installation media after recovery.

### Operating-system maintenance

1. Reviewed available Ubuntu package updates.
2. Identified Docker, container runtime, system, and kernel-related updates in the pending set.
3. Took a temporary Proxmox snapshot before applying changes.
4. Checked thin-pool utilization after Proxmox reported an overcommit warning.
5. Applied the package upgrade.
6. Confirmed that no failed systemd units remained.

### Guest-management remediation

Installed QEMU Guest Agent inside the VM and verified end-to-end communication from the Proxmox host.

### Monitoring stack refresh

The monitoring stack was managed through Docker Compose and included:

* Prometheus
* Grafana
* NUT exporter

The application images were pulled intentionally as a separate change from guest OS maintenance, then the Compose stack was recreated.

## Validation

Validation included:

* all monitoring containers running after recreation
* Grafana responding through its local HTTP endpoint
* Prometheus responding through its local HTTP endpoint
* Prometheus reporting the NUT exporter scrape target as healthy
* no failed systemd services
* successful QEMU Guest Agent ping from Proxmox

The NUT exporter was not expected to respond directly on the VM's localhost interface because its container port was exposed within Docker networking but not published to the host. Prometheus target health was therefore used as the correct service-level validation.

## Impact

The monitoring VM experienced planned restarts during maintenance. Monitoring availability was briefly interrupted while the guest and Compose services restarted.

## Rollback

A temporary Proxmox snapshot provided VM-level rollback protection throughout the maintenance window. The snapshot was removed only after operating-system, guest-agent, and monitoring application validation succeeded.

## Engineering considerations

Several operational improvements came out of this change:

* maintenance scope must be reconciled against the live guest inventory rather than relying on remembered guest lists
* guest-management tooling should be validated before it is needed for recovery
* Docker application updates should be treated separately from guest package maintenance
* thin-provisioning warnings should be evaluated using actual pool utilization, not only provisioned virtual capacity
* health checks should follow the actual network architecture of the service rather than assuming every exposed container port is published to the host

## Lessons

The most significant outcome was not the package update itself. The maintenance window exposed gaps in credential documentation, guest-agent deployment, inventory management, and service-validation assumptions. Correcting those gaps improved both the monitoring service and the maintenance process used for the rest of the homelab.
