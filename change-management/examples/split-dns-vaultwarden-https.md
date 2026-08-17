# Change Example: Split DNS and Vaultwarden HTTPS

## Summary

Migrated Vaultwarden from direct service-port access to a domain-based HTTPS access model using local split DNS and a reverse proxy with automated certificate issuance.

## Change type

Significant

## Risk

Medium

## Problem statement

Vaultwarden was reachable through a direct service port and did not use the desired trusted HTTPS access pattern. The goal was to improve certificate trust, reduce direct service exposure, avoid inbound WAN port forwarding, and establish a reusable domain-based access model for internal services.

## Scope

Affected components included:

* Vaultwarden application container
* local DNS filtering and resolution service
* reverse-proxy configuration
* router DNS behavior
* certificate automation

Exact domain names, IP addresses, and provider credentials are intentionally omitted from this public record.

## Implementation

1. Added a local DNS override so the service hostname resolves to the internal reverse-proxy path from the home network.
2. Added a reverse proxy in front of Vaultwarden.
3. Configured automated certificate issuance using a DNS challenge rather than inbound HTTP validation.
4. Removed direct application-port exposure from the normal access path.
5. Restarted affected services and validated name resolution and HTTPS access.

## Validation

The change was considered successful when:

* the service hostname resolved correctly on the local network
* the Vaultwarden web interface loaded through HTTPS
* the certificate chain was trusted by clients
* no inbound WAN port forwarding was required
* Vaultwarden remained functional through the new proxy path

## Impact

A brief service interruption occurred during configuration and restart. After validation, Vaultwarden was available through the new trusted HTTPS endpoint.

## Rollback

Rollback consists of:

1. restoring the previous local DNS behavior
2. removing the reverse proxy from the Vaultwarden access path
3. restoring the previous direct application-port mapping if required
4. removing the local DNS override
5. revoking the certificate-provider API credential if the integration is decommissioned

## Engineering considerations

This change separated service identity from a specific port number and established a pattern that can be reused for additional internal applications.

Using DNS challenge validation avoided opening inbound ports solely for certificate issuance. Local split DNS allowed the same logical service name to resolve appropriately from the internal network while keeping the service off a directly exposed WAN path.

## Lessons

A reverse proxy is not only an HTTPS convenience layer. In this design it became part of the service architecture, controlling how clients identify and reach the application while reducing the need for direct backend exposure.
