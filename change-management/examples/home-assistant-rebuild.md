# Change Example: Home Assistant VM Rebuild

## Summary

Rebuilt the Home Assistant virtual machine using a fresh Home Assistant OS KVM image after the previous VM became unreachable and could no longer provide a reliable web or network management path.

## Change type

Emergency

## Risk

High

## Problem statement

The existing Home Assistant VM was not reachable through the web interface, was not producing useful network activity, and appeared to have a broken boot or display state. Troubleshooting required a recovery path that restored service while preserving the previous VM for fallback during validation.

## Scope

Affected components included:

* Home Assistant VM
* Proxmox virtual hardware configuration
* Home Assistant OS boot disk
* console access
* network configuration
* HACS
* Alarm.com custom integration
* security dashboard configuration

Exact IP addresses and account identifiers are intentionally excluded.

## Implementation

1. Created a new Home Assistant VM in Proxmox.
2. Imported the Home Assistant OS KVM disk image.
3. Attached the imported disk as the VM boot disk.
4. Adjusted console output to support direct troubleshooting.
5. Verified that Home Assistant OS booted successfully.
6. Configured the Home Assistant network interface with the intended static address after DHCP behavior proved unreliable.
7. Installed HACS.
8. Installed the Alarm.com custom integration.
9. Investigated integration setup failures and identified a compatibility issue with the original integration version.
10. Replaced the failing integration with a compatible fork.
11. Created and populated a dedicated security dashboard.

## Validation

The rebuild was considered successful when:

* Home Assistant OS booted reliably
* the web interface became reachable
* the intended network address was active
* HACS loaded correctly
* Alarm.com entities became available through the compatible integration fork
* key security sensors appeared on the dedicated dashboard

## Impact

The rebuild restored access to Home Assistant after the previous VM became unusable. The old VM was retained during validation to preserve a fallback path.

## Rollback

Rollback consisted of stopping the new VM and restoring or re-enabling the previous Home Assistant VM if validation failed.

## Engineering considerations

The rebuild separated recovery from diagnosis. Rather than continuing to make changes to an unstable VM, a clean replacement was created while the old VM remained available as a fallback artifact.

Serial console access proved important because normal graphical output was not sufficient for troubleshooting. Network behavior was also treated as an independent failure domain rather than assuming the application itself was responsible for all symptoms.

## Lessons

Appliance-style workloads benefit from simple, replaceable deployment patterns. Preserving the previous VM during rebuild validation reduced recovery risk and made it possible to restore service without immediately destroying the prior state.
