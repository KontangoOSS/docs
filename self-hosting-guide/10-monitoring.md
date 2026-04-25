# Monitoring & Observability

You can't operate what you can't see. This chapter is about what to watch on a Kontango deployment, what stack we ship by default, and what alerts actually matter versus which ones add noise.

## What to watch

The signal-to-noise distinction is the whole game. Most monitoring setups fail not because they don't collect data — they collect everything — but because the operator stops looking when there are 47 dashboards and 200 alerts a day.

A useful taxonomy of what to monitor:

**Symptom-based:**
- Is the service responding?
- Is the response time acceptable?
- Is the error rate acceptable?
- Are users getting auth'd successfully?

**Cause-based:**
- Is CPU saturated?
- Is memory full?
- Is disk full?
- Is the certificate near expiry?
- Is there packet loss between nodes?

Symptom-based alerts wake you up. Cause-based metrics tell you why once you're up. Both matter. **Page on symptoms; investigate with causes.**

## The default stack

Kontango ships with this observability layer:

- **Telegraf** — collects metrics from each node and from select services
- **InfluxDB** — time-series storage
- **Loki** — log storage
- **Grafana** — dashboards and alerting

All four are open-source, self-hosted, and run on the overlay. None require external SaaS.

This isn't the only viable stack — Prometheus is excellent and many shops prefer it. We default to Telegraf+Influx because the per-host agent has lower friction for small deployments. For scale, Prometheus or VictoriaMetrics are well worth the swap.

## What gets collected by default

Out of the box, after a Kontango install:

**Per-node metrics:**
- CPU usage, load average
- Memory usage, swap usage
- Disk usage per filesystem, IOPS, latency
- Network bytes in/out per interface
- Systemd unit health (which services are up)

**Kontango-specific metrics:**
- Number of enrolled identities
- Active overlay connections
- Edge router throughput
- Controller Raft state (leader, follower, term)
- Cert expiry distance for both PKIs

**Service-specific metrics** (if you wire each service):
- Forgejo: requests/sec, response times, build queue depth
- OpenBao: secret read/write rate, seal status, leadership
- Grafana itself: dashboard view counts, alerting state
- Caddy: requests, response codes, TLS handshake counts

## What alerts matter

A short list of things that should page someone:

**Severity 1 — wake someone up:**
- Controller cluster has lost quorum
- Public TLS cert expires in less than 14 days and renewal is failing
- A service in production is down (5xx rate > 50% for 5 min)
- Disk on any node > 90% full
- Suspicious enrollment patterns (many failed enrollments from one IP)

**Severity 2 — tell someone in business hours:**
- A single service is degraded (response time 2x normal)
- A node has a hardware warning (SMART, ECC errors)
- Memory pressure on any node (> 90% used + active swap)
- Internal cert expiry in less than 30 days
- Backup hasn't run in over 24 hours

**Don't alert on:**
- Individual failed authentications (that's normal)
- A single 5xx response
- Brief network blips (< 30s)
- CPU spikes that resolve
- Anything that recovers in less than 5 minutes

## Dashboards we ship

The default Grafana setup has:

- **Cluster Overview** — controller Raft status, total identities, total active connections
- **Per-Node Health** — CPU/RAM/disk per controller and service host
- **Edge Traffic** — what's flowing through Caddy and the edge routers
- **Cert Expiry** — calendar view of every cert and its days-until-expiry
- **Service Health** — per-service uptime and response time
- **Audit** — recent enrollments, recent revocations, failed auth events

You can add more, but these eight cover most operational needs.

## Logging

Logs are a different beast from metrics. The ratio of useful-info to volume is much worse, and storage costs add up.

Default config:

- Each Kontango component writes to systemd journal
- A Loki agent ships journal to Loki
- Loki retains logs for 30 days by default; configurable
- Grafana queries Loki for log views

Log retention deserves a thought. Compliance contexts (financial, healthcare) often require longer retention; homelab use can usually get by with less. The trade is disk space — busy services log a lot.

**Don't log secrets.** This sounds obvious. It still happens. Audit your service log statements; redact tokens, passwords, session keys.

## Tracing

Tracing (OpenTelemetry, Jaeger) is excellent for diagnosing latency in distributed services. We don't ship it by default because most Kontango deployments don't have enough services for tracing to pay off.

When you do need it:

- Instrument the services with OpenTelemetry SDKs
- Run Jaeger on the overlay as another service
- Send traces to Jaeger; visualize via the Jaeger UI

The hard part is instrumentation, not infrastructure.

## SLOs and error budgets

If you're running services for a real audience, set explicit Service Level Objectives. A useful pattern:

- 99.9% uptime ≈ 43 minutes of unplanned downtime per month
- 95th percentile response time < 300ms for interactive services
- < 1% error rate on API endpoints

Display these on a dashboard. When you're inside the budget, ship features. When you're outside, focus on reliability. This is more discipline than tooling, but the dashboards make it visible.

## The observability paradox

Observability tooling itself is a service that needs observability. What if Grafana goes down? What if Loki fills the disk?

A few defenses:

- **The observability stack has its own minimal heartbeat.** A blackbox prober that just checks "is Grafana responding" via a different path
- **Disk pressure on the observability node should be aggressively alerted.** It often eats its own logs.
- **Don't rely on the same node for both observability and the thing being observed.** Run Grafana on a different node from your most critical services.

## Day-to-day rhythm

A working monitoring practice looks like:

- **Daily** — quick glance at the cluster overview dashboard during morning coffee. Anything red?
- **Weekly** — review last week's alerts. Were any false positives? Any silent failures?
- **Monthly** — review SLO performance. Are we meeting our targets? Where did the time go?
- **Quarterly** — review the alerting rules themselves. Add rules for incidents you've actually had. Remove rules that have never fired or always fire.

## What's next

[Chapter 11: Maintenance & Operations](11-maintenance) — backups, updates, day-2 operations, and the runbook practices that keep this all working over years.
