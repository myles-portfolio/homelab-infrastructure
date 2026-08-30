# DNS and Split DNS

## Overview

Local DNS is provided by Pi-hole. In addition to filtering, selected internal service names use local DNS overrides so clients on the home network resolve those names to internal service paths.

This allows a consistent service name to be used without requiring clients to connect through a public WAN path.

## Why split DNS

Without split DNS, an internal client resolving a public service name may receive a public-facing address and depend on router hairpin behavior or unnecessary WAN routing.

With split DNS, the same logical service name can resolve to an internal address for local clients while public DNS remains independent.

```text
Internal client
    |
    v
Pi-hole
    |
    +--> known internal service --> private destination
    |
    +--> all other queries ------> upstream resolver
```

## Operational role

Pi-hole therefore serves two distinct functions:

1. DNS filtering
2. local service-name resolution

This makes DNS a dependency for several internally hosted services even when those applications themselves are healthy.

## Trusted HTTPS administration

The DNS administration interface is accessed through a locally resolved service name over trusted HTTPS rather than plain HTTP.

Certificate issuance uses an ACME client with DNS validation. The DNS challenge is completed through a dedicated DNS-provider API credential, which allows certificate validation without exposing the administrative interface to the public internet.

The implementation follows this pattern:

```text
Administrator browser
        |
        v
Local DNS service name
        |
        v
HTTPS management interface
        |
        +--> ACME certificate
        |
        +--> automated DNS challenge
        |
        +--> automated renewal and service reload
```

HTTP access is configured to redirect to HTTPS so routine administration consistently uses the encrypted interface.

The certificate deployment workflow also performs a service reload after renewal so the newly issued certificate becomes active without manual intervention.

API credentials, private keys, live service names, and provider-specific account details are intentionally omitted from this repository.

## Validation

After DNS maintenance or configuration changes, validate both external and local resolution.

Example external test:

```bash
nslookup example.com <dns-server>
```

Example internal-service test:

```bash
nslookup internal-service.example <dns-server>
```

The exact internal names and addresses are intentionally omitted from this public repository.

A successful DNS test should confirm:

* the resolver responds
* external names resolve normally
* expected local overrides return the intended private destination
* dependent applications remain reachable through their normal service names

For HTTPS administration changes, additionally confirm:

* the management name resolves to the expected private destination
* the browser trusts the served certificate
* HTTP redirects to HTTPS
* the certificate subject matches the management name
* automated renewal completes successfully
* the DNS service remains healthy after the certificate reload

## Failure isolation

If a named application becomes unreachable, test the layers independently:

```text
Does the name resolve?
    |
    +--> no  --> investigate DNS
    |
    v
Does the resolved endpoint accept a connection?
    |
    +--> no  --> investigate network / proxy
    |
    v
Does the application respond normally?
    |
    +--> no  --> investigate backend service
```

This avoids treating every application access failure as an application problem.

## Recovery considerations

Because local DNS is a dependency for management-friendly service names, administrators should retain a documented direct management path that does not depend on local DNS.

The public documentation intentionally does not publish that live recovery address.

Certificate automation should also retain a recoverable local certificate path so a failed renewal does not remove the currently working certificate before a replacement is validated.

## Security considerations

Local DNS overrides are configuration data rather than authentication controls. They should not be relied on to protect a service.

Backend access should still be constrained appropriately through application authentication, host configuration, reverse-proxy policy, and network controls.

DNS-provider API credentials used for certificate validation should be dedicated to the automation workflow, stored outside source control, and granted only the access required for DNS validation.
