# When Things Break

This is the page to read when something isn't working. It lists the symptoms you're most likely to see, what each one usually means, and what to try before contacting support.

## "This site can't be reached" or a Chrome-style error

**What it means:** Your device isn't on the Kontango network right now. Either the agent isn't running, or your enrollment is broken, or the address you typed isn't a real `.tango` service.

**What to try:**

1. Check the agent: `systemctl status ziti-edge-tunnel.service` (Linux). It should say `active (running)`.
2. If it says `inactive` or `failed`, restart it: `sudo systemctl restart ziti-edge-tunnel.service`. Wait five seconds, try again.
3. If the agent is running but you still can't reach anything, your enrollment may have been revoked. Contact your admin with your device's hostname.

A note: some Kontango deployments serve a fake browser-error page to anyone who hits an internal hostname without proper enrollment — so you may see what looks like a Chrome error page complete with the dinosaur. That's actually the system politely telling you "you don't belong here yet."

## A page loads slowly, then works

**What it means:** Internal proxies refresh their session periodically. While that's happening, requests can stall briefly. This is expected, not a bug, and usually lasts under 30 seconds.

**What to try:** Wait, refresh, move on. If it persists for more than a minute, treat it as the agent-not-running case above.

## Login form rejects valid credentials

**What it means:** Either you mistyped your password, the auth service is restarting, or your account has been disabled.

**What to try:**

1. Re-type carefully — caps lock, password manager auto-fill, etc.
2. If three attempts fail, wait five minutes (the auth service may be cycling).
3. If it still fails, contact the admin to check your account status.

## A specific service won't load — but others do

**What it means:** This is almost always a [policy](policy.md) issue. Your device is fine, the network is fine, but you don't have a role that lets you reach this particular service.

**What to try:**

1. Confirm by trying a different `.tango` service you know works — if that loads, the network is fine.
2. Note the service name you can't reach.
3. Contact the admin: "I'm enrolled but `<service>.tango` isn't loading." They'll check the policy and either add you or explain why you don't have access.

## Some features inside a service are broken

**What it means:** A service you can reach has a feature that depends on another backend (a database, an upload service) and that backend is having trouble. The service is up, the feature is degraded.

**What to try:**

1. Try the action again in a couple of minutes — many of these are transient.
2. If it persists, capture: what you were doing, the exact error message, the time, and the URL. Send all of that.

## File uploads or attachments fail

**What it means:** The object storage backend is unhealthy or full.

**What to try:** Wait two minutes and retry. If it keeps failing, report it with the file name and the service you were using.

## You used to see a service in your list — now it's gone

**What it means:** A policy changed, or your role changed. This usually isn't an outage; usually somebody adjusted permissions.

**What to try:** Contact the admin. If you should still have access, it's a quick fix.

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

- [Where help comes from](getting-help.md) — what to send when you need a person
- [Policy](policy.md) — to understand "why am I denied"
- [Enrolling a device](enrollment.md) — if your enrollment really is broken
