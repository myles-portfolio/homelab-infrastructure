# Homelab Infrastructure

Sanitized documentation, automation, and configuration examples from my personal homelab environment.

This repository is a public portfolio of systems administration, infrastructure engineering, automation, observability, documentation, and systems design work. It deliberately excludes live credentials, exact location data, private endpoints, secrets, and other sensitive operational details.

## What this repository demonstrates

* Proxmox VE administration across virtual machines and Linux containers
* Linux server and service maintenance
* Docker Compose application operations
* Prometheus and Grafana monitoring
* PostgreSQL backup and validation
* Home Assistant automation and appliance maintenance
* Samba-based network storage and service accounts
* Workload-specific backup and rollback design
* Change validation beyond package-manager success
* Public documentation with deliberate security sanitization

## Architecture

The environment is centered on a Proxmox VE host running dedicated workloads for file services, monitoring, password management, DNS and filtering, development, and home automation.

See [`architecture/overview.md`](architecture/overview.md) for the sanitized architecture and operating model.

## Case studies

### Presence-aware residential HVAC control

A Home Assistant based HVAC control system that combines deterministic weekly scheduling with presence-aware daytime temperature control, neighborhood geofencing, delayed absence detection, and manual override capability.

```text
Phone location
    |
    v
Home Assistant person entity
    |
    v
Neighborhood zone logic
    |
    +--> resident nearby --> comfort target
    |
    +--> away > 30 min --> away target
                          |
Weekly scheduler ----------+
                          v
                 Versatile Thermostat
                          |
                          v
                      Alarm.com
                          |
                          v
                Physical thermostat
```

See [`home-assistant/hvac/`](home-assistant/hvac/) for the sanitized automation and design notes.

### Proxmox guest maintenance and service validation

Maintenance workflows treat the hypervisor and each guest as separate operational domains. The current guest inventory is reconciled before maintenance, rollback protection is selected based on workload risk, and application health is validated independently of operating-system package state.

Examples include:

* Docker-based Prometheus, Grafana, and NUT exporter maintenance
* QEMU Guest Agent deployment and validation
* PostgreSQL logical backup before VM maintenance
* Home Assistant local and external backup validation
* LVM thin-pool utilization checks before snapshot-heavy changes

See [`proxmox/`](proxmox/) for the operational model and runbooks.

## Repository map

```text
architecture/
  overview.md

home-assistant/
  hvac/
    README.md
    presence-based-hvac.yaml
    weekly-schedule-design.md

proxmox/
  README.md
  runbooks/
    development-vm-maintenance.md
    home-assistant-vm-maintenance.md
    monitoring-vm-maintenance.md

SECURITY.md
.gitignore
```

## Operating philosophy

A few principles recur throughout this environment:

1. Reconcile the live inventory before maintenance.
2. Validate the hosted application, not only the operating system.
3. Use workload-appropriate recovery controls such as snapshots, application backups, and database dumps.
4. Separate predictable schedules from dynamic automation logic.
5. Use dedicated service accounts for machine-to-machine access when practical.
6. Remove temporary rollback artifacts after successful validation.
7. Keep public documentation useful to reviewers without exposing the live environment unnecessarily.

## Planned additions

* File-services maintenance and Samba design
* Pi-hole DNS maintenance and validation
* Vaultwarden container maintenance
* Reverse-proxy configuration patterns
* Zigbee and IoT integration
* Monitoring architecture and alerting
* Change-management records and operational checklists

## Security and sanitization

The public repository is a curated documentation source, not a mirror of the live homelab configuration. Configuration examples are reviewed and sanitized before publication.

See [`SECURITY.md`](SECURITY.md) for publication rules.
