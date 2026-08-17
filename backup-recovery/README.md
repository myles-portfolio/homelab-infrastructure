# Backup and Recovery

## Overview

This section documents the layered backup and recovery model used across the homelab.

The environment does not rely on a single recovery mechanism. Different workloads use different controls because snapshots, application backups, logical database dumps, persistent volumes, and network backup copies protect against different failure modes.

## Backup and recovery architecture

The following diagram summarizes the major recovery layers and their intended use.

![Backup and recovery architecture](diagrams/backup-recovery-architecture.png)

The diagram is intentionally sanitized and does not expose the live environment's real domains, IP addresses, credentials, or storage paths.

## Recovery layers

### Proxmox snapshots

Temporary Proxmox snapshots provide fast VM-level rollback during selected maintenance windows.

They are useful when a change may affect the guest operating system or the VM as a whole, but they are not treated as long-term backups.

Typical use:

* take snapshot before higher-risk maintenance
* apply operating-system or application changes
* validate the workload
* remove the snapshot after successful validation

Snapshots are intentionally short-lived to avoid unnecessary storage consumption and dependence on stale rollback points.

### Home Assistant backups

Home Assistant uses its application-level backup system for configuration and appliance recovery.

The current design stores backup copies in two locations:

* local Home Assistant storage
* external Samba network storage hosted outside the Home Assistant VM

This provides a recovery path even if the Home Assistant VM itself is damaged or unavailable.

The backup workflow includes automatic backups, backup creation before updates, retention controls, and manual validation of both storage targets.

### PostgreSQL logical backups

The Development VM uses PostgreSQL logical dumps before higher-risk maintenance.

A logical dump protects the application database independently of the VM and provides a more granular recovery option than reverting the entire guest.

This is useful when the operating system remains healthy but database data or schema changes need to be restored.

### Docker persistent data

Stateful containerized applications depend on persistent volumes or bind-mounted application data.

Container recreation is therefore treated separately from data removal. Refreshing an image and recreating a Compose service should not delete persistent application data unless a recovery procedure explicitly requires it.

For workloads such as Vaultwarden, the running container is replaceable while the persistent application state is the critical recovery asset.

### Network storage

The file-services container provides external storage for selected application backups.

A dedicated Samba share and service account are used for the Home Assistant backup path so machine-to-machine access is scoped to the required resource rather than a general-purpose user account.

## Why multiple recovery methods exist

The recovery methods are complementary rather than interchangeable.

| Recovery control | Best suited for | Example failure |
|---|---|---|
| Proxmox snapshot | Fast guest-level rollback | VM maintenance introduces a boot or configuration failure |
| Home Assistant backup | Application or appliance restore | Home Assistant configuration becomes unusable |
| PostgreSQL logical dump | Database-level recovery | Database data or schema needs restoration |
| Docker persistent data | Application state preservation | Container image is recreated or replaced |
| External Samba backup | Recovery outside the source VM | Home Assistant VM or local backup storage fails |

## Validation model

A backup is not considered trustworthy simply because a job reports success.

Validation should confirm the expected artifact exists at the intended recovery layer.

Examples include:

* Home Assistant backup appears in both local and external locations
* PostgreSQL dump file exists and the application database remains queryable after maintenance
* Proxmox snapshot is visible before the change and intentionally removed afterward
* Docker services return with persistent application state intact after recreation
* Samba backup destination accepts an authenticated write

## Recovery decision model

Use the narrowest recovery mechanism that safely addresses the failure.

```text
Failure detected
      |
      v
Is the issue application data only?
      |
      +--> yes --> application backup or database restore
      |
      +--> no
             |
             v
Is the guest operating system or VM state affected?
             |
             +--> yes --> snapshot rollback or full guest recovery
             |
             +--> no --> service-level remediation
```

Restoring an entire VM when only a database needs recovery creates unnecessary impact. Conversely, restoring application data alone will not fix a broken guest operating system.

## Operational principles

1. **Choose recovery controls by workload.** Different applications require different recovery mechanisms.
2. **Keep at least one recovery copy outside the protected workload when practical.**
3. **Do not confuse snapshots with backups.** Snapshots are primarily short-term rollback tools.
4. **Validate artifacts before relying on them.** A reported success state is not sufficient by itself.
5. **Remove temporary rollback artifacts after validation.** Long-lived snapshots create unnecessary storage and operational risk.
6. **Preserve persistent application data when recreating containers.**
7. **Use the least disruptive recovery method that solves the problem.**

## Current improvement areas

Planned work includes automated backup verification and restore testing so recoverability is tested intentionally rather than inferred from backup creation alone.

See [`../ROADMAP.md`](../ROADMAP.md) for the broader infrastructure backlog.
