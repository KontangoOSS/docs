# Policy

## What policy is

Policy is the rulebook that says **who is allowed to reach what**. It's the difference between "you're on the network" and "you can reach this specific service."

On a normal office wifi, once you're connected you can usually reach anything else on that wifi. The network has a perimeter, and being inside the perimeter is the main check. Kontango is different — there is no perimeter. Being on the network gets you nothing on its own. Each service has its own rules about who can connect to it, and those rules are checked every single time.

Policy is also where the platform stops being one big shared bucket and starts being practical for actual organizations. A small business doesn't want every employee reaching every service; a homelab owner doesn't want a guest device touching the secrets vault. Policies are how you carve out the right access for the right people without standing up separate networks for each one.

## The mental picture

Imagine a venue with several private rooms. Getting through the front door is just the first check — and even that requires showing your card. To enter any specific room you have to be **on that room's list**.

In Kontango terms:

- You at the door = your [device's identity](identities)
- Your card = your certificate, plus the **role tags** attached to your identity
- Each room = a [service](services)
- The room's list = the policy that says which roles are allowed in
- The door staff = the controller, checking every time you knock

## How roles decide what you can reach

Identities carry **role attributes** — tags like `#stage-2`, `#web-clients`, `#admin-users`, `#trusted`, `#host-something`. You don't pick your own roles; the admin assigns them when your device is enrolled or promoted.

Each service has a policy that says something like:

> "Anyone with role `#stage-2` or higher may dial me."

When your device tries to reach the service, the controller compares your roles against that rule. Match → connection allowed. No match → connection denied, instantly, no error from the service itself (because your traffic never even reaches the service host).

You'll generally never see role names directly. But this is the underlying reason something does or doesn't work.

## The trust progression — stages

New devices don't get full access on day one. They start in a low-trust **stage** and earn more access over time. The stages roughly look like:

| Stage | Roles a typical device has | What it can usually reach |
|---|---|---|
| **Stage 0** (just enrolled) | `#stage-0`, `#tunnel`, `#quarantine` | SSH back to itself; minimal else |
| **Stage 1** (basic trust) | `#stage-1`, `#tunnel`, `#web-clients` | Read-only services like dashboards |
| **Stage 2** (trusted) | `#stage-2`, `#tunnel`, `#web-clients` | Most application services — git, secrets, etc |
| **Stage 3** (admin) | `#stage-3`, `#admin-users`, `#ssh-clients` | Everything, including infrastructure |

This protects the network. If a device gets compromised on day one, it can't immediately reach the secrets vault. Stages have to be earned — usually by an admin reviewing the device, sometimes by automated criteria like compliance checks.

If you're stuck at a stage you think is wrong, **ask the admin to promote you**. Usually a 30-second fix.

## "I'm enrolled but I can't reach `git.tango`"

This is the most common policy question. The likely causes, in order of frequency:

1. **You're at a stage that doesn't include `git.tango`.** Stage 0 and Stage 1 don't normally have it; Stage 2 does. Ask to be promoted to Stage 2.
2. **The service's policy was tightened.** Sometimes a service is restricted to a specific role and your device isn't in that role yet. Ask the admin to add the role.
3. **The agent on your device isn't running.** Not actually a policy problem — a quick service status check will tell you. See [When things break](when-things-break).

In all three cases, the fix is the same: tell the admin what you can't reach. They'll check both your roles and the service's policy and either fix it or explain why you shouldn't have access.

## Why default-deny matters

Kontango policies start from "no" and add explicit "yes" rules. If a service has no policy granting your roles access, you're denied. There's no way to "accidentally" reach a service because of a misconfigured firewall — to be reachable, a service must have an explicit rule, and the rule must mention roles you actually have.

This is why a brand-new enrollment often has very limited access. There aren't many policies that grant `#stage-0`, by design. As your stage goes up, more policies start including you, and more of the network opens up.

## Where to next

- [Services](services) — the things policies grant or deny access to
- [Identities](identities) — the things policies are written about
- [When things break](when-things-break) — distinguishing policy denials from network failures
