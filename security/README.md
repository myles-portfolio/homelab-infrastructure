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
* backend application ports kept private where practical
* secure remote access through Tailscale rather than exposing administrative interfaces directly

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

Backend migrations are staged so a DNS rollback can restore the previously validated direct path if a proxy change fails.

## Secure remote access

Tailscale is deployed as the overlay VPN for remote administrative access.

A dedicated unprivileged Debian container provides the subnet-router role. It runs separately from the hypervisor, reverse proxy, DNS service, monitoring platform, and application workloads.

The implementation separates remote network access from application ingress:

* Proxmox and selected SSH targets remain private and are reached through Tailscale
* selected private web applications may use Nginx for trusted named HTTPS while Tailscale controls remote network reachability
* Pi-hole administration is explicitly reachable through Tailscale while its DNS role remains local infrastructure
* backend services such as PostgreSQL, Prometheus, exporters, and monitoring agents remain LAN-only unless a documented requirement changes

The gateway uses narrowly scoped TUN device access inside an unprivileged LXC rather than privileged-container deployment. IPv4 forwarding is enabled only for the required subnet-routing role.

Tailscale Grants replace the default unrestricted allow-all rule. The current policy follows a deny-by-default approach with explicit destinations and ports for approved administrative and HTTPS paths.

Validation was performed from an external network and included both positive and negative tests. Intended Proxmox, reverse-proxy, Pi-hole, and selected SSH paths succeeded. A backend PostgreSQL path and an unapproved SSH path remained unavailable through Tailscale.

The gateway is monitored in Checkmk and included in the Core Infrastructure scheduled guest backup policy.

The initial subnet router shares the existing Proxmox host. Remote access therefore depends on that host remaining online. A future second physical host or other independent always-on infrastructure device could provide redundant subnet routing if remote-access availability becomes more important.

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
* API keys
* certificate-provider credentials
* private keys
* authentication cookies
* webhook URLs
* exact geofence coordinates
* public domains tied to the live environment
* detailed private IP inventories
* Tailscale authentication keys, tailnet identifiers, node addresses, and live access-policy identities

Where configuration examples require sensitive values, generic placeholders are used instead.

## Wildcard certificate considerations

A wildcard certificate simplifies TLS management but increases the importance of protecting its private key because that key can authenticate multiple service hostnames.

For that reason, the current design keeps the wildcard certificate on the dedicated reverse proxy rather than copying it to each backend host.

ACME validation uses a DNS-provider API credential stored outside source control. Automated renewal and certificate-expiration monitoring remain explicit lifecycle controls to complete.

## Reverse-proxy trust

Home Assistant and similar services may need to trust a reverse proxy so forwarded client information can be processed correctly.

Trusted-proxy configuration should be limited to the expected proxy source rather than broad private-network ranges whenever practical.

A misconfigured trust range can allow untrusted clients to spoof forwarded headers.

## Backup resilience

Security includes recoverability.

The environment uses multiple backup layers depending on the workload:

* Home Assistant local backups
* Home Assistant external network backup copies
* PostgreSQL logical dumps
* scheduled Proxmox guest backups
* temporary Proxmox snapshots for selected maintenance windows
* persistent Docker volumes for stateful container workloads

The reverse-proxy container, personal knowledge backend, and remote-access gateway are included in workload-appropriate scheduled guest backup policies. Backup creation and controlled restore validation remain separate recovery checks.

These mechanisms are not interchangeable. Each protects against a different failure mode.

## SSH hardening roadmap

SSH hardening has begun with the reverse-proxy workload and establishes the pattern for broader rollout:

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

* continue SSH key-based authentication rollout beyond the reverse proxy
* certificate-expiration monitoring
* automated backup restore testing
* reverse-proxy certificate renewal and reload automation validation
* expanded service-health alerting
* consider redundant subnet routing when independent infrastructure is available

These items are also tracked in the project wiki roadmap.
