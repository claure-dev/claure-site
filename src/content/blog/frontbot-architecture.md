---
title: "FrontBot: An Always-On AI Operator for My Homelab"
description: "How I built a Python service that monitors infrastructure, triages email, and manages dev sessions — running 24/7 on a $150 mini PC."
pubDate: 2026-03-22
tags: ["AI", "python", "automation", "homelab"]
readTime: 8
---

FrontBot started as a cron job that checked if my Docker containers were running. It's now a 6,000-line Python service with 20+ data providers, a scheduler, and the ability to restart things autonomously when they break.

The architecture is simple and that's the point. No Kubernetes. No message queues. One Python process, one SQLite database, one Matrix bot for communication.

## The core loop

FrontBot runs as a systemd service on brain (VM 101). It has a scheduler that fires tasks on intervals:

- **Every 30 minutes:** check calendar, send reminders
- **Every 2 hours:** triage email across 10 accounts, groom backlog
- **Every 6 hours:** full health check of all hosts, media queue cleanup, anomaly orchestration
- **Daily at 7:30 AM:** morning brief (the big one)
- **Weekly Sunday 8 AM:** financial + ops + media summary

Each task calls a provider. Every provider follows the same contract:

```python
@dataclass
class HealthReport:
    success: bool = False
    error: str = ""
    def summary(self) -> str: ...

async def gather() -> HealthReport:
    """Always returns a report. Never raises."""
```

That last line matters. A provider that throws an exception breaks the scheduler. A provider that returns `success=False` gets noted in the brief and life goes on. This pattern survived 10 months without a single scheduler crash.

## The morning brief

The brief is the killer feature. It collects reports from every provider, then feeds them to Claude for synthesis. The prompt is structured: here's the raw data, give me a concise summary organized by priority.

The output looks like a staff meeting briefing:

> **Infrastructure:** All hosts green. Disk on infra at 71%, trend stable.
> **Email:** 52 processed, 3 flagged — domain renewal notice (action: verify auto-renew), client question (action: reply today), shipping notification (info only).
> **Calendar:** Sprint planning at 10am (Teams link), dentist at 3pm.
> **Media:** 2 shows grabbed overnight, SABnzbd healthy.

I read this at 7:31 AM with coffee. It replaces checking 6 different apps.

## The email triage

This is the provider I'm most proud of. FrontBot has access to 10 email accounts via Himalaya CLI (IMAP). Every 2 hours it:

1. Fetches new envelopes since the last check
2. Reads full body text for anything that looks actionable
3. Scores each email: urgent / actionable / informational / noise
4. Auto-archives obvious noise (newsletters I never read, marketing, duplicate notifications)
5. Flags anything urgent in the brief and DMs me on Matrix

The scoring uses Claude with a tight prompt: "Given this email subject and first 500 chars of body, classify urgency." It's right about 95% of the time. The 5% it misses are edge cases — emails that look like marketing but contain account security notices.

## Health checks and remediation

The health provider SSHes into every host, collects CPU/memory/disk/container status, and runs it through analysis. If it finds a problem it knows how to fix, it fixes it.

Current auto-remediation scope:
- Restart crashed Docker containers
- Clear Docker build cache when disk gets high
- Restart stuck media queue items

That's intentionally limited. Network config, firewall rules, VM operations — those require my approval. The authority boundary is explicit in the code, not implicit in "the AI seems smart enough."

## The Operating Surface

FrontBot's memory is a SQLite database with four tables:

**traces** — every observation, event, signal. A firehose of "things that happened." FTS5 indexed so I can search by keyword.

**items** — dev tasks and decisions. These are the things someone needs to do or decide. Each has a severity, an owner, a status.

**sessions** — dev work tracking. When I start a Claude Code session, FrontBot knows what I'm working on. When I close it, FrontBot reviews the git log and writes up what changed.

**projects** — what we're building and where focus should go.

The session review is the continuity mechanism. I work in 2-4 hour Claude Code sessions. Between sessions, context is lost. FrontBot bridges that gap: it reads the git log, checks which items got resolved, and proposes what to work on next.

## What breaks

SQLite FTS5 triggers can corrupt on bulk updates. I learned this the hard way — 84 rows updated in one transaction fired 84 FTS delete+insert operations and corrupted the index. The fix was dropping the trigger before bulk operations and recreating it after.

Matrix rate limiting. Synapse (the Matrix server) aggressively rate-limits login attempts. FrontBot logs in once at startup and caches the token. If it logs in more than 3-4 times in a short window, it gets locked out for minutes.

Email OAuth token expiry. Google's OAuth tokens last 7 days for "testing" apps. The refresh flow works but occasionally hangs. A cron job refreshes tokens daily as a safety net.

## Would I recommend this?

If you run a homelab and you use Claude Code: yes, with caveats.

The caveats: it took weeks to build. It requires ongoing maintenance. You need to be comfortable with Python, SSH, and systemd. It's not a product — it's a tool I built for myself.

The payoff: I spend less time checking on infrastructure and more time building things. The morning brief alone saves me 15 minutes a day. The email triage saves more. The session continuity means I can pick up complex work without re-reading everything.

FrontBot is open source at [github.com/claure-lab/frontbot](https://github.com/claure-lab/frontbot).
