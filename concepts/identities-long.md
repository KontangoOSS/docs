# Identities

## What an identity is

An identity is the network's way of saying "I know who you are."

Every participant on Kontango has one — your laptop, the git server, the secrets vault, every controller, every router. Without an identity you don't exist on the network; with one, you're a known entity that the network can recognize, route, and either grant or deny access.

The closest analogy: an identity is like a passport. It has your name, it's tied to you specifically, it was issued by an authority everyone trusts, and showing it is the way you pass through controlled doors.

The "authority" in question is your own controller. Kontango isn't asking a third party to vouch for you — the network you run is the one that issues and trusts the certificates. That matters: it means there's no upstream service that can revoke your identities, no vendor lock-in, and no external dependency to keep paying for.

## The three flavors you'll encounter

You'll meet identities in three forms. They're all the same kind of thing technically, but they play different roles.

**A device identity — your laptop or phone.**
When you enroll a device, you create one of these. It says "I'm this person's laptop, here's the certificate that proves it." The device uses this identity every time it tries to reach an internal service.

**A host identity — a server in a data center.**
A server that hosts a service has its own identity. The git server has an identity that says "I'm the git host." This is how the network knows where `git.tango` actually lives.

**A service identity — a program that talks to other programs.**
Some services need to talk to each other (a CI runner pulling secrets from `bao.tango`, for example). The CI runner gets its own identity so the secrets vault knows it's allowed to ask.

You won't always be able to tell which type you're looking at by name, and you don't have to — they all work the same way under the hood.

## What an identity has

Three pieces of information matter:

| | What it is | Analogy |
|---|---|---|
| **Name** | A human-readable label like `alex-laptop` or `git.tango` | The name on your passport |
| **Certificate** | A cryptographic file that proves the identity is yours | The passport itself |
| **Roles (attributes)** | Tags like `#stage-2` or `#trusted` that decide what the identity can reach | Visa stamps showing which countries you're allowed in |

The certificate is what does the actual proving. Roles are what gate access — they're the connection between an identity and a [policy](policy).

## How the certificate is structured

If you want to understand what's really going on under the hood:

- Each identity has an X.509 certificate signed by your platform's intermediate CA, which is in turn signed by the root CA stored in your secrets vault.
- The certificate's Subject Alternative Name (SAN) carries a SPIFFE URI like `spiffe://kontango/device/alex-laptop` — that's the canonical identifier the network uses, separate from any human-readable name.
- Public key cryptography is ECDSA P-256 by default. RSA-2048 is supported for legacy clients but not the default.
- Validity is one year. Auto-renewal happens at 75% of lifetime.
- Revocation is enforced server-side. The controller maintains a revocation list; on every new connection, it checks the presented cert against the list. There's no CRL distribution to worry about because every connection terminates at the controller anyway.

This matters in two situations: when you're debugging "my cert is rejected" issues (check the SPIFFE URI matches the identity name in the controller), and when you're rotating a CA (you need to issue new certs to all identities; auto-renewal helps).

## Roles, attributes, and how policies see you

Roles are how identities get scoped to services. Practically:

- An identity has a list of role attribute strings: `#stage-2`, `#trusted`, `#host-something`, `#web-clients`, etc.
- Services have **dial policies** that say "anyone with these roles may connect to me" and **bind policies** that say "anyone with these roles may host me."
- When you try to reach a service, the controller checks your identity's roles against the service's dial policy. Match → allowed. No match → denied at the network layer, before any data reaches the service.

Roles are evaluated dynamically. If an admin removes `#stage-2` from your identity right now, your existing connection might keep flowing (TCP doesn't care about retroactive auth changes), but every *new* connection from your device will be denied. So role changes take effect within seconds for any practical workflow.

There's also a concept of **stages** — `#stage-0` through `#stage-3` — that represent a trust progression. A new device starts at `#stage-0` (very limited) and gets promoted as it earns trust. Most policies are written against stages, so promoting a device is the usual way to give it more access.

## What you don't have to worry about

You don't need to understand the cryptography. You don't need to know what an X.509 certificate is, or what mutual TLS does, or what trust anchors are. The agent on your device handles all of it. The only thing you might do, ever, is:

- See your device's name in an admin's list
- Ask for your device to be granted a different role

That's it. The rest is automatic.

## A quick test

> "Is my laptop a service or an identity?"

Your laptop is an identity. So is the git server. So is the secrets vault.

A *service* is the thing your laptop **connects to** — the git server, when it's serving git over the network, is a service. The git server's *identity* is what proves it's the real git server.

The two concepts overlap in a fuzzy way for service hosts: the git host has an identity (proving it's the git host) and provides a service (git, at `git.tango`). For end users, that distinction usually doesn't matter — what matters is "your laptop = identity," "the thing you're trying to reach = service."

See [Services](services) for the other side.

## Where to next

- [Services](services) — what's available to connect to
- [Policy](policy) — how roles decide what an identity can reach
- [When things break](when-things-break) — symptoms when an identity goes wrong
