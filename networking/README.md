# Networking

## Overview

This section documents the sanitized networking patterns used by the homelab.

The goal is to show how local name resolution, service access, reverse-proxy behavior, and internal dependencies are structured without publishing a complete live network map.

## Current design

The environment uses a private home LAN with Proxmox-hosted services and Home Assistant integrations distributed across virtual machines and Linux containers.

Key networking functions include:

- local DNS and filtering through Pi-hole
- split DNS for internal service names
- a dedicated Nginx reverse proxy for centralized HTTPS termination and hostname-based routing
- wildcard TLS for selected internal service names
- Samba/CIFS for network storage
- local-only service communication where practical
- trusted-proxy handling in Home Assistant
- no requirement for direct inbound WAN port forwarding for internally hosted web services documented here

## Logical flow

```text
Client device
    |
    v
Local DNS resolver
    |
    +--> internal service name --> reverse proxy
    |                                 |
    |                                 v
    |                            backend service
    |
    +--> public domain lookup --> upstream DNS
```

## Network and service flow

The following diagram shows the sanitized relationship between client devices, DNS, reverse proxy, core services, and selected external integrations.

![Network and service flow](diagrams/network-service-flow.png)

The diagram is intentionally sanitized and does not expose the live environment's real domains, IP addresses, or other sensitive addressing details.

## Design principles

1. **Name services by function rather than port number.** Clients should access services through stable names where practical.
2. **Keep backend services private.** The reverse proxy provides the user-facing HTTPS path while backend ports remain internal.
3. **Use split DNS deliberately.** Internal clients can resolve a service name to the proxy path without requiring hairpin WAN routing.
4. **Centralize TLS without distributing private keys broadly.** The wildcard certificate remains on the dedicated proxy rather than being copied to every backend.
5. **Avoid unnecessary WAN exposure.** Certificate automation and remote access should not require opening inbound ports when safer alternatives exist.
6. **Validate DNS separately from application health.** A service can be running while name resolution is broken, and the reverse can also be true.
7. **Document dependencies.** DNS, reverse proxy, authentication, certificate state, and backend services are separate failure domains.

## Key components

### Pi-hole

Pi-hole provides local DNS filtering and selected local DNS overrides. It is also a critical dependency for service names that rely on split DNS.

Selected internal service names now resolve to the dedicated reverse proxy rather than directly to application backends.

See [`dns-and-split-dns.md`](dns-and-split-dns.md).

### Reverse proxy

A dedicated unprivileged Linux container hosts Nginx as the centralized ingress and TLS termination layer for selected internal services.

The proxy uses a wildcard certificate issued through ACME DNS challenge validation. Service migrations are staged rather than moved all at once. The infrastructure monitoring interface was used as the first production validation target, while additional administrative services remain planned for later migration.

See [`reverse-proxy-pattern.md`](reverse-proxy-pattern.md).

### Home Assistant trusted proxy

Home Assistant is configured to trust only the expected reverse-proxy sources when processing forwarded client information. Public documentation preserves the concept but omits the real address ranges.

## Dependency examples

### Monitoring web access

```text
Client
  |
  v
Pi-hole / local DNS
  |
  v
Nginx reverse proxy
  |
  +--> wildcard TLS certificate
  |
  v
Monitoring backend
```

If DNS fails, the backend may remain healthy while the service name stops resolving.

If the reverse proxy fails, DNS may resolve correctly while HTTPS access fails.

If the backend fails, DNS and TLS may still appear healthy even though the application is unavailable.

### Vaultwarden access

```text
Client
  |
  v
Pi-hole / local DNS
  |
  v
Reverse proxy
  |
  v
Vaultwarden
```

The environment is being converged toward the dedicated proxy pattern so certificate and routing responsibilities are isolated from application workloads.

### Home Assistant reverse-proxy access

```text
Client
  |
  v
Reverse proxy
  |
  v
Home Assistant
      |
      +--> validates trusted proxy source
      +--> accepts forwarded client information
```

## Operational validation

Networking changes should be validated at more than one layer:

1. confirm expected DNS resolution
2. confirm TCP connectivity to the service path
3. confirm TLS behavior where HTTPS is used
4. confirm the reverse proxy selects the intended backend
5. confirm the backend application responds
6. confirm authentication and normal user workflows still function
7. confirm monitoring checks reflect the new ingress path

Before a DNS cutover, a client can force a single request to the new proxy address while preserving the production hostname. This validates certificate trust and proxy routing without changing normal resolution for other clients.

After DNS changes, direct resolver tests should confirm that stale host-level or duplicate local records are not returning an obsolete backend address alongside the proxy address.

## Security and sanitization

The public repository intentionally excludes:

- exact private IP assignments
- public service domains tied to the live environment
- router management details
- API credentials
- certificate-provider tokens
- wildcard private keys
- firewall-rule inventories
- exact trusted-proxy subnets

The architecture remains useful as a portfolio artifact without exposing a detailed attack map.
