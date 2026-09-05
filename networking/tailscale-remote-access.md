# Tailscale Secure Remote Access

## Purpose

This document describes the sanitized secure remote-access pattern used by the homelab.

The design provides authenticated encrypted access to private infrastructure without publishing administrative interfaces to the public internet or coupling remote administration to the application reverse proxy.

Exact addresses, tailnet identifiers, authentication material, user identities, device identities, live policy rules, and internal service mappings are intentionally excluded.

## Architecture

A dedicated lightweight Linux guest provides the Tailscale subnet-router role.

```text
Trusted remote device
        |
        v
     Tailscale
        |
        v
Dedicated subnet router
        |
        v
   Private homelab LAN
```

The remote-access role is kept separate from the hypervisor, reverse proxy, DNS service, monitoring platform, and application workloads. This reduces coupling and makes the gateway easier to maintain, monitor, replace, or relocate independently.

## Deployment model

The gateway is intentionally small, dedicated, and automatically started with the virtualization platform.

It runs as an unprivileged Linux container with only the capabilities required for the Tailscale networking function. The implementation avoids broad privilege escalation simply to support overlay networking.

The public repository does not publish exact guest sizing, live address assignments, device identifiers, or hypervisor configuration lines.

## Routing model

The gateway advertises only the private network route required for remote homelab access.

Forwarding is enabled only for the address family currently used by the remote-access design. The gateway is configured as a subnet router rather than as a general internet exit node.

Route advertisement and access authorization are treated as separate controls.

## Access-control model

The approved route provides reachability, while Tailscale access policy determines which remote connections are permitted.

The current design follows a deny-by-default approach and separates services by access requirement:

| Access class | Intended remote behavior |
|---|---|
| Private administrative services | Explicit Tailscale access only |
| Private user-facing web services | Tailscale reachability with HTTPS presentation where appropriate |
| Backend infrastructure | Remains local unless a documented remote requirement exists |
| Local data services | Remains local by default; remote access requires separate justification |

The public repository documents these classes rather than publishing the live policy file, exact destination addresses, or complete service allow list.

## Validation model

Remote-access validation is performed from a client on an external network so ordinary local routing cannot be mistaken for successful overlay routing.

Validation includes both positive and negative tests:

* confirm an intended administrative path is reachable through Tailscale
* confirm an intended private HTTPS path is reachable
* confirm an approved administrative protocol works only to selected systems
* confirm at least one backend-only service remains unreachable remotely
* confirm no inbound WAN port forwarding is required for the remote-access path

Positive tests prove the path works. Negative tests provide evidence that policy restrictions are being enforced.

## Reverse proxy relationship

Tailscale and Nginx solve different problems.

```text
Tailscale = authenticated private network reachability
Nginx     = HTTPS termination and hostname-based application routing
```

A private web application may use both layers, while infrastructure administration can use Tailscale directly without publishing the management interface through Nginx.

This keeps the reverse proxy from becoming the general security boundary for remote administration.

## Monitoring

The remote-access gateway is monitored as production core infrastructure.

Monitoring covers host health and the service state required to determine whether the gateway can perform its remote-access role. Agent communication is authenticated and remains an internal monitoring path rather than being published through the reverse proxy.

The public repository intentionally omits the monitoring object name, live address, registration identity, and internal receiver endpoint.

## Backup and recovery

The gateway is included in workload-appropriate scheduled guest backup coverage.

Recovery documentation focuses on the role and validation process rather than publishing the exact backup job, storage target, schedule, retention values, or live authentication state.

Restored gateways must be isolated during validation when duplicate network or overlay identity could conflict with production.

## Availability considerations

The current deployment provides a single remote-access gateway. As with other single-instance infrastructure, loss of the hosting platform or upstream connectivity can make remote access unavailable.

A future independent gateway can be added if higher remote-access availability becomes important enough to justify additional infrastructure.

The public documentation deliberately avoids mapping the exact shared failure domains of the live environment.

## Security principles

1. Keep the remote-access role separate from application workloads.
2. Use the least privilege required for overlay networking.
3. Do not publish administrative interfaces or backend services directly to the WAN.
4. Use explicit access policy rather than an unrestricted allow-all model.
5. Test denied paths as well as permitted paths.
6. Keep backend infrastructure local unless remote access has a documented requirement.
7. Do not publish live tailnet names, authentication keys, node addresses, private subnet details, user identities, or device-specific policy.
8. Preserve a local recovery path when changing remote-access or SSH controls.
