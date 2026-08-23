# Checkmk Notification Delivery

## Purpose

This document records the sanitized notification-delivery design used by Checkmk Community in the homelab.

The implementation provides a complete outbound email path for Checkmk infrastructure and service-state notifications without exposing SMTP credentials or live destination addresses in the public repository.

See [`checkmk-configuration-standards.md`](checkmk-configuration-standards.md) for contact-group and notification-rule standards and [`alerting-roadmap.md`](alerting-roadmap.md) for cross-platform alert ownership.

## Architecture

Checkmk Community uses the local Linux mail transport for email delivery. The monitoring VM runs Postfix as a relay-only mail transfer agent, and Postfix submits outbound mail to a managed SMTP relay over authenticated TLS.

```text
Checkmk notification engine
          |
          v
      local Postfix
          |
          | authenticated SMTP submission
          | STARTTLS on TCP 587
          v
     managed SMTP relay
          |
          v
   recipient mail system
```

The current managed relay is SMTP2GO. A dedicated sender subdomain is verified through public DNS, while SMTP authentication credentials are stored outside the repository in the password manager.

No inbound SMTP service or WAN port forwarding is required. The monitoring server initiates the outbound connection to the relay.

## Public DNS authentication

The sender subdomain is verified with provider-generated DNS records for mail authentication and tracking. The live hostnames and selectors are intentionally omitted from this repository.

The implementation uses provider-managed records for:

* return-path and SPF alignment
* DKIM signing
* tracking-domain validation

These records are published through the authoritative public DNS provider. They are not duplicated in internal DNS because they represent public mail-authentication namespaces rather than private service addresses.

## SMTP relay account

A dedicated SMTP credential is used for homelab monitoring rather than a personal mailbox credential.

Design requirements:

* credential purpose is limited to infrastructure monitoring
* password is stored in the password manager
* credential is not committed to source control
* SMTP submission requires TLS
* port 587 is the preferred submission port
* alternate provider ports are retained only as fallback options

## Postfix configuration

Postfix is installed on the Checkmk VM and configured as an outbound relay client.

Sanitized example:

```text
relayhost = [mail.smtp2go.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = encrypt
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
```

The SASL credential file contains the relay username and password and must remain restricted to root.

Example permissions and map generation:

```bash
chmod 600 /etc/postfix/sasl_passwd
postmap /etc/postfix/sasl_passwd
```

The credential itself is intentionally excluded from this repository.

## Checkmk notification model

The baseline Checkmk notification rule uses the built-in HTML email method and notifies all contacts assigned to the affected object.

Initial host-state events:

* any state to `DOWN`
* any state to `UP`

Initial service-state events:

* any state to `WARN`
* any state to `CRIT`
* any state to `UNKNOWN`
* any state to `OK`

This provides both problem and recovery notification coverage while operational alert volume is evaluated.

The rule currently has no time-period restriction, notification-count limit, or periodic-notification throttling. These controls should be introduced only if observed alert behavior demonstrates a need.

Notification-rule changes should be validated first through Checkmk's prediction workflow and then through an actual notification test.

## Contacts and routing

Notification routing is based on Checkmk contacts and contact groups rather than hard-coded recipient addresses in notification rules.

The operating model is:

```text
Monitored host
     |
     v
Contact-group assignment rule
     |
     v
Administrative contact group
     |
     v
Checkmk contact with email address
     |
     v
HTML email notification rule
```

A host contact-group assignment rule applies the administrative notification group to monitored hosts through reusable configuration. This allows additional administrators or future routing changes without rewriting the notification rule itself.

A fallback email destination is also configured in Checkmk so unmatched notifications still have a delivery target. Live addresses are not published here.

The contact-group assignment should be verified on a representative host whenever routing rules change.

## Validation

Notification delivery was validated in layers.

### Network and TLS validation

The Checkmk VM successfully resolved and connected to the SMTP relay on TCP 587. STARTTLS negotiation completed successfully and returned a valid certificate chain.

### Postfix relay validation

A direct test message was submitted through the local Postfix instance. Postfix authenticated to the managed SMTP relay, the relay recorded successful delivery activity, and the test message reached the destination mailbox.

### Checkmk notification validation

Checkmk's notification test workflow confirmed:

* the global HTML email rule matched the simulated state transition
* the expected contact was selected through contact-group assignment
* the HTML email notification plug-in was triggered
* Checkmk handed the message to the local mail transport
* the managed SMTP relay recorded the message
* the message reached the destination mailbox

This validates the complete path:

```text
Checkmk
  |
  v
Postfix
  |
  v
SMTP relay
  |
  v
Recipient mailbox
```

## Operational validation sequence

After any material notification or mail-transport change, validate in this order:

1. confirm DNS resolution for the relay endpoint
2. confirm TCP 587 connectivity
3. confirm STARTTLS negotiation
4. confirm Postfix configuration and service state
5. confirm direct Postfix relay delivery when transport changes are involved
6. confirm Checkmk contact-group assignment
7. confirm notification-rule prediction selects the intended contact and method
8. send an actual Checkmk notification
9. confirm relay activity and mailbox receipt

This sequence isolates transport failures from Checkmk routing failures.

## Maintenance considerations

Postfix is now part of the Checkmk notification dependency chain.

After Checkmk VM operating-system maintenance or Postfix package changes:

* confirm the Postfix service is running
* run a configuration check before restarting Postfix when mail settings have changed
* verify the effective relay and TLS settings
* inspect the mail queue if delivery is delayed
* send a real Checkmk notification test when the mail stack or notification configuration has materially changed

The SMTP credential should be rotated through the managed relay and password manager without committing the replacement secret to the repository.

## Current operating approach

The notification rule will initially remain broad enough to observe real-world alert volume. Warning notifications are intentionally retained during this evaluation period.

Future tuning should be evidence-based and may include:

* delaying notifications for transient WARN or CRIT states
* reducing non-actionable warning mail
* adding periodic reminders for persistent critical conditions
* refining routing by host or service classification
* validating acknowledgement and scheduled-downtime suppression
* validating recovery behavior with controlled failures

Alert quality should be reviewed before adding more notification rules.

## Security requirements

Public documentation must not include:

* SMTP passwords
* password-manager entries
* live destination email addresses
* live sender-domain DNS selectors where they unnecessarily expose implementation details
* internal IP addresses
* authentication tokens or automation secrets

The repository documents the architecture and operating model, not production secrets.
