# Home Assistant VM Maintenance

## Scope

This runbook documents maintenance for a Home Assistant OS virtual machine hosted on Proxmox VE.

Home Assistant OS is treated as an appliance-style workload. It is not maintained with `apt` or other general-purpose Linux package workflows. Maintenance is performed through Home Assistant's own update and backup interfaces, with Proxmox used for VM-level health and rollback support when needed.

## Monitoring suppression

Before disruptive maintenance, schedule Checkmk downtime for the monitored Home Assistant host and any separately monitored dependency that will intentionally become unavailable.

Follow the [Checkmk maintenance downtime standard](../../monitoring/checkmk/maintenance-downtime.md). Verify the downtime is active before applying an update or rebooting the VM. Keep it active through integration, automation, dashboard, reverse-proxy, and backup validation, then remove it early or allow it to expire after the host returns to its expected monitored state.

## Maintenance principles

* Confirm the VM is running normally before making changes.
* Check Home Assistant for available updates rather than treating the guest as a generic Linux server.
* Verify backups before applying updates.
* Prefer storing backups both locally and on separate network storage.
* Validate integrations, automations, and dashboard behavior after maintenance.
* Validate the named reverse-proxy path after changes that affect networking or HTTP settings.
* Avoid unnecessary Proxmox snapshots when no update is being applied and Home Assistant backups are current.

## Current access architecture

Home Assistant is presented to clients through the dedicated Nginx reverse proxy using split DNS and the centrally managed wildcard certificate.

```text
Client
  |
  v
Split DNS
  |
  v
Nginx reverse proxy
  |
  v
Home Assistant backend
```

The Home Assistant backend remains on its private application listener. Nginx provides the client-facing HTTPS endpoint and forwards the original client information through standard proxy headers.

Home Assistant's HTTP server is configured through the supported application settings to trust the dedicated reverse-proxy source. This trust relationship is required because Home Assistant rejects forwarded client headers received from an untrusted proxy.

When changing ingress, DNS, or HTTP-server settings, validate the Home Assistant trusted-proxy configuration before changing the service hostname to point at Nginx.

## Pre-maintenance checks

1. Confirm the Home Assistant VM is running normally in Proxmox.
2. Confirm the normal named HTTPS endpoint is reachable through Nginx.
3. In Home Assistant, open `Settings > System > Updates` and review available updates.
4. If no updates are available, treat the maintenance window as a validation and resilience check rather than forcing a change.
5. Open `Settings > System > Backups` and verify automatic backups are configured.
6. For networking or proxy changes, review `Settings > System > Network` and confirm the configured service URL, HTTP listener, forwarded-header handling, and trusted-proxy settings are consistent with the intended Nginx path.

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
5. Confirm the Home Assistant UI returns normally through the named HTTPS endpoint.
6. Validate critical integrations and automations.
7. Review `Settings > System > Repairs` for new issues.
8. Review logs for errors introduced by the update.
9. Confirm Nginx access, authentication, and WebSocket connectivity still work.

## Reverse-proxy change workflow

Treat reverse-proxy migration as an application change, not only a DNS or Nginx change.

Before DNS cutover:

1. Record the application's current canonical URL.
2. Confirm the proposed proxy hostname matches the intended canonical URL.
3. Confirm Nginx can reach the Home Assistant backend directly.
4. Configure the Nginx site and validate the configuration syntax.
5. In Home Assistant, enable forwarded-header handling and add the Nginx source to the trusted-proxy configuration.
6. Save and confirm the Home Assistant HTTP settings before proceeding.
7. Validate that the trusted-proxy setting persists after the application restarts.

Then perform the DNS cutover and validate:

* the service name resolves only to the reverse proxy
* Nginx presents the expected trusted certificate
* the Home Assistant UI loads normally
* authentication succeeds
* the WebSocket API remains connected
* dashboards and integrations update normally
* stale service-name records are removed after successful validation

If Home Assistant begins returning `400 Bad Request` immediately after proxying, review Home Assistant logs for untrusted forwarded-header messages before changing the Nginx backend or restarting unrelated services.

## Application-level validation

Examples of checks after Home Assistant maintenance include:

* thermostat control still propagates to the underlying climate device
* Scheduler entries remain enabled and execute correctly
* presence-based HVAC automation remains enabled
* Zigbee devices remain available
* Alarm.com entities remain reachable
* dashboard cards render correctly
* reverse-proxy access works through the canonical service name
* WebSocket connectivity remains stable
* backup destinations remain available
* Checkmk returns the Home Assistant host to its expected monitored state

## Rollback

Home Assistant backups are the primary application-level recovery mechanism.

Use a Proxmox snapshot when performing a higher-risk change that warrants VM-level rollback protection, but do not treat a snapshot as a substitute for Home Assistant backups.

For reverse-proxy changes, preserve the previous DNS path until the new ingress has been validated. If the centralized path fails, restore the prior DNS destination before changing the Home Assistant backend itself.

If Home Assistant becomes unavailable after an update, restore from a known-good Home Assistant backup or revert the VM snapshot when one was intentionally taken for the change.

## Maintenance outcome example

A maintenance cycle with no available Home Assistant updates may still produce meaningful improvements. In one validated cycle, backup automation was configured, a dedicated Samba backup target was created, and a manual backup was successfully written to both local and network storage. This improved recoverability without introducing unnecessary application changes.

A later networking change moved the Home Assistant web interface behind the centralized Nginx ingress. The final validated design used the application's canonical service name, split DNS to the reverse proxy, trusted forwarded-header handling on Home Assistant, centralized wildcard TLS, and WebSocket validation through the proxy path.
