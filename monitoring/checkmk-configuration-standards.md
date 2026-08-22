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
role:database
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

This pattern demonstrates the intended operating model:

```text
Folder inheritance
      +
Controlled host tags
      +
Rule conditions
      |
      v
Consistent effective service parameters
```

## Linux host onboarding standard

The validated Linux onboarding sequence is:

1. place the host in the appropriate folder
2. apply required inherited and explicit classifications
3. install the Checkmk Linux agent package
4. register the agent with the Checkmk site
5. restrict the agent listener to the monitoring server where host firewalling is enabled
6. validate agent connectivity
7. run service discovery
8. review discovered host labels and services before acceptance
9. activate the configuration
10. confirm host and accepted service state
11. review effective parameters when new rules or classifications are introduced

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
