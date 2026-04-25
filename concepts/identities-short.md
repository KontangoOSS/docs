# Identities

An identity is the network's way of saying "I know who you are." Every device, server, and service on Kontango has one — like a passport that proves you belong on the network.

**Three kinds of identities:**

- **Devices** — your laptop, phone, workstation
- **Hosts** — servers that run services
- **Services** — programs that talk to other programs (a CI runner reading from the secrets vault, for example)

Each one has a name, a certificate, and a set of "role" tags that decide what it's allowed to reach.

You don't have to think about cryptography. The agent on your device handles all of it. The only thing you might do is ask an admin to give your device a different role so you can reach a specific service.

Switch to [medium](?v=medium) for the full explanation, or [long](?v=long) for the operations detail.
