# Tailscale Secure Remote Access

## Purpose

This page records the secure remote-access capabilities currently implemented in the homelab.

Detailed topology, policy structure, addressing, device identities, authentication material, and service mappings are intentionally excluded from the public repository.

## Implemented capabilities

The environment currently uses Tailscale to provide authenticated encrypted remote access to approved private homelab services without requiring inbound WAN port forwarding.

Implemented capabilities include:

* a dedicated Tailscale subnet-router workload
* private remote access to approved administrative services
* remote access to selected private web applications
* split-DNS support for internal service names
* explicit access controls using a deny-by-default policy model
* separation of remote administration from centralized application ingress
* external validation of both permitted and denied access paths
* Checkmk monitoring of the remote-access gateway
* scheduled guest backup coverage for the gateway workload

## Current service roles

Tailscale provides remote network reachability for approved systems and services.

Nginx provides centralized HTTPS termination and hostname-based routing for selected private web applications.

Administrative services that do not require web ingress remain available through the private overlay rather than being published through the reverse proxy.

## Validation

The implemented remote-access service has been validated from an external client network.

Validation confirmed that:

* approved administrative services are reachable remotely
* approved private HTTPS services are reachable through the intended access path
* internal DNS resolution works for remote clients
* selected non-approved and backend-only paths remain unavailable
* no inbound WAN port forwarding is required for the remote-access implementation

## Monitoring and backup

The remote-access gateway is treated as core infrastructure.

It is monitored through Checkmk and included in scheduled guest backup coverage.

Operational validation includes confirming gateway availability, Tailscale service state, expected route availability, and successful backup completion.

## Availability

The current implementation uses a single remote-access gateway.

Redundant subnet routing may be added later if independent always-on infrastructure becomes available and the added resilience justifies the additional complexity.

## Public documentation scope

The public repository intentionally does not publish:

* tailnet names
* authentication keys
* device identities
* user identities
* node addresses
* private subnet details
* live access-policy rules
* exact destination mappings
* exact backup schedules or storage targets

The repository records that the remote-access controls are implemented and operational without exposing the detailed live security model.
