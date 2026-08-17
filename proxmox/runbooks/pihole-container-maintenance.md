# Pi-hole Container Maintenance

## Scope

This runbook documents maintenance for a Linux container hosting Pi-hole as a local DNS and filtering service.

Because DNS is infrastructure-critical, maintenance is validated through actual name resolution rather than package state alone.

## Pre-maintenance checks

1. Confirm the container is running.
2. Confirm Pi-hole services are active.
3. Record the current Pi-hole version.
4. Verify DNS resolution from at least one client before changing anything.

Example service checks:

```bash
pihole status
systemctl --failed
```

Example client validation:

```bash
nslookup example.com <dns-server>
```

## Operating-system maintenance

Refresh package metadata:

```bash
sudo apt update
apt list --upgradable
```

Apply package updates:

```bash
sudo apt full-upgrade
```

Afterward, confirm no failed units remain:

```bash
sudo systemctl --failed
```

Reboot only when required.

## Pi-hole application update

Pi-hole application maintenance should be treated separately from guest operating-system updates.

Review the installed version first:

```bash
pihole -v
```

When an application update is intentionally scheduled, use the supported Pi-hole update workflow for the installed release and review the resulting version afterward.

## Validation

Maintenance is not complete until DNS behavior is verified.

Validate:

* Pi-hole reports healthy service state
* DNS queries resolve successfully
* local DNS records still resolve where configured
* expected filtering remains active
* the administrative interface is reachable if used

Example:

```bash
pihole status
pihole -v
```

From a client:

```bash
nslookup example.com <dns-server>
```

If local records are part of the environment, test at least one known internal name as well.

## Rollback

If the guest update causes a broad failure, revert the temporary Proxmox snapshot when one was taken.

If only the Pi-hole application is affected, restore its configuration or backup using the supported Pi-hole recovery workflow rather than reverting unrelated guest changes.

## Maintenance completion criteria

Maintenance is complete only when:

* guest package maintenance succeeds
* no failed systemd units remain
* Pi-hole service status is healthy
* Pi-hole version is recorded after application maintenance
* external DNS resolution succeeds
* internal DNS resolution succeeds where applicable
