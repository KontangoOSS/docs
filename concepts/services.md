---
title: "Services"
parent: Concepts
nav_order: 5
---

# Services

## What a service is

A service is a thing on the network that you connect *to*. Apps, dashboards, code repositories, databases, SSH targets — anything you reach across the network is a service.

If an [identity](identities) is "who I am," a service is "what I'm trying to reach."

Services are how the platform earns its keep. The reason to run Kontango is so that the things you'd otherwise pay a SaaS for — git hosting, CI, secrets, monitoring, ticketing — can run on your own hardware, on your own terms, reachable from anywhere you happen to be working that day.

## Examples of services you might run

A typical Kontango deployment will host services like these. The exact list is up to you — Kontango isn't prescriptive about which open-source apps you run, only about how they're networked together.

| Address | What it might be |
|---|---|
| `git.tango` | A self-hosted git server (Forgejo, Gitea) |
| `bao.tango` | A secrets and credentials vault (OpenBao, Vault) |
| `auth.tango` | An identity provider for SSO (Zitadel, Authentik) |
| `dashboards.tango` | Monitoring dashboards (Grafana) |
| `tickets.tango` | An internal ticketing system |
| `ssh-<host>` | SSH access to any node |

You add a service by registering it on the network and pointing it at where the underlying app actually runs. From then on, anyone with the right [policy](policy) can reach it at the `.tango` address.

## Internal vs. public

A service can be either:

**Internal** — only reachable from enrolled devices. This is the default and it's where almost everything lives. The service has *no* public address. From the outside internet, it does not exist.

**Public** — reachable from anyone on the internet. Public services still benefit from Kontango (mutual auth, encryption end-to-end), but they accept connections from the public side too. A login portal is the obvious example — it has to be reachable before you're enrolled, by definition.

You can usually tell from the address: `.tango` is internal-only, anything with a real public domain is public.

## Why the `.tango` ending

`.tango` is a private namespace that only resolves on enrolled devices. It's not a real internet domain — DNS for `.tango` exists only inside the Kontango network.

This is what makes a service truly invisible. There's no DNS record for `git.tango` on any public name server. Even an attacker who knew the name couldn't find an IP. Inside the network, your device's agent intercepts `.tango` lookups, decides "that's a Kontango service," and routes the connection through the encrypted overlay.

It's like an internal extension number at a phone system — it works inside, it doesn't work outside.

## Discovering what's available

If you operate the platform, you can list services through the admin tools. As a regular user, the simpler answer is: **what your admin tells you you have access to**. Services are not advertised by default, and policies determine what each device can see in the first place. Ask your admin what's running, and they'll tell you.

## Where to next

- [Identities](identities) — the other side of the connection
- [Policy](policy) — what determines whether you can reach a given service
- [When things break](when-things-break) — what to do when a service won't load
