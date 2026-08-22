# Checkmk Configuration Standards

## Purpose

This document defines the sanitized configuration model used to scale Checkmk monitoring consistently across the homelab.

The objective is to avoid per-host configuration where inherited settings, controlled classifications, labels, and rules can express the same intent more reliably.

## Design principles

1. Use folders as configuration-inheritance boundaries, not merely visual organization.
2. Keep the folder hierarchy shallow until a real inheritance requirement justifies another level.
3. Use host tags for controlled classifications that will be referenced repeatedly by rules.
4. Use labels for flexible descriptive metadata that does not require a rigid taxonomy.
5. Avoid duplicating information already discovered by Checkmk or represented by another classification mechanism.
6. Prefer rules targeted by reusable classifications over explicit host lists.
7. Review effective parameters after introducing a new rule to verify the intended scope and resulting values.
8. Use stable semantic host identifiers rather than IP addresses as Checkmk host names.
9. Use dedicated least-privilege monitoring credentials for authenticated active checks rather than reusing application service accounts.
10. Tune only the specific parameter responsible for a non-actionable condition rather than suppressing an entire service check.

## Host naming standard

Checkmk host names follow this pattern:

```text
<environment>-<service>[-<role>]-<nn>
```

Examples:

```text
prod-dns-01
prod-files-01
prod-mon-01
dev-app-01
```

The optional role segment is used only when a service is split into distinct tiers, for example:

```text
dev-sam-app-01
dev-sam-db-01
```

Naming rules:

* use lowercase characters
* use hyphens as separators
* use short, controlled environment identifiers such as `prod`, `dev`, and `lab`
* identify the stable service or operational purpose rather than the product, operating system, hypervisor, or IP address
* use a two-digit instance suffix beginning with `01`
* keep the Checkmk host name stable when the implementation changes
* store the network address separately in the Checkmk IPv4 or IPv6 address field
* use the Alias field for a friendly product or workload name when useful

This separates system identity from addressing and implementation details.

## Folder model

The initial folder model separates lifecycle environments first, then introduces platform or operational groupings where they provide useful inheritance behavior.

```text
Main
├── Production
│   ├── Linux
│   │   ├── Infrastructure
│   │   └── Monitoring
│   ├── Appliances
│   │   └── Home Automation
│   └── Network
└── Development
    └── Linux
```

Additional folders should be created only when they provide a meaningful configuration or permission boundary.

## Custom host tags

Custom host tags are grouped under a common classification topic.

### Environment

| Tag ID | Display value |
|---|---|
| `production` | Production |
| `development` | Development |
| `lab` | Lab |

### Service Criticality

| Tag ID | Display value |
|---|---|
| `critical` | Critical |
| `high` | High |
| `normal` | Normal |
| `low` | Low |

### Platform

| Tag ID | Display value |
|---|---|
| `linux` | Linux |
| `windows` | Windows |
| `appliance` | Appliance |
| `network` | Network |

### Virtualization

| Tag ID | Display value |
|---|---|
| `vm` | VM |
| `container` | Container |
| `physical` | Physical |

### Service Class

| Tag ID | Display value |
|---|---|
| `core_infrastructure` | Core Infrastructure |
| `application` | Application |
| `monitoring` | Monitoring |
| `development` | Development |
| `utility` | Utility |

## Labels

Labels provide flexible metadata that complements the controlled host tags.

Current label patterns include:

```text
role:<service-role>
backup:<backup-policy>
hypervisor:<platform>
```

Examples:

```text
role:dns
role:application-development
role:fileshare
role:monitoring
backup:daily
hypervisor:proxmox
```

Do not create labels that duplicate existing tag values or Checkmk-discovered metadata.

## Rule targeting standard

Rules should normally target reusable classifications rather than individual hosts.

Validated patterns include:

* development filesystem thresholds targeted through environment, platform, virtualization, and folder scope
* active DNS checks targeted through production, core-infrastructure, and DNS-role classifications
* authenticated SMB checks targeted through production, core-infrastructure, and file-share role metadata
* application web checks targeted through service class and service-role metadata
* Linux-container memory tuning targeted through platform and virtualization tags

The intended operating model is:

```text
Folder inheritance
      +
Controlled host tags
      +
Flexible labels
      +
Rule conditions
      |
      v
Consistent effective monitoring behavior
```

## Linux host onboarding standard

The validated Linux onboarding sequence is:

1. choose a host name using the naming standard
2. place the host in the appropriate folder
3. set the explicit network address separately from the host name
4. apply required inherited and explicit classifications
5. install the Checkmk Linux agent package
6. register the agent using the same Checkmk host name
7. restrict the agent listener to the monitoring server where host firewalling is enabled
8. validate agent connectivity
9. run service discovery
10. review discovered host labels and services before acceptance
11. activate the configuration
12. confirm host and accepted service state
13. review effective parameters when new rules or classifications are introduced
14. add application-level active checks for important services where practical

## Authenticated active-check credentials

Authenticated service checks should use dedicated monitoring identities wherever practical.

Requirements:

* do not reuse a write-capable application service account when read-only monitoring is sufficient
* use non-login operating-system identities where the monitored protocol supports separate service credentials
* grant only the minimum protocol and resource permissions required by the active check
* validate the credential manually before placing it into Checkmk
* verify denied operations as well as allowed operations when enforcing least privilege
* never commit monitoring passwords or automation secrets to the public repository

For SMB monitoring, the validated pattern uses a dedicated Samba account that can authenticate and enumerate a monitored share but cannot create or modify content.

## Linux container memory thresholds

Small Linux containers can show elevated page-table usage as a percentage of their limited assigned RAM even when ordinary memory availability is healthy.

The effective parameters of the Linux memory service should be inspected before changing thresholds. In the current container baseline, the non-actionable warning was isolated specifically to the page-table percentage component.

A dedicated rule therefore changes only the page-table thresholds for hosts matching:

```text
Platform = Linux
Virtualization = Container
```

Current page-table levels:

* warning: 15 percent
* critical: 25 percent

All other Linux memory parameters remain at their normal defaults. This preserves RAM, swap, committed-memory, and other memory alerts while reducing known container-specific noise.

## Container considerations

Linux containers may require additional runtime capabilities for modern systemd services used by monitoring agents. When a service fails with a namespace-related systemd status, prefer correcting the container capability model rather than disabling service hardening.

For the current unprivileged Proxmox LXC model, enabling the required nesting capability resolved namespace failures affecting the Checkmk agent controller and other systemd services.

Container network configuration should also be kept intentional. Unused DHCPv6 configuration can delay network readiness and dependent service startup, so unused address-family configuration should be removed rather than left to time out during boot.

A privileged file-services container exposed a separate expected limitation: the guest AppArmor loader could not replace kernel profiles while the container remained confined by the Proxmox host. The guest AppArmor loader was disabled after confirming that host-level confinement remained in place, preventing a non-actionable failed-unit alert without weakening the container boundary.

## Publication requirements

Public examples must not expose:

* private IP addresses
* live internal or public hostnames
* site secrets or registration credentials
* authentication material
* notification destinations
* SNMP credentials
* API tokens

The public documentation should preserve architecture, taxonomy, workflow, and operating decisions while omitting live topology and secrets.
