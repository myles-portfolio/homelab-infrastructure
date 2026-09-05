# Vaultwarden Container Maintenance

## Scope

This runbook documents maintenance for a Linux container hosting Vaultwarden through Docker Compose.

The workload has two separate maintenance layers:

1. container operating-system packages
2. the Vaultwarden application container itself

Updating only the guest OS is not considered complete maintenance.

## Current access architecture

Vaultwarden no longer performs client-facing TLS termination on the application host.

The current access path is:

```text
Client
  |
  v
Split DNS
  |
  v
Dedicated Nginx reverse proxy
  |
  v
Vaultwarden backend
```

Nginx is the sole client-facing HTTPS layer for the service and presents the centrally managed wildcard certificate. Vaultwarden serves its application listener only on the internal backend path required by the proxy.

The previous application-local proxy layer has been removed from the Compose workload. Certificate issuance, renewal, and HTTPS presentation are therefore responsibilities of the dedicated reverse-proxy service rather than the Vaultwarden container host.

## Monitoring suppression

Before disruptive maintenance, schedule Checkmk downtime for the monitored password-management host and any separately monitored dependency that will intentionally become unavailable.

Follow the [Checkmk maintenance downtime standard](../../monitoring/checkmk/maintenance-downtime.md). Verify the downtime is active before rebooting, recreating the application container, or restarting services. Keep it active through client and synchronization validation, then remove it early or allow it to expire after the host returns to its expected monitored state.

## Pre-maintenance checks

1. Confirm the container is running.
2. Verify the Vaultwarden web interface is reachable through the normal Nginx HTTPS path.
3. Confirm the Docker service is healthy.
4. Identify the Compose working directory and current running image.
5. Record the currently deployed Vaultwarden version.
6. Confirm the backend listener remains reachable only through the intended private path.

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
2. Confirm the backend listener responds on the intended private path.
3. Confirm the public service hostname resolves to the dedicated Nginx reverse proxy rather than directly to the backend.
4. Confirm the web interface loads through the normal HTTPS hostname.
5. Confirm the expected centrally managed certificate is presented by Nginx.
6. Sign in through the browser client or extension.
7. Verify the deployed Vaultwarden version.
8. Confirm vault synchronization succeeds.
9. Confirm no obsolete application-local proxy service has been recreated by Docker Compose.

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
* restore the previous Nginx site configuration if the failure is isolated to the centralized ingress path

Do not remove application data volumes when recreating the service unless the recovery procedure explicitly requires it.

Reintroducing an application-local TLS proxy is not part of the normal rollback path unless the centralized reverse-proxy architecture itself is intentionally abandoned.

## Maintenance completion criteria

Maintenance is complete only when:

* guest package maintenance succeeds
* no failed systemd units remain
* the Vaultwarden Docker image has been refreshed intentionally
* the Compose service has been recreated when required
* the backend listener remains private
* the normal hostname resolves to the dedicated reverse proxy
* Nginx presents the expected trusted certificate
* the web interface is reachable
* the deployed version has been verified
* client synchronization succeeds
* the Compose workload contains no obsolete application-local proxy service
* Checkmk reports the expected final host and service states before downtime is removed
