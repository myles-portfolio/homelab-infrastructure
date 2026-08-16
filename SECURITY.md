# Security and Sanitization Policy

This repository contains sanitized examples from a live personal homelab. It is intentionally curated for public viewing and must never be treated as a direct synchronization target for production configuration.

## Do not publish

The following information must not be committed to this repository:

* Passwords, API keys, tokens, cookies, session data, or recovery codes
* `secrets.yaml` or equivalent secret stores
* Exact home address, GPS coordinates, or precise geofence boundaries
* Public IP addresses or private service endpoints that are not intentionally disclosed
* VPN credentials, private keys, certificates containing private key material, or authentication exports
* Alarm, security, or monitoring account identifiers
* Personally identifying device tracker data
* Backup archives or raw application databases
* Home Assistant `.storage/` contents
* Unreviewed exports of live configuration

## Sanitization rules

Public examples should use generic identifiers where disclosure would add no technical value. Examples:

```yaml
person.primary_resident
zone.neighborhood
sensor.room_temperature
```

Device nicknames may be retained when they do not disclose sensitive information.

## Publication workflow

1. Copy only the configuration or documentation required for the public example.
2. Replace personally identifying names, location names, endpoints, and account identifiers.
3. Review the complete diff before committing.
4. Confirm no secrets or precise location information are present.
5. Treat any accidentally committed credential as compromised and rotate it immediately.

## Repository boundary

The live homelab remains the operational source of truth. This repository is a sanitized portfolio and documentation layer only.
