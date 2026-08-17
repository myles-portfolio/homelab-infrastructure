# File Services Container Maintenance

## Scope

This runbook documents maintenance for a Linux container providing Samba-based network file services and dedicated backup shares.

The maintenance model treats file-sharing availability, share configuration, permissions, and client access as first-class validation targets.

## Pre-maintenance checks

1. Confirm the container is running.
2. Record the configured Samba shares.
3. Validate the Samba configuration.
4. Confirm the Samba service is healthy.
5. Test at least one representative share from a client.

Example checks:

```bash
testparm -s
systemctl status smbd --no-pager
systemctl --failed
```

## Operating-system maintenance

Refresh package metadata and review pending updates:

```bash
sudo apt update
apt list --upgradable
```

Apply package maintenance:

```bash
sudo apt full-upgrade
```

Then verify system health:

```bash
sudo systemctl --failed
```

Reboot only when required.

## Samba validation

After package maintenance:

```bash
testparm -s
systemctl status smbd --no-pager
```

Validate at least one representative share from a client rather than relying only on server-side service status.

## Dedicated service accounts

Machine-to-machine access should use purpose-specific Samba accounts where practical.

Example pattern for a Home Assistant backup target:

```text
service account: habackup
share: home-assistant-backups
path: /srv/home-assistant-backups
```

The account should not require an interactive shell and should only have the filesystem access required for its purpose.

Example share configuration:

```ini
[home-assistant-backups]
   path = /srv/home-assistant-backups
   browseable = yes
   read only = no
   guest ok = no
   valid users = habackup
   force user = habackup
   create mask = 0600
   directory mask = 0700
```

Validate changes before reloading Samba:

```bash
testparm -s
systemctl reload smbd
systemctl status smbd --no-pager
```

## Backup-target validation

When the container hosts backup storage for another service, validation must include an actual write from that service.

For example, a Home Assistant backup target is considered working only after Home Assistant successfully creates a backup on the Samba share and the backup appears in the expected location.

## Rollback

Rollback depends on the failure domain:

* revert a temporary Proxmox snapshot for guest-wide failure
* restore the previous Samba configuration for share-definition errors
* restore filesystem permissions when access control is the cause
* preserve data directories when recreating or repairing share configuration

## Maintenance completion criteria

Maintenance is complete only when:

* guest package maintenance succeeds
* no failed systemd units remain
* `testparm` validates the Samba configuration
* `smbd` is healthy
* representative client access succeeds
* machine-to-machine backup targets can perform real writes where applicable
