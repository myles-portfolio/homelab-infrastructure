# Vaultwarden Container Maintenance

## Scope

This runbook documents maintenance for a Linux container hosting Vaultwarden through Docker Compose.

The workload has two separate maintenance layers:

1. container operating-system packages
2. the Vaultwarden application container itself

Updating only the guest OS is not considered complete maintenance.

## Monitoring suppression

Before disruptive maintenance, schedule Checkmk downtime for the monitored password-management host and any separately monitored dependency that will intentionally become unavailable.

Follow the [Checkmk maintenance downtime standard](../../monitoring/checkmk/maintenance-downtime.md). Verify the downtime is active before rebooting, recreating the application container, or restarting services. Keep it active through client and synchronization validation, then remove it early or allow it to expire after the host returns to its expected monitored state.

## Pre-maintenance checks

1. Confirm the container is running.
2. Verify the Vaultwarden web interface is reachable.
3. Confirm the Docker service is healthy.
4. Identify the Compose working directory and current running image.
5. Record the currently deployed Vaultwarden version.

Example checks:

```bash
sudo docker ps
sudo docker compose ps
```

## Operating-system maintenance

Refresh package metadata and review pending updates:

```bash
sudo apt update
apt list --upgradable
```

Apply normal package maintenance:

```bash
sudo apt full-upgrade
```

Then verify system health:

```bash
sudo systemctl --failed
```

Reboot only when required by the guest operating system or package changes.

## Vaultwarden application update

From the Compose project directory:

```bash
sudo docker compose pull
sudo docker compose up -d
```

The pull step downloads the current image defined by the Compose configuration. The `up -d` step recreates the service when the image has changed.

## Validation

After recreation:

1. Confirm the Vaultwarden container is running.
2. Confirm the web interface loads.
3. Sign in through the browser client or extension.
4. Verify the deployed Vaultwarden version.
5. Confirm vault synchronization succeeds.

Example:

```bash
sudo docker ps
sudo docker compose ps
```

## Browser extension behavior

If the Bitwarden-compatible browser extension behaves abnormally after the server update, perform a full extension logout and sign back in rather than only reinstalling the extension.

This is an application-session recovery step, not evidence that the server update failed.

## Rollback

Rollback options depend on the failure mode:

* revert the guest snapshot if one was taken for the change
* restore the previous application data backup if the database is damaged
* pin and redeploy a known-good Vaultwarden image if an application regression is isolated to the new image

Do not remove application data volumes when recreating the service unless the recovery procedure explicitly requires it.

## Maintenance completion criteria

Maintenance is complete only when:

* guest package maintenance succeeds
* no failed systemd units remain
* the Vaultwarden Docker image has been refreshed intentionally
* the Compose service has been recreated when required
* the web interface is reachable
* the deployed version has been verified
* client synchronization succeeds
* Checkmk reports the expected final host and service states before downtime is removed
