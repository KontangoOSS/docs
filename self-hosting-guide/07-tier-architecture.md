# Three-Tier Architecture

Every service you run on Kontango lives in one of three tiers. Putting a service in the right tier is the most important security decision after enrollment itself. Get it wrong and you've either exposed something internal or made internal users sign in to read a status page.

## The three tiers

**Public** — anyone on the internet can reach it.

**Protected** — anyone on the internet can reach it, but they must authenticate first (OIDC at Caddy).

**Internal** — only enrolled devices reach it. The service has no public address at all.

Each tier has different reachability, different attack surface, different operational concerns.

## What goes in each tier

### Public tier

Things that *must* work for unenrolled users, plus things that have low intrinsic risk.

**Required public:**
- The auth portal (Zitadel) — users have to log in to get access to anything else
- Webhook receivers — external systems POST in
- The marketing/landing site — your front door

**Optional public:**
- A status page
- Public API endpoints, if any
- Documentation that you want findable in search

What goes in public is a security decision: more public surface = more places to attack. Don't put anything in public that doesn't need to be there.

### Protected tier

Things that need to be reachable from any browser, but not by random scanners. The user has to log in via Caddy's OIDC flow, *then* gets proxied to the service.

**Typical protected services:**
- Your team's internal blog or wiki
- A service-status dashboard (the human-readable kind)
- Some CI tools that need browser access but don't have their own auth

The pattern: Caddy intercepts the request, sees no auth cookie, redirects to the auth portal, accepts the OIDC callback, sets a session cookie, then forwards to the service. The service itself doesn't need to know about auth — Caddy enforces it.

This is useful when you want a SaaS-like login flow but don't want to put the underlying service on the public internet directly.

### Internal tier

The default. Everything you can put here, should go here.

**Typical internal services:**
- Git (Forgejo, Gitea)
- Secrets vault (OpenBao)
- CI runners (Woodpecker, Jenkins)
- Monitoring (Grafana, Prometheus)
- Issue tracking (Ticketarr)
- Databases (Postgres, Redis)
- Message buses (NATS)
- SSH to your servers

These are reachable only by enrolled devices. The public internet doesn't see them. Even if attackers know the names, they can't dial.

The trade: users have to be enrolled to reach these. That's a feature, not a limitation. It means your security perimeter is "the device is enrolled," not "anyone in the world."

## How traffic actually routes

A Kontango deployment with all three tiers looks like:

```
Public client → [Caddy public route] → service
Public client → [Caddy OIDC + protected route] → auth portal → service
Enrolled client → [overlay tunnel + policy check] → service
```

Same edge node, three different paths. The tier of a service decides which paths exist.

## Deciding which tier

A heuristic that holds up in practice:

> If the service has its own user authentication and would survive being on the public internet at AWS, it can be public.
>
> If the service has no authentication or has weak authentication, but the audience is non-technical users, it can be protected.
>
> Everything else is internal.

The default is internal. Don't promote a service to a higher tier without a specific reason.

## Worked examples

**Forgejo (git server).** Forgejo has its own user accounts and decent auth. But: the data inside is private code, the audience is enrolled developers, there's no reason for the public to see it. Verdict: **internal**. Developers reach it via the overlay; outsiders never see it.

**Zitadel (auth portal).** This is the single most public-facing service you have. Users have to reach it before they're enrolled. Verdict: **public**.

**Grafana (monitoring).** Grafana has its own auth, but the dashboards reveal operational details that benefit attackers. Verdict: **internal**, possibly with an OIDC-protected variant if you want to share specific dashboards externally.

**Ticketarr (issue tracking).** Internal. There's no reason for randos to file tickets in your private system.

**A status page.** Probably **protected** — the public has no business seeing detailed status; your customers might.

**A landing page describing your services.** **Public** — this is marketing.

## Cross-tier traffic

Services often need to talk to each other across tiers. A few patterns:

- **Public service writes to an internal service.** The public service has its own enrolled identity for the overlay; uses that identity to dial the internal service.
- **Internal service responds to a public webhook.** The webhook hits Caddy (public), Caddy validates and forwards into the overlay to the internal handler.
- **Two internal services talk.** Both are on the overlay; one dials the other; policy decides.

The pattern: every internal-to-anything connection goes through an identity. There's no anonymous "service mesh" in the back. This is what makes the lateral-movement story strong.

## Common mistakes to avoid

A few real failures we've seen:

- **Putting Grafana in protected because "we trust the SSO."** OK but you've now created a path for any session-hijacked account to read your operational details. Internal is safer.
- **Putting Forgejo in public because "we want to show off our open-source repos."** Use a separate public mirror. Don't make the host of your internal code public.
- **Trying to make everything internal, including the auth portal.** Then nobody can enroll the first time. Auth is a chicken-and-egg; it has to be public.
- **Putting databases in public.** This still happens. Databases are internal-tier, no exceptions.

## Migrating tiers

Sometimes a service starts internal and you decide it should be protected. The mechanics:

1. Add a Caddy route for the new public hostname with OIDC
2. Test that the route works (it forwards into the overlay to the same service)
3. Announce the change to users
4. (Optional) Tighten the overlay policy on the original service so direct overlay access is restricted to admins

You can also move the other direction: a protected service becomes internal-only by removing its Caddy route. Users now have to enroll to reach it.

## Tier and policy interact

Internal-tier services *also* have policies. A device being enrolled isn't enough to reach every internal service — the device's roles still have to match the service's policy. So there are really four conceptual tiers:

- Public — no auth
- Protected — public auth at edge
- Internal/permissive — overlay + minimal policy
- Internal/restricted — overlay + strict policy (e.g. admin-only)

Most internal services start permissive (any enrolled stage-2+ device) and tighten as you understand actual usage.

## What's next

[Chapter 8: Zero-Trust Network Access](08-zero-trust-access) — the OpenZiti overlay in depth: enrollment, policies, identity, the whole identity-based routing model.
