# Home Assistant

## Overview

This section documents the Home Assistant environment used in the homelab, including dashboard design, automation patterns, climate control, energy monitoring, backup practices, and selected configuration examples.

The public repository contains curated and sanitized examples rather than a mirror of the live `/config` directory. Live credentials, private endpoints, precise location data, device identifiers, alarm details, and other sensitive operational values are intentionally excluded.

## Platform

The Home Assistant deployment runs as Home Assistant OS in a dedicated Proxmox virtual machine.

Operational responsibilities include:

* Home Assistant OS and Core maintenance
* integration and custom-card management
* Zigbee device integration
* dashboard design and YAML configuration
* automation and script development
* climate scheduling and presence-aware control
* energy monitoring
* local and external backup validation
* reverse-proxy compatibility and trusted-proxy configuration

## Dashboard architecture

The primary custom dashboard is developed as a YAML-managed Lovelace dashboard rather than being maintained only through the graphical editor.

The design uses a three-column command-center layout organized around:

* quick actions and home state
* climate, time, and weather
* energy production, consumption, storage, and grid activity

The dashboard also uses state-aware colors so visual emphasis communicates operational state rather than serving only as decoration.

Custom frontend components currently used include:

* Mushroom
* button-card
* card-mod
* ApexCharts Card
* Scheduler Card

A custom dark theme provides consistent card surfaces, typography, rounded corners, and accent colors while retaining a subtle astronomical background image.

See [`dashboard/README.md`](dashboard/README.md) for design notes and publication guidance.

## Energy monitoring

The Home Assistant energy view integrates solar, household load, grid import and export, battery activity, and battery state of charge.

Instantaneous power metrics are kept distinct from cumulative energy counters.

Daily energy values are derived from cumulative FranklinWH energy sensors with Home Assistant utility meters. This supports dashboard views for daily solar generation, household consumption, grid import and export, and battery charge and discharge activity.

Public examples use generic entity names so the repository demonstrates the design without exposing live device naming.

## Climate automation

The HVAC implementation combines deterministic scheduling with presence-aware logic and manual control.

The design separates scheduled setpoint changes from dynamic presence corrections so short absences do not unnecessarily alter the climate target.

See [`hvac/`](hvac/) for the case study, sanitized automation, and schedule design.

## Backup and recovery

Home Assistant uses application-level backups in addition to Proxmox recovery controls.

Backup copies are stored locally and on external Samba storage hosted outside the Home Assistant VM. Automatic backups, update-time backups, retention settings, and manual validation are treated as separate controls.

See [`../backup-recovery/README.md`](../backup-recovery/README.md) for the broader recovery model.

## Configuration publication model

Only configuration that demonstrates a reusable technical pattern is published.

Examples may include:

* YAML dashboard registration
* custom theme loading
* daily utility meters
* sanitized dashboard configuration
* automation patterns
* helper configuration

The following are excluded from the public repository:

* `secrets.yaml`
* access tokens and API keys
* passwords and alarm codes
* internal or external live URLs
* private IP addresses
* exact person or household identifiers
* precise location and zone coordinates
* webhook identifiers
* serial numbers and MAC addresses
* raw `.storage` contents
* unsanitized device and entity inventories

See [`configuration/configuration.example.yaml`](configuration/configuration.example.yaml) for a minimal sanitized configuration example.

## Design principles

1. **Prefer explicit state over decorative complexity.** Color and emphasis should communicate something meaningful.
2. **Separate current power from cumulative energy.** Power answers what is happening now; energy answers what happened over time.
3. **Keep automation behavior inspectable.** Scheduling, presence logic, and scripts should remain understandable and recoverable.
4. **Treat dashboards as interfaces, not data dumps.** Frequently used actions and high-value status should be visible before deeper telemetry.
5. **Use application-aware backups.** Hypervisor snapshots and Home Assistant backups solve different recovery problems.
6. **Sanitize before publication.** Public examples should remain useful without exposing the live environment.
