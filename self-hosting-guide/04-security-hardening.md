# Security Hardening

Before you trust a node with your secrets, code, or infrastructure, lock it down. This chapter covers the OS-level hardening that makes Kontango's network-layer security actually meaningful. A perfectly-configured zero-trust overlay on top of a wide-open server doesn't help — both layers have to be solid.

The good news: hardening Linux for a Kontango node is mostly applying the same baseline you'd want on any internet-facing server. There's nothing exotic.

## The threat model

Be specific about what you're defending against:

- **Opportunistic scanning** — bots checking every IP on the internet for default passwords, exposed admin panels, known CVEs
- **Targeted attack** — someone who knows you're running Kontango and wants in
- **Lateral movement** — a service gets compromised; can the attacker pivot to others?
- **Insider mistakes** — a misconfigured rule or a leaked secret

The first one is the loudest noise on the internet. The second is what scares people. The third and fourth are what actually causes most real-world breaches. Hardening should address all four.

## Baseline OS hardening

A short list that covers most of the common ground:

### 1. SSH

Three changes to `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

Then restart sshd. Now your SSH only accepts public-key auth, never passwords, never root.

If you can disable SSH entirely once Kontango is set up — you don't need it; you can SSH over the overlay — even better. But until you've enrolled a client device, you need SSH for bootstrapping.

### 2. Firewall

`ufw` on Debian/Ubuntu, `firewalld` on RHEL-family. Default-deny inbound, allow only what you specifically need:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp     # SSH (remove later if you can)
sudo ufw allow 443/tcp    # Kontango public TLS
sudo ufw allow 1280/tcp   # Kontango controller
sudo ufw allow 3022/tcp   # Ziti edge router
sudo ufw enable
```

Adjust if you're running services that need other public ports.

### 3. Automatic updates

Install `unattended-upgrades` and let security patches apply automatically:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Yes, automatic updates can break things. The alternative — running unpatched servers — breaks more things, more often, with worse consequences. Apply security updates automatically; manage feature updates with a maintenance window.

### 4. Fail2ban

Install `fail2ban` to throttle brute-force attempts on SSH:

```bash
sudo apt install fail2ban
```

Default config is fine. After a few failed login attempts from one IP, that IP gets banned for 10 minutes. If you've already disabled password auth this is mostly defense-in-depth, but it's cheap and reduces log noise.

### 5. Time sync

Sounds boring, matters a lot. Ziti uses time for cert validation; clock drift breaks everything. Make sure `chrony` or `systemd-timesyncd` is running:

```bash
timedatectl
```

Should show "System clock synchronized: yes."

### 6. Disable swap on low-memory nodes

If your node has 8 GB RAM and 8 GB swap, the system will swap aggressively, which kills performance and can leak memory contents to disk. For production nodes, either size the RAM appropriately and disable swap, or use encrypted swap.

## Kontango-specific hardening

A few things that aren't generic Linux advice but matter for Kontango:

### Bao (secrets vault) lives in memory

When OpenBao is unsealed, its secrets are decrypted in memory. If someone has root on that host, they can read them. So:

- Run OpenBao on a dedicated host where it's the only sensitive workload, or
- Use OpenBao's auto-unseal with a cloud KMS so the seal key isn't on disk
- Either way, **encrypt the disk** so a stolen drive doesn't reveal anything

### The PKI root key

The root CA private key is the most sensitive thing in your deployment. Anyone with it can impersonate any identity on the network. So:

- It should never live on a controller — it lives in OpenBao
- OpenBao should be on a separate host from the controllers
- Take an offline backup of the root key (sealed) and store it somewhere physically secure

### Audit logging

Kontango logs every connection to the controller. Pipe these somewhere — Grafana Loki or a syslog server — and keep them at least 90 days. When something looks weird, you'll want a record.

## What hardening doesn't fix

Be honest about the limits:

- **Hardening doesn't help if your password manager is compromised.** Use 2FA on the few accounts that matter (Cloudflare, GitHub, your DNS provider, your VPS provider).
- **Hardening doesn't help if you commit secrets to git.** Use OpenBao for everything sensitive; never put secrets in environment files that get committed.
- **Hardening doesn't help if the supply chain is compromised.** Pin versions, verify checksums, use reproducible builds where you can.

## A hardening verification checklist

After hardening, verify:

- [ ] `ssh root@host` fails (no root login)
- [ ] `ssh -o PubkeyAuthentication=no user@host` fails (no passwords)
- [ ] `ufw status` shows only the ports you intended
- [ ] `apt list --upgradable` is empty (or auto-upgrade has applied them)
- [ ] `timedatectl` shows synchronized
- [ ] `fail2ban-client status sshd` shows it's running
- [ ] Disk encryption is on (or you've decided you don't need it)

## What about something like CIS benchmarks?

CIS hardening benchmarks are excellent and worth applying if you're going for a serious posture. They go further than this chapter — disabling unused kernel modules, restricting cron, locking down network sysctls.

For homelab use, the basics in this chapter are enough. For business or compliance contexts, run a CIS benchmark scan after you've applied the basics and address what it flags.

## What's next

[Chapter 5: Reverse Proxy & Tunnels](05-reverse-proxy) — putting Caddy at the edge, terminating TLS, and making services reachable.
