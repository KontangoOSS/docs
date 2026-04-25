---
title: "Identities"
parent: Concepts
nav_order: 4
---

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

## The one thing not to worry about

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
