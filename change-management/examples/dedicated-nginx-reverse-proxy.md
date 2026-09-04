# Change Example: Dedicated Nginx Reverse Proxy and Wildcard TLS

## Summary

Deployed a dedicated unprivileged Linux container for Nginx, centralized TLS termination with a wildcard certificate, onboarded the proxy into Checkmk, and migrated the infrastructure monitoring web interface through the new ingress path.

## Change type

Significant

## Risk

Medium

## Problem statement

Trusted HTTPS had previously been implemented independently on several internal services. That model worked, but it distributed certificate lifecycle responsibilities across application hosts and did not provide a single ingress layer for hostname-based routing.

The goal was to establish a dedicated reverse-proxy workload that could centralize TLS termination, reduce private-key distribution, isolate ingress from backend applications, and provide a reusable migration pattern for additional internal services.

## Scope

Affected components included:

* Proxmox VE
* a new unprivileged Linux container
* Nginx
* local DNS
* ACME certificate automation
* DNS provider API integration
* Checkmk monitoring
* the monitoring web interface used as the first migrated backend

Exact guest IDs, addresses, live hostnames, public domains, API credentials, and certificate private keys are intentionally omitted.

## Implementation

1. Created a lightweight unprivileged Debian container with a small CPU, memory, swap, and disk allocation appropriate for reverse-proxy traffic.
2. Configured a static internal address and local DNS dependency.
3. Installed Nginx and validated HTTP service availability.
4. Created a non-root administrative account and configured Ed25519 key-based SSH access.
5. Disabled password-based SSH only after validating a second independent key-authenticated session.
6. Disabled direct root password SSH and retained virtualization-console access as the recovery path.
7. Removed an unnecessary local mail service so only required listeners remained.
8. Installed and registered the Checkmk Linux agent with TLS.
9. Issued a wildcard certificate through ACME DNS challenge validation using a dedicated DNS provider API credential stored outside source control.
10. Added the first Nginx site configuration for the monitoring web interface.
11. Validated the proxy before DNS cutover by forcing a client request to the new proxy address while preserving the normal production hostname.
12. Changed local DNS so the monitoring hostname resolved to the reverse proxy rather than directly to the backend.
13. Validated browser trust, application access, proxy routing, and monitoring behavior.
14. Updated the Checkmk DNS active check to expect the new ingress address.
15. Identified and removed a stale host-level DNS entry that caused both the old backend address and the new proxy address to be returned.
16. Reloaded local DNS and confirmed direct resolver tests, the DNS monitoring plug-in, and the Checkmk service all returned to the expected healthy state.

## Validation

The change was considered successful when:

* the reverse-proxy host was healthy in Checkmk
* only the intended administrative and web listeners remained active
* key-based SSH access succeeded after password authentication was disabled
* the wildcard certificate contained the expected apex and wildcard names
* the certificate chain was trusted by the client
* a forced pre-cutover request reached the proxy and routed successfully to the backend
* local DNS returned only the proxy address after migration
* the monitoring web interface loaded normally through the proxy
* the Checkmk active DNS service returned to OK after stale DNS data was removed

## Impact

The first service migration was performed with a reversible DNS change. No public inbound port forwarding was introduced. Backend application availability remained independently testable during the migration.

## Rollback

Rollback consists of:

1. restoring the affected local DNS record to the previous backend destination
2. disabling the corresponding Nginx site if required
3. validating direct backend access
4. preserving the wildcard certificate and proxy workload for troubleshooting unless the proxy platform itself is being decommissioned

## Security considerations

The wildcard certificate private key is intentionally kept on the dedicated proxy instead of being copied to every backend service.

DNS provider API credentials and private key material are excluded from the public repository.

The proxy uses an unprivileged container, non-root administration, key-based SSH, restricted service listeners, and Checkmk TLS agent registration.

## Follow-up work

The implementation is operational but intentionally not treated as complete until the remaining lifecycle controls are addressed:

* add the reverse-proxy container to scheduled Proxmox backup coverage
* validate successful backup creation
* configure and test automatic wildcard certificate renewal
* reload Nginx automatically after successful renewal
* add certificate-expiration monitoring in Checkmk
* migrate the DNS administration interface through the dedicated proxy
* migrate the virtualization management interface only after additional soak time
* review thin-pool auto-extension protection as a separate storage-hardening task

## Engineering considerations

A reverse proxy becomes a shared dependency as additional services are migrated behind it. That changes its role from a convenience layer into core infrastructure.

For that reason, monitoring, backup, certificate lifecycle, rollback, and service dependency documentation must evolve with the proxy rather than treating it as a single configuration file.

The staged migration also demonstrated the value of testing the new ingress path before changing DNS and of validating DNS from the same resolver path used by monitoring rather than relying on one client cache or lookup tool.
