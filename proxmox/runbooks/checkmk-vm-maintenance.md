# Checkmk VM Maintenance Runbook

## Purpose

This runbook documents maintenance for a Debian virtual machine hosting Checkmk Community and the local outbound mail transport used for infrastructure notifications.

Environment-specific site names, hostnames, addresses, credentials, and mail destinations are intentionally omitted.

## Monitoring suppression

Before disruptive maintenance, schedule Checkmk downtime for the Checkmk server itself and any other monitored hosts intentionally affected during the same window.

Follow the [Checkmk maintenance downtime standard](../../monitoring/checkmk/maintenance-downtime.md). When the Checkmk server itself is restarted or rebooted, it cannot evaluate or deliver notifications while unavailable, so schedule downtime before taking it offline and retain enough time for monitoring state to refresh afterward.

## Pre-maintenance checks

Confirm the guest is running and reachable.

Check operating-system and kernel state:

```bash
cat /etc/os-release
uname -r
systemctl --failed
```

Confirm Checkmk version and site state:

```bash
omd version
omd sites
omd status <site>
```

Confirm the local mail transport is healthy when it is part of the notification path:

```bash
systemctl status postfix --no-pager
postfix check
```

Confirm adequate filesystem headroom before package maintenance:

```bash
df -h
```

Create VM-level rollback protection when the change risk justifies it and confirm current Proxmox backups are available.

## Package maintenance

Refresh repository metadata and review pending updates:

```bash
sudo apt update
apt list --upgradable
```

Apply package maintenance:

```bash
sudo apt full-upgrade
```

Review the transaction summary before accepting unexpected package removals.

After package changes:

```bash
sudo systemctl --failed
omd status <site>
```

If the kernel or core libraries require a reboot, reboot the VM and repeat the validation after it returns.

## Checkmk application validation

Confirm the Checkmk site is running:

```bash
omd status <site>
```

Validate through the web interface that:

* the site loads normally
* monitored hosts and services refresh
* agent-based hosts remain reachable
* active checks continue to execute
* scheduled downtimes remain visible when the broader maintenance window is still in progress

## Notification transport validation

If Postfix, mail libraries, Checkmk notification configuration, or related network settings changed during maintenance:

```bash
postfix check
systemctl status postfix --no-pager
mailq
```

Then validate notification delivery according to the [Checkmk notification-delivery documentation](../../monitoring/checkmk/checkmk-notifications.md).

A material change to the notification path should be followed by a real Checkmk notification test after the broader maintenance window is no longer suppressing the test condition.

## Post-maintenance monitoring recovery

Allow Checkmk enough time to refresh monitored state after the VM returns.

Confirm:

* the Checkmk server host object is healthy
* retained services on the Checkmk server are healthy
* other maintained hosts have returned to their expected states
* no maintenance-generated stale problem remains
* outbound notification transport is healthy

Remove the scheduled downtime only after validation, or allow the fixed downtime to expire.

## Cleanup

After successful validation:

* remove temporary VM snapshots used only for maintenance
* verify backup coverage remains intact
* record material package, monitoring, or notification changes
* document any warning that remains intentionally accepted

## Rollback

If the VM fails broadly after maintenance, revert the temporary VM snapshot or restore from the established Proxmox backup workflow.

If Checkmk configuration is the isolated failure domain, restore the relevant Checkmk configuration or site backup rather than reverting unrelated guest changes when practical.

If only notification delivery fails, troubleshoot Postfix, DNS, TLS, SMTP authentication, contact routing, and notification-rule evaluation separately before reverting the entire VM.

## Completion criteria

Maintenance is complete only when:

* guest package maintenance succeeds
* no unexplained failed systemd units remain
* the Checkmk site is running
* the Checkmk web interface is reachable
* monitored hosts and services refresh normally
* agent and active-check paths remain functional
* notification transport is healthy
* Checkmk reports the expected final state before downtime is removed
