# Homelab Infrastructure

Sanitized documentation, automation, and configuration examples from my personal homelab environment.

This repository is intended as a public portfolio of systems administration, infrastructure engineering, automation, observability, documentation, and systems design work. It deliberately excludes live credentials, exact location data, private endpoints, secrets, and other sensitive operational details.

## Current case studies

### Presence-aware residential HVAC control

A Home Assistant based HVAC control system that combines deterministic weekly scheduling with presence-aware daytime temperature control, neighborhood geofencing, delayed absence detection, and manual override capability.

Control path:

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
                Honeywell thermostat
```

See [`home-assistant/hvac/`](home-assistant/hvac/) for the sanitized automation and design notes.

## Planned areas

* Home Assistant automation and dashboard patterns
* Proxmox VE architecture and maintenance
* Pi-hole DNS services
* Vaultwarden operations and maintenance
* Reverse proxy configuration patterns
* Zigbee and IoT integration
* Monitoring and service health
* Change management and runbooks

## Security and sanitization

The public repository is a curated documentation source, not a mirror of the live homelab configuration. Configuration examples are reviewed and sanitized before publication.

See [`SECURITY.md`](SECURITY.md) for publication rules.
