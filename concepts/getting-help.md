---
title: "Getting Help"
parent: Concepts
nav_order: 8
---

# Getting Help

When something's wrong and the [troubleshooting page](when-things-break) hasn't fixed it, here's how to get a person involved and what to send them.

Kontango is open source, but help is paid work. The same way RedHat ships free software and sells support, anyone running Kontango can either operate it themselves using the docs and community, or pay for hands-on help — deployment, on-call, custom services, training. Both paths are real. Neither is hidden behind the other.

## How to reach us

| Channel | Best for |
|---|---|
| **Discord** — [join here](https://discord.gg/CrXWJfwD) | Quick questions, community discussion, casual help. The fastest way to get a human in the loop. |
| **`hello@kontango.io`** | General questions, partnerships, "is Kontango right for me?", press, anything that isn't urgent |
| **`support@kontango.io`** | Real issues — broken enrollments, services that won't load, security concerns, anything that needs an actual response |

Discord is moving fast right now and is the best place to start. If the invite link above has expired, send a note to `hello@kontango.io` and we'll get you a fresh one.

## When to use which

The decision tree, more or less:

| Situation | What to do |
|---|---|
| Quick question or want to chat with the community | Discord |
| You think you're seeing a security issue | `support@kontango.io`, mark URGENT in the subject |
| Your enrollment is stuck or rejected | `support@kontango.io` |
| A service has been down for more than ten minutes | `support@kontango.io` |
| You'd like Kontango deployed for you / want to talk pricing | `hello@kontango.io` |
| You found a bug in the docs | Open an issue or PR on the docs repo |

## What to include when you email

Including all of this saves a round-trip:

```
What I was trying to do:
  (e.g., "load git.tango in my browser")

What happened:
  (e.g., "the page loaded but says 'this site can't be reached'")

When it started:
  (e.g., "this morning around 9:30 AM Pacific")

Is it still happening:
  (e.g., "yes" or "it's intermittent")

My device:
  Hostname:        (run `hostname`)
  Operating system: (e.g., Ubuntu 24.04 / macOS Sonoma)

Agent status:
  (output of `systemctl status ziti-edge-tunnel.service` if you can run it)

Anything I tried:
  (e.g., "restarted the agent, didn't help")
```

If you can't run any commands because the network or device is broken, that's fine — just say so. We can usually figure things out from the symptom and the hostname alone.

## What you can do without help

These are safe and often resolve the problem:

- **Restart the agent**: `sudo systemctl restart ziti-edge-tunnel.service`
- **Re-run the installer**: it's idempotent — running it again won't break anything that's already working
- **Hard refresh your browser**: `Ctrl+Shift+R` (Mac: `Cmd+Shift+R`)
- **Try a different `.tango` service** to isolate whether the issue is one service or your whole connection

## What you should not do alone

- **Don't disable the agent.** That doesn't fix things; it guarantees nothing works.
- **Don't change DNS settings** to try to make `.tango` resolve "another way." It won't, and you'll make diagnosis harder.
- **Don't share your enrollment files** with anyone, including support, unless they explicitly ask. We almost never need them.
- **Don't try to "speed up" enrollment** by killing the installer mid-run. The 60-second wait is intentional.

## A note on response times

This is a small platform. Responses are usually fast during working hours and slower at night and weekends. If something is genuinely urgent, say so in the subject line — "URGENT: locked out before customer demo" gets a different priority than "question about git access."

Discord is generally faster than email for casual questions; email is the right call for anything that needs a record.

## Where to next

- [When things break](when-things-break) — try this first
- [The eight concepts](index) — useful background if you're new
