# Security and Sanitization Policy

This repository contains sanitized examples from a live personal homelab. It is intentionally curated for public viewing and must never be treated as a direct synchronization target for production configuration.

## Do not publish

The following information must not be committed to this repository:

* passwords, API keys, tokens, cookies, session data, recovery codes, or reusable authentication material
* `secrets.yaml` or equivalent secret stores
* exact home address, GPS coordinates, or precise geofence boundaries
* public IP addresses or private service endpoints that are not intentionally disclosed
* VPN credentials, private keys, certificates containing private key material, or authentication exports
* alarm, security, monitoring, or remote-access account identifiers
* personally identifying device tracker data
* backup archives, raw application databases, or restore artifacts
* Home Assistant `.storage/` contents
* unreviewed exports of live configuration
* live access-control policy files that reveal user identities, device identities, internal addresses, or service-specific allow rules
* exact internal address maps, MAC addresses, guest IDs, or hostnames where disclosure adds little technical value
* detailed backup placement, media topology, or recovery gaps that reveal exactly which failures would defeat the current protection model
* combinations of software versions, internal topology, service ports, and host roles that materially improve external reconnaissance

## Sanitization rules

Public examples should preserve engineering decisions while removing details that materially reduce uncertainty for an attacker.

Prefer generic identifiers and architectural descriptions over live values. Examples:

```yaml
person.primary_resident
zone.neighborhood
sensor.room_temperature
```

Examples of preferred abstractions:

* `private subnet` instead of the live CIDR
* `managed SMTP relay` instead of a live provider endpoint when the provider name adds no technical value
* `workload-specific scheduled backup` instead of the exact backup job, storage location, and retention values
* `current Linux release` instead of a patch-level operating-system fingerprint when the exact release is not central to the example
* `dedicated monitoring VM` instead of exact CPU, memory, and disk allocation unless sizing itself is the subject being documented

Device nicknames may be retained when they do not disclose sensitive information or create a useful map of the live environment.

## Publication workflow

1. Copy only the configuration or documentation required for the public example.
2. Replace personally identifying names, location names, endpoints, account identifiers, exact addresses, and unnecessary topology details.
3. Review the complete diff for both secrets and reconnaissance value.
4. Ask whether the same technical lesson can be shown with less precise infrastructure detail.
5. Confirm no secrets, precise location information, live access policy, or unnecessary recovery weakness is present.
6. Treat any accidentally committed credential as compromised and rotate it immediately.

## Repository boundary

The live homelab remains the operational source of truth. This repository is a sanitized portfolio and documentation layer only.

The public repository should explain patterns, controls, tradeoffs, and validation methods. It should not provide enough detail to reconstruct the live network, identify exact administrative paths, or determine the most effective way to defeat recovery controls.
