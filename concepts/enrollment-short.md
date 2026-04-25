# Enrolling a Device

Enrollment is how your laptop joins the Kontango network. You run one command, wait about a minute, and your device gets a certificate that lets it reach internal services for the next year.

```bash
curl -fsSL https://your-controller.example.com/install | bash
```

**What gets sent:** your hostname, your OS, a hardware ID, and about a minute of CPU/memory readings to prove you're a real computer (not a bot). **What doesn't get sent:** your files, your browsing, your activity. The performance numbers are thrown away after the decision.

**What changes after:** addresses ending in `.tango` start working in your browser. SSH to internal hosts works. A small background agent runs from then on.

Most enrollments auto-approve in 60–90 seconds. If your device is enrolling from an unusual location, an admin reviews it.

Switch to [medium](?v=medium) for the full explanation of what's happening and why, or [long](?v=long) for the cryptography and operations detail.
