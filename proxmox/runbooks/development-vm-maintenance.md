# Development VM Maintenance

## Scope

This runbook documents maintenance for a sanitized Ubuntu development VM running PostgreSQL and development tooling.

The workflow separates operating-system maintenance from database validation and preserves both application-level and VM-level rollback options.

## Current service profile

Validated workload characteristics:

* Ubuntu 24.04 LTS
* PostgreSQL 16
* SSH access
* QEMU Guest Agent
* No Docker workload
* No failed systemd services at baseline
* One application database in addition to standard PostgreSQL databases

## Pre-maintenance checks

1. Confirm the VM is running and reachable over SSH.
2. Verify the operating system and kernel.

```bash
cat /etc/os-release
uname -r
```

3. Check for failed services.

```bash
sudo systemctl --failed
```

4. Confirm PostgreSQL is running.

```bash
systemctl status postgresql@16-main --no-pager
```

5. Record listening services.

```bash
ss -tulpn
```

6. Confirm QEMU Guest Agent is running.

```bash
systemctl status qemu-guest-agent --no-pager
```

7. List PostgreSQL databases.

```bash
sudo -u postgres psql -c '\l'
```

## Database backup

Create a logical backup of the application database before package maintenance.

Example:

```bash
sudo -u postgres pg_dump -Fc app_database > ~/app_database-pre-maintenance.dump
```

Verify that the backup exists and is nonzero in size.

```bash
ls -lh ~/app_database-pre-maintenance.dump
```

Keep the logical dump through at least the next known-good backup cycle.

## Proxmox snapshot

Take a temporary pre-maintenance snapshot before applying operating-system updates.

Do not include RAM state unless there is a specific reason to preserve it.

If Proxmox reports thin-pool overcommit warnings, verify actual thin-pool utilization before proceeding.

Example host check:

```bash
lvs
```

Review both `Data%` and `Meta%` for the thin pool rather than interpreting provisioned virtual-disk capacity as actual storage consumption.

## Package maintenance

Refresh package metadata and review pending changes.

```bash
sudo apt update
apt list --upgradable
```

Then perform the upgrade.

```bash
sudo apt full-upgrade
```

Before confirming, review the summary for unexpected removals or held packages.

## Post-upgrade validation

Do not consider maintenance complete only because the package manager exits successfully.

Validate systemd health:

```bash
sudo systemctl --failed
```

Validate PostgreSQL service state:

```bash
sudo systemctl status postgresql@16-main --no-pager
```

Validate that the application database accepts queries:

```bash
sudo -u postgres psql -d app_database -c 'SELECT now();'
```

Check whether Ubuntu requires a reboot:

```bash
test -f /var/run/reboot-required && cat /var/run/reboot-required || echo "No reboot required"
```

If a reboot is required, reboot the VM and repeat the systemd, PostgreSQL, and database-query validation after it returns.

## Proxmox guest validation

From the Proxmox host, confirm QEMU Guest Agent communication:

```bash
qm guest cmd <vmid> ping
```

A successful response confirms Proxmox can communicate with the guest.

## Cleanup

After all validation passes:

1. Delete the temporary pre-maintenance snapshot.
2. Retain the logical PostgreSQL dump until the next successful backup cycle.
3. Record package or service changes that materially affect the workload.

## Rollback

If package maintenance causes guest-wide failure, revert the Proxmox snapshot.

If the VM remains healthy but the application database requires recovery, restore from the logical PostgreSQL dump using standard `pg_restore` procedures.

The logical database backup and VM snapshot serve different rollback purposes and should not be treated as interchangeable controls.
