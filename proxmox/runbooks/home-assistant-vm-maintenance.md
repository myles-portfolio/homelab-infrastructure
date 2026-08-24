# Home Assistant VM Maintenance

## Scope

This runbook documents maintenance for a Home Assistant OS virtual machine hosted on Proxmox VE.

Home Assistant OS is treated as an appliance-style workload. It is not maintained with `apt` or other general-purpose Linux package workflows. Maintenance is performed through Home Assistant's own update and backup interfaces, with Proxmox used for VM-level health and rollback support when needed.

## Monitoring suppression

Before disruptive maintenance, schedule Checkmk downtime for the monitored Home Assistant host and any separately monitored dependency that will intentionally become unavailable.

Follow the [Checkmk maintenance downtime standard](../../monitoring/checkmk/maintenance-downtime.md). Verify the downtime is active before applying an update or rebooting the VM. Keep it active through integration, automation, dashboard, and backup validation, then remove it early or allow it to expire after the host returns to its expected monitored state.

## Maintenance principles

* Confirm the VM is running normally before making changes.
* Check Home Assistant for available updates rather than treating the guest as a generic Linux server.
* Verify backups before applying updates.
* Prefer storing backups both locally and on separate network storage.
* Validate integrations, automations, and dashboard behavior after maintenance.
* Avoid unnecessary Proxmox snapshots when no update is being applied and Home Assistant backups are current.

## Pre-maintenance checks

1. Confirm the Home Assistant VM is running normally in Proxmox.
2. In Home Assistant, open `Settings > System > Updates` and review available updates.
3. If no updates are available, treat the maintenance window as a validation and resilience check rather than forcing a change.
4. Open `Settings > System > Backups` and verify automatic backups are configured.

## Backup configuration

A resilient configuration should keep more than one copy of Home Assistant backups.

Recommended pattern:

* automatic backups enabled
* daily schedule
* seven retained backups
* backup before updates enabled
* Home Assistant settings included
* history included
* apps included
* share folder included when needed by the local configuration
* media excluded unless specifically required
* local Home Assistant storage retained
* secondary copy written to network storage hosted outside the Home Assistant VM

## Network backup target

A dedicated Samba share can be used as the external backup location.

Example sanitized server-side design:

```text
/srv/home-assistant-backups
```

Create a dedicated service account rather than reusing an interactive user account.

Example share definition:

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

Validate the Samba configuration before reload:

```bash
testparm -s
systemctl reload smbd
systemctl status smbd --no-pager
```

In Home Assistant, add the location under network storage using CIFS/Samba and mark the usage as Backup.

## Backup validation

After configuring the network target, create a manual validation backup.

Example naming convention:

```text
VM107-HA-Backup-Validation-YYYY-MM-DD
```

Validation is successful when the backup appears in both:

* the local Home Assistant backup location
* the external network backup location

Do not consider backup configuration complete until a real backup has been created successfully and is visible in both locations.

## Update workflow

If Home Assistant reports updates:

1. Confirm current backups exist locally and externally.
2. Allow Home Assistant to create a backup before updating.
3. Apply updates through the Home Assistant UI.
4. Allow the appliance to restart as required.
5. Confirm the Home Assistant UI returns normally.
6. Validate critical integrations and automations.
7. Review `Settings > System > Repairs` for new issues.
8. Review logs for errors introduced by the update.

## Application-level validation

Examples of checks after Home Assistant maintenance include:

* thermostat control still propagates to the underlying climate device
* Scheduler entries remain enabled and execute correctly
* presence-based HVAC automation remains enabled
* Zigbee devices remain available
* Alarm.com entities remain reachable
* dashboard cards render correctly
* reverse-proxy access still works
* backup destinations remain available
* Checkmk returns the Home Assistant host to its expected monitored state

## Rollback

Home Assistant backups are the primary application-level recovery mechanism.

Use a Proxmox snapshot when performing a higher-risk change that warrants VM-level rollback protection, but do not treat a snapshot as a substitute for Home Assistant backups.

If Home Assistant becomes unavailable after an update, restore from a known-good Home Assistant backup or revert the VM snapshot when one was intentionally taken for the change.

## Maintenance outcome example

A maintenance cycle with no available Home Assistant updates may still produce meaningful improvements. In one validated cycle, backup automation was configured, a dedicated Samba backup target was created, and a manual backup was successfully written to both local and network storage. This improved recoverability without introducing unnecessary application changes.
