# The Network

Imagine the internet as a public hallway anyone can walk down. Kontango adds a **private hallway** alongside it. Only your devices have keys. Only you decide what doors are inside.

That's the Kontango network: a private, encrypted layer running alongside the public internet. Your normal browsing still works the same way. The Kontango network is *additional* — it's how your devices reach internal services like `git.tango` or `bao.tango` that nobody else on the internet can see.

**Three things to know:**

1. The network doesn't care where your hardware lives — Pi, desktop, cloud VM, all the same
2. Internal addresses end in `.tango` and only resolve on enrolled devices
3. It's not a VPN; only `.tango` traffic uses the private hallway

Switch to the [medium](?v=medium) version for the full mental model, or [long](?v=long) for the architecture.
