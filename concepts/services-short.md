# Services

A service is a thing on the network you connect *to*. Apps, dashboards, code repositories, databases, SSH targets — anything you reach across the Kontango network is a service.

If an [identity](identities) is "who I am," a service is "what I'm trying to reach."

**Examples of services you might run:**

| Address | What it might be |
|---|---|
| `git.tango` | Self-hosted git (Forgejo, Gitea) |
| `bao.tango` | Secrets vault (OpenBao) |
| `auth.tango` | SSO / login (Zitadel) |
| `dashboards.tango` | Monitoring (Grafana) |

Each gets a `.tango` address that only resolves on enrolled devices. From the public internet, these services don't exist — there's nothing to scan, nothing to attack.

You add a service by registering it on the network and pointing it at the underlying app. From then on, anyone with the right [policy](policy) can reach it.

Switch to [medium](?v=medium) for the full set, or [long](?v=long) for the operations side.
