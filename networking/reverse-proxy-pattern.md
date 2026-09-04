# Reverse Proxy Pattern

## Overview

Selected homelab web services are accessed through a dedicated reverse proxy rather than by exposing each backend application's port directly to users.

The reverse proxy provides a stable service name, centralized HTTPS termination, and a single routing layer between clients and internal applications.

## Current implementation

A dedicated unprivileged Linux container now hosts Nginx as the centralized reverse proxy and TLS termination layer.

The container is intentionally narrow in scope:

* Nginx provides hostname based routing and HTTPS termination
* SSH administration uses key based authentication through a non-root administrative account
* unnecessary local services are disabled or removed
* Checkmk monitors the guest through its Linux agent with TLS registration
* ACME certificate issuance uses DNS challenge validation through a DNS provider API
* the live wildcard private key and DNS provider credentials remain outside source control

The first service migrated through this path is the infrastructure monitoring web interface. Additional internal services will be migrated incrementally after validation and soak time.

## Logical pattern

```text
Client
  |
  v
Local DNS
  |
  v
Dedicated Nginx reverse proxy
  |
  +--> monitoring backend
  +--> DNS administration backend, planned
  +--> virtualization management backend, planned last
```

Client facing service names resolve to the reverse proxy. Nginx then selects the appropriate internal backend based on the requested hostname.

## Why use a dedicated reverse proxy

A dedicated reverse proxy provides several operational benefits:

* users connect to named HTTPS services rather than memorizing host ports
* TLS configuration is centralized at the ingress layer
* backend services can remain on private ports
* certificate automation can be separated from application configuration
* additional services can be introduced without exposing a new client-facing port for each application
* ingress maintenance is isolated from the applications being proxied
* certificate and routing behavior can be monitored as infrastructure rather than as an incidental application feature

## Certificate strategy

The reverse proxy uses a wildcard certificate that covers the public domain apex and one level of subdomains.

The public repository intentionally represents this generically as:

```text
example.net
*.example.net
```

Wildcard issuance requires DNS challenge validation. An ACME client creates the temporary DNS proof through a dedicated DNS provider API credential.

Conceptually:

```text
Reverse proxy
    |
    +--> ACME certificate request
              |
              v
        DNS provider API
              |
              v
       DNS challenge proof
              |
              v
       wildcard certificate
```

The wildcard certificate is stored only on the reverse proxy rather than copied to every backend service. This reduces private-key distribution and centralizes certificate lifecycle management.

Certificate renewal automation, automatic Nginx reload after successful renewal, and certificate-expiration monitoring remain explicit follow-up controls before the implementation is considered complete.

## DNS migration pattern

Before migration, a service name may resolve directly to the backend application.

After migration, local DNS resolves that service name to the reverse proxy instead:

```text
Before
service.example.net --> backend application

After
service.example.net --> reverse proxy --> backend application
```

The backend address does not become part of the user-facing access path.

During the first migration, validation also identified a stale host-level DNS entry that caused two addresses to be returned for the same service name. The stale entry was removed, the local resolver was reloaded, and both direct DNS testing and the Checkmk active DNS check were validated before the change was considered complete.

## Backend isolation

The reverse proxy must know how to reach each backend service, but clients do not need direct knowledge of that backend port or address.

This distinction allows the access path to evolve independently from the application process itself.

Where practical, backend applications continue to use encrypted transport between the reverse proxy and the application. Applications that only expose HTTP may be proxied over the trusted internal network when risk and application capability justify it.

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
6. validate monitoring and active checks after the DNS path changes
7. validate client applications or browser extensions when applicable

A useful pre-cutover test is to force a client request to the reverse proxy while preserving the production hostname. This validates Nginx routing and certificate behavior before local DNS is changed.

## Failure isolation

Common symptoms can point to different layers:

| Symptom | Likely investigation area |
|---|---|
| Name does not resolve | DNS |
| Multiple addresses returned unexpectedly | duplicate DNS or host-level records |
| Connection refused | listener, firewall, or proxy service |
| TLS warning | certificate or hostname configuration |
| 502 or 504 response | proxy-to-backend communication |
| Login or synchronization failure | backend application or client compatibility |
| Monitoring check remains critical after migration | stale expected value, resolver path, or cached configuration |

The reverse proxy should not be assumed to be the cause simply because it sits in the request path.

## Rollback pattern

If a proxy migration fails:

1. restore the previous DNS destination
2. disable the affected Nginx site configuration if needed
3. restore the previously validated direct backend access path
4. verify application health directly before troubleshooting the new ingress path further
5. preserve the working certificate and credentials unless the reverse-proxy platform itself is being decommissioned

## Remaining implementation work

The dedicated reverse proxy is operational, but the broader migration is intentionally staged.

Remaining work includes:

* add the reverse-proxy container to scheduled Proxmox backup coverage and validate a successful backup
* configure and test certificate renewal automation
* reload Nginx automatically after successful certificate renewal
* add certificate-expiration monitoring in Checkmk
* migrate the DNS administration interface through the reverse proxy
* migrate the Proxmox management interface only after additional soak time
* continue reviewing thin-pool capacity and protection before adding larger workloads
