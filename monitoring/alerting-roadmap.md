# Monitoring and Alerting Roadmap

## Purpose

This roadmap defines the planned alerting architecture for the existing Prometheus and Grafana monitoring stack.

The selected implementation path uses Prometheus alert rules for condition evaluation and Alertmanager for grouping, deduplication, silencing, routing, and notification delivery.

The goal is to expand monitoring coverage gradually while keeping alerts actionable and avoiding unnecessary noise.

Checkmk Community is also planned as a complementary infrastructure and service-monitoring platform. Its notification capabilities will be introduced deliberately so that overlapping conditions do not produce duplicate alerts. See [`checkmk-plan.md`](checkmk-plan.md) for the Checkmk deployment plan and alert ownership model.

## Target architecture

```text
Metric sources
    |
    v
Prometheus
    |
    +--> stores time-series data
    |
    +--> evaluates alert rules
              |
              v
         Alertmanager
              |
              v
     Notification channels
```

Grafana remains the primary visualization layer and queries Prometheus for dashboards and analysis. Alertmanager is a separate operational component responsible for notification handling.

Where Checkmk monitors the same infrastructure condition, one platform should be designated as the authoritative notification path. The intent is to use Alertmanager for metrics-based conditions and Checkmk for infrastructure or service-state conditions where that distinction is operationally useful.

See [`README.md`](README.md) for the monitoring architecture diagram.

## Phase 1: Alerting foundation

### Deploy Alertmanager

Add Alertmanager as a managed service in the Monitoring VM's Docker Compose stack.

Implementation goals:

* use a persistent configuration file
* keep Alertmanager on the internal monitoring network
* configure Prometheus to send alerts to Alertmanager
* avoid publishing the Alertmanager interface externally unless there is a clear operational requirement
* document the deployment and maintenance workflow

### Configure notification routing

Establish at least one tested notification destination.

Initial options may include:

* email
* chat or mobile notification integration

Success criteria:

* a known test alert reaches Alertmanager
* Alertmanager routes the test alert to the intended destination
* resolved notifications are delivered when appropriate
* notification credentials are stored outside the public repository

### Create baseline alert rules

Begin with a small set of high-value alerts rather than broad metric coverage.

Initial rules should be easy to understand, validate, and respond to operationally.

## Phase 2: Near-term alert coverage

### UPS and power events

Use the existing NUT exporter metrics to notify on meaningful power conditions.

Candidate conditions include:

* utility power loss
* UPS operating on battery
* low battery
* communication loss with the UPS or exporter
* return to normal power

Alerts should distinguish informational state changes from conditions that require operator action.

### Monitoring target failures

Alert when critical Prometheus scrape targets become unavailable.

Initial targets include:

* Node Exporter on the Proxmox host
* Proxmox/PVE exporter
* NUT exporter path

A short delay should be used where appropriate so brief network interruptions do not immediately generate noise.

### Host resource conditions

Add selected Proxmox host alerts using the existing Node Exporter and PVE metrics.

Candidate signals include:

* high sustained CPU utilization
* memory pressure
* critically low filesystem capacity
* thin-pool capacity risk
* exporter or host monitoring loss

Thresholds should be based on conditions that require intervention rather than arbitrary utilization percentages.

## Phase 3: Service and infrastructure coverage

### Certificate expiration monitoring

Add visibility into TLS certificate expiration for internally hosted HTTPS services.

Success criteria:

* expiration date is collected as a metric
* alert threshold provides enough lead time for remediation
* alert identifies the affected service clearly
* certificate monitoring does not depend solely on manual browser inspection

### Core service availability

Add availability checks for critical internal services such as:

* DNS
* Vaultwarden
* Home Assistant
* file services
* monitoring endpoints

Availability checks should validate the path users actually depend on, not only process state.

### Backup monitoring

Add visibility into backup recency and failure states where metrics or status data can be exposed safely.

Targets include:

* Home Assistant backup age
* application backup completion
* restore-test status
* stale or missing backup conditions

### Guest health

Expand Proxmox monitoring to identify guest-level conditions that warrant attention.

Useful signals include:

* unexpected stopped state
* sustained guest resource pressure
* storage-capacity concerns
* monitoring loss for critical workloads

## Phase 4: Application-specific health

Where practical, add health checks that confirm applications are usable rather than only listening on a port.

Examples include:

* Prometheus target health
* Grafana data-source availability
* DNS lookup success
* Vaultwarden HTTPS response
* Home Assistant HTTP response
* PostgreSQL query success

These checks should be dependency-aware where possible so a single infrastructure failure does not create many redundant alerts.

## Alert severity model

A simple severity model will be used initially:

| Severity | Meaning | Example |
|---|---|---|
| Info | State change worth recording but not requiring immediate action | Utility power restored |
| Warning | Condition requires attention but service is not yet critically affected | Certificate nearing expiration |
| Critical | Immediate intervention may be required | Low UPS battery during outage |

Additional severity levels should be added only if operational need justifies them.

## Alert ownership

As Checkmk is introduced, each monitored condition should have a defined notification owner.

Examples:

* sustained resource thresholds may remain owned by Prometheus and Alertmanager
* Linux service state, filesystem state, and SNMP device state may be owned by Checkmk
* external service availability may be assigned to either platform based on which check provides the clearest operational signal

The same condition should not normally notify through both platforms.

## Alerting principles

1. **Alert on actionable conditions.** Every enabled alert should correspond to an expected operator decision or action.
2. **Avoid duplicate symptoms.** One dependency failure should not generate an unnecessary storm of secondary alerts.
3. **Use sustained conditions where appropriate.** Brief spikes and transient network loss should not automatically page the operator.
4. **Use warning thresholds before critical thresholds when early intervention is useful.**
5. **Document the expected response.** Every persistent alert should eventually link to a response runbook.
6. **Test the entire path.** Rule evaluation alone is not sufficient; Alertmanager routing and notification delivery must also be validated.
7. **Review alert quality.** Noisy, redundant, or non-actionable alerts should be tuned or removed.
8. **Keep secrets private.** Notification credentials, tokens, and addresses are excluded from public configuration examples.
9. **Define alert ownership across monitoring platforms.** Overlapping checks should have one authoritative notification path.

## Validation requirements

An alert is not considered operational until the full workflow has been tested intentionally:

```text
Condition triggered
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

For Checkmk-owned conditions, equivalent end-to-end testing should verify service-state detection, notification routing, acknowledgement or maintenance behavior where applicable, and recovery notification.

Test alerts should be documented so maintenance can distinguish intentional validation from genuine incidents.

## Planned documentation

As the implementation progresses, this section should gain:

* sanitized Prometheus alert-rule examples
* Alertmanager configuration examples
* notification-routing documentation
* alert ownership documentation for Checkmk and Prometheus
* alert response runbooks
* validation records showing successful end-to-end tests
* maintenance procedures for Alertmanager

Once the alerting foundation is deployed and validated, completed items should move from this roadmap into the main monitoring documentation and changelog.
