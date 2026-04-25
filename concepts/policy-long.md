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

## How policies are written

If you operate the platform, here's what a policy actually looks like under the hood. There are two policy types per service:

- **Dial policy** — "identities with role X may connect to me"
- **Bind policy** — "identities with role X may host me as a service endpoint"

A typical service registration creates both. For `git.tango`:

```yaml
# Dial policy — clients
identityRoles: ["#stage-2", "#stage-3"]
serviceRoles: ["@git.tango"]

# Bind policy — host
identityRoles: ["@forgejo-host"]
serviceRoles: ["@git.tango"]
```

That says: any identity with `#stage-2` or higher can dial git.tango; only the `forgejo-host` identity can register as the binding endpoint. Both are evaluated at connection time, not just at registration.

Role attribute strings can be either explicit identity references (`@some-identity`) or attribute tags (`#tag-name`). Tags are the usual choice because they're how you scale — adding a role to a tag automatically grants that role to every identity carrying the tag.

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

## Promotion mechanics

How does a device move from one stage to the next? Two paths:

- **Manual promotion** — an admin reviews the device and adds the next stage tag. Effective immediately on the next connection.
- **Automated promotion** — a rule in the platform's decision engine watches for criteria (the device's been online N days, it's running a current OS, it's never failed an auth check, etc) and promotes automatically. This is opt-in per deployment.

Demotion works the same way, in reverse. If a device fails compliance, exhibits suspicious behavior, or is just being decommissioned, the admin (or the engine) removes the higher stage tags and the device's access narrows automatically.

## "I'm enrolled but I can't reach git.tango"

This is the most common policy question. The likely causes, in order of frequency:

1. **You're at a stage that doesn't include `git.tango`.** Stage 0 and Stage 1 don't normally have it; Stage 2 does. Ask to be promoted to Stage 2.
2. **The service's policy was tightened.** Sometimes a service is restricted to a specific role and your device isn't in that role yet. Ask the admin to add the role.
3. **The agent on your device isn't running.** Not actually a policy problem — a quick service status check will tell you. See [When things break](when-things-break).

In all three cases, the fix is the same: tell the admin what you can't reach. They'll check both your roles and the service's policy and either fix it or explain why you shouldn't have access.

## Why default-deny matters

Kontango policies start from "no" and add explicit "yes" rules. If a service has no policy granting your roles access, you're denied. There's no way to "accidentally" reach a service because of a misconfigured firewall — to be reachable, a service must have an explicit rule, and the rule must mention roles you actually have.

This is why a brand-new enrollment often has very limited access. There aren't many policies that grant `#stage-0`, by design. As your stage goes up, more policies start including you, and more of the network opens up.

## Beyond stages — task-scoped roles

Stages aren't the only roles. You'll also see purpose-specific tags:

- `#web-clients` — can dial HTTP services
- `#ssh-clients` — can dial SSH services
- `#nats-clients` — can publish to the message bus
- `#admin-users` — can reach administrative services
- `#host-<something>` — can host the named service

These compose with stages. A typical CI runner might have `#stage-2`, `#nats-clients`, `#host-ci-runner` — three independent permissions. Removing any one narrows access; adding any one widens it.

This is what lets policy scale to dozens of services without explosion. Each service's policy mentions only the roles it cares about, and roles are reused across services.

## Where to next

- [Services](services) — the things policies grant or deny access to
- [Identities](identities) — the things policies are written about
- [When things break](when-things-break) — distinguishing policy denials from network failures
