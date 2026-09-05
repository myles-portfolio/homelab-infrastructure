# Backup and Recovery

## Overview

This section documents the layered backup and recovery model used across the homelab without publishing the live storage topology, exact job structure, retention values, or recovery weaknesses.

The environment uses multiple recovery controls because guest backups, snapshots, application backups, logical database exports, persistent application data, and secondary copies address different failure modes.

## Backup and recovery architecture

The following diagram presents the recovery layers at a conceptual level.

![Backup and recovery architecture](diagrams/backup-recovery-architecture.png)

The diagram is intentionally sanitized and does not expose live domains, addresses, storage paths, guest identifiers, backup schedules, or retention values.

## Recovery layers

### Scheduled guest backups

Proxmox VE provides scheduled VM and container backups for active infrastructure and application workloads.

Backup jobs are organized by workload characteristics rather than relying on a single undifferentiated job. This allows backup behavior to be adjusted when a workload has different consistency, availability, or recovery requirements.

New infrastructure workloads are reviewed for backup coverage before they are treated as fully operational.

A successful backup task confirms that an artifact was created. It does not, by itself, prove that the guest can be recovered successfully. Recoverability is validated separately through controlled restore testing.

### Proxmox snapshots

Temporary Proxmox snapshots are used as short-lived rollback protection for selected maintenance windows.

They are not treated as long-term backups. Snapshots are removed after successful validation so they do not become stale recovery assumptions or unnecessary storage consumers.

### Application-level backups

Some workloads also use application-native recovery mechanisms where those provide better granularity than a full guest restore.

Examples include:

* application-generated configuration backups
* logical database exports
* persistent container data
* secondary copies written outside the protected application guest

Application-level recovery is preferred when the operating system remains healthy and only application state must be restored.

### Database recovery

Database-backed workloads use logical exports where appropriate before higher-risk maintenance or data changes.

Logical recovery provides a narrower option than reverting an entire guest and helps separate application data recovery from operating-system recovery.

### Persistent container data

Containerized applications treat runtime images and persistent state as separate recovery concerns.

Refreshing or recreating an application container should not remove persistent application data unless a documented recovery procedure explicitly requires it.

### Infrastructure configuration state

Shared infrastructure workloads such as reverse proxy and remote-access services are included in workload-appropriate guest backup coverage.

Their recovery scope includes the operating-system configuration and service state required to rebuild the role. Secrets and reusable authentication material are handled separately and are not stored in this public repository.

Restored infrastructure guests should be isolated during validation when duplicate host identity, network identity, certificates, or remote-access state could conflict with production systems.

## Backup storage design

The live environment uses redundant local storage and additional workload-specific recovery controls.

The public repository intentionally does not document the exact placement of primary data and backup datasets, physical disk layout, backup target names, or which failure combinations would defeat the current recovery model.

The design objective is to increase independence between primary workloads and recovery copies over time. Additional independent storage remains part of the resilience roadmap as hardware and budget permit.

## Backup scope limitations

Guest-level backup coverage does not automatically protect every dataset visible from inside a VM or container.

Externally mounted datasets, bind mounts, network storage, or other data outside normal guest-managed volumes may require separate protection.

Backup review therefore asks two distinct questions:

1. Is the guest itself protected by an appropriate scheduled backup?
2. Is important data outside the guest backup boundary protected through another recovery mechanism?

A successful guest backup must not be interpreted as proof that every accessible dataset is protected.

## Recovery control selection

The available controls are complementary rather than interchangeable.

| Recovery control | Best suited for |
|---|---|
| Scheduled guest backup | Full VM or container recovery |
| Proxmox snapshot | Short-term rollback during selected maintenance |
| Application backup | Application-specific configuration or appliance recovery |
| Logical database export | Database-level recovery |
| Persistent container data | Application-state preservation across container recreation |
| Secondary copy | Recovery outside the protected application workload |

## Validation model

A backup is not considered trustworthy simply because a job reports success.

Validation is performed in two stages.

### Backup creation validation

Confirm that:

* the expected scheduled job completes successfully
* the intended workload is included
* the expected artifact exists
* retention behavior is applied as designed
* known exclusions are understood and documented internally

### Restore validation

Controlled restore testing provides stronger evidence of recoverability.

The general pattern is:

1. restore a recent recovery point into an isolated temporary guest
2. prevent duplicate network or application identity from affecting production
3. boot the restored guest
4. validate operating-system startup and expected application state
5. validate the primary service using local or isolated access where practical
6. remove the temporary restored guest after validation

The environment has successfully exercised this pattern with a critical infrastructure workload. Public documentation records the method and outcome without publishing the exact restored guest identity, storage target, or live recovery topology.

## Recovery decision model

Use the narrowest recovery mechanism that safely addresses the failure.

```text
Failure detected
      |
      v
Application data only?
      |
      +--> yes --> application or database recovery
      |
      +--> no
             |
             v
Guest operating system or full guest state affected?
             |
             +--> yes --> snapshot rollback or full guest recovery
             |
             +--> no --> service-level remediation
```

Restoring an entire guest when only a database needs recovery creates unnecessary impact. Conversely, restoring application data alone will not repair a broken operating system.

## Operational principles

1. Choose recovery controls by workload.
2. Keep recovery copies independent from the protected workload where practical.
3. Do not confuse snapshots with backups.
4. Validate recoverability, not only backup creation.
5. Understand backup boundaries and external datasets.
6. Isolate backup failures by workload where practical.
7. Remove temporary rollback and restore-test artifacts after validation.
8. Preserve persistent application state when recreating containers.
9. Use the least disruptive recovery method that solves the problem.
10. Review backup coverage whenever a new infrastructure workload is deployed.

## Current improvement areas

Current work focuses on strengthening recoverability without publishing the exact live protection topology.

Planned improvements include:

* continued backup-creation validation for newer workloads
* periodic controlled restore testing for critical services
* review of externally mounted or otherwise excluded datasets
* additional application-native recovery layers where useful
* defined restore-test frequency for critical workloads
* greater independence of recovery storage as additional hardware becomes practical

See the project wiki roadmap for the broader infrastructure backlog.
