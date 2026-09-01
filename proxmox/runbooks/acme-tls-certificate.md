# Proxmox VE ACME TLS Certificate

## Purpose

This runbook documents a sanitized method for replacing the default Proxmox VE web certificate with a publicly trusted certificate issued through ACME.

The implementation uses Proxmox VE's built-in ACME client with a DNS challenge. DNS validation is performed through the registrar or DNS provider API, allowing certificate issuance without exposing an HTTP challenge endpoint.

## Design

* Proxmox VE terminates TLS directly on the native management interface.
* A publicly resolvable management hostname is used for certificate validation.
* Let's Encrypt provides the certificate through ACME.
* A DNS API challenge plugin creates the required temporary TXT record.
* API credentials are stored outside this repository and are never committed to source control.
* Automatic renewal is handled by Proxmox VE.

## Configuration outline

1. Create an ACME account under **Datacenter > ACME**.
2. Add a DNS challenge plugin for the configured DNS provider.
3. Store the provider API credentials only in the Proxmox ACME plugin configuration and the password manager.
4. Restrict the API credential to the required registered domain when supported by the provider.
5. On the Proxmox node, open **System > Certificates**.
6. Add the management FQDN as an ACME domain.
7. Select the DNS challenge plugin.
8. Order the certificate.
9. Allow Proxmox to install the certificate and reload the web proxy.

## Validation

After issuance:

* Confirm the installed certificate subject matches the management FQDN.
* Confirm the issuer is the expected public certificate authority.
* Confirm the browser reports a trusted HTTPS connection when accessing the Proxmox management interface.
* Confirm the Proxmox GUI remains reachable after the web proxy reload.
* Confirm the certificate appears under the node's certificate inventory.

## Renewal

Proxmox VE manages ACME renewal automatically. The DNS challenge plugin must remain functional, and its API credential must remain valid for future renewals.

Periodic operational checks should confirm:

* the ACME account remains registered;
* the DNS challenge plugin remains configured;
* the DNS API credential has not been revoked or expired;
* the management hostname still resolves as intended;
* certificate expiration monitoring is in place.

## Security notes

* Never commit DNS provider API keys or ACME credentials to this repository.
* Use a dedicated API key when practical.
* Restrict the key to the required domain when supported.
* Restrict by source IP only when the originating public IP is stable enough to avoid breaking automated renewal.
* Avoid publishing internal hostnames, addresses, node identifiers, or provider secrets in public documentation.

## Rollback

If the ACME certificate must be removed, delete the ACME domain configuration and custom certificate from the node, then restore or regenerate the Proxmox-managed local certificate. Browser trust warnings will return when using a hostname not covered by a publicly trusted certificate.
