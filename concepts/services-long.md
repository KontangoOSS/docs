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

## How a service is registered

If you operate the platform, here's the actual flow when you stand up a new service:

1. **Pick the app.** Forgejo, OpenBao, your own custom service — anything that listens on a TCP port and speaks HTTP, SSH, NATS, or any other protocol.
2. **Run it on a host that has a Kontango identity.** The host's identity is what gives it permission to advertise services on the overlay.
3. **Register the service in the controller.** This creates a `.tango` name and binds it to the host's identity. You give it role attributes that determine who can reach it.
4. **Write the policy.** The policy says "identities with role X can dial this service; identities with role Y can host it." Without a policy, the service is registered but unreachable.
5. **Restart the host's tunnel agent.** It picks up the new binding and starts accepting connections.

That's it. The service is live. Any enrolled device with a matching role can dial `<service>.tango` and connect.

For HTTP services that need to be public (a login portal, a status page), there's an extra step: add a Caddy route on the controller. Caddy sits at the edge, terminates public TLS, and proxies into the overlay. Adding or removing the Caddy route is the toggle for "public vs internal."

## Internal vs. public

A service can be either:

**Internal** — only reachable from enrolled devices. This is the default and it's where almost everything lives. The service has *no* public address. From the outside internet, it does not exist.

**Public** — reachable from anyone on the internet. Public services still benefit from Kontango (mutual auth, encryption end-to-end), but they accept connections from the public side too. A login portal is the obvious example — it has to be reachable before you're enrolled, by definition.

You can usually tell from the address: `.tango` is internal-only, anything with a real public domain is public.

The split matters operationally. Internal services don't need rate limiting or DDoS protection — the only reachable clients are the ones you've enrolled. Public services do, and Caddy provides both at the edge.

## Why the `.tango` ending

`.tango` is a private namespace that only resolves on enrolled devices. It's not a real internet domain — DNS for `.tango` exists only inside the Kontango network.

This is what makes a service truly invisible. There's no DNS record for `git.tango` on any public name server. Even an attacker who knew the name couldn't find an IP. Inside the network, your device's agent intercepts `.tango` lookups, decides "that's a Kontango service," and routes the connection through the encrypted overlay.

It's like an internal extension number at a phone system — it works inside, it doesn't work outside.

## Service types and protocols

Services aren't limited to HTTP. The overlay is protocol-agnostic, so any TCP service works:

- **HTTP/HTTPS** — web apps, APIs, dashboards
- **SSH** — `ssh user@host` over the overlay; no need to expose port 22 anywhere public
- **Database protocols** — Postgres, Redis, MongoDB, etc, accessed by service identity instead of password
- **Custom TCP** — internal RPC, message buses (NATS), anything else
- **UDP** — supported but less common; works for DNS-over-overlay and similar

Each service type gets the same security properties: mutual auth, end-to-end encryption, identity-based access. You don't have to add TLS to your internal HTTP services — the overlay already encrypts. You don't have to add an SSH proxy — the overlay is one.

## Discovering what's available

If you operate the platform, you can list services through the admin tools. As a regular user, the simpler answer is: **what your admin tells you you have access to**. Services are not advertised by default, and policies determine what each device can see in the first place. Ask your admin what's running, and they'll tell you.

There's a discovery story planned (a `.tango` directory service that lists what an identity is allowed to reach), but for now it's intentionally manual.

## Where to next

- [Identities](identities) — the other side of the connection
- [Policy](policy) — what determines whether you can reach a given service
- [When things break](when-things-break) — what to do when a service won't load
