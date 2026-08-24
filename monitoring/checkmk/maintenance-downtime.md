# Checkmk Maintenance Downtime Standard

## Purpose

Planned maintenance must not generate actionable incident notifications. Checkmk scheduled downtime is the standard suppression mechanism for all monitored hosts that are intentionally affected by maintenance.

A scheduled downtime suppresses problem notifications for the selected host. Services belonging to that host inherit the host downtime automatically.

## Policy

Before beginning maintenance on any monitored host:

1. identify every Checkmk host object expected to be affected
2. schedule downtime before the first disruptive action
3. include enough time for validation and rollback, not only the estimated update duration
4. enter a maintenance comment that identifies the work being performed
5. verify the downtime is active before shutting down, rebooting, restarting, or otherwise disrupting the host
6. complete maintenance and validation while the downtime remains active
7. remove the downtime early when work is complete, or allow it to expire at the planned end time
8. confirm the host and retained services return to healthy monitoring state

Do not disable notification rules globally for routine maintenance. Scheduled downtime preserves normal alerting for infrastructure outside the maintenance scope.

## Checkmk UI workflow

From a host or host list in Checkmk:

1. open the relevant host view
2. use `Commands`
3. select `Schedule downtimes`
4. choose a fixed maintenance window that covers the expected work plus validation and rollback buffer
5. enter a clear comment such as `Planned maintenance`
6. execute and confirm the command

For multiple affected hosts, use a host view with checkboxes and apply the command to the selected objects as a group.

Active downtimes can be reviewed under `Monitor > Overview > Scheduled downtimes`.

## Scope rules

### Single guest maintenance

Schedule downtime for the guest host object being maintained.

If the work intentionally disrupts another monitored dependency, include that dependency as well.

### Hypervisor maintenance

A hypervisor reboot disrupts the hypervisor and every running guest on that host. Before disruptive hypervisor maintenance, schedule downtime for:

* the Proxmox hypervisor host object
* every monitored guest expected to stop, reboot, or become unreachable
* dependent service objects represented by separate Checkmk hosts when they will also become unavailable

For a full single-host homelab maintenance window, this normally means placing the entire monitored Proxmox host and guest set into scheduled downtime before the hypervisor reboot.

### Monitoring-server maintenance

Schedule downtime for the Checkmk server itself and for any other monitored hosts being intentionally maintained during the same window before taking the Checkmk server offline.

Because the Checkmk server cannot deliver notifications while it is down, maintenance of the monitoring server should normally occur near the end of a broader maintenance cycle, after other workloads have been validated.

## Duration guidance

Prefer fixed downtimes for planned maintenance.

Choose a window that includes:

* package download and installation time
* service restart or reboot time
* guest startup time
* application-level validation
* troubleshooting buffer
* rollback time if validation fails

If work completes early, the downtime may be removed after validation. Do not remove it simply because the package manager has finished.

## Completion criteria

Monitoring suppression is complete only when:

* all intentionally affected hosts were covered before disruption
* no maintenance-generated problem notification escaped the downtime window
* each host returned to its expected monitored state
* unexpected problems remaining after maintenance are no longer hidden by an unnecessary downtime

## Relationship to acknowledgements

Acknowledgements and scheduled downtimes serve different purposes.

Use scheduled downtime for planned work. Use acknowledgements for known unplanned problems that are being investigated or tolerated temporarily.

## Security and repository hygiene

Public maintenance documentation must not include live administrative credentials, tokens, private addresses, or other secrets. Hostnames and addresses should be represented generically where publishing them would expose unnecessary environment detail.
