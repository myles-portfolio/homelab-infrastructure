# Checkmk Notification Delivery

## Purpose

This document records the sanitized notification-delivery design used by Checkmk Community in the homelab.

The implementation provides a complete outbound email path for infrastructure and service-state notifications without exposing SMTP credentials, live destination addresses, provider-specific endpoints, or sender-domain details in the public repository.

See [`checkmk-configuration-standards.md`](checkmk-configuration-standards.md) for contact-group and notification-rule standards and [`alerting-roadmap.md`](alerting-roadmap.md) for cross-platform alert ownership.

## Architecture

Checkmk uses a local Linux mail transport for outbound delivery. The monitoring workload relays mail through a managed SMTP service over authenticated TLS.

```text
Checkmk notification engine
          |
          v
   local mail transport
          |
          | authenticated TLS submission
          v
     managed SMTP relay
          |
          v
   recipient mail system
```

A dedicated sender identity is used for infrastructure monitoring, while authentication credentials remain outside the repository in the password manager.

No inbound SMTP service or WAN port forwarding is required. The monitoring server initiates the outbound connection to the relay.

## Public DNS authentication

The sender namespace is verified with provider-generated public DNS records for mail authentication.

The live domain names, selectors, record targets, and tracking identifiers are intentionally omitted.

The implementation uses standard provider-managed mechanisms for sender authentication and message signing.

## SMTP relay identity

A dedicated SMTP credential is used for homelab monitoring rather than a personal mailbox credential.

Design requirements:

* credential purpose is limited to infrastructure monitoring
* password is stored outside source control
* authenticated submission requires TLS
* provider-specific endpoints and fallback ports remain part of the live configuration, not the public documentation

## Postfix configuration model

Postfix is configured as an outbound relay client.

Sanitized example:

```text
relayhost = [mail.example-relay.net]:<submission-port>
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = encrypt
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
```

The SASL credential file contains the relay username and password and remains restricted to the operating-system administrator.

Example permissions and map generation:

```bash
chmod 600 /etc/postfix/sasl_passwd
postmap /etc/postfix/sasl_passwd
```

The credential itself is intentionally excluded from this repository.

## Checkmk notification model

The baseline Checkmk notification rule uses the built-in HTML email method and routes notifications through contacts and contact groups rather than embedding destination addresses directly in rules.

The baseline covers representative problem and recovery state transitions while operational alert volume is evaluated.

Notification throttling, delays, and periodic reminders are introduced only when observed behavior demonstrates a need.

Notification-rule changes should be validated first through Checkmk's prediction workflow and then through an actual notification test.

## Contacts and routing

Notification routing is based on reusable contacts and contact groups.

```text
Monitored object
      |
      v
Contact-group assignment
      |
      v
Administrative contact group
      |
      v
Email notification method
```

A fallback destination is maintained for unmatched events. Live contact names and addresses are not published.

## Validation

Notification delivery is validated in layers so transport failures can be distinguished from monitoring-rule failures.

Validation includes:

* DNS resolution for the relay service
* outbound connectivity to the managed relay
* successful TLS negotiation
* local mail-transport configuration and service state
* relay authentication and provider acceptance
* Checkmk rule matching and contact selection
* handoff from Checkmk to the local mail transport
* final mailbox receipt

This validates the complete conceptual path:

```text
Checkmk
  |
  v
Local mail transport
  |
  v
Managed SMTP relay
  |
  v
Recipient mailbox
```

## Maintenance considerations

The local mail transport is part of the monitoring notification dependency chain.

After relevant operating-system or mail-transport maintenance:

* confirm the mail service is healthy
* validate configuration before restart when settings changed
* inspect the queue if delivery is delayed
* send a real Checkmk notification test after material notification-path changes

SMTP credentials should be rotated through the managed relay and password manager without committing replacement secrets to the repository.

## Current operating approach

Notification tuning is evidence-based. Candidate improvements include:

* delaying notifications for transient states
* reducing non-actionable warning mail
* periodic reminders for persistent critical conditions
* routing by host or service classification
* acknowledgement behavior
* scheduled-downtime suppression
* recovery validation with controlled failures

The preferred result remains one authoritative notification path per operational condition.

## Security requirements

Public documentation must not include:

* SMTP passwords or usernames
* password-manager entries
* live destination email addresses
* live sender-domain records or selectors
* provider-specific relay endpoints when they add no technical value
* internal IP addresses
* authentication tokens or automation secrets

The repository documents the architecture and operating model, not production secrets or live service endpoints.
