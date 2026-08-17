# Security

## Overview

This section documents the security principles applied across the homelab without exposing sensitive implementation details from the live environment.

The design emphasizes reducing unnecessary exposure, separating credentials by function, limiting trust boundaries, preserving recovery options, and documenting changes in a way that supports secure operations.

## Security principles

1. **Minimize exposure.** Internal services should not be directly exposed to the WAN unless there is a clear requirement.
2. **Prefer named HTTPS access paths.** Reverse proxies and trusted certificates reduce direct backend exposure and improve service consistency.
3. **Use dedicated service accounts.** Machine-to-machine workflows should avoid reusing interactive administrator accounts where practical.
4. **Protect secrets from source control.** Credentials, API tokens, private keys, exact location data, and sensitive endpoints are never intended for the public repository.
5. **Limit trust relationships.** Reverse-proxy trust, service permissions, and file-share access should be scoped to the smallest practical set of systems.
6. **Keep recovery controls independent.** Snapshots, application backups, database dumps, and external backup copies address different failure modes.
7. **Harden remote administration.** SSH hardening and secure remote access are treated as explicit security work rather than default assumptions.
8. **Validate after security changes.** Authentication, service access, and rollback paths must be tested after modifying security controls.

## Exposure reduction

Several architectural choices reduce attack surface:

* internal split DNS for selected services
* reverse-proxy based HTTPS access
* avoidance of unnecessary inbound WAN port forwarding
* backend application ports kept private where practical
* planned secure remote access through an overlay VPN rather than exposing administrative interfaces directly

See [`../networking/`](../networking/) for the networking architecture that supports these controls.

## Service accounts

Dedicated service accounts are used where practical for non-interactive access.

One example is the Home Assistant backup workflow, which uses a dedicated Samba account with access only to the backup share rather than reusing a general-purpose user account.

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

Where configuration examples require sensitive values, generic placeholders are used instead.

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
* temporary Proxmox snapshots for selected maintenance windows
* persistent Docker volumes for stateful container workloads

These mechanisms are not interchangeable. Each protects against a different failure mode.

## SSH hardening roadmap

Current roadmap work includes hardening SSH administration by:

* disabling direct root password login
* moving toward key-based authentication
* preserving an emergency recovery path
* validating console access before restricting remote authentication

This work is intentionally staged so security improvements do not create an avoidable lockout condition.

## Remote access roadmap

Secure remote access is planned through an overlay VPN approach rather than exposing Proxmox, SSH, or internal web applications directly to the internet.

The goals are:

* authenticated encrypted remote access
* minimal WAN exposure
* centralized remote-access control
* reduced dependence on router port forwarding

## Public documentation model

The repository itself is treated as part of the security model.

A public portfolio should demonstrate engineering decisions without becoming a reconnaissance package for the live environment.

The repository therefore avoids publishing combinations of details that are individually harmless but collectively reveal too much, such as exact hostnames, IP addresses, software versions, external domains, service ports, and recovery commands in one place.

See [`../SECURITY.md`](../SECURITY.md) for publication-specific sanitization rules.

## Security improvement backlog

Current planned improvements include:

* SSH key-based authentication
* secure remote access deployment
* certificate-expiration monitoring
* automated backup restore testing
* further reverse-proxy isolation
* expanded service-health alerting

These items are also tracked in [`../ROADMAP.md`](../ROADMAP.md).
