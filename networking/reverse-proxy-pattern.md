# Reverse Proxy Pattern

## Overview

Selected homelab web services are accessed through a reverse proxy rather than by exposing each backend application's port directly to users.

The reverse proxy provides a stable service name, HTTPS termination, and a single routing layer between clients and internal applications.

## Logical pattern

```text
Client
  |
  v
DNS
  |
  v
Reverse proxy
  |
  +--> service A backend
  +--> service B backend
  +--> service C backend
```

The current environment uses this pattern for selected services. A dedicated reverse-proxy container is planned to further isolate ingress from individual application workloads.

## Why use a reverse proxy

A reverse proxy provides several operational benefits:

* users connect to named HTTPS services rather than memorizing host ports
* TLS configuration is centralized at the ingress layer
* backend services can remain on private ports
* certificate automation can be separated from application configuration
* additional services can be introduced without exposing a new client-facing port for each application

## Certificate strategy

For services that require trusted certificates without opening inbound WAN ports, DNS challenge validation can be used with the certificate authority.

Conceptually:

```text
Reverse proxy
    |
    +--> certificate request
              |
              v
        DNS provider API
              |
              v
       DNS challenge proof
```

No provider credentials or live domain information are stored in this public repository.

## Backend isolation

The reverse proxy should know how to reach the backend service, but clients do not need direct knowledge of that backend port.

This distinction allows the access path to evolve independently from the application process itself.

## Home Assistant considerations

Applications that depend on forwarded client information must explicitly trust the expected proxy source.

For Home Assistant, trusted-proxy configuration is kept narrowly scoped to known proxy sources rather than accepting forwarded headers from arbitrary clients.

This protects against untrusted clients spoofing forwarded source information.

## Validation

A reverse-proxy change should be tested at several layers:

1. confirm DNS resolves to the expected ingress path
2. confirm HTTPS negotiation succeeds
3. inspect certificate trust and hostname validity
4. confirm the reverse proxy routes to the expected backend
5. confirm the application loads and authentication works
6. validate client applications or browser extensions when applicable

## Failure isolation

Common symptoms can point to different layers:

| Symptom | Likely investigation area |
|---|---|
| Name does not resolve | DNS |
| Connection refused | listener, firewall, or proxy service |
| TLS warning | certificate or hostname configuration |
| 502 or 504 response | proxy-to-backend communication |
| Login or synchronization failure | backend application or client compatibility |

The reverse proxy should not be assumed to be the cause simply because it sits in the request path.

## Planned improvement

The current roadmap includes moving reverse-proxy responsibility into a dedicated LXC workload.

The intended benefits are:

* stronger service isolation
* clearer ownership of ingress configuration
* easier expansion to additional HTTPS services
* simpler reverse-proxy maintenance independent of application containers

## Rollback pattern

If a proxy migration fails:

1. restore the previous DNS destination if it changed
2. restore the previous proxy configuration
3. re-enable the previously validated backend access path when necessary
4. verify application health directly before troubleshooting the new ingress path further
