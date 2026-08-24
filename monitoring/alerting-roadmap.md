# Monitoring and Alerting Roadmap

## Purpose

This roadmap defines notification ownership across Checkmk, Prometheus, and Grafana.

Checkmk is the primary platform for infrastructure and service-state monitoring. Prometheus remains the primary time-series metrics platform, while Grafana remains the primary visualization layer. Alertmanager is optional and should be introduced only where Prometheus-owned metric conditions require a separate routing and notification component.

The goal is one authoritative notification path per operational condition.

See [`checkmk/README.md`](checkmk/README.md) for the Checkmk documentation index and [`README.md`](README.md) for the overall monitoring architecture.

## Current state

Checkmk deployment phases 1 through 6 are complete.

Validated capabilities include:

* Linux host and service-state monitoring
* controlled host, agent, and service failure detection
* internal and upstream DNS checks
* authenticated SMB share monitoring
* user-facing web checks
* certificate-expiration checks where HTTPS is used
* Home Assistant, Prometheus, and Grafana availability checks
* current network-device reachability and management-interface monitoring
* Proxmox host, ZFS, process, interface, and physical-disk health monitoring
* reusable classification-based rule targeting
* contact-group-based notification routing
* HTML email notification delivery through Postfix and a managed SMTP relay
* fallback notification delivery
* successful end-to-end Checkmk notification testing

Prometheus continues to collect time-series host, virtualization, and UPS telemetry. Grafana remains the visualization layer.

## Target architecture

```text
Infrastructure and services
          |
          +--> Checkmk agents / active checks / supported network monitoring
          |                 |
          |                 v
          |              Checkmk
          |                 |
          |                 v
          |        Checkmk notifications
          |                 |
          |                 v
          |              Postfix
          |                 |
          |                 v
          |          managed SMTP relay
          |                 |
          |                 v
          |        notification mailbox
          |
          +--> exporters
                    |
                    v
                Prometheus
                    |
                    +--> Grafana
                    |
                    +--> Prometheus alert rules
                              |
                              v
                         Alertmanager
```

The Alertmanager branch remains conditional.

## Alert ownership model

### Checkmk-owned conditions

Checkmk should normally own conditions where current operational state is the primary question.

Examples include:

* host availability
* monitoring-agent availability
* Linux service state
* filesystem state
* DNS resolution failure
* authenticated share failure
* application endpoint availability
* certificate expiration detected by active checks
* supported network-device and interface state
* hypervisor host state
* ZFS health
* physical-disk SMART health

### Prometheus-owned conditions

Prometheus should normally own conditions where historical context, sustained thresholds, rates, or aggregation are required.

Examples include:

* sustained CPU utilization
* sustained memory pressure
* capacity trends
* thin-pool or storage-growth risk
* exporter-derived UPS telemetry
* metric conditions requiring time-window evaluation

### Shared-condition rule

A condition may be observed by multiple systems, but only one should normally send notifications.

## Phase 1: Checkmk notification foundation

Status: **Complete**

Completed work:

* created an administrative Checkmk contact group
* assigned the contact group through a reusable host rule
* configured contact and fallback email destinations
* retained the built-in HTML email method
* configured host DOWN and UP notifications
* configured service WARN, CRIT, UNKNOWN, and OK notifications
* installed and configured Postfix as the local outbound mail transport
* configured authenticated STARTTLS delivery through a managed SMTP relay
* verified the sender domain through public DNS
* stored credentials outside the public repository
* validated direct Postfix delivery
* validated real Checkmk notification delivery end to end

See [`checkmk/checkmk-notifications.md`](checkmk/checkmk-notifications.md) for the sanitized implementation.

## Phase 2: Notification quality and lifecycle validation

Status: **In progress**

The baseline rule remains intentionally broad while real alert volume is observed.

Validation and tuning priorities:

* confirm recovery emails for controlled service and host failures
* validate acknowledgement behavior
* validate scheduled-downtime suppression
* identify transient WARN or CRIT states that justify a short delay
* suppress only confirmed non-actionable noise
* consider periodic reminders only for persistent critical conditions
* refine routing by classification if multiple operational owners are introduced

Warning notifications remain enabled during this observation period.

## Phase 3: High-value infrastructure notification refinement

Checkmk notifications should remain focused on conditions that require operator awareness or action.

Priority conditions include:

* core host DOWN states
* Checkmk agent failures on critical Linux systems
* DNS resolution failures
* password-management availability failures
* file-share failures
* Home Assistant availability failures
* Prometheus or Grafana endpoint failures
* filesystem capacity thresholds
* certificate expiration thresholds
* ZFS or physical-disk health faults

## Phase 4: Prometheus alerting evaluation

Alertmanager should be deployed only if Prometheus-owned metric conditions justify it.

Candidate conditions include:

* utility power loss
* UPS on battery
* low UPS battery
* UPS or exporter communication loss
* sustained CPU utilization
* sustained memory pressure
* unusual capacity growth
* thin-pool risk

If Alertmanager is introduced, implementation should include:

* persistent configuration
* internal-only management access unless external access is explicitly required
* tested Prometheus integration
* at least one tested notification destination
* grouping, deduplication, silencing, and resolved-notification validation
* credentials excluded from the public repository

A future `prometheus/` directory should contain Prometheus-specific alert rules and Alertmanager procedures if this phase is implemented.

## Phase 5: Backup and recovery monitoring

Future monitoring should add reliable visibility into backup recency, failure, and recoverability.

Potential targets include:

* Home Assistant backup age
* scheduled guest backup failures
* stale or missing application backups
* restore-test recency
* independent backup-destination health

Backup creation alone is not proof of recoverability.

## Alert severity model

| Severity | Meaning | Example |
|---|---|---|
| Info | State change worth recording but not requiring immediate action | Utility power restored |
| Warning | Attention required before critical service impact | Certificate nearing expiration |
| Critical | Immediate intervention may be required | Core service unavailable |

Additional severity levels should be introduced only when an operational requirement exists.

## Alerting principles

1. Alert on actionable conditions.
2. Assign one authoritative owner per condition.
3. Avoid duplicate symptom notifications.
4. Use Prometheus for sustained and time-window conditions where appropriate.
5. Tune warning and critical thresholds according to response needs.
6. Document expected operator response for persistent alerts.
7. Test the entire notification path.
8. Validate recovery notifications deliberately.
9. Use acknowledgement and scheduled downtime for planned work.
10. Keep credentials, notification addresses, and topology details out of public examples.
11. Review noisy or redundant alerts and tune them narrowly.

## Validation requirements

A Checkmk delivery path is considered operational when this sequence succeeds:

```text
Test event
   |
   v
Notification rule matches
   |
   v
Contact is selected
   |
   v
Notification plug-in executes
   |
   v
Postfix accepts the message
   |
   v
Managed SMTP relay accepts the message
   |
   v
Recipient mailbox receives the message
```

This baseline path is validated.

Full lifecycle validation also requires a controlled problem to generate a notification and its recovery to generate the expected recovery notification.

A future Prometheus-owned alert is not operational until the metric condition, Prometheus rule, Alertmanager route, destination delivery, and resolved state have all been tested.

## Documentation model

Cross-platform architecture stays at the `monitoring/` level. Product-specific implementation documentation belongs in product subdirectories.

Current structure:

```text
monitoring/
├── README.md
├── alerting-roadmap.md
├── checkmk/
│   ├── README.md
│   ├── checkmk-plan.md
│   ├── checkmk-configuration-standards.md
│   └── checkmk-notifications.md
└── diagrams/
```

Prometheus and Grafana should receive their own directories when their documentation grows beyond the shared overview.
