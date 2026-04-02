---
title: "AI Email Triage Across 10 Accounts"
description: "How FrontBot reads, scores, and auto-archives email from 10 IMAP accounts — catching what matters, ignoring what doesn't."
pubDate: 2026-04-01
tags: ["AI", "email", "automation", "python"]
readTime: 6
---

I have 10 email accounts. Personal, work, domains, aliases. Across all of them, I get maybe 50 emails a day. Three or four actually need a response. The rest is noise.

FrontBot reads them all every two hours and tells me which ones matter.

## The setup

All 10 accounts are accessible via IMAP through Himalaya CLI running on brain. Himalaya handles the auth complexity — Gmail uses OAuth2, Proton goes through Bridge, the domain emails use standard IMAP.

The triage loop:

1. Fetch envelopes since last check (Himalaya tracks high-water marks)
2. For anything that looks potentially important, read the full body
3. Score each email: urgent / actionable / informational / noise
4. Auto-archive obvious noise
5. Report flagged items in the morning brief

## The scoring

The scoring prompt is tight. Claude gets the sender, subject, and first 500 characters of body. It returns one of four levels:

- **Urgent:** needs response within hours. Account security, time-sensitive business, system alerts.
- **Actionable:** needs response within a day or two. Client questions, renewals, things with deadlines.
- **Informational:** worth knowing but no action needed. Shipping notifications, receipts, account updates.
- **Noise:** auto-archive candidates. Marketing, newsletters I don't read, duplicate notifications.

The scoring is conservative — it flags borderline cases as informational rather than noise. I'd rather see a false positive in the brief than miss something.

## What gets auto-archived

FrontBot auto-archives emails that match patterns I've confirmed as noise:

- Marketing emails from services I use but don't read updates from
- Duplicate delivery notifications (the third "your package is on the way" email)
- Social media digests
- Generic newsletters I subscribed to and never unsubscribed from

The auto-archive list started empty. Each time I told FrontBot "yes, archive that category," it added the pattern. After a few weeks, about 60% of incoming email gets auto-archived without me seeing it.

## The edge cases

**Proton aliases.** Proton's SimpleLogin aliases route to a single IMAP account. An email to a @passmail.net address shows up in the `proton` account. FrontBot needs to know which account to check for which alias. This is a config mapping, not smart detection.

**OAuth token refresh.** Google's OAuth tokens expire. Himalaya handles refresh tokens, but "testing" app status means tokens only last 7 days. A daily cron job forces a refresh so the token never actually expires during a triage run.

**Rate limiting.** Hitting 10 IMAP servers in rapid succession sometimes triggers rate limits, especially on Gmail. The triage loop has a 2-second delay between accounts. Boring but necessary.

**Shipping detection.** Some emails that look like noise (marketing from a retailer) actually contain tracking numbers for things I ordered. FrontBot reads the body and checks for tracking number patterns. If found, the email gets bumped from noise to informational and the tracking number gets extracted.

## What it catches

In a typical week:

- 3-4 emails that genuinely need a response
- 5-10 that are worth knowing about
- ~300 that get auto-archived or marked as noise

The brief consolidates the flagged ones into a two-line summary each. On a light day, the email section of the brief says "47 processed, nothing flagged." That's the best possible outcome — it means nothing needs my attention and I don't have to think about email.

## The honest limitations

It's not a replacement for actually checking email. FrontBot reads text and scores urgency. It doesn't understand context — it doesn't know that an email from a specific person about a specific project is more important than the subject line suggests.

It misses things that require visual context — emails with important information in images or PDFs. It reads text only.

And the auto-archive is aggressive by design. Once a month I spot-check the archive folders to make sure nothing important got buried. So far: one false positive in three months (a "your subscription is expiring" email that I actually needed to act on). I added that sender to the exception list.

The value isn't perfection. It's that I check email once a day instead of five times, and I still catch everything that matters.
