# Monitoring and Alerting Roadmap

## Purpose

This roadmap identifies practical observability improvements that build on the existing Prometheus and Grafana stack.

The goal is to expand coverage gradually while keeping alerts actionable and avoiding unnecessary noise.

## Near-term priorities

### Certificate expiration monitoring

Add visibility into TLS certificate expiration for internally hosted HTTPS services.

Success criteria:

* expiration date is collected as a metric
* alert threshold provides enough lead time for remediation
* alert identifies the affected service clearly
* certificate monitoring does not depend solely on manual browser inspection

### UPS and power-event notifications

Add notification delivery for meaningful UPS events already surfaced through NUT-related monitoring.

Potential conditions include:

* utility power loss
* battery operation
* low battery
* communication loss with UPS
* return to normal power

Alerts should distinguish informational state changes from conditions that require action.

### Core service availability

Add simple availability checks for critical internal services such as:

* DNS
* Vaultwarden
* Home Assistant
* file services
* monitoring endpoints

Availability checks should validate the service path that users depend on, not only process state.

## Medium-term priorities

### Backup monitoring

Add visibility into backup recency and failure states where metrics or status data can be exposed safely.

Targets include:

* Home Assistant backup age
* application backup completion
* restore-test status
* stale or missing backup conditions

### Proxmox and guest health

Expand host and guest-level monitoring to make capacity and failure conditions easier to identify.

Useful signals include:

* CPU utilization
* memory pressure
* storage utilization
* thin-pool utilization
* guest availability
* unexpected stopped state

### Application-specific health

Where practical, add health checks that confirm applications are usable rather than only listening on a port.

Examples:

* Prometheus target health
* Grafana data-source availability
* DNS lookup success
* Vaultwarden HTTPS response
* PostgreSQL query success

## Alerting principles

1. Alert on actionable conditions.
2. Avoid duplicate alerts for the same underlying failure.
3. Prefer dependency-aware alerting where possible.
4. Use warning thresholds before critical thresholds when early intervention is useful.
5. Document the expected operator response for every alert that remains enabled.
6. Review noisy or non-actionable alerts and remove them.

## Future documentation

As alerting is implemented, this roadmap should evolve into:

* alert rule examples
* notification-routing documentation
* response runbooks
* validation records showing that alerts were tested intentionally
