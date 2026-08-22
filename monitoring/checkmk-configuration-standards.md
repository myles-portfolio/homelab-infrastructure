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

This separates system identity from addressing and implementation details. For example, a DNS host may remain `prod-dns-01` even if the DNS software changes later.

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

This classification is intentionally separate from Checkmk's existing built-in criticality tag group so operational importance is not mixed with lifecycle or monitoring-state semantics.

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
backup:daily
hypervisor:proxmox
```

Do not create labels that duplicate existing tag values or Checkmk-discovered metadata. For example, operating system, device type, and environment should not be repeated as labels when they already exist elsewhere in the configuration model.

## Rule targeting standard

Rules should normally target reusable classifications rather than individual hosts.

A validated example applies development filesystem thresholds to systems matching all of the following conditions:

* folder: Development Linux systems
* environment: Development
* service criticality: Normal
* platform: Linux
* virtualization: VM

The rule sets filesystem used-space thresholds to:

* warning: 80 percent
* critical: 90 percent

Effective service parameters were reviewed after activation and confirmed that the intended rule supplied the resulting thresholds.

A second validated pattern uses an active DNS check targeted by reusable classifications and a DNS-role label. The check queries the monitored DNS host directly and validates that a known internal record resolves to its expected address. This tests application behavior rather than only host or process state.

This pattern demonstrates the intended operating model:

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

## Container considerations

Linux containers may require additional runtime capabilities for modern systemd services used by monitoring agents. When a service fails with a namespace-related systemd status, prefer correcting the container capability model rather than disabling service hardening.

For the current unprivileged Proxmox LXC model, enabling the required nesting capability resolved namespace failures affecting the Checkmk agent controller and other systemd services.

Container network configuration should also be kept intentional. A production DNS container was found to have an unused DHCPv6 setting alongside static IPv4 addressing. The unresolved DHCPv6 request delayed network readiness and therefore delayed DNS service startup. Removing the unused DHCPv6 configuration restored prompt service startup after reboot.

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
