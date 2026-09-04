# Change Management

## Purpose

This section documents the lightweight change-management approach used for homelab infrastructure and service changes.

The goal is not to reproduce enterprise bureaucracy. The goal is to preserve enough context that a future operator can answer five questions:

1. What changed?
2. Why was it changed?
3. What systems were affected?
4. How was success validated?
5. How can the change be rolled back?

## Change categories

### Routine

Low-risk, repeatable maintenance with a known procedure and established rollback path.

Examples:

* operating-system package updates
* container image refresh using an existing Compose workflow
* validation of backup jobs
* routine application maintenance

### Significant

Changes that alter architecture, control logic, service dependencies, authentication, network paths, or recovery behavior.

Examples:

* moving thermostat scheduling from a vendor application into Home Assistant
* adding presence-aware HVAC control
* creating a new network backup path
* introducing a new VM or container role
* centralizing reverse-proxy and TLS responsibilities for multiple services

### Emergency

Unplanned changes required to restore service, recover access, or reduce immediate risk.

Examples:

* password recovery for an inaccessible VM
* emergency rollback after a failed application update
* restoring DNS after service failure

## Risk model

A simple five-level scale is used when additional risk context is useful:

| Risk | General meaning |
|---|---|
| Trivial | No meaningful service impact expected |
| Low | Limited, easily reversible impact |
| Medium | Service interruption or dependency impact possible |
| High | Broad service impact or recovery complexity possible |
| Critical | Failure could cause major loss of service or data |

## Change record minimum fields

Every material change should capture:

| Field | Purpose |
|---|---|
| Date | When the change occurred |
| Status | Planned, in progress, complete, rolled back, or failed |
| Change made | Concise description of the implementation |
| Reason | Business or technical rationale |
| Systems affected | Guests, services, devices, or integrations in scope |
| Impact | User-facing or operational effect |
| Validation | Evidence that the change succeeded |
| Rollback notes | How to return to the previous known-good state |

## Workflow

```text
Identify need
    |
    v
Define scope and risk
    |
    v
Capture rollback path
    |
    v
Implement change
    |
    v
Validate system health
    |
    +--> validation fails --> rollback / troubleshoot
    |
    v
Document outcome
```

## Templates

* [Change record template](templates/change-record.md)
* [Maintenance record template](templates/maintenance-record.md)

## Examples in this repository

Detailed examples include:

* [Split DNS and Vaultwarden HTTPS](examples/split-dns-vaultwarden-https.md)
* [Dedicated Nginx reverse proxy and wildcard TLS](examples/dedicated-nginx-reverse-proxy.md)
* [Home Assistant rebuild](examples/home-assistant-rebuild.md)
* [Monitoring VM maintenance](examples/monitoring-vm-maintenance.md)

The runbooks under `proxmox/runbooks/` demonstrate the same operating model in practice. They separate pre-checks, implementation, validation, cleanup, and rollback rather than treating a package-manager success message as proof of service health.
