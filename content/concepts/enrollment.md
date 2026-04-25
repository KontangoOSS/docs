+++
title = "Enrolling a Device"
weight = 3
+++

# Enrolling a Device

## What enrollment actually is

Enrollment is how a device proves it's real, gets a digital certificate, and joins the Kontango network. The certificate is what the network checks every time the device tries to reach an internal service.

The closest everyday analogy: it's like getting a building access card on your first day at a job. The card identifies you, the building knows it issued the card, and from then on the doors recognize you when you swipe.

Once enrolled, your device's certificate is good for about a year. You don't have to renew it manually — the agent handles that.

It works on whatever device you bring — a brand-new laptop, an old desktop you had lying around, a Raspberry Pi, a cloud VM, a phone (when the mobile agent is ready). The whole point is that you don't need specific hardware to participate. If it can run a small background agent, it can join.

## The thirty-second version

You run one command — the install URL is the one your admin or the platform's installer gives you. The installer collects basic system info, streams about a minute of "is this a real computer" data to the controller, and the controller decides whether to issue a certificate. Most devices auto-approve. Once approved, the agent runs in the background from then on, and `.tango` addresses start working.

```bash
curl -fsSL https://your-controller.example.com/install | bash
```

That's the whole thing for most people.

## What information leaves your device

Enrollment is honest about what it sends. There are exactly two phases:

**Phase 1 — System fingerprint (sent once):**

- Your hostname (computer name)
- OS and architecture (e.g. Linux x86_64)
- A unique machine ID derived from your hardware

That's the device-identification phase. It's the same kind of information any IT inventory tool collects.

**Phase 2 — Liveness telemetry (streamed for ~60 seconds):**

- CPU usage percentage
- Memory usage
- System load average

That's it. **Not** sent: your files, your browsing history, your processes, your network traffic, your keystrokes, who you're logged in as.

The telemetry exists for one reason: to prove a real computer is enrolling, not a script trying to fake its way in. A bot can't easily produce 60 seconds of believable system behavior. After enrollment, the telemetry is **discarded** — it's only kept in memory during the decision, never written to a database, never persisted unless something looks suspicious enough to flag for review.

## What happens during the wait

The 60-second wait isn't dead time. The controller is doing three things:

1. **Collecting metrics** — every 5 seconds your device reports a number; over 60 seconds the shape of those numbers either looks like a real computer or doesn't
2. **Running a decision engine** — the controller checks the device against rules: is it from a trusted network, is the hostname plausible, do the metrics look genuine
3. **Scoring** — score above the threshold and your device auto-approves; below it, a human reviews

You'll see something like this in the installer output:

```
streaming telemetry... (58s remaining)
streaming telemetry... (12s remaining)
waiting for approval...
approved — enrolling identity
tunnel started — device is on the overlay
```

Most enrollments complete in 60–90 seconds.

## Auto-approval vs. manual approval

**Auto-approve** is the normal path. If your device looks legitimate and is coming from a trusted source, the certificate is issued automatically. No human in the loop.

**Manual approval** kicks in when the score is borderline — for example, a device enrolling from an unusual location, or a hostname that doesn't fit the pattern. In that case, an admin reviews the device and clicks approve. This usually takes a few minutes if someone's at a keyboard, longer if not.

You don't get to choose between auto and manual. The system decides based on what your device looks like.

## What the user actually does

For most people, the answer is: nothing. Run the installer, wait, done.

If you want to attribute the device explicitly (for example, "this is Alex's laptop"), the installer accepts a custom ID:

```bash
curl -fsSL https://your-controller.example.com/install | bash -s -- --id alex-laptop
```

That custom ID shows up in the admin's view so they know whose device this is. Optional but recommended.

## What changes after enrollment

Three things start working that didn't before:

1. **A background agent is running.** It's the thing that holds your certificate and routes traffic to `.tango` addresses. You won't see it in normal operation, but a quick service status check will show it.
2. **`.tango` addresses resolve.** Type `git.tango` in your browser, run `ssh your-host`, point your git client at `git@git.tango` — it all just works.
3. **You're now a participant on the network.** You have an identity (see [Identities](identities)), and policies will start applying to your device.

## Is it safe?

Reasonable question. The honest answers:

- **The certificate is tied to your device.** It can't be lifted off and used on another machine without going through the same enrollment process.
- **Telemetry is ephemeral.** It's used to decide and then discarded. Nothing is kept on disk unless a threat is detected.
- **The traffic is encrypted end-to-end.** Even the controllers can't see what's inside your connections.
- **You can revoke it.** If your laptop is lost, an admin can revoke the identity and the certificate stops working immediately.

## Where to next

- [Identities](identities) — what your device's identity actually is
- [Policy](policy) — what the device can and can't reach, and why
- [When things break](when-things-break) — what to do if enrollment fails or stops working
