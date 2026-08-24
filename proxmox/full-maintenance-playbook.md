# Full Proxmox Environment Maintenance Playbook

## Purpose

This playbook coordinates a complete maintenance cycle across the Proxmox hypervisor and active guest workloads.

It is intentionally broader than the individual guest runbooks. The playbook defines sequencing, monitoring suppression, inventory reconciliation, dependency handling, validation, and completion criteria for the entire environment.

Environment-specific hostnames, addresses, credentials, and secrets are intentionally omitted.

## Operating model

A full maintenance window is not considered complete when package updates finish. Completion requires:

* monitoring suppression before disruption
* live inventory reconciliation
* hypervisor maintenance
* guest operating-system or appliance maintenance as appropriate
* application-specific maintenance where applicable
* reboot handling
* service validation
* monitoring recovery
* cleanup and documentation

## Phase 1: Prepare the maintenance window

### Reconcile the live inventory

From the Proxmox host:

```bash
qm list
pct list
```

Classify each guest as:

* maintain now
* intentionally stopped
* intentionally excluded
* application-specific maintenance required

Do not rely solely on an older checklist.

### Review backup and rollback readiness

Before disruptive work:

* confirm scheduled backups have been completing
* confirm the backup target is available
* take temporary snapshots where the individual runbook calls for them
* create application-native backups when required, such as database dumps or Home Assistant backups
* confirm enough storage headroom exists for temporary snapshots or update writes

### Schedule Checkmk downtime

Follow the [Checkmk maintenance downtime standard](../monitoring/checkmk/maintenance-downtime.md).

For a full environment maintenance window, schedule downtime for every monitored host that will be intentionally disrupted.

When the hypervisor will reboot, this normally includes:

* the hypervisor host object
* every running monitored VM
* every running monitored container
* separately represented dependent services that will intentionally become unavailable

Schedule downtime before the first reboot, shutdown, service restart, or other disruptive action. Include time for validation and rollback.

Do not globally disable notifications. Hosts outside the maintenance scope should continue to alert normally.

## Phase 2: Establish the baseline

Before making changes, capture the current state.

On the Proxmox host:

```bash
pveversion
uname -r
pvesm status
zpool status
systemctl --failed
apt-mark showhold
```

For each guest, record the baseline required by its runbook, including service state, application version, backup state, and expected functionality.

Any unexplained pre-existing failure should be documented before maintenance begins so it is not incorrectly attributed to the change window.

## Phase 3: Maintain critical guest services before the hypervisor

When practical, maintain active guests before rebooting the hypervisor so package or application problems can be isolated from host-level changes.

Recommended sequence for a small single-host environment:

1. file services
2. password-management service
3. DNS and filtering service
4. development workload
5. Home Assistant
6. general monitoring stack
7. other active guests
8. Proxmox hypervisor
9. Checkmk monitoring server, when it is maintained in the same window

Dependency awareness matters more than the exact numeric order. A service that other maintenance steps actively depend on should remain available until those steps no longer require it.

## Phase 4: Execute guest-specific maintenance

Use the corresponding runbook for each workload:

* [File services container maintenance](runbooks/fileshare-container-maintenance.md)
* [Vaultwarden container maintenance](runbooks/vaultwarden-container-maintenance.md)
* [Pi-hole container maintenance](runbooks/pihole-container-maintenance.md)
* [Development VM maintenance](runbooks/development-vm-maintenance.md)
* [Home Assistant VM maintenance](runbooks/home-assistant-vm-maintenance.md)
* [Monitoring VM maintenance](runbooks/monitoring-vm-maintenance.md)

Each runbook must include or inherit these minimum controls:

* Checkmk downtime active before disruption
* pre-maintenance health check
* package or application update review
* rollback protection appropriate to the workload
* post-change system health validation
* application-level validation
* cleanup of temporary recovery artifacts

## Phase 5: Maintain the Proxmox hypervisor

Use the [Proxmox VE Hypervisor Maintenance Runbook](runbooks/hypervisor-maintenance.md).

Before rebooting the host, confirm that the Checkmk downtime covers every monitored workload that will disappear with the hypervisor.

Key host checks include:

```bash
apt update
apt list --upgradable
apt full-upgrade
pvesm status
systemctl --failed
```

If a new kernel is installed, reboot the host and perform the full post-reboot validation defined in the hypervisor runbook.

## Phase 6: Recover and validate guests after hypervisor reboot

After the hypervisor returns:

```bash
qm list
pct list
```

Compare the running state against the pre-maintenance inventory.

If an expected VM is stopped, inspect its autostart configuration:

```bash
qm config <vmid> | grep -E '^onboot|^startup'
```

Correct autostart configuration where policy requires the guest to return automatically after future host reboots.

Then validate each workload at the application level. A guest showing `running` is not sufficient evidence of service recovery.

## Phase 7: Maintain the Checkmk server

If the Checkmk server itself requires maintenance during the same window, perform it after the other infrastructure has returned to a known-good state whenever practical.

Before taking Checkmk offline:

* ensure scheduled downtime already covers the Checkmk server
* ensure other hosts being intentionally maintained remain covered for the expected duration
* record the last known monitoring state

After Checkmk maintenance:

* validate the Checkmk site services
* validate the web interface
* validate agent communication
* validate notification transport when relevant
* allow monitored hosts and services to refresh

If the Checkmk VM also hosts outbound notification infrastructure, validate that mail delivery remains operational after relevant package or configuration changes.

## Phase 8: Monitoring recovery

Do not remove downtime immediately when systems first become reachable.

First confirm:

* the hypervisor is healthy
* expected storage is active
* all intended guests are running
* application-level checks pass
* Checkmk has refreshed host and service states
* no unexpected WARN, CRIT, UNKNOWN, or DOWN conditions remain

Then remove scheduled downtime early or allow the fixed downtime to expire.

Any problem that remains after maintenance should become visible to normal alerting once the planned suppression window ends.

## Phase 9: Cleanup

After successful validation:

* remove temporary Proxmox snapshots created only for maintenance
* detach temporary installation or rescue media
* restore normal boot order where changed
* retain application-native backups according to their retention policy
* remove temporary files or dumps only when their required retention period has passed
* verify intentionally stopped guests remain stopped
* record material deviations and new operational requirements

## Phase 10: Record the change

Document at minimum:

* maintenance date
* affected systems
* package or application updates applied
* repository, DNS, storage, or boot changes made
* reboots performed
* problems encountered
* corrective actions
* validation results
* rollback actions if any
* monitoring downtime coverage and final monitoring state

Public repository entries must remain sanitized and must not expose live credentials, internal addresses, secrets, or unnecessary identifying details.

## Full maintenance completion criteria

The maintenance window is complete only when:

* all planned hosts were placed into Checkmk downtime before disruption
* backups or rollback controls required by the individual runbooks were verified
* guest maintenance completed according to workload-specific runbooks
* hypervisor maintenance completed successfully
* the expected kernel and Proxmox versions are active after reboot
* storage and systemd health are normal
* DNS and networking function normally
* all intended guests are running
* required autostart behavior is configured
* application-level validation passes for each maintained workload
* Checkmk has refreshed and reports the expected final state
* scheduled downtime has been removed or expired
* temporary rollback artifacts have been cleaned up
* the maintenance result has been documented
