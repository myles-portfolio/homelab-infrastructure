# Proxmox System Mail Relay

## Purpose

This runbook documents a sanitized outbound-mail design for a Proxmox VE hypervisor using Postfix as an authenticated SMTP relay client.

The goal is to avoid direct Internet SMTP delivery from the homelab while preserving useful Proxmox and operating-system mail.

Environment-specific sender domains, recipient addresses, SMTP usernames, passwords, hostnames, and IP addresses are intentionally omitted.

## Architecture

```text
Proxmox VE / local system mail
          |
          v
       Postfix
          |
          | authenticated SMTP submission
          | STARTTLS on TCP 587
          v
    managed SMTP relay
          |
          v
 administrative mailbox
```

The hypervisor uses a dedicated SMTP credential rather than sharing the credential used by another workload.

## Why a relay is required

A default Postfix installation may attempt direct delivery to the recipient domain's MX servers over TCP 25 when no `relayhost` is configured.

In a residential or otherwise restricted network, direct delivery can fail because:

* outbound TCP 25 is filtered
* IPv6 MX addresses are returned but the host has no working IPv6 route
* the public source address lacks the reputation and DNS configuration expected for direct mail delivery
* the sender domain is not authorized by the receiving or relay service

Symptoms include a growing deferred queue and log entries showing repeated connection timeouts to external MX servers.

Do not solve this by loosening the mail-queue monitoring threshold. Fix the delivery path instead.

## SMTP relay identity

Create a dedicated SMTP user in the managed relay service for the Proxmox host.

Requirements:

* use a unique SMTP username and password for the hypervisor
* store the credential outside source control
* use a verified sender domain or verified sender address
* use authenticated submission over TCP 587 with STARTTLS
* do not use the web-console account password as the SMTP password

## Postfix credential map

Create a root-only SASL credential file:

```text
/etc/postfix/sasl_passwd
```

Sanitized format:

```text
[mail.example-relay.net]:587 SMTP_USERNAME:SMTP_PASSWORD
```

Protect and compile the map:

```bash
chmod 600 /etc/postfix/sasl_passwd
postmap /etc/postfix/sasl_passwd
```

The generated database file and source credential file must not be committed to the repository.

## Postfix relay configuration

Sanitized example:

```bash
postconf -e 'relayhost = [mail.example-relay.net]:587'
postconf -e 'smtp_sasl_auth_enable = yes'
postconf -e 'smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd'
postconf -e 'smtp_sasl_security_options = noanonymous'
postconf -e 'smtp_tls_security_level = encrypt'
postconf -e 'smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt'
```

Validate before restart:

```bash
postfix check
```

Then restart Postfix:

```bash
systemctl restart postfix
systemctl status postfix --no-pager
```

## Sender rewriting

Local system mail may originate from an address such as:

```text
root@localhost-name.local
```

A managed relay will normally reject that sender because the local pseudo-domain is not verified.

Use a Postfix generic map to rewrite local mail to an address under the verified sender domain.

Example `/etc/postfix/generic`:

```text
root@localhost-name.local hypervisor@alerts.example.net
root                      hypervisor@alerts.example.net
```

Compile and enable the map:

```bash
postmap /etc/postfix/generic
postconf -e 'smtp_generic_maps = hash:/etc/postfix/generic'
systemctl restart postfix
```

Use a functional sender identity rather than a personal mailbox address where practical.

## Proxmox administrative destination

The Proxmox administrative user should contain the current operational notification destination.

Update it through the Proxmox user-management interface rather than editing cluster configuration files directly.

The destination address and the SMTP sender address serve different purposes:

* sender identifies the infrastructure system that generated the message
* recipient identifies the administrative mailbox that receives the message

Do not assume changing the recipient fixes mail transport. A host without a relay will still attempt direct SMTP delivery.

## Validation

Send a test message after configuration:

```bash
echo "Proxmox SMTP relay test" | mail -s "PVE SMTP relay test" <administrative-address>
```

Check the queue:

```bash
postqueue -p
```

Expected healthy result:

```text
Mail queue is empty
```

Review recent delivery logs:

```bash
journalctl -u postfix --since "5 minutes ago" --no-pager | grep -E 'relay=|status='
```

Successful delivery should show the managed relay and:

```text
status=sent
```

Validate actual mailbox receipt as the final end-to-end check.

## Deferred queue troubleshooting

Inspect the queue:

```bash
postqueue -p
```

Inspect recent Postfix logs:

```bash
journalctl -u postfix --since "6 hours ago" --no-pager | grep -Ei 'deferred|warning|error|reject|timeout|connect|status='
```

Common causes include:

* direct TCP 25 delivery because `relayhost` is blank
* SMTP authentication failure
* sender domain not verified by the relay provider
* TLS negotiation failure
* DNS resolution failure
* network path failure to the relay endpoint

If stale deferred messages must be discarded after the root cause is understood:

```bash
postsuper -d ALL deferred
```

Deleting queued mail is not a substitute for fixing the delivery failure.

## Monitoring

The hypervisor's Postfix queue should remain monitored.

A growing deferred queue is operationally meaningful because it can indicate that system notifications are not leaving the host. Do not raise queue thresholds simply to hide a persistent delivery problem.

After mail changes, verify the Checkmk Postfix queue and status services return to healthy state.

## Security requirements

Public documentation must not contain:

* SMTP usernames or passwords
* live recipient addresses
* private hostnames or IP addresses
* password-manager entries
* provider API keys
* generated SASL credential maps

Use separate SMTP credentials for distinct infrastructure senders when practical so a credential can be rotated or revoked without affecting unrelated systems.

## Completion criteria

System mail relay configuration is complete only when:

* Postfix passes `postfix check`
* the Postfix service is healthy
* the relay endpoint is configured on authenticated TCP 587 submission
* local sender addresses are rewritten to a verified sender domain
* the administrative recipient is current
* a test message records `status=sent`
* the message is received at the administrative mailbox
* the deferred queue is empty
* Checkmk reports the Postfix queue and service state as healthy
