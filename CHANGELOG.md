# Changelog

This changelog is a curated, sanitized history of material homelab changes. It is derived from the internal operational log but intentionally omits exact private IPs, public-facing domains, personal entity identifiers, and other sensitive implementation details.

## 2026-08-17

### Built YAML-managed Home Assistant Command Center dashboard

Created and implemented a new YAML-managed Home Assistant dashboard organized around three functional domains: quick actions and home state, climate and weather, and energy monitoring.

Key changes:

* introduced a three-column Sections-based layout
* added state-aware home, security, door, garage, lighting, and routine controls
* grouped time, thermostat control, indoor and outdoor environmental data, weather, climate trends, and HVAC scheduling into a single climate domain
* added energy production, household load, grid activity, battery state, daily energy values, and trend visualizations
* retained the existing astronomical background while improving information density and visual hierarchy

### Added custom Home Assistant dashboard theme and frontend components

Expanded the Home Assistant frontend stack and standardized the dashboard visual language.

Key changes:

* installed Mushroom, card-mod, button-card, and ApexCharts Card through HACS
* created a custom Command Center dark theme
* standardized rounded card surfaces, typography, borders, transparency, and semantic state colors
* used custom cards for compact controls, state-aware presentation, and time-series visualization

### Added daily FranklinWH energy telemetry

Created daily Home Assistant utility meters from cumulative FranklinWH energy sensors so the dashboard can distinguish instantaneous power from daily energy usage.

Daily metrics now cover:

* solar generation
* household consumption
* grid import
* grid export
* battery charge energy
* battery discharge energy

The utility meters use daily cycles and are configured for cumulative source meters that do not reset themselves.

## 2026-08-16

### Added presence-aware HVAC control

Implemented Home Assistant based climate control using Versatile Thermostat, Scheduler Component, Alarm.com integration, and presence-aware automation.

Key changes:

* established Aeolus as the primary Home Assistant thermostat control layer
* created separate weekly schedules for workdays, Friday, Saturday, and Sunday
* added a neighborhood zone and delayed absence threshold to avoid reacting to short trips
* added a manual Work From Home helper and dashboard control
* validated the full control path from Home Assistant to the physical thermostat

### Expanded Proxmox guest maintenance coverage

Reconciled the live guest inventory and identified previously omitted VMs in the maintenance workflow.

Key changes:

* added Monitoring VM maintenance
* added Development VM maintenance
* added Home Assistant VM maintenance
* established inventory reconciliation as a required maintenance step

### Monitoring VM maintenance and recovery

Performed full maintenance on the Monitoring VM.

Key changes:

* recovered administrative access
* updated Ubuntu packages and Docker components
* installed and validated QEMU Guest Agent
* refreshed Prometheus, Grafana, and NUT exporter images
* validated Grafana and Prometheus HTTP responses
* validated NUT exporter health through Prometheus target status
* removed temporary rollback snapshot after successful validation

### Development VM maintenance

Performed maintenance on the PostgreSQL development VM.

Key changes:

* created a logical backup of the application database
* took a temporary Proxmox snapshot
* updated Ubuntu packages
* validated PostgreSQL service state and application database queries
* verified QEMU Guest Agent communication
* removed the snapshot after validation while retaining the logical database backup

### Home Assistant backup resilience

Improved Home Assistant recovery capability.

Key changes:

* configured automatic backups
* enabled backup creation before Home Assistant updates
* configured seven retained backups
* included Home Assistant settings, history, apps, and share data
* created a dedicated Samba backup share with a service account
* configured both local and external network backup locations
* validated a manual backup to both locations

## 2026-08-12

### Migrated Home Assistant reverse-proxy configuration

Moved Home Assistant HTTP reverse-proxy settings from YAML into the supported network configuration interface while preserving trusted-proxy behavior.

The change was made to maintain compatibility with future Home Assistant releases as YAML-based HTTP configuration is deprecated.

## 2026-08-08

### Core homelab maintenance

Updated the Proxmox host and core service guests, including DNS, password management, and file services.

Validation included:

* host health
* DNS resolution
* Vaultwarden web access and container health
* Samba service health
* intentionally stopped media workload state

### Vaultwarden application refresh

Updated the Vaultwarden application image and restored normal browser-extension operation after client compatibility issues.

The successful recovery required both the server update and a full browser-extension logout and sign-in.

## 2026-06-19

### Proxmox and core-service maintenance

Performed routine maintenance across the Proxmox host and core service guests.

Key changes:

* updated host packages
* updated DNS, Vaultwarden, and file-service guest packages
* refreshed the Vaultwarden Docker image and recreated the Compose service
* preserved local configuration files during package prompts
* enabled automatic start for critical service guests
* validated DNS, password-management, and file-service functionality

## 2026-05-27

### Rebuilt Home Assistant as a new VM

Rebuilt the Home Assistant deployment using the Home Assistant OS KVM image after the previous VM became unreachable.

Additional work included:

* serial-console troubleshooting
* static network configuration
* HACS installation
* Alarm.com custom integration installation
* identification of an integration compatibility defect
* migration to a compatible fork
* creation of a dedicated security dashboard

## 2026-02-22

### Implemented split DNS and HTTPS for Vaultwarden

Introduced local split DNS and moved Vaultwarden behind a trusted HTTPS reverse-proxy path without opening inbound WAN ports.

The design standardized domain-based service access, removed direct service-port exposure, and improved certificate trust.

## 2026-02-15

### Deployed observability stack

Deployed Prometheus and Grafana to provide infrastructure monitoring and dashboards.

The stack later evolved to include NUT exporter monitoring for power-related metrics.

## 2026-02-14

### Added UPS monitoring and host power tuning

Configured Network UPS Tools for power-event awareness and graceful shutdown capability.

Also updated CPU power-management tooling and tuned host CPU scaling behavior.

## 2026-02-09

### Centralized file services

Configured centralized file shares for backups and media storage.

## 2026-02-07

### Added Steam backup storage

Configured centralized storage for selected game libraries to reduce rebuild and re-download time.

## 2026-02-06

### Deployed Home Assistant and media services

Deployed the initial Home Assistant VM and Plex container.

## 2026-02-05

### Built Proxmox host

Completed the initial Proxmox VE host build, including base operating-system configuration, networking, storage pools, and initial provisioning.
