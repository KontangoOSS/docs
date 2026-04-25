# Zero-Trust Network Access

This chapter is the core of why Kontango exists. Everything else — Caddy, certs, tier architecture — is plumbing around the central idea of zero-trust networking. If you understand this chapter, you understand why your private cloud works the way it does.

## The traditional model

For thirty years, network security worked like a castle:

- Build a wall (firewall)
- Anyone inside the wall is trusted
- Anyone outside the wall is not

It scaled poorly. Once an attacker is inside the wall (phishing, a compromised laptop, a malicious insider), they have free rein. Lateral movement is the standard playbook in modern breaches.

Worse, "inside vs outside" stopped being meaningful when work went remote. Where's the wall when half your engineers are working from coffee shops?

## What zero-trust means here

Zero-trust isn't a product. It's a posture: **every connection is verified, every time, regardless of where it's coming from.**

Concretely, on Kontango:

- Every connection presents a cryptographic identity (mTLS)
- Every connection is checked against the destination's policy
- Every connection is logged
- There's no privileged "inside" — your laptop on hotel wifi gets exactly the same treatment as your laptop in the office, gated by exactly the same rules

If you compromise a single device, you get only what that device's policy allowed. There's no flat network to roam.

## The OpenZiti substrate

Kontango's overlay network is built on **OpenZiti**, an open-source zero-trust networking project. You don't have to know OpenZiti to use Kontango — the platform abstracts it — but it helps to know what's underneath.

The key OpenZiti concepts:

- **Identities** — every participant (device, host, service) has one
- **Services** — named endpoints (e.g. `git.tango`)
- **Policies** — who can dial / host which services
- **Edge routers** — entry points clients connect to
- **Controllers** — central authority that issues identities and enforces policies

When your laptop dials `git.tango`:

1. Laptop's tunnel agent looks up `git.tango` against the controller
2. Controller checks the laptop's identity against `git.tango`'s dial policy
3. If allowed, controller sets up a session: an encrypted tunnel from your laptop's edge router to the service's edge router
4. Traffic flows; both ends are mutually authenticated; everything is encrypted

The traffic never goes "to the public internet" the way a VPN would route it. It's an end-to-end encrypted tunnel between your laptop and the service host, optionally relayed through edge routers.

## What identity-based routing buys you

Several things, all of which compound:

**No exposed services.** Internal services have no public IP, no DNS record, no firewall hole. They can't be scanned. They can't be brute-forced. They can't be DDoSed. They simply don't exist from the outside.

**Identity is enforced at the network layer.** Even if a service has a bug — say, an authentication bypass — attackers still can't reach the service to exploit it. The bypass is gated by the network.

**Lateral movement is hard.** Compromise a developer laptop, you get the developer laptop's policy. You don't get the secrets vault. You don't get the production database. You get whatever that one laptop was scoped to.

**Auditing is universal.** Every connection is logged, with cryptographic identity attached. "Who did X" is always answerable.

**Geography doesn't matter.** Your office, your home, a coffee shop, a foreign country — same security posture. There's no special "I'm inside the office" exception.

## Practical enrollment

How a real device joins the network:

```bash
curl -fsSL https://your-controller.example.org/install | bash
```

That's the user-facing thing. Underneath:

1. Installer downloads the agent (OpenZiti's `ziti-edge-tunnel`)
2. Generates a CSR locally
3. Calls the platform's enrollment API with the CSR plus a one-time-token (OTT)
4. Controller validates the OTT, signs the cert, returns it
5. Agent installs as a systemd service
6. From now on, `.tango` addresses resolve and route through the overlay

The OTT can be issued in two ways:

- **Auto-approval** — the platform's decision engine looks at the device fingerprint and decides; most legitimate devices auto-approve in 60-90 seconds
- **Manual approval** — borderline cases queue for an admin to review

Once enrolled, the device's cert is valid for a year and renews automatically.

## Roles and policies in practice

A device's identity carries **role attributes** — short tags that policies match against. Examples from a real deployment:

```
your-laptop: #stage-2, #web-clients, #host-your-laptop
ci-runner-1: #stage-2, #web-clients, #ci-runners, #host-ci-runner-1
forgejo-host: #host-forgejo, #web-services
admin-laptop: #stage-3, #admin-users, #ssh-clients
```

Services have policies that mention these roles:

```
git.tango — dial: #stage-2 OR higher; bind: #host-forgejo
bao.tango — dial: #stage-3 OR #ci-runners; bind: #host-bao
ssh-* — dial: #ssh-clients; bind: #infrastructure-hosts
```

The controller evaluates these on every connection. Match → allowed. No match → denied at the network layer.

You don't write policies as a user. As an operator, you write them once per service and rarely touch them after.

## Stages — earning trust over time

New devices start at `#stage-0` — minimal access. They're enrolled but they can't reach much. Over time, the device earns higher stages:

- **Stage 0** (just enrolled) — quarantine; can SSH back to itself, that's it
- **Stage 1** (basic trust) — read-only access to dashboards, status pages
- **Stage 2** (working trust) — most application services (git, secrets, etc.)
- **Stage 3** (admin) — infrastructure access (SSH to servers, admin services)

Promotion happens manually (admin reviews and promotes) or automatically (rule-based — device has been online N days, OS is current, etc.). Demotion works the same way in reverse.

This protects you from a freshly-compromised device. A laptop that just enrolled doesn't get full access — it earns it.

## Testing zero-trust

Validate the security properties yourself:

**Test 1: scan from outside.** From a non-enrolled machine, run `nmap` against your controller's public IP. You should see port 443 (Caddy) and nothing else of interest. The Ziti control port should be SNI-multiplexed behind 443.

**Test 2: bare connect.** From a non-enrolled machine, try `curl https://git.tango`. It won't resolve. Try the controller's IP directly — Caddy returns the honeypot or a 404. There's no "git.tango" surface.

**Test 3: revoke and confirm.** Enroll a test device. Verify it can reach a service. Revoke its identity from the admin UI. Within seconds, the device loses access on its next connection.

If all three pass, the zero-trust model is working as designed.

## Limitations to be honest about

Zero-trust is strong but not magic:

- **A compromised device gets the device's policy.** If you've assigned `#stage-3` to a workstation that doesn't really need it, that's still a footgun. Right-size roles.
- **Policy is only as good as it's written.** A service with a permissive dial policy (anyone with `#stage-2`) doesn't differ much from a flat network for those clients. Tighten where it matters.
- **The overlay doesn't protect against application bugs that an authorized client triggers.** If a `#stage-2` device exploits a bug in git.tango, the network won't stop it. Application security still matters.
- **The controller is critical.** Lose the controller, lose policy enforcement. HA cluster + good backups are not optional in production.

## What's next

[Chapter 9: Identity & Access Management](09-identity-management) — single sign-on across services with Zitadel, and how it integrates with the overlay's identities.
