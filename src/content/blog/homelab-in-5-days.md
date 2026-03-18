---
title: "How I Built a Complete Homelab in 10 Months with AI"
description: "Starting from zero to 26 services, 3 VMs, and a fully autonomous sysadmin bot — with Claude doing 95% of the work."
pubDate: 2026-03-18
tags: ["homelab", "automation", "AI", "proxmox", "docker"]
readTime: 12
---

I didn't intend to build a homelab. I wanted to stop paying for services I didn't control.

Ten months later: Proxmox hypervisor running 3 VMs, 17 Docker containers, a self-hosted email stack, a media server replacing Netflix, a full monitoring and alerting pipeline, and an autonomous bot that manages all of it while I sleep.

The part that's hard to explain to most people: Claude Code did 95% of the implementation work. I reviewed, approved, and pressed enter. The AI wrote the Ansible playbooks, debugged the DNS, figured out why containers couldn't reach each other, and wrote the bot that now sends me a morning brief every day.

This is the story of how that happened.

## The starting point

A GL-iNet Flint 2 router, a Beelink mini PC I bought for $150, and a strong opinion that I shouldn't have to pay someone else to run my own data.

The first decision was Proxmox instead of bare Docker on the mini PC. That turned out to be correct — VMs are significantly easier to manage than trying to keep everything on one machine. Snapshots before risky changes. Easy rollback. The ability to blow up VM 100 without touching VM 101.

## How AI-assisted development actually works

The workflow isn't "ask AI, get code, done." It's closer to pair programming where one partner (me) sets direction and reviews output, and the other (Claude) does the research, writes the implementation, and catches things I'd miss.

A concrete example: setting up Nginx Proxy Manager with AdGuard DNS.

The naive approach would be: install NPM, install AdGuard, configure DNS, done. What actually happens: AdGuard needs to be the DNS server for the router, but the router has a fallback DNS that sometimes wins the race. Cloudflare responds faster than AdGuard and returns the wrong IP for internal services. This took two hours to debug and involved packet captures, router config analysis, and understanding how dnsmasq handles dual-server configurations.

Claude worked through this with me. Identified the race condition. Explained why removing the fallback entirely (rather than changing its priority) was the right fix. Wrote the AdGuard config and documented the gotcha for future reference.

That gotcha is now in my auto-memory system. Every future session, Claude knows not to touch the router's DNS fallback. The lesson compounds.

## The infrastructure stack (what ended up running)

Three VMs, all Proxmox:

**infra (VM 100):** Network services. Nginx Proxy Manager, AdGuard Home, Uptime Kuma, Homepage dashboard, Omada controller, Speedtest Tracker, Dozzle. Everything that touches DNS and routing lives here.

**brain (VM 101):** The smart layer. FrontBot, Matrix (Synapse + PostgreSQL), Proton Mail Bridge, 10 email accounts managed via Himalaya CLI. This is where the AI agent lives and runs.

**media (VM 102):** Entertainment. Jellyfin, Radarr, Sonarr, Prowlarr, SABnzbd, Jellyseerr. The whole *arr stack with automated content requests.

Each VM has:
- An Ansible playbook for deployment
- A service documentation file
- Restic backups to Backblaze B2
- Proxmox snapshots on Sundays

## The thing nobody talks about: managing state

The hardest part of running a homelab isn't deploying services — it's knowing what you deployed, why, and how it connects to everything else.

I built a documentation system (the Vault) that lives alongside the code. Every service has a doc. Every decision has a date and reasoning. Every gotcha gets captured and becomes permanent institutional knowledge.

This matters more with AI assistance than it does solo. The AI doesn't remember between sessions. The Vault is the external memory. When I start a new session, Claude reads the HANDOFF.md, pulls the git log, checks the auto-memory, and picks up exactly where we left off.

The documentation discipline that feels like overhead in the moment pays massive dividends when something breaks at 11pm and you need to know exactly what changed.

## FrontBot: the part that surprised me

About four months in, I started thinking about automation beyond "services running." What if the infrastructure could monitor itself? Triage email? Tell me what happened overnight?

FrontBot started as a simple health-check bot. It's now a 10-provider modular system that:

- Checks all hosts every 6 hours, analyzes trends, remediates known issues automatically
- Triages 10 email accounts with AI priority scoring — urgent vs. actionable vs. noise
- Sends a morning brief every day at 7:30 AM: weather, infrastructure health, email summary, upcoming calendar, media activity, finance snapshot
- Watches calendar and DMs me 15 minutes before any appointment
- Monitors the media stack and requests content based on watch history
- Tracks business revenue and notifies on new sales

The brief is the thing I interact with every morning. A structured synthesis of 10+ data sources, delivered as a Matrix message, that tells me everything I need to know in 60 seconds.

Building each provider was a session of work. The architecture compounds: each new provider plugs into the same scheduler, the same brief pipeline, the same insight analysis layer.

## What this actually cost

Infrastructure costs:
- Proxmox + mini PC: ~$200 one-time hardware (VMs are free)
- Backblaze B2: ~$0.50/month for config backups (media files are re-downloadable)
- Porkbun domain (claure.org): ~$12/year
- Usenet access for media: ~$7/month

Development costs:
- Claude Code Max subscription: ~$100/month (zero marginal cost per session)
- Time: ~15-30 minutes per day reviewing and approving changes

The interesting thing about the Claude Max subscription: it enables true zero-marginal-cost AI development. FrontBot makes ~20 Claude API calls per day (briefs, triage, health synthesis). At pay-as-you-go pricing, that would add up. With Max, it's included.

## The honest take on AI-assisted homelab building

It's not magic. The AI makes mistakes. It sometimes suggests architecturally wrong approaches that I have to push back on. It occasionally misses context that changes the answer.

The value is in:

1. **Speed on the boring parts.** Ansible playbooks are tedious to write. Claude writes them correctly in 30 seconds. I review, test, deploy.

2. **Research synthesis.** "How does Proton Mail Bridge handle IMAP authentication in a container?" would take 45 minutes of reading docs and StackOverflow. Claude knows the answer and gives it in 30 seconds, with the relevant gotchas.

3. **Compounding knowledge.** The auto-memory and Vault documentation create a feedback loop. Each session is faster than the last because the AI has more context. Year 2 of this is going to be qualitatively different from year 1.

4. **The 3am problem.** Something breaks at 3am. With AI assistance, you can actually fix it at 3am without a full engineering-brain context load. "Something is wrong with NPM, here's the logs" plus `ssh infra docker logs npm` gets you to root cause in 10 minutes.

The stack is live. The bot is running. If you want the configs, they're in the products section.

---

*Next post: Building FrontBot from scratch — architecture decisions, the provider pattern, and why the scheduler matters more than you'd think.*
