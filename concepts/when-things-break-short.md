# When Things Break

Most problems on Kontango fall into one of three buckets:

**1. "This site can't be reached" — your agent isn't running.**
On Linux: `sudo systemctl restart ziti-edge-tunnel.service`. Wait 5 seconds, try again.

**2. One service won't load, others work.**
That's a [policy](policy) issue — your device doesn't have the right role for that service. Contact your admin: "I'm enrolled but `<service>.tango` isn't loading."

**3. Login rejects valid credentials.**
Either you mistyped, or the auth service is restarting. Wait 5 minutes and try again. If it persists, ask your admin to check your account.

**Don't do these things:**
- Don't disable the agent (you'll lose access entirely)
- Don't edit DNS or firewall rules
- Don't share your enrollment files

If none of the above fixes work, [reach out for help](getting-help) — `support@kontango.io` or Discord.

Switch to [medium](?v=medium) for the full troubleshooting list, or [long](?v=long) for the diagnostic detail.
