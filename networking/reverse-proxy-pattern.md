# Reverse Proxy Pattern

## Overview

Selected homelab web services are accessed through a dedicated reverse proxy rather than by exposing each backend application's listener directly to users.

The reverse proxy provides stable service naming, centralized HTTPS termination, and a single routing layer between clients and internal applications.

## Current implementation

A dedicated unprivileged Linux container hosts Nginx as the centralized reverse proxy and TLS termination layer.

The container is intentionally narrow in scope:

* Nginx provides hostname-based routing and HTTPS termination
* SSH administration uses key-based authentication through a non-root administrative account
* unnecessary local services are disabled or removed
* Checkmk monitors the guest independently from the applications routed through it
* ACME certificate issuance uses DNS challenge validation through a DNS provider API
* live certificate private keys and DNS provider credentials remain outside source control

Service migration is staged. Monitoring was used as the first production validation target, and additional private applications are moved behind the proxy only after direct backend reachability, proxy routing, certificate behavior, and rollback options have been tested.

## Logical pattern

```text
Client
  |
  v
Private network reachability
  |
  v
Split DNS
  |
  v
Dedicated Nginx reverse proxy
  |
  +--> monitoring backend
  +--> password-management backend
  +--> other selected private web applications
```

Administrative interfaces such as the hypervisor and SSH are intentionally excluded from this pattern. They use the secure overlay network directly rather than being published through Nginx.

## Why use a dedicated reverse proxy

A dedicated reverse proxy provides several operational benefits:

* users connect to named HTTPS services rather than backend ports
* TLS configuration is centralized at the ingress layer
* backend services can remain private
* certificate automation is separated from application configuration
* additional services can be introduced without adding a new client-facing listener for each application
* ingress maintenance is isolated from the applications being proxied
* certificate and routing behavior can be monitored as infrastructure rather than as an incidental application feature

## Certificate strategy

The reverse proxy uses a wildcard certificate covering the public domain apex and one level of subdomains.

The public repository represents this generically as:

```text
example.net
*.example.net
```

Wildcard issuance uses DNS challenge validation. An ACME client creates the temporary proof through a dedicated DNS provider credential.

The wildcard certificate is stored only on the reverse proxy rather than copied to every backend service. This reduces private-key distribution and centralizes certificate lifecycle management.

Automated renewal, automatic Nginx reload after successful renewal, and certificate-expiration monitoring remain explicit lifecycle controls.

## DNS migration pattern

Before migration, a service name may resolve directly to an application host.

After migration, split DNS resolves that service name to the reverse proxy instead:

```text
Before
service.example.net --> application host

After
service.example.net --> reverse proxy --> application backend
```

The backend address and listener are no longer part of the normal client-facing path.

For remote clients connected through the secure overlay, split DNS provides the same private service naming model used on the LAN.

## Backend simplification

Where an application previously ran its own dedicated TLS proxy, that proxy can be removed after Nginx has been validated as the sole client-facing HTTPS layer.

This reduces duplicate certificate handling, duplicate proxy configuration, and unnecessary software on the application host.

The migration sequence is deliberately staged:

1. make the application reachable from Nginx over a private backend path
2. validate the backend independently
3. create and test the Nginx site without changing normal DNS
4. change split DNS to the reverse proxy
5. validate browser and native-client behavior from LAN and remote networks
6. remove the previous application-local proxy only after the new path is proven
7. remove any now-unnecessary direct overlay-network membership from the application host

## Access-control interaction

Nginx is not the remote-access security boundary.

The secure overlay controls whether a remote device can reach the private network. Nginx then provides HTTPS presentation and hostname routing for applications that use the proxy pattern.

This separation prevents application ingress and infrastructure administration from becoming unnecessarily coupled.

## Validation

A reverse-proxy change should be tested at several layers:

1. confirm private DNS resolves to the expected ingress path
2. confirm HTTPS negotiation succeeds
3. inspect certificate trust and hostname validity
4. confirm the reverse proxy routes to the expected backend
5. confirm the application loads and authentication works
6. validate native clients or browser extensions where applicable
7. confirm monitoring reflects the new access path
8. test from an external network through the secure overlay when the service is intended for remote use

A useful pre-cutover test is to force a single request to the reverse proxy while preserving the production hostname. This validates routing and certificate behavior before DNS is changed.

## Failure isolation

Common symptoms can point to different layers:

| Symptom | Likely investigation area |
|---|---|
| Name does not resolve | split DNS or remote DNS path |
| Connection refused or timeout | listener, firewall, route, or access policy |
| TLS warning | certificate chain, hostname, or incorrect ingress path |
| 502 or 504 response | proxy-to-backend communication |
| Login or synchronization failure | backend application, proxy behavior, DNS, or client compatibility |
| Monitoring check remains critical after migration | stale expected value, resolver path, or cached configuration |

The reverse proxy should not be assumed to be the cause simply because it sits in the request path.

## Rollback pattern

If a proxy migration fails:

1. restore the previously validated DNS destination or backend path
2. disable the affected Nginx site configuration if needed
3. verify application health directly
4. troubleshoot the new ingress path without changing unrelated services
5. preserve certificate and credential state unless the reverse-proxy platform itself is being rebuilt

## Remaining implementation work

The dedicated reverse proxy is operational and multiple private service patterns have been validated. Remaining work includes:

* configure and test certificate renewal automation
* reload Nginx automatically after successful certificate renewal
* add certificate-expiration monitoring
* migrate additional private web applications where the pattern provides a clear benefit
* continue validating application-specific proxy requirements before each migration
