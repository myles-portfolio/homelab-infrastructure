# Monitoring and Alerting Roadmap

## Purpose

This roadmap defines the notification architecture across Checkmk, Prometheus, and Grafana.

Checkmk is now the primary platform for infrastructure and service-state monitoring. Prometheus remains the primary time-series metrics platform, while Grafana remains the primary visualization layer. Alertmanager is no longer a prerequisite for infrastructure alerting and should be introduced only where Prometheus-owned metric conditions require routing and notification handling.

The goal is to establish one authoritative notification path per operational condition, keep alerts actionable, and avoid duplicate notifications across monitoring platforms.

See [`checkmk-plan.md`](checkmk-plan.md) for the Checkmk deployment plan, [`checkmk-notifications.md`](checkmk-notifications.md) for the sanitized Checkmk email-delivery implementation, and [`README.md`](README.md) for the overall monitoring architecture.

## Current monitoring state

Checkmk platform deployment, low-risk onboarding, core guest coverage, network monitoring, hypervisor monitoring, and baseline email notification delivery are complete.

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
* Proxmox host, ZFS, process, interface, and physical-disk health monitoring
* contact-group-based notification routing
* HTML email notifications through a local Postfix relay and managed SMTP provider
* a fallback email destination for unmatched notifications
* successful end-to-end Checkmk notification delivery testing

Prometheus continues to collect time-series metrics for host, virtualization, and UPS telemetry. Grafana remains the primary visualization layer.

The next alerting work should focus on alert quality, lifecycle validation, and clearly separated ownership rather than adding another monitoring platform or notification path by default.

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
                              |
                              v
                    Notification channels
```

The Prometheus-to-Alertmanager branch remains conditional. It should be implemented only where Prometheus-owned metric conditions require alert routing.

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
* hypervisor host state, ZFS health, and physical-disk health where Checkmk already has authoritative state coverage

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

Status: **Complete**

### Notification recipients and routing

The initial Checkmk notification path is operational.

Completed work:

* created an administrative contact group for monitored infrastructure
* assigned the contact group to monitored hosts through a reusable assignment rule
* configured a contact email destination outside the public repository
* configured a fallback email destination
* retained the built-in HTML email notification method
* configured the baseline notification rule for host DOWN and UP events and service WARN, CRIT, UNKNOWN, and OK events

### Outbound delivery

Checkmk Community uses the local Linux mail transport rather than commercial direct-SMTP delivery.

Completed work:

* installed Postfix on the Checkmk VM
* configured a managed SMTP relay as the Postfix smarthost
* enabled SMTP authentication and required TLS for outbound submission
* verified a dedicated sender subdomain through public DNS
* stored SMTP credentials outside the public repository
* validated direct Postfix relay delivery
* validated a real Checkmk HTML notification from rule match through final mailbox delivery

See [`checkmk-notifications.md`](checkmk-notifications.md) for the sanitized implementation.

### Remaining lifecycle validation

Baseline delivery is complete, but operational behavior still needs deliberate validation over time.

Candidate tests include:

* temporary service failure followed by recovery
* temporary agent communication failure followed by recovery
* controlled host outage followed by recovery
* acknowledgement behavior
* scheduled-downtime suppression

Success criteria:

* problem state is detected
* problem notification is delivered
* acknowledgement behavior is understood
* scheduled downtime suppresses expected notifications
* recovery notification is delivered

## Phase 2: High-value infrastructure notifications

Enable notifications only for conditions that require operator awareness or action.

Initial candidates include:

* core host DOWN states
* Checkmk agent communication failure on critical Linux systems
* DNS resolution failure
* password-management availability failure
* file-share availability failure
* Home Assistant availability failure
* Prometheus or Grafana endpoint failure
* filesystem capacity thresholds
* certificate expiration thresholds
* ZFS or physical-disk health faults

The baseline rule currently includes WARN notifications so real-world volume can be observed. Warning and critical thresholds should reflect operational response needs rather than dashboard aesthetics.

Potential tuning after observation includes:

* short delays for transient WARN or CRIT states
* suppression of known non-actionable warnings
* periodic reminders for persistent critical conditions
* routing differences based on host or service classification

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

Current network and hypervisor coverage is operational, but notification ownership should remain deliberate as deeper checks are added.

Expected Checkmk candidates include:

* network device availability
* interface state on future SNMP-capable devices
* hardware or environmental faults exposed through SNMP
* hypervisor host availability
* operating-system service state
* ZFS health
* physical-disk SMART health

Prometheus may continue to own historical performance and capacity conditions for the same infrastructure.

The current Proxmox API special-agent integration remains disabled because of a compatibility failure. Existing Linux-agent and Prometheus exporter coverage remains authoritative for the supported checks until that integration is revisited.

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

A Checkmk-owned notification is considered delivery-operational when this path has been tested:

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
Postfix accepts message
      |
      v
Managed relay accepts message
      |
      v
Recipient mailbox receives message
```

That baseline path is now validated.

Full lifecycle validation additionally requires:

```text
Real or controlled problem occurs
      |
      v
Checkmk detects state
      |
      v
Problem notification is delivered
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

Completed notification-delivery architecture is documented in [`checkmk-notifications.md`](checkmk-notifications.md).

Additional documentation should be added as the alerting model matures:

* sanitized Checkmk notification-rule examples where useful
* alert ownership matrix
* response runbooks for persistent alerts
* validation records for recovery, acknowledgement, and downtime behavior
* sanitized Prometheus alert-rule examples if Prometheus-owned alerts are introduced
* Alertmanager configuration and maintenance procedures only if Alertmanager is ultimately deployed

Completed implementation work should continue moving from this roadmap into the main monitoring documentation and changelog.
