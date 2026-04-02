---
title: "Gotcha-Driven Development"
description: "I maintain a list of 43 things that have broken my homelab. It's the most valuable file in the repo."
pubDate: 2026-03-30
tags: ["homelab", "AI", "debugging"]
readTime: 5
---

My most-read file isn't a README or architecture doc. It's `gotchas.md` — a numbered list of things that have broken my homelab, how they broke, and how to never break them again.

It's at 43 entries and growing.

## What a gotcha looks like

```markdown
2. **Tailscale accept-routes on LAN devices**: NEVER enable 
   `--accept-routes` on machines already on 192.168.8.0/24. 
   It routes LAN traffic through Tailscale, breaking NFS and 
   direct connectivity. Only enable on remote devices.
```

Short. Specific. Emphatic. The format is: what went wrong, why, and the rule that prevents it from happening again.

Every gotcha started as a production incident. Some were 10-minute fixes. Some were 2-hour debugging sessions. #40 (FTS5 triggers corrupt on bulk UPDATE) cost me an afternoon and a database rebuild.

## Why a list, not documentation

Documentation describes how things work. Gotchas describe how things break.

The difference matters when you're working with AI agents. Claude Code reads `gotchas.md` at the start of every session. It doesn't need to understand my full network topology. It needs to know: "don't touch the router DNS fallback" and "Omada controller must use explicit bind mount paths, not Docker volumes."

These are things the AI would get wrong without the gotcha, because the default behavior is the wrong behavior in my environment. The gotcha list is a set of constraints that override defaults.

## How gotchas accumulate

The pattern is always the same:

1. Something breaks in a way I didn't expect
2. I figure out why
3. I write down the rule in one sentence
4. That rule prevents the same break forever

The discipline is writing it down immediately. Not "I'll remember this." Not "it's obvious once you know." Write it down. The AI doesn't remember. Future-you barely remembers. The gotcha list remembers.

Some of my favorites:

**#1 (Omada bind mounts):** Wrong Docker volume paths nuked my entire Omada controller config. Every access point needed re-adoption. This was the gotcha that started the list.

**#14 (Synapse rate limiting):** Matrix server rate-limits login at 3-4 attempts. FrontBot was logging in on every restart instead of caching the token. Took 20 minutes to debug why the bot was randomly offline.

**#33 (Docker restart corrupts iptables):** Restarting individual containers on infra can break NAT rules for all other containers. The fix is restarting the entire Docker daemon. I learned this at 11 PM when nothing on my network could reach the internet.

**#42 (SQL LIMIT before Python filter):** A query with `LIMIT 50` followed by a Python filter was silently dropping items. The severity sort pushed low-priority items past the limit boundary. This was invisible in testing because the test database was small.

## The compounding effect

Each gotcha makes every future session slightly faster. When I start a Claude Code session and say "set up a new Docker service on infra," Claude already knows about #33 (don't restart individual containers), #4 (source .env for NPM secrets), and #3 (Watchtower needs DOCKER_API_VERSION).

Without the gotcha list, I'd rediscover these constraints. With it, we skip straight to the thing that's actually new.

Over 10 months, that time savings compounds. The first month had the most gotchas per session. Now, most sessions encounter zero previously-documented gotchas. The ones that surface are genuinely new.

## The meta-gotcha

The gotcha list itself has a gotcha: keeping it up to date.

When I refactor something and a gotcha no longer applies, I need to remove it. Stale gotchas are worse than no gotchas — they create false constraints that prevent correct solutions.

FrontBot now syncs the gotcha list as part of its maintenance cycle, flagging entries that reference files or services that no longer exist. Automating the grooming because I know I won't do it manually.

Forty-three gotchas and counting. Every one is a bug I'll never hit twice.
