# When Things Break

This is the page to read when something isn't working. It lists the symptoms you're most likely to see, what each one usually means, and what to try before contacting support.

## "This site can't be reached" or a Chrome-style error

**What it means:** Your device isn't on the Kontango network right now. Either the agent isn't running, or your enrollment is broken, or the address you typed isn't a real `.tango` service.

**What to try:**

1. Check the agent: `systemctl status ziti-edge-tunnel.service` (Linux). It should say `active (running)`.
2. If it says `inactive` or `failed`, restart it: `sudo systemctl restart ziti-edge-tunnel.service`. Wait five seconds, try again.
3. If the agent is running but you still can't reach anything, your enrollment may have been revoked. Contact the admin with your device's hostname.

**Diagnostic checklist for operators:**

- `journalctl -u ziti-edge-tunnel.service -n 50` shows recent agent logs. Look for `OIDC` errors (auth-policy mismatch), `tls: bad certificate` (cert chain mismatch — usually a CA rotation issue), or repeated reconnect attempts (controller unreachable).
- `ziti edge list edge-routers` from the controller shows whether your device's edge router is online. If it's offline, the device can't reach anything regardless of enrollment status.
- Check the controller logs for `enrollment denied` events tied to this hostname — if the device was previously revoked, re-enrolling won't auto-restore unless the admin clears the revocation first.

A note: some Kontango deployments serve a fake browser-error page to anyone who hits an internal hostname without proper enrollment — so you may see what looks like a Chrome error page complete with the dinosaur. That's actually the system politely telling you "you don't belong here yet."

## A page loads slowly, then works

**What it means:** Internal proxies refresh their session periodically. While that's happening, requests can stall briefly. This is expected, not a bug, and usually lasts under 30 seconds.

**What to try:** Wait, refresh, move on. If it persists for more than a minute, treat it as the agent-not-running case above.

**Operator detail:** Caddy refreshes its Ziti session every ~25 minutes by default to renew the API session token. The refresh is fast but blocking. You'll see a brief connection delay during the refresh window, which is why staggering refresh times across multiple Caddy instances matters when you're scaling. The refresh window is logged at INFO level in Caddy.

## Login form rejects valid credentials

**What it means:** Either you mistyped your password, the auth service is restarting, or your account has been disabled.

**What to try:**

1. Re-type carefully — caps lock, password manager auto-fill, etc.
2. If three attempts fail, wait five minutes (the auth service may be cycling).
3. If it still fails, contact the admin to check your account status.

**Operator detail:** Auth lives in Zitadel by default. Three failure modes:
- **Account locked** — too many failed attempts; admin can unlock from the Zitadel admin UI
- **OIDC actions broken** — pre-existing Zitadel bug where actions throw "function not found"; check Zitadel logs for `actions.*function not found`
- **Token expired** — reload the page; the auth flow restarts and gets a fresh token

## A specific service won't load — but others do

**What it means:** This is almost always a [policy](policy) issue. Your device is fine, the network is fine, but you don't have a role that lets you reach this particular service.

**What to try:**

1. Confirm by trying a different `.tango` service you know works — if that loads, the network is fine.
2. Note the service name you can't reach.
3. Contact the admin: "I'm enrolled but `<service>.tango` isn't loading." They'll check the policy and either add you or explain why you don't have access.

**Operator detail:** From the controller side, run `ziti edge list service-policies 'service.name="<service>.tango"'` to see who's allowed. Then `ziti edge list identities --filter "<device-name>"` to see the device's current role attributes. Compare. If the device should have access, add the missing tag with `ziti edge update identity <name> --role-attributes <new-tag>`.

## Some features inside a service are broken

**What it means:** A service you can reach has a feature that depends on another backend (a database, an upload service) and that backend is having trouble. The service is up, the feature is degraded.

**What to try:**

1. Try the action again in a couple of minutes — many of these are transient.
2. If it persists, capture: what you were doing, the exact error message, the time, and the URL. Send all of that.

**Operator detail:** Most "feature degraded" issues are upstream service health problems — postgres connection pool exhausted, MinIO bucket full, NATS message queue backed up. Grafana's service-health dashboard is the right place to triage.

## File uploads or attachments fail

**What it means:** The object storage backend is unhealthy or full.

**What to try:** Wait two minutes and retry. If it keeps failing, report it with the file name and the service you were using.

**Operator detail:** MinIO storage capacity, bucket policy, or service identity permission. Check `mc admin info` and the MinIO operator logs.

## You used to see a service in your list — now it's gone

**What it means:** A policy changed, or your role changed. This usually isn't an outage; usually somebody adjusted permissions.

**What to try:** Contact the admin. If you should still have access, it's a quick fix.

## Edge router or controller is hard down

**Operator-only.** If the entire `.tango` namespace stops resolving, suspect controller cluster issues:

1. `ziti agent cluster list` from any controller shows the Raft cluster state. Healthy = leader + 2 followers.
2. If quorum is lost, the cluster is in read-only mode — existing connections work, new ones can't be authorized.
3. Single-node failures recover automatically. Two-node failures need manual intervention; consult the runbook.

Edge router failures are softer: clients fall back to other routers automatically. A single dead router degrades but doesn't break the overlay.

---

## What NOT to do

- **Don't disable or uninstall the agent.** It's how everything works. Removing it doesn't fix anything; it just guarantees you can't reach internal services until you re-enroll.
- **Don't edit DNS or firewall rules manually.** Kontango handles routing for `.tango` addresses. Changing your local DNS will break things and make them harder to diagnose.
- **Don't share your enrollment files.** The certificate on your device is tied to you. If someone else has it, they have your access — and worse, the audit trail will say it was you.
- **Don't try to enroll the same machine twice.** It will work, but it'll create a second identity and cause confusion. If you need to re-enroll, ask the admin to delete the old identity first.

## Self-service fixes that are safe

These are okay to try on your own:

| Symptom | Try this |
|---|---|
| Agent not running | `sudo systemctl restart ziti-edge-tunnel.service` |
| Browser shows cached error | Hard refresh — `Ctrl+Shift+R` (Mac: `Cmd+Shift+R`) |
| You think your enrollment is broken | Re-run the installer; it's safe to run again |
| You don't know if something is up | Try a known-working `.tango` service first to isolate |

## Where to next

- [Where help comes from](getting-help) — what to send when you need a person
- [Policy](policy) — to understand "why am I denied"
- [Enrolling a device](enrollment) — if your enrollment really is broken
