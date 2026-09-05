# Change Example: Split DNS and Vaultwarden HTTPS

## Summary

Migrated Vaultwarden from direct service-port access to a domain-based HTTPS access model using local split DNS and a centralized Nginx reverse proxy with automated certificate issuance.

The final architecture removes application-local TLS proxying from the Vaultwarden workload so HTTPS termination and certificate lifecycle management are centralized on the dedicated reverse-proxy service.

## Change type

Significant

## Risk

Medium

## Problem statement

Vaultwarden was reachable through a direct service port and did not use the desired trusted HTTPS access pattern. The goal was to improve certificate trust, reduce direct service exposure, avoid inbound WAN port forwarding, and establish a reusable domain-based access model for internal services.

As the centralized ingress architecture matured, the application-local proxy layer also became redundant and was removed to eliminate duplicate proxying and certificate-management responsibilities.

## Scope

Affected components included:

* Vaultwarden application container
* Docker Compose workload definition
* local DNS filtering and resolution service
* dedicated Nginx reverse-proxy configuration
* certificate automation

Exact domain names, IP addresses, guest identifiers, and provider credentials are intentionally omitted from this public record.

## Implementation

1. Added a local DNS override so the service hostname resolves to the internal reverse-proxy path from the home network.
2. Added the dedicated Nginx reverse proxy in front of Vaultwarden.
3. Configured automated certificate issuance using a DNS challenge rather than inbound HTTP validation.
4. Removed direct application-port exposure from the normal client access path.
5. Validated the centralized HTTPS path and native client behavior.
6. Removed the redundant application-local proxy service from the Vaultwarden Compose workload.
7. Confirmed Vaultwarden now serves only the internal backend path required by Nginx.
8. Confirmed Nginx is the sole client-facing HTTPS termination point for Vaultwarden.

## Validation

The change was considered successful when:

* the service hostname resolved correctly on the local network
* the Vaultwarden web interface loaded through the centralized HTTPS endpoint
* the certificate chain was trusted by clients
* no inbound WAN port forwarding was required
* Vaultwarden remained functional through the Nginx proxy path
* browser and native client sign-in and synchronization continued to work
* the backend remained available only through the intended private service path
* the obsolete application-local proxy was no longer present in the Compose workload

## Impact

Brief service interruptions occurred during configuration, restart, and final proxy cleanup. After validation, Vaultwarden remained available through the trusted centralized HTTPS endpoint.

## Rollback

Rollback depends on the failure domain.

For application or container regressions:

1. restore the previous known-good Vaultwarden image or guest state
2. preserve existing application data volumes
3. validate backend service health before restoring normal client access

For centralized ingress failures:

1. restore the previous Nginx site configuration
2. restore the previous local DNS destination if necessary
3. validate the backend directly from an administrative path

Reintroducing the removed application-local TLS proxy is not part of the normal rollback path unless the centralized reverse-proxy architecture is intentionally abandoned.

## Engineering considerations

This change separated service identity from a specific port number and established a pattern that can be reused for additional internal applications.

Using DNS challenge validation avoided opening inbound ports solely for certificate issuance. Local split DNS allowed the same logical service name to resolve appropriately from the internal network while keeping the service off a directly exposed WAN path.

Centralizing TLS termination also reduced duplicate proxy configuration and certificate lifecycle responsibilities on individual application hosts. Vaultwarden now owns the application service while the dedicated Nginx workload owns client-facing HTTPS presentation.

## Lessons

A reverse proxy is not only an HTTPS convenience layer. In this design it became part of the service architecture, controlling how clients identify and reach the application while reducing direct backend exposure.

Once the centralized ingress path is validated, leaving an application-local TLS proxy in place creates unnecessary complexity. Removing the redundant layer simplifies troubleshooting, maintenance, certificate renewal, and recovery boundaries.
