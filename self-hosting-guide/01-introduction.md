# Introduction & Cost Analysis

Welcome to the Self-Hosting Guide. This is the long version of "how do I actually do this." Eleven chapters, each one builds on the last, and by the end you'll have a real working private cloud on your own hardware.

This first chapter is about **why**. If you're already convinced and you just want to install things, skip to [Chapter 2: Prerequisites](02-prerequisites). But if you're trying to make the case to yourself, your team, or your boss, this is where the numbers and the stories live.

## The cloud bill is real

Cloud providers got cheap on purpose. The first VM is free, the next one is a few dollars, and then one day the spreadsheet shows you're spending five figures a month and nobody can explain exactly why.

Three patterns drive that:

- **Egress fees.** Moving data *into* the cloud is free. Moving it *out* is metered. Once your data is in, leaving is expensive — by design.
- **Managed services markup.** A managed Postgres on AWS is roughly 4x the underlying compute. The convenience tax compounds across every service.
- **License inflation.** SaaS pricing climbs faster than inflation. The product you signed up for at $15/seat is now $30/seat with no real change.

None of this is a moral judgment. Cloud is the right answer for many workloads. It's just that the assumption "of course we use cloud for everything" stopped being defensible a few years ago.

## Real-world examples

A few well-documented stories:

**37signals (Basecamp, HEY)** — moved off AWS in 2023-2024. Reported saving $7M over five years on cloud spend. Their CTO David Heinemeier Hansson has been public about the math: their on-prem hardware cost was a fraction of the cloud equivalent, and they have engineers who can run the boxes.

**Dropbox** — left AWS S3 between 2015 and 2017. Built their own storage stack ("Magic Pocket"). Reported saving $74M over two years.

**GitLab** — discussed leaving major cloud and built out their own infrastructure for the most expensive workloads.

These are big companies with big workloads, and the lesson isn't "cloud is bad." The lesson is that **the math changes** at some scale, and that scale is lower than most people think. For a small team or a homelab, you're often past the break-even point at the very first dollar.

## Where Kontango fits

Kontango isn't competing with AWS for hyperscaler workloads. It's the platform layer for **the things you'd otherwise rent**: source control, secrets, dashboards, monitoring, identity, ticketing, the small services every team needs.

The math in our world looks like this:

| Service | SaaS monthly cost | Self-hosted equivalent |
|---|---|---|
| Git (10 users) | GitHub Team — $40 | Forgejo on a Pi — $0 + Pi cost |
| Secrets management | HashiCorp Cloud — $80–$500 | OpenBao on a VM — $0 |
| Monitoring (10 dashboards) | Datadog Pro — $200+ | Grafana on a VM — $0 |
| Identity / SSO (50 users) | Auth0 — $250+ | Zitadel — $0 |
| Issue tracking | Linear — $80+ | Ticketarr — $0 |

These are illustrative monthly numbers; your actual SaaS bill depends on team size and usage. But the pattern is consistent: a few hundred dollars a month of SaaS, replaced by hardware you already own and software that's free.

The trade is **operational time**. Self-hosted costs ops effort. Kontango is opinionated about minimizing that — by giving you the network, identity, and policy layers as a coherent platform, you spend your time on services, not on plumbing.

## What "private cloud" actually means here

People say "private cloud" and mean different things. Let's be specific about what Kontango gives you:

- **All your services on hardware you control**, anywhere — home, office, colo, cheap VPS, mix and match
- **One private network** that connects everything, regardless of where it physically lives
- **Identity-based access** — every device, every service, every user has an identity, and policies decide who reaches what
- **No public exposure by default** — internal services don't have public addresses, period
- **Open source end to end** — Forgejo, OpenBao, Zitadel, Grafana, Caddy, OpenZiti, all your data, all your code

What it *isn't*: an automatic horizontal scaler, a multi-tenant managed service, a replacement for AWS for petabyte-scale storage. Match the tool to the workload.

## When self-hosting is the wrong answer

Honest list:

- **You have nobody who can run a server.** Not "nobody who wants to" — nobody who *can*. Then either find someone, or stay on SaaS.
- **Your workload genuinely needs hyperscale elasticity.** A traffic spike from 10 to 100,000 RPS in five minutes is not a self-hosted homelab problem.
- **You have a regulatory requirement that mandates a specific provider.** Some defense or financial work requires FedRAMP'd vendors. Kontango doesn't help.
- **The thing you're hosting is genuinely public** — a marketing site, a public API. Cloudflare or a small VPS is fine; you don't need a Kontango cluster for that.

For everything else — internal tools, team services, infrastructure, the things that should be private anyway — self-hosted on Kontango is usually the right answer.

## What this guide will give you

By Chapter 11 you'll have:

- A 1-to-3 node cluster running a private cloud
- A working private overlay network with `.tango` addresses
- Hardened nodes with automated certs and updates
- Caddy as the public edge, with TLS, OIDC, and rate limiting
- A tier architecture (public, protected, internal)
- Identity-based access via Zitadel SSO
- Monitoring via Grafana, with metrics flowing
- Backup and update runbooks you can actually follow

That's a real platform — not a toy.

## What's next

[Chapter 2: Prerequisites & Planning](02-prerequisites) — what you need before you start: hardware, network, expectations.
