# Networking

## Overview

This section documents the sanitized networking patterns used by the homelab.

The goal is to show how local name resolution, service access, reverse-proxy behavior, and internal dependencies are structured without publishing a complete live network map.

## Current design

The environment uses a private home LAN with Proxmox-hosted services and Home Assistant integrations distributed across virtual machines and Linux containers.

Key networking functions include:

* local DNS and filtering through Pi-hole
* split DNS for internal service names
* reverse-proxy based HTTPS access for selected services
* Samba/CIFS for network storage
* local-only service communication where practical
* trusted-proxy handling in Home Assistant
* no requirement for direct inbound WAN port forwarding for internally hosted web services documented here

## Logical flow

```text
Client device
    |
    v
Local DNS resolver
    |
    +--> internal service name --> private service path
    |
    +--> public domain lookup --> upstream DNS

Internal web service request
    |
    v
Reverse proxy
    |
    v
Backend application
```

## Design principles

1. **Name services by function rather than port number.** Clients should access services through stable names where practical.
2. **Keep backend services private.** Reverse proxies provide the user-facing HTTPS path while backend ports remain internal.
3. **Use split DNS deliberately.** Internal clients can resolve a service name to its private path without requiring hairpin WAN routing.
4. **Avoid unnecessary WAN exposure.** Certificate automation and remote access should not require opening inbound ports when safer alternatives exist.
5. **Validate DNS separately from application health.** A service can be running while name resolution is broken, and the reverse can also be true.
6. **Document dependencies.** DNS, reverse proxy, authentication, and backend services are separate failure domains.

## Key components

### Pi-hole

Pi-hole provides local DNS filtering and selected local DNS overrides. It is also a critical dependency for service names that rely on split DNS.

See [`dns-and-split-dns.md`](dns-and-split-dns.md).

### Reverse proxy

A reverse proxy provides HTTPS termination and routes named services to internal backends. Vaultwarden currently uses this pattern.

A dedicated reverse-proxy container remains a planned architecture improvement so ingress can be isolated from individual application containers.

See [`reverse-proxy-pattern.md`](reverse-proxy-pattern.md).

### Home Assistant trusted proxy

Home Assistant is configured to trust only the expected reverse-proxy sources when processing forwarded client information. Public documentation preserves the concept but omits the real address ranges.

## Dependency examples

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

If DNS fails, the backend may remain healthy while the service name stops resolving.

If the reverse proxy fails, DNS may resolve correctly while HTTPS access fails.

If Vaultwarden fails, DNS and TLS may still appear healthy even though the application is unavailable.

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
4. confirm the backend application responds
5. confirm authentication and normal user workflows still function

This layered validation helps distinguish DNS, proxy, certificate, network, and application failures.

## Security and sanitization

The public repository intentionally excludes:

* exact private IP assignments
* public service domains tied to the live environment
* router management details
* API credentials
* certificate-provider tokens
* firewall-rule inventories
* exact trusted-proxy subnets

The architecture remains useful as a portfolio artifact without exposing a detailed attack map.
