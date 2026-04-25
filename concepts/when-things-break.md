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
