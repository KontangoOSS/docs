# Prerequisites & Planning

This chapter is what to gather and decide before you install anything. Spending an hour here saves you ten hours of rework later.

## Hardware

You need at least **one machine**. For a real cluster, you need three. Kontango runs on what you have.

**Minimum viable single node:**

- 2 CPU cores
- 4 GB RAM
- 40 GB disk
- A network interface, wired ideally

That's a Raspberry Pi 5, an old desktop, a $5/mo VPS, or a small VM. A single node gets you a working private cloud — what you lose is HA. If the node goes down, the cloud goes down with it. Acceptable for a homelab.

**Recommended for a real cluster (3 nodes):**

- 4 CPU cores per node
- 8 GB RAM per node
- 100 GB disk per node
- Wired ethernet, ideally with the nodes on the same LAN segment for low latency

Three nodes give you tolerance for one failure. Five nodes give you tolerance for two. Most non-hyperscale deployments don't need more than three.

**Disk speed matters more than disk size** for the controller workload. The controller writes Raft logs, audit trails, identity records. NVMe or a decent SSD is meaningfully faster than spinning rust. For service hosts (the boxes that actually run Forgejo, OpenBao, Grafana), the disk profile depends on the service — git is mostly read-heavy, secrets are tiny, dashboards are write-heavy.

## Network

A few decisions, in order of how much they bite you later:

**Public vs private control plane.** Are your controllers reachable from the public internet, or only from your LAN?

- **Public** (cloud VPS, e.g. DigitalOcean): nodes can be anywhere; clients connect from anywhere; you need a real public DNS name and TLS certs from Let's Encrypt
- **Private** (LAN only): nodes are at home or in your office; clients on other networks need a tunnel home (which Kontango itself provides)

Most production deployments are **public** because that's the only way for your laptop on hotel wifi to reach the controller without a separate VPN.

**Static or dynamic IPs.** Controllers should have stable addresses. On a cloud VPS, that's the default. On home hardware, set DHCP reservations (or assign static IPs) so the controller isn't moving around.

**Open ports.** A Kontango controller needs:

- `:443` — public TLS, the edge entry for Caddy
- `:1280` — controller management/API (TLS-only)
- `:3022` — edge router (the Ziti port)

If you're behind NAT, port-forward those three. If you're on a cloud VPS, just allow them in the firewall.

**DNS.** You need at least one public domain you control. The domain doesn't have to be the *primary* one for your business — `kontango.example.org` works fine if you already own `example.org`. Internal services use `.tango` and don't need public DNS at all.

## Operating system

Kontango runs on Linux. Anything modern works:

- **Ubuntu 22.04 / 24.04 LTS** — best supported, most testing
- **Debian 12** — stable and minimal, also fine
- **AlmaLinux / Rocky 9** — the RHEL family works
- **Proxmox VE** — works as the underlying hypervisor, with Kontango running in LXC or VMs

ARM and x86_64 are both fine. The Raspberry Pi 5 (arm64) is a perfectly capable controller node.

What we don't recommend:

- **Alpine** — the Ziti binaries assume glibc; musl works in some places but creates surprises
- **NixOS** — works but the install path isn't well-documented yet
- **Anything older than 5 years** — kernels too old to support modern systemd features the platform needs

## DNS planning

Before you install, write down:

- **Your public domain** for the cluster — e.g. `cloud.example.org`
- **Per-controller subdomains** — e.g. `ctrl-1.cloud.example.org`, `ctrl-2`, `ctrl-3`
- **A wildcard or service-specific domain** for public services — e.g. `*.cloud.example.org`
- **Your internal `.tango` namespace** — defaults to `.tango`, but you can change it

For the public domain, plan to use Cloudflare (the platform integrates well with Cloudflare DNS for ACME challenges and DDoS protection). Other DNS providers work but you'll need to wire the ACME DNS-01 challenge yourself.

## Trust anchors

Two pieces of trust have to come from somewhere:

**The root CA**. Kontango's PKI is rooted in your secrets vault (OpenBao). The intermediate CA used to sign device certificates lives there too. You'll create both during install.

**The admin identity**. Someone has to be allowed to administer the network at the start. That's you. Your initial admin identity is created during install and used to bootstrap everything else.

Plan for both. Decide who else (if anyone) gets admin access on day one. Two-person admin is a good practice; one-person admin is fine for a homelab.

## Time budget

Realistic expectations:

| Task | Time |
|---|---|
| Read this guide cover to cover | 4–6 hours |
| First single-node install on a fresh machine | 2–3 hours |
| First HA 3-node install | 4–6 hours |
| Stand up your first service (git, secrets, etc) | 1–2 hours each |
| Wire SSO across services | 2–4 hours |
| Tune monitoring and backups | 4–8 hours |

Total: about a long weekend for a single node, about a full week of evenings for a real cluster with services running. **Less than the time you spent setting up your last cloud account, in most cases.**

## A planning checklist

Before you move on, fill this out:

- [ ] How many nodes (1 or 3)?
- [ ] Where do they live (cloud, home, mix)?
- [ ] OS choice
- [ ] Public domain name
- [ ] DNS provider (Cloudflare or other)
- [ ] Static IPs assigned (or VPS public IPs noted)
- [ ] Ports open at the firewall (443, 1280, 3022)
- [ ] Admin identity name (e.g. your-handle-laptop)
- [ ] Backup plan — where backups go (S3-compatible, another box, both)

If any of those are blank, finish them before installing.

## What's next

[Chapter 3: Network Foundation](03-network-foundation) — VLANs, segmentation, and the topology you'll build on top of.
