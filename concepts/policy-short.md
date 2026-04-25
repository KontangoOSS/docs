# Policy

Policy is the rulebook that says **who is allowed to reach what**.

On normal office wifi, once you connect, you can reach everything else on that network. Kontango doesn't work that way. Being on the network is just step 1. **Each service has its own list** of who's allowed in.

Think of it like a venue with private rooms. Getting through the front door is one check. Getting into a specific room is another — your name has to be on that room's list.

**The mechanism:** every device has role tags (like `#stage-2` or `#trusted`). Every service has rules that say which roles can connect. The controller checks both, every time, instantly.

**If you can't reach a service:** ask the admin to give your device the right role. Usually a 30-second fix.

**Why this matters:** if any one device gets compromised, it can only reach what its roles allow — not the whole network. Default-deny by design.

Switch to [medium](?v=medium) for the trust progression detail, or [long](?v=long) for the policy authoring side.
