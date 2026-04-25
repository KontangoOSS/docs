# What Is Kontango?

## The one-sentence version

Kontango is an open-source platform that lets you run your own cloud — on whatever hardware you already own, anywhere in the world — using the same industrial-grade tools the big providers use, without needing a Kubernetes degree to do it.

## The one-paragraph version

Kontango exists because your data should be yours. Not on someone else's servers, not under someone else's terms of service, not paywalled behind a subscription that doubles every year. Kontango gives you a real cloud — secrets management, source control, dashboards, monitoring, identity — running on hardware **you own**, in places **you choose**, connected by a private, encrypted network that nobody outside the system can see. The whole stack is open source. The whole design assumes you own the infrastructure end to end. Security and zero-trust networking aren't the headline; they're the property that makes self-hosting safe enough to actually trust with your data.

## Why this exists

There's an enormous amount you can already do with the hardware you have. An old desktop, a Raspberry Pi, a laptop that's not doing much, a small server in a closet, a cheap VPS — any of those can run real services. The reason most people don't is that the path from "I have a computer" to "I have a working private cloud" used to look like this:

- Learn Kubernetes
- Stand up a registry, a CA, a secrets manager, a CI runner, a reverse proxy, an identity provider
- Wire them all together
- Decide what to expose and how
- Hope you got the security right
- Pay a SaaS provider to do all of the above instead

Kontango is the third option: **the integrated path**. You bring hardware. Kontango brings the platform. The result is a homelab — or a small business stack, or a whole company's internal infrastructure — that runs on your equipment and nobody else's, with the same kind of tooling the big clouds use internally.

## What you actually get

When you run Kontango, you get:

- **A private overlay network** that connects all your devices and services securely, regardless of where they physically live. Office, home, VPS, friend's basement — all one logical network.
- **Industry-grade tools** for the things every cloud needs: secrets management, git hosting, CI, dashboards, identity, and more — all open source, all self-hosted.
- **Hardware freedom**. Run on a Raspberry Pi, a desktop, a rack server, a cheap cloud VM, or all of the above at once.
- **Zero exposure by default**. Internal services have no public address. They don't show up in scans. There's no login page sitting on the open internet for attackers to attack.
- **One enrollment per device**. After the first install, your laptop reaches everything on your private network — no VPN to toggle, no port forwards to maintain.

## Why "your data should be yours"

The current default, when you want to do anything serious with computers, is to rent. Rent storage, rent compute, rent SSO, rent monitoring, rent the place your code lives. Each of those rentals comes with terms — terms about who can read your data, who can shut your account down, what happens when the company gets bought, what happens when the price goes up.

Self-hosting is the alternative, and it's been technically possible for years. The reason it isn't more common is that doing it well — with proper security, automation, and the kind of resilience a real workload needs — has been a full-time job. Kontango's whole reason for existing is to collapse that into a small, opinionated, open-source platform that anyone with hardware can run.

## Open source, with help available

The whole platform is open source. The code is in the open, the design is in the open, and you can run it without paying anyone. That's the same model RedHat made popular: the software is free, the help is the product. If you want to set Kontango up yourself, read the docs and go. If you want someone to deploy it, run it for you, or build something specific on top of it, that's available too. We're not selling licenses. We're not gating features. The way you choose to use this is up to you.

Open source doesn't mean unsupported, and it doesn't mean free as in "no one ever gets paid." It means the code belongs to everyone, and the people who build and maintain it get paid for the help they offer on top.

## Three things you actually do with it

1. **Reach your stuff from anywhere.** Open `git.tango` in your browser from your couch and your private git repository is right there — the same as if you were sitting next to the server.
2. **Run real workloads on hardware you own.** A small business runs its ticketing system, its accounting, its file sharing, and its source code on a single server in the back office. None of it is on the public internet. None of it is on someone else's cloud.
3. **Build a homelab that's actually useful.** Spin up a service, give it a `.tango` address, and your laptop and your phone can both reach it without you ever opening a port on your router.

## What it isn't

- **Not a SaaS.** There's no Kontango cloud you log into. You run the whole thing yourself.
- **Not a fork of Kubernetes.** It's a different bet — that most workloads don't need orchestration at that scale, and most people would rather not run Kubernetes if they don't have to.
- **Not an exit-from-the-internet manifesto.** Public services still work, public internet still works, and Kontango plays nicely with both. The point is that *your* infrastructure can be private without giving up reach.

## Where to next

- If you want to picture how this works as a network, read [The network](the-network).
- If you're about to enroll a device, read [Enrolling a device](enrollment).
- If you're wondering what's actually live, see [Services](services).
