# Maintenance & Operations

Setting up a Kontango deployment is one weekend. Operating it for years is the rest of the time. This final chapter covers the practices that keep your platform working when you're not actively touching it: backups, updates, runbooks, and the operational rhythm that prevents the slow drift toward "I forgot how this works."

## The maintenance mindset

Two truths that frame everything:

1. **The deployment will rot if you don't tend it.** Certs expire. Disks fill. Software gets vulnerable. Your future self forgets the layout. None of these are dramatic; they're slow erosions.

2. **Most of the rot is preventable with simple, written-down practice.** Not fancy automation — checklists, calendars, runbooks, automatic backups, automatic security updates.

The good news: a Kontango deployment is small. A homelab might be 1-3 nodes; a small business 5-10. The maintenance burden is hours per month, not hours per day. As long as you don't let it pile up.

## Backups

What to back up, in priority order:

**Tier 1 — the vault itself:**
- OpenBao data (root keys, intermediate CA, all your secrets, all the cert state)
- This is the most sensitive thing in your stack. Lose it, lose recovery.

**Tier 2 — service data:**
- Postgres databases (Forgejo, Zitadel, Ticketarr, etc.)
- Forgejo repos (yes, even though they're git — recovering 200 repos one-by-one is awful)
- Object storage if you're using MinIO

**Tier 3 — configuration:**
- Caddyfiles
- Systemd units
- The platform's config directory
- All the small files that say "how this deployment is set up"

**Tier 4 — observability data:**
- Grafana dashboards (config, not the metrics themselves)
- Alert rules
- Loki logs if you have a retention requirement

### Where to back up

Two patterns, both are real:

**3-2-1 the easy way:** local snapshot + cloud object storage + offline copy

- Local: Borg, restic, or LVM snapshots, every hour
- Cloud: encrypted upload to Backblaze B2, S3, or another cheap object store, every day
- Offline: external drive, plugged in once a month, kept in a safe or off-site

**3-2-1 the homelab way:** local + a friend's Kontango + a USB drive

- Local snapshots
- Encrypted backup pushed to a friend's Kontango cluster (you both run Kontango, you both back each other up)
- Periodic USB cold storage

For a small business, the cloud option is right. For a homelab, the friend swap is genuinely viable.

### Recovery testing

A backup you've never restored is a hypothesis. Once a quarter, do this:

1. Spin up a fresh node
2. Restore your backup
3. Verify the data is intact
4. Throw the test node away

Yes, it's tedious. Yes, it's the difference between "we have backups" and "we have working backups." Most people who think they have backups don't actually have working backups.

## Updates

Three categories, three different cadences:

**Security updates** — automatic, daily.
- `unattended-upgrades` for OS-level CVEs
- Container image bumps for services with announced CVEs
- Kontango platform CVEs as they ship

**Feature updates** — manual, monthly.
- New service versions, new platform features
- Run in a maintenance window; test before broadcasting
- Read the release notes; assume nothing is fully backward compatible

**Major version updates** — manual, quarterly or per-release.
- Things like Postgres major versions, Zitadel major versions
- Plan in advance; have a rollback path; communicate to users

Don't get stuck on the update treadmill, but don't fall behind by years either. A reasonable steady state is ~2-3 minor versions behind latest for major services, with security patches always current.

## Cert lifecycle

The most common silent failure mode in self-hosted deployments. To avoid it:

- **Public certs:** Caddy auto-renews. The Grafana cert dashboard shows distance-to-expiry. Alert at 14 days; investigate at 30.
- **Internal device certs:** auto-renew at 75% of lifetime. If a device has been offline for too long, it might miss its window. Periodically scan for devices with certs in the last 30 days of life.
- **Internal intermediate CA:** lasts 5 years. Calendar reminder for 6 months before expiry. Plan a rollover.
- **Root CA:** lasts 10 years. Calendar reminder for 12 months before expiry. This is a much bigger event — you'll cycle the intermediate CA at the same time.

## User and identity churn

People come and go. A practice for keeping access lists clean:

- **Joiner:** create Zitadel account, add to appropriate groups, send enrollment instructions
- **Mover:** group memberships updated when role changes
- **Leaver:** disable Zitadel account, revoke any device identities they had on the overlay, remove from project memberships

Calendar a quarterly review: is the list of users in Zitadel still the people who should have access? Same for device identities — is each one still owned by someone who should have it?

## The runbook habit

For every operational thing you might do at 3 AM:

1. The first time you do it, do it manually and learn what's involved
2. The second time, write a runbook — exactly the steps, with the actual commands
3. The third time, follow the runbook exactly and update it where it's wrong
4. The fourth time, consider automating

This is how operational knowledge stops living in one person's head. The runbooks live in your private docs (Kontango's own MkDocs is fine for this).

A starter list of runbooks every Kontango deployment should have:

- Add a new node
- Decommission a node
- Re-enroll a device
- Rotate the OpenBao seal key
- Restore from backup
- Add a new service
- Promote/demote a device's stage
- Investigate a suspicious enrollment

Each runbook is short — half a page to two pages. The discipline is having them written before you need them.

## The "two boxes" rule

A practice that prevents a lot of pain: never have only one of anything that matters.

- One controller? Add a second.
- One backup target? Add a second.
- One admin? Train another.
- One Bao server? Run two.
- One person who knows the password? Two should.

The cost of redundancy is usually less than the cost of the one outage where the single thing was missing. This isn't paranoia, it's professionalism.

## Capacity planning

When to add more nodes / services:

- **CPU sustained > 70% for a week** — add capacity
- **RAM > 80% for a week** — same
- **Disk growth rate predicts < 6 months runway** — order/provision more
- **Service response time creeping up** — investigate; might be capacity, might be code

Don't over-provision. A homelab that runs at 30% utilization is wasted electricity. A small business that runs at 95% utilization is one bad day from a real problem. 50-70% steady state is healthy.

## When something breaks

The framework that's served well:

1. **Stop and breathe.** Don't make it worse with quick reflexes.
2. **Check the obvious.** Is the controller cluster healthy? Are certs valid? Is disk full?
3. **Read the logs.** Real logs, not summaries. The error message is usually the fastest answer.
4. **Bisect.** What changed recently? Last deploy? Last reboot? Last patch?
5. **Ask for help if stuck.** [Discord, support email](../concepts/getting-help). Not heroically alone for hours.

Most production issues turn out to be simple — full disks, expired certs, misconfigured rules. The dramatic failures (corrupted databases, sophisticated attacks) are real but rare.

## The closing word

Running your own infrastructure is a discipline, not a project. The win isn't the install; it's the years of working systems that follow. The maintenance practices in this chapter are what turn a clever weekend project into something you trust with your team's, your business's, or your family's data.

You now know enough to run a real Kontango deployment. The rest is doing it.

Welcome aboard.

---

## Where to next

- [The eight Concepts](../concepts/) — referenceable summaries you can come back to
- [Offerings: Basic / Intermediate / Enterprise](../offerings/) — pre-shaped paths for specific deployment scales
- [Discord](https://discord.gg/CrXWJfwD) — the people who run this stuff, including the people who built it

If you found errors in this guide, the source is on [GitHub](https://github.com/KontangoOSS/docs) and PRs are welcome.
