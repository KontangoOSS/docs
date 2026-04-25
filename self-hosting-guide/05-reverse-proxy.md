# Reverse Proxy & Tunnels

Caddy is the public-facing edge of a Kontango deployment. It's the only thing the public internet sees: it terminates TLS, handles authentication for public services, and proxies traffic into the overlay network. Everything else is hidden behind it.

This chapter covers what Caddy does in a Kontango context, why we use it specifically, and how it relates to the Ziti tunnel that runs alongside it.

## The job at the edge

Imagine your cluster from the perspective of someone on the public internet. What can they see?

- The IP addresses of your controllers
- Port 443 listening with a TLS certificate
- A handful of named hostnames responding (your auth portal, maybe a public landing page)
- That's it

Everything else — git, secrets, dashboards, your tickets system — is invisible. Caddy is the gatekeeper that makes that work. It accepts public connections, decides which hostname they're for, and routes them appropriately:

- **Public service** (login portal, status page) → proxy to the service host inside the overlay
- **Internal-only service** (most of them) → reject; they shouldn't be reachable from outside
- **Auth-required service** → present an OIDC challenge; proxy only on success

## Why Caddy specifically

Several real reasons over nginx, traefik, or HAProxy:

- **ACME built-in.** Caddy gets and renews TLS certs automatically with no separate certbot dance. Out of the box it'll handle HTTP-01 challenges; with our DNS plugin it'll do DNS-01 against Cloudflare.
- **Layer 4 SNI routing.** A standard Kontango deployment uses the same port 443 for *everything* — public web traffic, Ziti edge router, Caddy. Caddy at L4 looks at the SNI hostname and decides where each connection goes. One port, multiple services.
- **OIDC plugin.** caddy-security gives us SSO via Zitadel without writing nginx config files.
- **Single binary.** No PHP-FPM-style sidecars; one process, easy to reason about.

We ship a custom Caddy build (`caddy-kontango`) that includes the plugins we need: Cloudflare DNS, OpenBao storage for certs, OpenZiti, NATS for telemetry, OIDC, Layer 4 SNI splitter. Source is at `KontangoOSS/caddy-kontango`.

## How traffic flows on port 443

When a packet arrives at the controller's port 443:

1. Caddy's L4 layer reads the SNI from the TLS handshake
2. If the SNI is `ctrl.example.org` (the Ziti control plane hostname) → route to the Ziti edge router process listening locally on a different port
3. If the SNI is `auth.example.org` (a public service) → route to Caddy's HTTP layer, terminate TLS, then proxy
4. If the SNI is anything else → drop or serve the honeypot

This split is what lets you have only port 443 open and still serve both Ziti's mTLS traffic and public HTTPS services. Without it, you'd need separate ports.

## TLS certificates

A working Kontango deployment has at least three certs:

- **The wildcard `*.example.org` cert** for public services that share the apex domain
- **Per-controller certs** (`ctrl-1.example.org`, `ctrl-2`, `ctrl-3`) — issued by Let's Encrypt via DNS-01 so the controllers don't need to be reachable for HTTP-01
- **Internal Ziti PKI** — the root and intermediate CAs that sign device and service identities. This PKI is internal-only; never shows up on public certs.

Caddy manages the public certs. The Ziti controller manages the internal PKI. **Don't mix them.** Don't try to use Let's Encrypt certs for Ziti identity — Ziti needs its own PKI for the security model to work.

## Configuring Caddy

A minimal Caddyfile for a Kontango edge looks like:

```caddyfile
# Layer 4 splitter — runs first, decides where to send each connection
{
    layer4 {
        :443 {
            @ziti tls sni ctrl.example.org
            route @ziti {
                proxy 127.0.0.1:1280
            }
            @http tls sni *.example.org
            route @http {
                tls
                http
            }
        }
    }
}

# Public auth portal
auth.example.org {
    reverse_proxy 127.0.0.1:8080
}

# Static landing page
example.org, www.example.org {
    root * /var/www/html
    file_server
}
```

This is the bones. Real deployments add per-service routes, OIDC, headers, rate limiting. We have a [reference Caddyfile](https://github.com/KontangoOSS/caddy-kontango) in the platform repo.

## Where the Ziti tunnel fits

Caddy handles **public** ingress. The Ziti tunnel handles **overlay** ingress: it's the thing on each controller that accepts encrypted overlay connections from enrolled clients and forwards them to the right service host.

Conceptually:

```
Public client → Caddy (public TLS) → service host
Enrolled client → Ziti tunnel (mutual mTLS overlay) → service host
```

Both paths terminate at the same service host, but they get there differently. Public clients go through Caddy and have to satisfy whatever auth Caddy enforces. Enrolled clients come through the overlay and present their device identity, which the policy system checks.

## Public services we usually expose

A typical small deployment has these public-facing pieces:

- **The auth portal** (Zitadel) — `auth.example.org` — needs to be public so users can log in before they're enrolled
- **A status / landing page** — optional, low value, low risk
- **Webhooks** — if your CI runs on private compute but needs to receive GitHub webhooks, those webhooks have to land somewhere public

That's usually it. Everything else lives entirely on the overlay.

## Webhooks and inbound integrations

A common challenge: an external system (GitHub, Stripe, a webhook from a SaaS you use) needs to POST to one of your services. The service is internal. What to do?

Two patterns:

1. **Receive at Caddy, forward over overlay.** Make a public route at Caddy (e.g. `webhooks.example.org/github`), validate the signature, then forward into the overlay to the actual handler.
2. **Use a webhook-relay service.** ngrok, smee.io, or self-hosted equivalents. Less clean, but sometimes simpler.

Pattern 1 is what most production deployments do. The Caddy validation gives you a chance to drop garbage before it ever touches your internal service.

## Avoiding common mistakes

A few things that bite people:

- **Don't expose internal services through Caddy "just for testing."** Once a public route exists, attackers find it. If the service was meant to be internal, keep it on the overlay.
- **Don't share certificates across hosts.** Each controller should have its own LE-signed cert. Sharing certs creates blast radius if one host is compromised.
- **Don't forget the wildcard renewal.** Let's Encrypt wildcards via DNS-01 work, but if your DNS provider API access breaks, renewals silently fail until you notice. Monitor renewal status.

## What's next

[Chapter 6: SSL Certificates](06-ssl-certificates) — deeper into the cert lifecycle: what gets issued, by whom, when, and how renewals work.
