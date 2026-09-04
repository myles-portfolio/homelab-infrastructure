# Reverse Proxy Container Maintenance

## Purpose

This runbook defines the sanitized maintenance workflow for the dedicated Nginx reverse-proxy container.

The workload is treated as core infrastructure because multiple internal web services may depend on it for DNS-directed ingress, TLS termination, and hostname-based backend routing.

Exact guest IDs, addresses, live domains, certificate paths tied to the production environment, and credentials are intentionally omitted.

## Scope

The maintenance workflow covers:

* Debian operating-system updates
* Nginx package and service health
* SSH administration
* wildcard certificate presence and validity
* ACME renewal configuration
* backend routing validation
* Checkmk monitoring state
* scheduled backup coverage and recovery readiness

## Pre-maintenance

1. Reconcile the current proxy configuration and list the services that depend on it.
2. Confirm the proxy host is healthy in Checkmk.
3. Schedule Checkmk downtime for the proxy and any dependent hosts expected to show intentional access-path failures during maintenance.
4. Confirm a recent scheduled guest backup exists once backup coverage has been enabled.
5. Confirm a direct backend recovery path is documented for each critical proxied service.
6. Confirm DNS rollback can restore the previous backend destination if ingress fails.
7. Check Proxmox storage health and thin-pool utilization before creating additional temporary rollback artifacts.
8. Confirm console access through Proxmox remains available before modifying SSH configuration.

## Operating-system maintenance

Update the guest operating system using the standard Debian package workflow.

After package maintenance, validate:

```bash
systemctl --failed
systemctl status nginx --no-pager
ss -tulpn
nginx -t
```

Only expected listeners should remain active.

## Nginx validation

Confirm configuration syntax before every reload or restart:

```bash
sudo nginx -t
```

If successful, reload Nginx rather than restarting it when a reload is sufficient:

```bash
sudo systemctl reload nginx
```

Validate that configured service hostnames still route to the intended backends.

A pre-DNS or isolated validation can force a request to the proxy address while preserving the service hostname so certificate and routing behavior can be tested independently from normal resolver state.

## TLS validation

Confirm the proxy presents the expected trusted certificate and that the certificate includes the intended wildcard coverage.

Validation should include:

* certificate issuer
* validity dates
* subject alternative names
* trusted client negotiation
* expected hostname match

The wildcard private key must remain outside source control and should not be copied to backend systems unless a separate design requirement explicitly justifies it.

## Certificate renewal

Once renewal automation is fully implemented, maintenance validation should include:

1. verify the ACME client can access the DNS-provider plug-in
2. confirm the protected credential file retains restrictive permissions
3. run a safe renewal test or dry run when supported
4. confirm the deploy hook reloads Nginx only after successful renewal
5. confirm the newly active certificate is presented to clients
6. confirm Checkmk certificate-expiration monitoring returns the expected state

## SSH validation

The proxy uses a non-root administrative account with Ed25519 key authentication.

After SSH-related changes:

1. validate the SSH daemon configuration before reload
2. keep the current session open
3. open a second independent key-authenticated session
4. confirm password authentication remains disabled
5. retain Proxmox console access as the emergency recovery path

Do not remove the recovery path before the new remote-authentication path is proven.

## DNS and routing validation

For each proxied service:

1. confirm local DNS resolves the service name to the proxy path
2. confirm the reverse proxy presents the expected certificate
3. confirm Nginx selects the intended backend
4. confirm the backend application responds normally
5. confirm application authentication or client workflows still succeed
6. confirm monitoring checks reflect the current ingress path

If DNS returns more than one unexpected address, inspect both local DNS records and host-level resolver data before changing the application.

## Checkmk validation

Confirm:

* proxy host is reachable
* Linux agent TLS registration is healthy
* accepted services return to their expected state
* no new systemd or filesystem problems were introduced
* active checks for proxied services reflect the current DNS and ingress path
* certificate-expiration monitoring is healthy once implemented

Remove scheduled downtime only after the proxy and dependent services have returned to their expected states.

## Backup validation

The reverse-proxy container should be included in the core-infrastructure scheduled Proxmox backup job.

After adding or changing backup coverage:

1. confirm the scheduled job includes the proxy guest
2. run or observe a successful backup
3. confirm the expected backup artifact exists
4. verify retention behavior
5. periodically perform an isolated restore test

A restore test should validate Nginx configuration, certificate state, system startup, and local service behavior without allowing a duplicate proxy identity to conflict with production.

## Rollback

If maintenance breaks the proxy path:

1. restore the previous Nginx configuration if the failure is configuration-specific
2. restore DNS to the previously validated backend destination when necessary
3. validate direct backend application health
4. use the most appropriate guest-level rollback or backup recovery control if the container itself is damaged
5. preserve evidence needed to determine whether the failure occurred in DNS, TLS, Nginx, networking, or the backend application

## Post-maintenance acceptance

Maintenance is complete only when:

* Debian reports no unexpected failed units
* Nginx configuration validation succeeds
* expected listeners are active
* proxied service hostnames resolve correctly
* trusted HTTPS works
* backend routing succeeds
* Checkmk returns to the expected state
* backup coverage remains intact
* any temporary rollback artifacts are removed when no longer required
