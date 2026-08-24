# Proxmox VE Hypervisor Maintenance Runbook

## Purpose

This runbook documents a sanitized maintenance workflow for a standalone Proxmox VE hypervisor hosting Linux containers and virtual machines.

The workflow covers repository validation, DNS validation, package maintenance, kernel updates, guest startup behavior, storage validation, monitoring suppression, and post-reboot recovery.

Environment-specific hostnames, addresses, credentials, and secrets are intentionally omitted.

## Maintenance principles

1. Schedule Checkmk downtime before any disruptive action.
2. Reconcile the live guest inventory before maintenance.
3. Confirm storage health before package or reboot activity.
4. Validate package repositories before upgrading.
5. Review upgrade transactions before accepting removals or dependency changes.
6. Reboot when a new Proxmox kernel is installed.
7. Validate storage, systemd health, networking, DNS, and guest recovery after reboot.
8. Verify guest autostart behavior rather than assuming all workloads will return automatically.
9. Validate unexpected firmware or EFI warnings before treating maintenance as complete.
10. Remove monitoring downtime only after the environment has returned to its expected state.

## Pre-maintenance monitoring suppression

Follow the [Checkmk maintenance downtime standard](../../monitoring/checkmk/maintenance-downtime.md).

For hypervisor maintenance that will reboot the host, schedule downtime for:

* the Proxmox hypervisor host object
* every monitored guest expected to become unavailable
* any separately monitored dependent service that will intentionally become unavailable

Use a fixed downtime long enough to include package installation, reboot, guest startup, application validation, troubleshooting, and rollback buffer.

## Pre-maintenance checks

### Reconcile guest inventory

List virtual machines and containers:

```bash
qm list
pct list
```

Record which guests are:

* running and expected to return after reboot
* intentionally stopped
* intentionally excluded from the maintenance window

### Check Proxmox version and kernel

```bash
pveversion
uname -r
```

### Check Debian release

```bash
cat /etc/os-release
```

The configured Debian suite and Proxmox repository suite must match the supported release combination for the installed Proxmox major version.

### Validate repositories

```bash
apt update
```

Investigate repository warnings before applying upgrades.

Review repository definitions when needed:

```bash
cat /etc/apt/sources.list
cat /etc/apt/sources.list.d/*.list 2>/dev/null
cat /etc/apt/sources.list.d/*.sources 2>/dev/null
```

Do not proceed when the Proxmox repository is disabled, points at an incorrect Debian suite, or repeatedly fails to refresh.

### Validate DNS

If repository access reports name-resolution errors, identify the active resolver:

```bash
cat /etc/resolv.conf
```

Test both normal host resolution and the configured DNS server directly.

Examples:

```bash
getent hosts download.proxmox.com
nslookup download.proxmox.com <dns-server>
```

A resolver that is reachable as a gateway but does not answer DNS queries should not be retained as the host's sole nameserver.

### Check storage

```bash
pvesm status
zpool status
```

Expected Proxmox storage should report `active`, and ZFS pools should report healthy state before maintenance begins.

### Check systemd health

```bash
systemctl --failed
```

Investigate failed units before introducing additional change.

### Review held packages

```bash
apt-mark showhold
```

Unexpected held packages should be understood before upgrading.

## Package maintenance

Refresh metadata:

```bash
apt update
```

Review pending changes:

```bash
apt list --upgradable
```

Apply the full upgrade:

```bash
apt full-upgrade
```

Before confirming the transaction, review:

* packages to upgrade
* newly installed packages
* packages to remove
* packages held back

Unexpected removal of core Proxmox packages such as `proxmox-ve`, `pve-manager`, `qemu-server`, `pve-container`, or required storage components is a stop condition.

Package-news and service-restart prompts may appear during the upgrade. Review them and allow selected services to restart when appropriate.

## Pre-reboot validation

After the package transaction finishes:

```bash
pveversion
uname -r
pvesm status
systemctl --failed
```

If a new kernel was installed, the running kernel will remain unchanged until reboot.

Do not reboot if storage is unexpectedly inactive or failed systemd units indicate an unresolved platform problem.

## Reboot

Reboot the hypervisor:

```bash
reboot
```

Allow sufficient time for the host, storage, networking, containers, and VMs to initialize.

## Post-reboot validation

After the host returns:

```bash
uname -r
pveversion
pvesm status
systemctl --failed
cat /etc/resolv.conf
```

Confirm:

* the expected new kernel is running
* Proxmox reports the expected package version
* all configured storage is active
* no failed systemd units remain
* DNS configuration survived the reboot

For ZFS-backed environments:

```bash
zpool status
```

Confirm pool health before declaring the host recovered.

## Guest recovery

Review all guests:

```bash
qm list
pct list
```

Do not assume a stopped VM represents a boot failure. Check its autostart configuration:

```bash
qm config <vmid> | grep -E '^onboot|^startup'
```

For guests that should start with the hypervisor, configure:

```bash
qm set <vmid> --onboot 1
```

Use Proxmox startup ordering and delay settings when dependency sequencing is required.

Start any expected guest that remains stopped:

```bash
qm start <vmid>
pct start <ctid>
```

Then verify all intended workloads are running.

## UEFI certificate maintenance

Recent Proxmox releases may warn that an OVMF EFI disk has not enrolled the updated Microsoft 2023 UEFI certificates.

Identify the guest operating system and EFI configuration first:

```bash
qm config <vmid> | grep -E '^(name|ostype|bios|machine|efidisk|tpmstate):'
```

When the EFI disk is configured for the updated certificate set and the guest can be safely stopped, enroll the keys while the VM is shut down:

```bash
qm shutdown <vmid>
qm status <vmid>
qm enroll-efi-keys <vmid>
qm start <vmid>
```

For Windows guests using BitLocker, follow the Proxmox warning guidance and suspend or disable BitLocker protection as appropriate before changing EFI keys. Linux guests do not require the BitLocker step.

Confirm the guest boots normally afterward.

## Application validation

Hypervisor recovery is not complete merely because guests show `running`.

Validate each guest through its service-specific runbook. Examples include:

* DNS resolution succeeds
* password-manager web access and client synchronization succeed
* file shares are reachable
* monitoring applications and targets are healthy
* development databases accept queries
* Home Assistant integrations and dashboards remain operational

## Checkmk validation

Before removing scheduled downtime:

1. confirm the Checkmk server is reachable
2. allow hosts and services to refresh
3. confirm the hypervisor returns to its expected monitoring state
4. confirm all maintained guests return to their expected monitoring state
5. investigate any new WARN, CRIT, UNKNOWN, or DOWN state that remains after the maintenance window
6. remove the downtime early only after validation, or allow the fixed downtime to expire

## Rollback

Rollback depends on the failure domain.

### Repository or DNS configuration

Restore the prior known-good repository or resolver configuration only when it is compatible with the currently installed Proxmox and Debian release.

### Package regression

Use known-good package versions or restore the hypervisor from the established recovery method. Avoid mixing package sets from unsupported Debian or Proxmox release combinations.

### Guest startup configuration

Autostart can be disabled when needed:

```bash
qm set <vmid> --onboot 0
```

### EFI enrollment

If EFI changes prevent a guest from booting, restore the guest from a known-good backup or VM-level recovery point.

## Completion criteria

Hypervisor maintenance is complete only when:

* repository configuration is valid
* package maintenance succeeds
* the expected kernel is running
* storage is active and healthy
* no unexplained failed systemd units remain
* DNS resolution works and persists
* all intended guests are running
* guest autostart configuration matches policy
* application-level validation succeeds
* affected Checkmk hosts return to expected monitored states
* scheduled downtime is removed or allowed to expire after validation
* material changes and deviations are recorded
