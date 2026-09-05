# Tailscale Secure Remote Access

## Purpose

This document describes the sanitized secure remote-access pattern used by the homelab.

The design provides authenticated encrypted access to private infrastructure without publishing administrative interfaces to the public internet or coupling remote administration to the Nginx reverse proxy.

Exact addresses, tailnet identifiers, authentication material, user identities, device names, and live access-policy details are intentionally excluded.

## Architecture

A dedicated unprivileged Debian Linux container provides the Tailscale subnet-router role.

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
        |
        +--> Proxmox management
        +--> selected SSH targets
        +--> Pi-hole administration
        +--> Nginx private HTTPS ingress
```

The subnet router is a separate workload from:

* the Proxmox hypervisor
* the Nginx reverse proxy
* Pi-hole
* Checkmk
* application workloads

This separation keeps remote-access routing independent from application ingress, DNS, monitoring, and virtualization responsibilities.

## Container profile

The gateway is intentionally lightweight:

```text
Guest type: unprivileged LXC
Operating system: Debian 13
vCPU: 1
RAM: 512 MB
Swap: 512 MB
Root disk: 8 GB
Autostart: enabled
```

The container uses a static LAN address and the normal internal DNS resolver.

Because Tailscale requires a Linux TUN interface, the container receives narrowly scoped access to `/dev/net/tun` rather than being converted to a privileged container. Nesting is enabled to support the Debian systemd environment used by the workload.

## Routing model

The gateway advertises the homelab IPv4 subnet to authenticated Tailscale clients.

IPv4 forwarding is enabled persistently inside the gateway. IPv6 forwarding remains disabled because the current remote-access design does not advertise an IPv6 subnet.

The gateway is a subnet router, not an exit node. General internet traffic is not intentionally routed through the homelab.

## Access-control model

Route advertisement and access authorization are separate controls.

The approved subnet route makes the private network reachable through the Tailscale overlay. Tailscale Grants then limit which remote connections are permitted.

The current policy follows a deny-by-default model and separates services into access classes:

| Access class | Examples | Remote access behavior |
|---|---|---|
| Private administrative | Proxmox management, selected SSH targets | Tailscale only |
| Private user-facing web | Checkmk and other applications presented through Nginx | Tailscale reachability plus Nginx HTTPS |
| Local administrative web | Pi-hole administration | LAN plus explicitly granted Tailscale access |
| Backend infrastructure | PostgreSQL, Prometheus, exporters, monitoring agents | LAN only unless a documented requirement changes |
| Local data services | Samba and similar storage | LAN by default; remote access requires separate justification |

The public repository documents the model rather than publishing the live policy file.

## Validated behavior

The deployment was validated from a client on an external network rather than from the home LAN. This prevents a successful local route from being mistaken for successful overlay routing.

Validation confirmed:

* remote TCP connectivity to the Proxmox management interface through the subnet router
* remote HTTPS connectivity to the Nginx reverse proxy
* remote access to the Pi-hole administration path
* SSH connectivity to explicitly approved Linux administrative targets
* SSH denial to a system outside the approved SSH target set
* PostgreSQL access denial over Tailscale for a backend database service that remains LAN-only
* no inbound WAN port forwarding is required for the remote-access path

The positive and negative tests are both important. Successful access proves the route works, while denied access provides evidence that the policy is restricting paths that should remain local.

## Reverse proxy relationship

Tailscale and Nginx solve different problems.

```text
Tailscale = authenticated private network reachability
Nginx     = HTTPS termination and hostname-based application routing
```

For a private web application, the combined path can be:

```text
Remote client
    |
    v
Tailscale
    |
    v
Private LAN
    |
    v
Nginx
    |
    v
Application backend
```

For administrative interfaces such as Proxmox, Nginx is intentionally omitted:

```text
Remote client
    |
    v
Tailscale
    |
    v
Proxmox management
```

This prevents the reverse proxy from becoming the general security boundary for infrastructure administration.

## Monitoring

The remote-access gateway is monitored in Checkmk as a production core-infrastructure Linux container.

The Checkmk Linux agent is registered with TLS. Agent registration uses the internal Checkmk backend path rather than the reverse-proxied web hostname because the agent receiver is a separate internal service and is not published through Nginx.

Monitoring should cover at minimum:

* host availability
* CPU and memory state
* filesystem state
* systemd health
* Tailscale daemon state
* network reachability appropriate to the gateway role

## Backup and recovery

The gateway is included in the Core Infrastructure scheduled Proxmox backup policy.

Backup coverage protects the guest operating system, Tailscale configuration state, routing configuration, Checkmk agent configuration, and other local system configuration captured by the guest backup.

Reusable authentication material and other secrets are not stored in the public repository.

Successful backup creation and controlled restore validation remain separate requirements. A restored gateway should be validated carefully because duplicate network identity or Tailscale state could conflict with the production gateway if both are online simultaneously.

## Failure domains

The initial subnet router runs as a guest on the same physical Proxmox host as the services it exposes remotely.

This creates a known dependency:

* if the Proxmox host is offline, the subnet router is also offline
* if the home internet connection is unavailable, remote Tailscale access cannot reach the site
* local LAN access remains independent of the Tailscale control path for services that support local access

A future second physical server or other independent always-on device can provide a redundant subnet router if remote-access availability becomes important enough to justify additional infrastructure.

## Security principles

1. Keep the subnet-router role separate from application workloads.
2. Use an unprivileged container and grant only the device capability required for TUN networking.
3. Do not publish Proxmox, SSH, databases, or internal agent ports directly to the WAN.
4. Use explicit Tailscale Grants instead of an unrestricted allow-all policy.
5. Test denied paths as well as permitted paths.
6. Keep backend infrastructure LAN-only unless remote access has a documented requirement.
7. Do not publish live tailnet names, authentication keys, node addresses, private subnet details, or user-specific policy identities.
8. Preserve a local console recovery path when changing remote-access or SSH controls.
