# Identity & Access Management

The previous chapter was about *device* identity — which laptop, which server, which service. This chapter is about *user* identity — who's actually sitting at the keyboard and what they're allowed to do inside the services they reach.

These are two different things, and Kontango uses both. A device gets onto the network (zero-trust overlay). A user gets into a service (SSO). You need both for a complete story.

## The two-layer model

When you log into Forgejo on Kontango, two things have to be true:

1. **Your device is enrolled** and policy permits it to reach `git.tango`
2. **Your user account is valid** and Forgejo permits you to do what you're doing

Layer 1 is OpenZiti and the overlay. Layer 2 is Zitadel and the SSO that ties services together.

Without layer 1, an attacker can't even reach the login page. Without layer 2, anyone on the device can do anything in the service. Both layers matter.

## Why Zitadel

There are several open-source identity providers (Keycloak, Authentik, FusionAuth). We default to **Zitadel** for a few practical reasons:

- **Modern** — written in Go, designed around OIDC from the start
- **Self-hosted, no compromise** — runs in a single container with Postgres
- **Strong API** — most operations (creating users, projects, applications) work via API, which means GitOps-style workflows are realistic
- **Multi-tenancy** — you can host multiple isolated identity domains on one Zitadel instance, useful for managed-service scenarios

Authentik and Keycloak are also fine choices. If you're already operating one of those, use it. The integration patterns described in this chapter port across with minor changes.

## What gets centralized in Zitadel

In a typical Kontango deployment:

- **User accounts** — name, email, password, MFA, group memberships
- **Service accounts** — for CI runners, automation, headless services
- **Applications** — each service that uses SSO is registered as an application
- **Projects** — group of applications that share user/permission scope
- **Actions** — custom logic that runs on auth events (custom claims, hooks)

The pattern: one Zitadel project per logical environment. Each service is an application inside that project. Users are granted access to specific applications.

## How services connect

Every service that does SSO is, from Zitadel's view, an OIDC client. The flow:

1. User browses to `git.tango`
2. Forgejo sees no auth cookie, redirects to `auth.example.org/oidc/login?...`
3. User enters credentials at Zitadel; MFA if configured
4. Zitadel redirects back to Forgejo with an auth code
5. Forgejo exchanges the code for an ID token + access token
6. Forgejo creates/updates the local user from the ID token, sets session cookie
7. User now has a Forgejo session

The one-time setup per service: register an OIDC application in Zitadel, configure the service to use it (issuer URL, client ID, client secret, redirect URI). Most modern services have OIDC built in or via plugin.

## What's worth centralizing

A common question: "Should I use SSO for everything, or only some services?"

**Centralize:**
- Anything where users come and go (employees, contractors, family members on the homelab)
- Anything that requires MFA — easier to enforce in one place than in every service
- Compliance-relevant services where you need an auth audit trail

**Don't centralize:**
- Public services that have anonymous access (a public landing page)
- Service-to-service authentication — use service identities on the overlay, not user SSO
- Highly-trusted infrastructure — the SSO failure mode is "nobody can log in"; don't make recovery dependent on the thing that's down

## MFA and session policy

Once Zitadel is the front door, configure it well:

- **Require MFA for admin-level groups.** TOTP, WebAuthn, push notifications — Zitadel supports all three.
- **Session lifetimes.** Default is 12 hours. Consider 8 hours for normal users, 1 hour for admin sessions.
- **Idle timeout.** 30 minutes for sensitive applications.
- **Password policy.** Modern guidance: long passwords, no forced rotation, breached-password detection.

## User vs service identities

Two distinct kinds of accounts in Zitadel:

**User accounts** — a human. Has a password, MFA, email, can log in via a browser.

**Service accounts** — a non-human. Has a long-lived secret or a JWT. Used by CI, automation, internal services that need to authenticate.

Use service accounts for non-human callers. **Don't have a CI runner log in as "ci@example.org" with a password.** That's a footgun. Service accounts can be scoped tightly and rotated independently.

## Group structure

A pattern that holds up:

```
admins             — full access to platform
operators          — can deploy services, can't manage users
developers         — can use developer tools (git, CI, secrets read for their projects)
contributors       — read-only access to most things; specific writes by exception
service-accounts   — non-human entities, scoped by app
```

Map these to OIDC groups. Configure each application to map groups to its own permissions. Keep the group structure simple; resist adding groups for every project (the admin overhead grows fast).

## How this interacts with the overlay's stages

Stages on the overlay are *device* trust. Groups in Zitadel are *user* permissions. They compose:

- A new device at `#stage-0` can't reach Forgejo at all (overlay denies)
- A `#stage-2` device can reach Forgejo, but if the user logging in is in the `contributors` group, they can only browse, not push
- A `#stage-3` device with an `admins` user can do anything

You generally want both layers tight. Don't rely solely on the overlay (a stolen device with `#stage-3` is bad). Don't rely solely on the SSO (a stage-0 quarantined device with admin SSO is also bad). Together, they cover each other.

## A worked setup

For a small business deployment, the IAM rollout looks like:

1. Stand up Zitadel internally (`auth.example.org` is its public face)
2. Create one project: `kontango`
3. Register applications: `forgejo`, `grafana`, `bao`, `tickets`, etc.
4. Create groups: `admins`, `developers`, `support`
5. Map group → application permissions in each app
6. Invite users via email; they set passwords and MFA on first login
7. As users join the team, add them to the right group
8. As users leave, disable their account in Zitadel — they lose access everywhere at once

That last point is the win. A leaver is one Zitadel disable away from losing access to everything. No hunting through individual services.

## Backup and DR for IAM

Identity is the keys-to-everything-else. Treat it accordingly:

- **Database backups** — Zitadel runs on Postgres. Back up the Postgres regularly (every hour for write-heavy environments).
- **Recovery codes** — every admin should have offline recovery codes for their MFA. Store them somewhere physical and safe.
- **Break-glass account** — one admin account with offline-stored credentials, used only if the SSO itself is broken and you need to recover.
- **DNS resilience** — `auth.example.org` going down means nobody can log in. If it's hosted on a controller that's also being patched, you're in for a bad day. Plan for this.

## What's next

[Chapter 10: Monitoring & Observability](10-monitoring) — Grafana, metrics, alerts, and what to actually watch.
