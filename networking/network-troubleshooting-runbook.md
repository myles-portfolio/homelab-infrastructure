# Network Troubleshooting Runbook

## Scope

This runbook provides a layered troubleshooting workflow for internally hosted services that depend on DNS, network reachability, reverse proxies, TLS, and backend applications.

The goal is to identify the failing layer before restarting services or changing configuration.

## Troubleshooting order

Work from the client inward:

```text
Client
  |
  v
DNS
  |
  v
Network reachability
  |
  v
TLS / reverse proxy
  |
  v
Backend application
  |
  v
Dependent service
```

## 1. Confirm the client symptom

Record the exact failure mode:

* name does not resolve
* connection times out
* connection is refused
* certificate warning appears
* HTTP error is returned
* login fails
* page loads but application data does not

Different symptoms point to different layers.

## 2. Test DNS resolution

From the client:

```bash
nslookup service.example.internal
```

or:

```bash
dig service.example.internal
```

Expected result:

* the hostname resolves to the intended internal destination

If DNS fails:

1. confirm the client is using the expected DNS server
2. confirm Pi-hole is running
3. confirm upstream DNS works
4. confirm the local override exists when split DNS is required

Do not restart the backend application when the hostname itself does not resolve.

## 3. Test basic network reachability

Where ICMP is appropriate:

```bash
ping <service-host>
```

For service ports, use a TCP test instead of relying only on ping.

Examples:

```bash
nc -vz <host> 443
nc -vz <host> 22
```

On Windows PowerShell:

```powershell
Test-NetConnection <host> -Port 443
```

A successful DNS lookup with failed TCP connectivity indicates the problem is beyond name resolution.

## 4. Test the HTTP or HTTPS path

Use `curl` to distinguish transport, TLS, proxy, and application behavior.

```bash
curl -I https://service.example.internal
```

For more TLS detail:

```bash
curl -v https://service.example.internal
```

Validate:

* TCP connection succeeds
* TLS negotiation succeeds
* certificate subject and trust are expected
* an HTTP response is returned

A valid HTTP error is different from a connection failure. An HTTP `502` or `504`, for example, often indicates that the proxy is reachable but the backend is not.

## 5. Validate the reverse proxy

If the service uses a reverse proxy:

1. confirm the proxy process or container is running
2. inspect recent logs
3. confirm the configured backend name and port are correct
4. verify the proxy can reach the backend directly
5. confirm certificate automation has not failed

Avoid changing DNS while the proxy itself is the failing layer.

## 6. Validate the backend application

Test the backend from its local host or trusted internal path.

Examples:

```bash
curl -I http://localhost:<port>
```

or:

```bash
sudo docker ps
sudo docker compose ps
```

For systemd workloads:

```bash
sudo systemctl status <service> --no-pager
sudo systemctl --failed
```

If the backend works locally but fails through the proxy, focus on proxy routing, firewall behavior, TLS, or network reachability between layers.

## 7. Validate application dependencies

If the application is running but the user workflow fails, check dependencies such as:

* databases
* authentication providers
* external APIs
* local DNS
* storage mounts
* exporter targets
* Home Assistant integrations

Example for PostgreSQL:

```bash
sudo -u postgres psql -d app_database -c 'SELECT now();'
```

Example for Prometheus targets:

```bash
curl -s http://localhost:9090/api/v1/targets
```

## 8. Validate the user workflow

A service is not considered restored only because a process is running.

Examples:

* sign in to Vaultwarden and confirm sync
* open Grafana and confirm dashboards have data
* query Prometheus and verify targets are healthy
* resolve a local DNS record through Pi-hole
* access a Samba share and perform an authenticated write test
* trigger or inspect a Home Assistant automation

## Common symptom mapping

| Symptom | Likely layers to check first |
|---|---|
| Hostname not found | DNS |
| Name resolves but connection times out | network path, firewall, host availability |
| Connection refused | service not listening, wrong port, backend stopped |
| Certificate warning | TLS, reverse proxy, certificate issuance |
| HTTP 502 / 504 | reverse proxy to backend path |
| Login page works but authentication fails | application, identity, session, backend compatibility |
| Dashboard loads but has no data | data source, exporter, database, scrape target |

## Escalation rule

Do not make broad configuration changes until the failing layer has been isolated.

Capture the command, expected result, actual result, and timestamp for each validation step when troubleshooting a persistent issue.

This creates an evidence trail that is useful for rollback, documentation, and future problem analysis.
