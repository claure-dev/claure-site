---
title: "10 Months of AI-Assisted Homelab: What Actually Happened"
description: "I gave an AI agent sudo access and told it to build me a homelab. Here's the honest version."
pubDate: 2026-03-18
tags: ["homelab", "AI", "proxmox", "docker"]
readTime: 8
---

I didn't plan to build a homelab. I planned to stop paying Dropbox $12/month. Ten months later I have three VMs, 17 containers, an autonomous sysadmin bot, and a very confused power bill.

The trajectory was: self-host one thing → realize you need DNS → realize you need a reverse proxy → realize you need monitoring for the reverse proxy → realize you need a bot to watch the monitoring. Classic scope creep, except the AI was writing code faster than I could scope-creep.

## What I actually deployed

A $150 Beelink mini PC running Proxmox, split into three VMs:

**infra** runs the network layer — Nginx Proxy Manager, AdGuard, Uptime Kuma, the Omada controller for my access point. Everything that touches DNS or routing lives here because when this VM goes down, it should be obvious.

**brain** is where the AI lives. FrontBot (the sysadmin bot), a Matrix server for communication, Proton Mail Bridge for email access. This is the only VM that costs me sleep when it has problems.

**media** is the entertainment stack — Jellyfin, the *arr suite, SABnzbd. The thing that actually got my wife to stop asking why there's a mini PC in the closet.

## The AI-assisted part

I use Claude Code for almost everything. The workflow isn't "generate code and paste it." It's more like having a very fast junior engineer who has read every man page but has never operated a production system.

Example: I asked Claude to set up AdGuard as the network DNS. It wrote perfect config. The config worked. Then nothing on my network could resolve for 30 seconds every few minutes. Turns out my router had a DNS fallback set to Cloudflare, and Cloudflare was winning the race sometimes and returning public IPs for internal services.

Claude identified the race condition after I described the symptoms. It proposed removing the fallback entirely. I confirmed that was safe (it was — AdGuard itself falls back to upstream resolvers). The fix was one router setting.

That kind of loop — AI writes the thing, I discover the edge case in production, AI diagnoses it — happens daily. The gotcha list is up to 43 entries. Each one is a lesson that now persists across sessions.

## The thing that actually works well

Morning briefs. Every day at 7:30 AM, FrontBot collects data from 10+ sources — weather, all email accounts, calendar, infrastructure health, media activity — and synthesizes it into one message. "Here's what happened overnight, here's what needs attention, here's your day."

I read it in 60 seconds over coffee. I've caught infrastructure issues, appointment conflicts, and important emails before opening a single app.

The email triage alone is worth it. 10 accounts. Most of it is noise. FrontBot reads subject lines and body text, scores urgency, and tells me which 3 out of 50 emails actually need a response.

## What I'd do differently

Less Docker complexity upfront. I containerized everything from day one. Some things (Proton Mail Bridge, the CI runner) would have been simpler as native systemd services. Containers add a layer of networking indirection that burns you when things like IMAP bridges need to bind to specific interfaces.

More snapshot discipline earlier. The first time I broke a VM and had to rebuild from scratch, I started taking Proxmox snapshots before risky changes. The second time, I automated Sunday snapshots. The third time didn't happen.

The backup strategy should have been day one, not month three. Config backup to B2 costs literal cents per month. There's no excuse for not having it from the start.

## The numbers

| Category | Cost |
|----------|------|
| Hardware (one-time) | ~$200 |
| Backblaze B2 | $0.50/month |
| Domain | $12/year |
| Usenet | $7/month |
| Claude Max | $100/month |
| Electricity | ~$5/month |

The Claude subscription is the expensive line. But it's not really a homelab cost — I use it for everything. The homelab rides on it.

The honest ROI: I replaced Netflix ($16), Dropbox ($12), a VPN ($5), and Google storage ($3) with self-hosted alternatives. That's $36/month in services I no longer pay for, against $12.50/month in operating costs. Net savings exist but they're not why I do this.

I do this because debugging a DNS race condition at midnight is more fun than scrolling Twitter. Your mileage may vary.
