# Security

## Overview

This section documents the security principles applied across the homelab without exposing sensitive implementation details from the live environment.

The design emphasizes reducing unnecessary exposure, separating credentials by function, limiting trust boundaries, preserving recovery options, and documenting changes in a way that supports secure operations.

## Security principles

1. **Minimize exposure.** Internal services should not be directly exposed to the WAN unless there is a clear requirement.
2. **Prefer named HTTPS access paths.** Reverse proxies and trusted certificates reduce direct backend exposure and improve service consistency.
3. **Use dedicated service accounts.** Machine-to-machine workflows should avoid reusing interactive administrator accounts where practical.
4. **Protect secrets from source control.** Credentials, API tokens, private keys, exact location data, and sensitive endpoints are never intended for the public repository.
5. **Limit trust relationships.** Reverse-proxy trust, service permissions, file-share access, and remote-access permissions should be scoped to the smallest practical set of systems.
6. **Keep recovery controls independent.** Snapshots, application backups, database dumps, and external backup copies address different failure modes.
7. **Harden remote administration.** SSH hardening and secure remote access are treated as explicit security work rather than default assumptions.
8. **Validate after security changes.** Authentication, service access, denied paths, and rollback options must be tested after modifying security controls.

## Exposure reduction

Several architectural choices reduce attack surface:

* internal split DNS for selected services
* dedicated reverse-proxy based HTTPS access
* centralized TLS termination for selected internal services
* avoidance of unnecessary inbound WAN port forwarding
* backend application listeners kept private where practical
* secure remote access through Tailscale rather than exposing administrative interfaces directly
* removal of application-local TLS proxies and direct overlay membership when centralized infrastructure makes them unnecessary

See [`../networking/`](../networking/) for the networking architecture that supports these controls.

## Reverse proxy isolation

A dedicated unprivileged Linux container hosts the Nginx reverse proxy and wildcard TLS certificate used for selected internal services.

This separates ingress and certificate lifecycle responsibilities from application workloads. The wildcard private key is kept on the proxy rather than distributed across every backend system.

The proxy itself is hardened as a core infrastructure workload:

* unprivileged container deployment
* non-root administrative account
* Ed25519 key based SSH authentication
* password based SSH disabled after successful key-login validation
* direct root password SSH disabled
* unnecessary local services removed or disabled
* Checkmk Linux agent registered with TLS
* only required service listeners retained

Backend migrations are staged so the old path is preserved until the new proxy path has been validated.

One private application migration also removed the application-local TLS proxy after Nginx had been validated as the sole client-facing HTTPS layer. This reduced duplicate certificate handling and removed an unnecessary service from the application host.

## Secure remote access

Tailscale is deployed as the overlay VPN for remote administrative access.

A dedicated unprivileged Linux container provides the subnet-router role. It runs separately from the hypervisor, reverse proxy, DNS service, monitoring platform, and application workloads.

The implementation separates remote network access from application ingress:

* Proxmox and selected SSH targets remain private and are reached through Tailscale
* selected private web applications use Nginx for trusted named HTTPS while Tailscale controls remote network reachability
* internal DNS is available to remote clients only through the private overlay so split-DNS service names continue to work outside the LAN
* DNS administration is explicitly reachable through a restricted Tailscale path
* backend services such as PostgreSQL, Prometheus, exporters, and monitoring agents remain LAN-only unless a documented requirement changes

Tailscale Grants replace the default unrestricted allow-all rule. The current policy follows a deny-by-default approach with explicit approved paths.

Validation was performed from an external network and included both positive and negative tests. Intended administrative and HTTPS paths succeeded, while selected backend and non-approved paths remained unavailable through Tailscale.

Private application hosts do not require direct Tailscale membership when their remote-access path is provided through the subnet router and reverse proxy. Direct node membership is removed where it no longer provides a distinct security or operational benefit.

The gateway is monitored in Checkmk and included in the appropriate scheduled guest backup policy.

A future independent always-on infrastructure device could provide redundant subnet routing if remote-access availability becomes more important.

See [`../networking/tailscale-remote-access.md`](../networking/tailscale-remote-access.md).

## Service accounts

Dedicated service accounts are used where practical for non-interactive access.

One example is the Home Assistant backup workflow, which uses a dedicated Samba account with access only to the backup share rather than reusing a general-purpose user account.

Monitoring agent registration and certificate automation also use purpose-specific credentials rather than general administrator identities where supported.

The intended pattern is:

```text
Application
   |
   v
Dedicated service identity
   |
   v
Single required resource
```

This reduces the blast radius of a compromised credential.

## Secrets handling

The public repository follows a simple rule: documentation should describe the control without publishing the secret.

Examples of information excluded from public configuration include:

* passwords
* password hashes used for privileged application administration
* API keys
* certificate-provider credentials
* private keys
* authentication cookies
* webhook URLs
* exact geofence coordinates
* public domains tied to the live environment
* detailed private IP inventories
* Tailscale authentication keys, tailnet identifiers, node addresses, and live access-policy identities

When sensitive authentication material is exposed during troubleshooting, it is rotated even if the exposed value is a derived credential representation rather than plaintext.

Where configuration examples require sensitive values, generic placeholders are used instead.

## Wildcard certificate considerations

A wildcard certificate simplifies TLS management but increases the importance of protecting its private key because that key can authenticate multiple service hostnames.

For that reason, the current design keeps the wildcard certificate on the dedicated reverse proxy rather than copying it to each backend host.

ACME validation uses a DNS-provider API credential stored outside source control. Automated renewal and certificate-expiration monitoring remain explicit lifecycle controls to complete.

## Reverse-proxy trust

Applications that process forwarded client information must trust only the expected proxy source.

Trusted-proxy configuration should be limited to known proxy sources rather than broad private-network ranges whenever practical.

A misconfigured trust range can allow untrusted clients to spoof forwarded headers.

## Backup resilience

Security includes recoverability.

The environment uses multiple backup layers depending on the workload:

* application-level backups where supported
* logical database dumps where useful
* scheduled Proxmox guest backups
* temporary Proxmox snapshots for selected maintenance windows
* persistent application storage for stateful container workloads
* additional network backup copies for selected services

Backup creation and controlled restore validation remain separate recovery checks.

These mechanisms are not interchangeable. Each protects against a different failure mode.

## SSH hardening roadmap

SSH hardening has begun with selected infrastructure workloads and establishes the pattern for broader rollout:

* disable direct root password login
* use key-based authentication
* preserve an emergency recovery path through the virtualization console
* validate a second remote session before disabling password authentication
* repeat the pattern on additional Linux workloads after service-specific validation

Tailscale restricts which remote SSH paths are reachable, but it does not replace host-level SSH authentication and hardening.

## Public documentation model

The repository itself is treated as part of the security model.

A public portfolio should demonstrate engineering decisions without becoming a reconnaissance package for the live environment.

The repository therefore avoids publishing combinations of details that are individually harmless but collectively reveal too much, such as exact hostnames, IP addresses, software versions, external domains, service ports, and recovery commands in one place.

See [`../SECURITY.md`](../SECURITY.md) for publication-specific sanitization rules.

## Security improvement backlog

Current planned improvements include:

* continue SSH key-based authentication rollout
* certificate-expiration monitoring
* automated backup restore testing
* reverse-proxy certificate renewal and reload automation validation
* expanded service-health alerting
* consider redundant subnet routing when independent infrastructure is available

These items are also tracked in the project wiki roadmap.
