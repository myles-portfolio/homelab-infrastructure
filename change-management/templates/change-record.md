# Change Record Template

## Summary

**Date:** YYYY-MM-DD  
**Status:** Planned | In Progress | Complete | Rolled Back | Failed  
**Change type:** Routine | Significant | Emergency  
**Risk:** Trivial | Low | Medium | High | Critical

## Change made

Describe the implementation in concise operational terms.

## Reason

Explain the technical or operational problem being solved.

## Systems affected

List affected guests, services, devices, integrations, or dependencies.

## Expected impact

Describe expected downtime, behavior changes, or user-facing effects.

## Implementation

1. Step one
2. Step two
3. Step three

## Validation

Document evidence that the change succeeded.

Examples:

* service status healthy
* endpoint responds
* application login succeeds
* expected automation executes
* database query succeeds
* client can reach the service

## Rollback notes

Describe the known-good rollback path before making the change.

Examples:

* revert Proxmox snapshot
* restore application backup
* redeploy known-good container image
* restore previous configuration
* re-enable previous scheduler

## Outcome

Record deviations, lessons learned, or follow-up work discovered during implementation.
