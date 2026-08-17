# Service Dependency Mapping

## Purpose

This document maps major homelab services to the supporting components they depend on. The goal is to improve troubleshooting by making failure domains explicit.

Exact IP addresses, public domains, and sensitive network details are intentionally omitted.

## Dependency model

A user-facing service may depend on several separate layers:

```text
Client
  |
  v
DNS
  |
  v
Network path
  |
  v
Reverse proxy / ingress
  |
  v
Backend application
  |
  v
Authentication / data store / external integration
```

A failure at any layer can produce a similar user symptom, such as "the site is down," even when the root cause is different.

## Service matrix

| Service | Primary dependencies | Validation focus |
|---|---|---|
| Vaultwarden | local DNS, reverse proxy, TLS certificate, Docker service, persistent application data | DNS resolution, HTTPS response, container health, client sign-in and sync |
| Home Assistant | VM health, network path, reverse proxy where used, trusted-proxy configuration, integrations | UI access, integration availability, automation execution, backup access |
| Pi-hole | container health, network path, upstream DNS | local resolution, upstream resolution, filtering behavior |
| Grafana | Monitoring VM, Docker, network path, persistent Grafana data | HTTP response, dashboard access, data-source connectivity |
| Prometheus | Monitoring VM, Docker, target network reachability, configuration file | HTTP response, active target health, query execution |
| NUT exporter | Monitoring VM Docker network, UPS/NUT source, Prometheus scrape configuration | Prometheus target reports `up` and returns expected metrics |
| Samba file services | file-services container, smbd, storage path, permissions, network path | share listing, authenticated read/write, expected file persistence |
| Home Assistant backups | Home Assistant backup subsystem, Samba share, dedicated service account, file-services container | backup completes locally and externally, backup file exists on share |
| PostgreSQL development database | Development VM, PostgreSQL service, local storage | service status, successful application database query, logical backup |

## Example: Vaultwarden

```text
Client
  |
  v
Local DNS
  |
  v
Reverse proxy
  |
  +--> TLS certificate
  |
  v
Vaultwarden container
  |
  v
Persistent vault data
```

Possible failure domains:

* DNS name does not resolve
* reverse proxy is unavailable
* certificate is invalid or expired
* backend container is stopped
* application is responding but client compatibility is broken
* persistent data is unavailable

## Example: Home Assistant backup path

```text
Home Assistant
    |
    v
Backup subsystem
    |
    +--> local backup storage
    |
    v
CIFS / Samba
    |
    v
File-services container
    |
    v
Dedicated backup directory
```

A successful Home Assistant backup should be validated at both the application layer and the storage layer.

## Example: Monitoring stack

```text
Prometheus
   |
   +--> host / service targets
   |
   +--> NUT exporter
   |
   v
Time-series data
   |
   v
Grafana
```

Grafana can be available while Prometheus targets are unhealthy. Prometheus can be available while an individual exporter is down. These are separate validation domains.

## Operational use

When a service fails, use the dependency map to test from the outside inward:

1. Can the client resolve the expected name?
2. Can the client reach the expected network endpoint?
3. Does the proxy or ingress layer respond?
4. Does the backend application respond directly where appropriate?
5. Are dependent services healthy?
6. Does the user workflow succeed end to end?

This approach reduces the tendency to restart the application before proving which layer is actually failing.
