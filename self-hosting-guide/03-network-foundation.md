# Network Foundation

Before you install Kontango, your underlying network needs to be sane. This chapter covers the network layer your platform sits on top of — segmentation, VLANs, and the topology decisions that make later chapters easier or harder.

If you're running everything on cloud VPSes, much of this is implicit (the cloud provider handles the network). Skip to [Chapter 4](04-security-hardening) if that's you.

## The two networks you care about

Every Kontango deployment has two network layers:

1. **The underlay** — your physical or virtual network. Wires, switches, NICs, routes, DNS.
2. **The overlay** — Kontango's encrypted virtual network. `.tango` addresses, identity-based routing, mutual TLS everywhere.

This chapter is about the underlay. The overlay is built on top of it in later chapters.

## Topology — three patterns

Pick the one that matches your situation.

### Pattern A: All-cloud

All your nodes are VPSes (DigitalOcean, Hetzner, AWS Lightsail, whatever). Each has a public IP. Cloud provider handles the underlay.

- ✅ Simplest to bring up
- ✅ Public access works without port forwarding
- ❌ Recurring monthly cost (~$5–20 per node)
- ❌ You don't actually own the hardware

Use this when: you don't have hardware lying around, or you specifically want geographic distribution.

### Pattern B: All-on-premises

All your nodes are at home, in your office, or in a colo. Your router does NAT. Public reach requires either port forwarding or a tunnel.

- ✅ Lowest ongoing cost (electricity)
- ✅ You own the hardware end to end
- ❌ Public reach requires more setup
- ❌ Single ISP outage takes you offline

Use this when: you have decent home/office connectivity and you really want to own your stack.

### Pattern C: Hybrid

Most real deployments. Controllers live in the cloud (cheap, public, always-on), service hosts live on-premises (where the data is, where it's cheap to run). The Kontango overlay stitches them together.

- ✅ Best of both — public reach where you need it, cheap on-prem where you don't
- ✅ Controllers are small enough to be cheap (one CPU, 1 GB)
- ⚠️ Two networks to think about

Use this when: you're past "homelab" but you don't want to pay cloud rates for everything.

## VLAN segmentation

If your network is anything beyond "one switch, one subnet," consider segmenting. The reason is blast radius: when something gets compromised, segmentation limits how far it can reach.

A simple, defensible setup:

| VLAN | Purpose | What's allowed in |
|---|---|---|
| **VLAN 10 — Trusted** | Your laptop, admin workstation | Outbound only |
| **VLAN 20 — Servers** | Kontango controllers and service hosts | Inter-VLAN to specific ports |
| **VLAN 30 — IoT** | Cameras, smart home, the garage door | No inter-VLAN; outbound only |
| **VLAN 99 — Guest** | Visitors, untrusted devices | Internet only, no LAN access |

Kontango doesn't *require* this — it's not an OPNsense replacement. But Kontango works *with* this, and the combination is much stronger than either alone. Network segmentation gives you defense-in-depth even when you've also got identity-based access on every connection.

If you don't have VLAN-capable equipment, that's fine. Many home networks are flat, and Kontango still gives you all of its security properties on a flat network. Segmentation just makes the failure modes shallower.

## What needs to talk to what

Map the actual network paths:

**Each controller** must reach:
- All other controllers (Raft consensus, port 1280 outbound and inbound)
- Public internet for ACME challenges and updates
- Your DNS provider's API (e.g. Cloudflare) for DNS-01 challenge

**Each service host** must reach:
- At least one controller (for Ziti edge router connection, port 3022)
- Other service hosts only if the services need to talk

**Each client device** must reach:
- At least one controller (for enrollment and the overlay edge)
- That's it — everything else routes via the overlay

Notice: **service hosts don't need direct connectivity to each other** in the underlay. The overlay handles that. This means you can put service hosts on separate networks, even on different continents, and Kontango makes them act like neighbors.

## DNS — public and private

Two namespaces, two strategies:

**Public DNS** — for the controller hostnames and any public services. Use a real provider (Cloudflare, Route 53). Set A or AAAA records pointing to the controllers' public IPs. For wildcard certificates, you'll need API access to your DNS provider for DNS-01 challenges.

**Private `.tango` DNS** — handled by the Kontango overlay. Each enrolled device's local DNS resolver intercepts `.tango` lookups and routes them through the overlay. No public DNS records exist for `.tango` services. This is by design.

**Important:** the `.tango` DNS interceptor must be **scoped** to `.tango` only. If it tries to be the default resolver for everything, it breaks public DNS resolution on the device. Most installs do this correctly out of the box, but if you've customized things, double-check.

## NAT and firewalls

For the underlay:

- **Inbound** — controllers need 443 (public TLS), 1280 (controller API), 3022 (edge router). Forward these from your edge router to the controller(s).
- **Outbound** — controllers need to reach the public internet on 443 (ACME), and your DNS provider's API.
- **Inter-controller** — if controllers are behind different firewalls, they need to reach each other on 1280. WAN-side or via a VPN tunnel between sites.

For overlay traffic, **nothing else needs to be open**. The overlay tunnels everything through the controller's public ports. That's the security property — your service hosts don't need any inbound ports open at all.

## Latency expectations

Some rough numbers for capacity planning:

- **Controller-to-controller (Raft)** — works fine across a continent (~80ms RTT). Across the Pacific (~150ms RTT) is also fine. Don't try to span both Pacifics in the same Raft group.
- **Client-to-controller (enrollment, session)** — 50ms is great, 200ms is fine, 500ms is noticeable on the first connect.
- **Client-to-service (overlay traffic)** — basically the same as the underlying internet. The overlay adds 1–3ms of TLS handshake overhead per new connection. Existing connections have no measurable overhead.

If you're running everything on one LAN, you'll never notice any of this. If you're geographically distributed, plan accordingly.

## What this looks like in practice

A homelab deployment might be:

- One controller on a $5/mo Hetzner VPS in Germany
- Two service hosts at home (a Pi running Forgejo, a desktop running OpenBao and Grafana)
- Three laptops/phones as enrolled clients

A small business deployment might be:

- Three controllers in DigitalOcean (one per region for HA)
- Service hosts on a Proxmox box in the office
- 10–20 laptops/phones as enrolled clients
- Optional edge routers in branch offices for low-latency access

A bigger deployment scales the controller count up and adds geographic edge routers.

## Validation checklist

Before moving on:

- [ ] Decided on topology (all-cloud, on-prem, hybrid)
- [ ] Static IPs (or DHCP reservations) for all controller and service-host nodes
- [ ] Public DNS records pointing at controller IPs
- [ ] Firewall rules for 443/1280/3022 inbound to controllers
- [ ] Verified each controller can reach the others on 1280
- [ ] Verified each controller can reach the public internet on 443
- [ ] DNS provider API token saved (you'll need it for ACME later)

## What's next

[Chapter 4: Security Hardening](04-security-hardening) — locking down the OS before you put anything important on it.
