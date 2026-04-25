# SSL Certificates

This chapter is about the cryptographic identity layer that makes everything else work. Two completely separate PKIs run in a Kontango deployment, and conflating them is one of the most common mistakes. Understand the split early.

## The two PKIs

**Public PKI (Let's Encrypt).** For services that talk to the public internet. Anyone with a real browser must trust the cert. Issued by Let's Encrypt; lives on Caddy; visible to public clients.

**Internal PKI (Kontango's own).** For everything on the overlay. Controllers, edge routers, devices, services-talking-to-services. Rooted in your secrets vault; never trusted by browsers; never visible to the public internet.

These don't overlap. Don't try to make them. The public certs are for public eyes; the internal certs are for internal trust. Mixing them creates bizarre failure modes and weakens both.

## Public certificates with Let's Encrypt

For the services Caddy exposes publicly, Caddy handles the entire ACME flow:

1. On startup, Caddy generates a private key for each public hostname
2. Caddy sends a CSR to Let's Encrypt
3. LE replies with a challenge — either a file to serve at `/.well-known/acme-challenge/...` (HTTP-01) or a DNS TXT record to add (DNS-01)
4. Caddy completes the challenge
5. LE returns a signed cert valid for 90 days
6. Caddy stores the cert (in OpenBao, in our default setup) and starts using it
7. ~30 days before expiry, Caddy renews automatically

You configure this once in the Caddyfile and forget about it.

### HTTP-01 vs DNS-01

The two challenge types matter operationally:

**HTTP-01** — LE GETs a path on your server. Requires your server to be reachable on port 80 from the public internet at the hostname being validated. Simple, works for most public-facing services.

**DNS-01** — LE checks a TXT record at your DNS provider. Requires API access to your DNS. Works for hostnames that aren't reachable on port 80 (like wildcard certs, or hosts behind a firewall).

For most Kontango deployments, **DNS-01 is the right default**. Reasons:

- Wildcard certs (`*.example.org`) require it
- Controllers behind firewalls or with port 80 closed can still get certs
- More resilient — doesn't depend on inbound connectivity at the moment of renewal

Our default config uses DNS-01 against Cloudflare. Other providers (Route 53, Gandi, deSEC, etc.) have plugins; the Caddy config changes are a few lines.

### Cert storage

Caddy needs to store the cert and key somewhere. Default Caddy stores them on disk. Our build of Caddy stores them in **OpenBao**, which has two real benefits:

- All controllers share the same cert (no per-host renewal storms)
- Backup is one of OpenBao's normal jobs; you don't need a separate cert backup process

The downside: bootstrap dependency. Caddy can't get its first cert until OpenBao is up. We handle this with a chicken-and-egg-aware install sequence.

### Renewal monitoring

Set up an alert for cert renewal failures. The pattern that bites people is: renewal fails silently for 60 days, then the cert expires at midnight and everything breaks.

The Grafana dashboard ships with a panel watching cert expiry. If anything is within 14 days, you get an alert. You should never see this in normal operation.

## Internal Kontango PKI

The overlay's security model rests on internal certificates. Each device, each service, each controller has a cert. Mutual TLS on every connection. The trust anchor for all of it is **your root CA**, which lives in OpenBao and never moves.

### The chain

Three layers:

1. **Root CA** — generated once, kept offline. Only used to sign intermediate CAs.
2. **Intermediate CA** — actually used to sign device and service certs. Stored in OpenBao.
3. **Identity certs** — the per-device, per-service certs issued during enrollment.

Why a separate intermediate? Two reasons:

- The root can stay offline (or in a sealed vault) and only get touched when you rotate intermediates — which is rare
- If an intermediate is compromised, you can revoke it and issue a new one without throwing away the root

### Cert lifetimes

| Type | Lifetime | Renewal |
|---|---|---|
| Root CA | 10 years | Manual when it expires |
| Intermediate CA | 5 years | Manual; plan to rotate before expiry |
| Device cert | 1 year | Auto-renewed at 75% of lifetime |
| Service cert | 1 year | Auto-renewed at 75% of lifetime |

Device certs renew automatically — the agent on each device handles this. Service certs renew automatically — the controller does it.

You only manually intervene when intermediate or root expiry is approaching. Set a calendar reminder for 6 months before either expires.

### What happens during enrollment

Enrollment is the moment a device gets its first internal cert. Walking through it:

1. Device runs the installer with a one-time-token (OTT)
2. Device generates its private key locally — **the private key never leaves the device**
3. Device sends a CSR + the OTT to the controller
4. Controller validates the OTT, signs the cert with the intermediate CA, returns the signed cert
5. Device stores the cert and starts using it

The OTT is one-shot — once redeemed, it's invalid. The cert is the long-term credential.

### Revocation

Revoking a device's cert is how you disable access. The pattern:

```
admin runs:  ziti edge delete identity <device-name>
```

The controller marks the identity revoked. From that moment, every new connection that presents the revoked cert is denied. Existing TCP connections might keep flowing for a few seconds (until they need a new session token), then they break.

This is faster than CRL distribution because Kontango doesn't *use* CRLs — every connection terminates at the controller, so the controller's own state is authoritative. Stolen device → admin runs delete → device is locked out within seconds.

## Cert-related operational practice

A short list of things that matter:

- **Back up the root CA private key, sealed.** Print the QR code, store in a fireproof box. If your OpenBao goes sideways, this is your only path to recovery.
- **Don't email cert files.** They're not secret in the same way as keys, but you should use OpenBao or the platform's identity service to provision them, not human-handled file transfers.
- **Watch the cert dashboard.** Grafana has a built-in panel; cert expiries shouldn't be a surprise.
- **Test revocation.** Once a quarter, revoke a test device, verify it loses access, re-enroll it. Mechanical practice.

## When things go wrong

Common cert problems:

| Symptom | Cause | Fix |
|---|---|---|
| `tls: bad certificate` from a service | Cert chain mismatch — usually intermediate not present in trust bundle | Reissue the cert; make sure the bundle has the full chain |
| Device can't reach overlay after restart | Cert expired and renewal failed | Re-run installer on the device |
| Caddy renewal fails | DNS-01 challenge fails — usually API token revoked or DNS provider issue | Update Cloudflare token in OpenBao; restart Caddy |
| Internal mutual auth fails after PKI rotation | Old certs still trusted; new ones not yet on all clients | Wait for auto-renewal; force re-enroll the laggards |

## What's next

[Chapter 7: Three-Tier Architecture](07-tier-architecture) — public, protected, internal — and how to decide which tier each service belongs in.
