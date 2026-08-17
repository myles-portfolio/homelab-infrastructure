# Homelab Infrastructure

Sanitized documentation, automation, configuration examples, and operational records from my personal homelab environment.

This repository is a public portfolio of systems administration, infrastructure engineering, automation, observability, service operations, documentation, and systems design work. It deliberately excludes live credentials, exact location data, private endpoints, secrets, and other sensitive operational details.

## Start here

If you are reviewing this repository as a portfolio, these are the best entry points:

1. **Architecture:** [`architecture/overview.md`](architecture/overview.md) provides the sanitized system topology, workload roles, backup model, and design principles.
2. **Systems operations:** [`proxmox/README.md`](proxmox/README.md) explains the maintenance model and links to workload-specific runbooks.
3. **Automation case study:** [`home-assistant/hvac/README.md`](home-assistant/hvac/README.md) documents a presence-aware HVAC control system built in Home Assistant.
4. **Change management:** [`change-management/README.md`](change-management/README.md) describes the lightweight change-control framework used throughout the lab.
5. **Detailed change examples:** [`change-management/examples/`](change-management/examples/) contains deeper records showing problem analysis, implementation, validation, rollback, and lessons learned.
6. **History:** [`CHANGELOG.md`](CHANGELOG.md) shows how the environment has evolved over time.
7. **Planned work:** [`ROADMAP.md`](ROADMAP.md) tracks future infrastructure and security improvements.

## What this repository demonstrates

* Proxmox VE administration across virtual machines and Linux containers
* Linux server and service maintenance
* Docker Compose application operations
* Prometheus and Grafana observability
* PostgreSQL backup and validation
* Home Assistant automation and appliance maintenance
* Samba-based network storage and dedicated service accounts
* DNS, reverse-proxy, and TLS architecture
* Workload-specific backup and rollback design
* Change management and maintenance documentation
* Validation of application health beyond package-manager success
* Public technical documentation with deliberate security sanitization

## Architecture

The environment is centered on a Proxmox VE host running dedicated workloads for file services, monitoring, password management, DNS and filtering, development, and home automation.

See [`architecture/overview.md`](architecture/overview.md) for the sanitized architecture and operating model.

## Featured case studies

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

The implementation separates predictable schedule transitions from dynamic presence corrections. It also applies a continuous absence threshold so short errands and neighborhood activity do not unnecessarily change the HVAC setpoint.

See [`home-assistant/hvac/`](home-assistant/hvac/) for the design notes, schedule model, and sanitized automation.

### Proxmox guest maintenance and service validation

Maintenance workflows treat the hypervisor and each guest as separate operational domains. The live guest inventory is reconciled before maintenance, rollback protection is selected based on workload risk, and application health is validated independently of operating-system package state.

Examples include:

* Docker-based Prometheus, Grafana, and NUT exporter maintenance
* QEMU Guest Agent deployment and validation
* PostgreSQL logical backup before VM maintenance
* Home Assistant local and external backup validation
* LVM thin-pool utilization checks before snapshot-heavy changes
* Vaultwarden application image refresh in addition to guest OS maintenance
* Pi-hole DNS validation after package maintenance
* Samba configuration and service validation

See [`proxmox/`](proxmox/) for the operational model and runbooks.

### Change management in practice

The lab uses a lightweight change-management model designed to preserve technical intent without introducing unnecessary administrative overhead.

Detailed examples include:

* [`Split DNS and Vaultwarden HTTPS`](change-management/examples/split-dns-vaultwarden-https.md)
* [`Home Assistant rebuild`](change-management/examples/home-assistant-rebuild.md)
* [`Monitoring VM maintenance`](change-management/examples/monitoring-vm-maintenance.md)

These examples document the problem, scope, risk, implementation, validation evidence, rollback path, and lessons learned.

## Repository map

```text
architecture/
  overview.md

change-management/
  README.md
  examples/
    home-assistant-rebuild.md
    monitoring-vm-maintenance.md
    split-dns-vaultwarden-https.md
  templates/
    change-record.md
    maintenance-record.md

home-assistant/
  hvac/
    README.md
    presence-based-hvac.yaml
    weekly-schedule-design.md

proxmox/
  README.md
  runbooks/
    development-vm-maintenance.md
    fileshare-container-maintenance.md
    home-assistant-vm-maintenance.md
    monitoring-vm-maintenance.md
    pihole-container-maintenance.md
    vaultwarden-container-maintenance.md

CHANGELOG.md
ROADMAP.md
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
7. Treat application updates and guest operating-system updates as separate maintenance layers when appropriate.
8. Keep public documentation useful to reviewers without exposing the live environment unnecessarily.

## Current technology areas

| Area | Technologies and concepts |
|---|---|
| Virtualization | Proxmox VE, KVM, LXC, snapshots, QEMU Guest Agent |
| Linux operations | Ubuntu, Debian-based package management, systemd, SSH |
| Containers | Docker, Docker Compose, persistent volumes |
| Observability | Prometheus, Grafana, NUT exporter |
| Data | PostgreSQL, logical backups, query validation |
| Network services | Pi-hole, split DNS, Samba, reverse proxy, TLS |
| Identity and secrets | Vaultwarden, dedicated service accounts |
| Home automation | Home Assistant OS, HACS, Zigbee, climate automation |
| Operations | Change records, maintenance runbooks, rollback planning, validation |

## Roadmap

The lab remains an active engineering environment. Planned work includes secure remote access, dedicated reverse-proxy isolation, certificate-expiration monitoring, automated backup restore testing, SSH hardening, enhanced UPS notifications, and additional observability.

See [`ROADMAP.md`](ROADMAP.md) for the current backlog.

## Security and sanitization

The public repository is a curated documentation source, not a mirror of the live homelab configuration. Configuration examples are reviewed and sanitized before publication.

See [`SECURITY.md`](SECURITY.md) for publication rules.
