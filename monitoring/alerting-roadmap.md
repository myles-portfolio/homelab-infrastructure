# Monitoring and Alerting Roadmap

## Purpose

This roadmap defines the planned notification architecture across Checkmk, Prometheus, and Grafana.

Checkmk is now the primary platform for infrastructure and service-state monitoring. Prometheus remains the primary time-series metrics platform, while Grafana remains the primary visualization layer. Alertmanager is no longer a prerequisite for infrastructure alerting and should be introduced only where Prometheus-owned metric conditions require routing and notification handling.

The goal is to establish one authoritative notification path per operational condition, keep alerts actionable, and avoid duplicate notifications across monitoring platforms.

See [`checkmk-plan.md`](checkmk-plan.md) for the Checkmk deployment plan and [`README.md`](README.md) for the overall monitoring architecture.

## Current monitoring state

Checkmk platform deployment, low-risk onboarding, and core guest coverage are complete.

Validated Checkmk coverage now includes:

* Linux host and service-state monitoring
* controlled host, agent, and service failure detection
* internal and upstream DNS resolution checks
* authenticated SMB share availability
* user-facing web availability checks
* certificate-expiration checks where HTTPS is used
* Home Assistant availability
* Prometheus availability
* Grafana availability
* reusable rule targeting through folders, classifications, and labels
* container-specific memory threshold tuning where the default page-table percentage produced non-actionable warnings

Prometheus continues to collect time-series metrics for host, virtualization, and UPS telemetry. Grafana remains the primary visualization layer.

The next alerting work should focus on notification ownership and delivery rather than adding another monitoring platform by default.

## Target architecture

```text
Infrastructure and services
          |
          +--> Checkmk agents / active checks / SNMP
          |                 |
          |                 v
          |              Checkmk
          |                 |
          |                 v
          |        Checkmk notifications
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
                              |
                              v
                    Notification channels
```

This architecture does not imply that every condition should exist in both Checkmk and Prometheus. Each alert-worthy condition should have one authoritative owner.

## Alert ownership model

### Checkmk-owned conditions

Checkmk should normally own conditions where current operational state is the primary question.

Examples include:

* host availability
* Linux service state
* monitoring-agent availability
* filesystem state
* DNS resolution failure
* authenticated SMB share failure
* web application availability
* certificate expiration detected by active checks
* SNMP device and interface state when network monitoring is introduced

### Prometheus-owned conditions

Prometheus should normally own conditions where time-series behavior, sustained thresholds, rates, or historical context are required.

Examples include:

* sustained CPU utilization
* sustained memory pressure
* capacity trends
* exporter-derived UPS conditions
* thin-pool or storage-growth risk
* metrics that require rate, aggregation, or trend evaluation

### Shared-condition rule

The same operational condition should not normally send notifications through both platforms.

If both systems observe the same dependency, one may retain the telemetry while only the designated owner sends notifications.

## Phase 1: Checkmk notification foundation

### Define notification recipients and routing

Establish an initial Checkmk notification path for infrastructure and service-state conditions.

Success criteria:

* at least one notification destination is configured
* credentials and destination details remain outside the public repository
* warning, critical, and recovery notifications are delivered successfully
* routing can be scoped by host or service classification where useful

### Validate notification lifecycle

Test the complete Checkmk notification path using controlled conditions already proven during monitoring validation.

Candidate tests include:

* temporary service failure
* temporary agent communication failure
* controlled host outage
* active HTTP or DNS check failure

Success criteria:

* problem state is detected
* notification is delivered
* acknowledgement behavior is understood
* scheduled downtime suppresses expected notifications
* recovery notification is delivered

## Phase 2: High-value infrastructure notifications

Enable notifications only for conditions that require operator awareness or action.

Initial candidates include:

* core host DOWN states
* Checkmk agent communication failure on critical Linux systems
* DNS resolution failure
* Vaultwarden availability failure
* file-share availability failure
* Home Assistant availability failure
* Prometheus or Grafana endpoint failure
* filesystem capacity thresholds
* certificate expiration thresholds

Warning and critical thresholds should reflect operational response needs rather than dashboard aesthetics.

## Phase 3: Prometheus alerting evaluation

Alertmanager should be introduced only after identifying Prometheus-owned conditions that Checkmk does not represent as clearly.

### UPS and power events

The existing NUT exporter metrics are strong candidates for Prometheus alert rules because they represent state and telemetry already collected through the metrics stack.

Candidate conditions include:

* utility power loss
* UPS operating on battery
* low battery
* UPS or exporter communication loss
* return to normal power

### Sustained resource conditions

Prometheus may be preferable for conditions requiring time-window evaluation.

Examples include:

* sustained high CPU utilization
* sustained memory pressure
* unusual resource growth
* thin-pool capacity risk
* capacity trends that require historical context

### Alertmanager deployment decision

Deploy Alertmanager only if the selected Prometheus-owned conditions justify a separate routing component.

If deployed, implementation goals include:

* persistent configuration
* internal-only management access unless external access is explicitly required
* tested Prometheus-to-Alertmanager integration
* at least one tested notification destination
* grouping, deduplication, silencing, and recovery behavior validation
* credentials excluded from the public repository

## Phase 4: Network and hypervisor alert ownership

As Checkmk coverage expands to network devices and the Proxmox hypervisor, define alert ownership before enabling notifications.

Expected Checkmk candidates include:

* network device availability
* interface state
* hardware or environmental faults exposed through SNMP
* hypervisor host availability
* operating-system service state
* filesystem and ZFS health where available

Prometheus may continue to own historical performance and capacity conditions for the same infrastructure.

## Phase 5: Backup and recovery monitoring

Add visibility into backup recency, failure, and recoverability where reliable status data can be exposed.

Targets may include:

* Home Assistant backup age
* scheduled guest backup failures
* stale or missing application backups
* restore-test recency
* independent backup destination health when introduced

Backup creation alone should not be treated as proof of recoverability. Restore validation remains a separate operational requirement.

## Alert severity model

A simple severity model will be used initially:

| Severity | Meaning | Example |
|---|---|---|
| Info | State change worth recording but not requiring immediate action | Utility power restored |
| Warning | Condition requires attention but service is not yet critically affected | Certificate nearing expiration |
| Critical | Immediate intervention may be required | Core service unavailable |

Additional severity levels should be introduced only when a real operational requirement exists.

## Alerting principles

1. **Alert on actionable conditions.** Every enabled notification should correspond to an expected operator decision or action.
2. **Assign one authoritative owner.** Avoid sending the same operational condition through both Checkmk and Alertmanager.
3. **Avoid duplicate symptoms.** A single dependency failure should not create an unnecessary notification storm.
4. **Use sustained conditions where appropriate.** Prometheus is better suited to time-window and trend-based evaluation.
5. **Use warning thresholds before critical thresholds when early intervention is useful.**
6. **Document the expected response.** Persistent alerts should eventually link to response procedures or runbooks.
7. **Test the entire notification path.** Detection without delivery is not operational alerting.
8. **Validate recovery.** Recovery notifications and state clearing must be tested as deliberately as failure notifications.
9. **Use maintenance controls.** Planned outages should use acknowledgement or scheduled-downtime mechanisms rather than generating known noise.
10. **Keep secrets private.** Notification credentials, tokens, addresses, automation secrets, and internal topology remain outside public examples.
11. **Review alert quality.** Noisy, redundant, or non-actionable alerts should be tuned or removed.

## Validation requirements

A Checkmk-owned notification is not considered operational until this workflow has been tested:

```text
Condition triggered
      |
      v
Checkmk detects state
      |
      v
Notification rule matches
      |
      v
Notification is delivered
      |
      v
Condition resolves
      |
      v
Recovery notification verified
```

A Prometheus-owned notification is not considered operational until this workflow has been tested:

```text
Metric condition occurs
      |
      v
Prometheus rule fires
      |
      v
Alert reaches Alertmanager
      |
      v
Routing policy matches
      |
      v
Notification is delivered
      |
      v
Condition resolves
      |
      v
Resolved notification verified
```

Test alerts should be documented so intentional validation can be distinguished from genuine incidents.

## Planned documentation

As notification coverage is implemented, this section should gain:

* sanitized Checkmk notification-rule examples
* alert ownership matrix
* notification-routing documentation
* response runbooks for persistent alerts
* validation records for warning, critical, recovery, acknowledgement, and downtime behavior
* sanitized Prometheus alert-rule examples if Prometheus-owned alerts are introduced
* Alertmanager configuration and maintenance procedures only if Alertmanager is ultimately deployed

Completed implementation work should move from this roadmap into the main monitoring documentation and changelog.
