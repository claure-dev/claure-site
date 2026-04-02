---
title: "Running a Two-AI Dev Team"
description: "One agent builds, one agent manages. How I coordinate Claude Code sessions with an always-on project manager."
pubDate: 2026-03-25
tags: ["AI", "workflow", "automation"]
readTime: 6
---

I have two AI agents. One builds things. The other manages the backlog and reviews the work. I'm the product owner who decides what matters and occasionally resolves disputes.

This sounds absurd. It works better than it should.

## The roles

**Desk** is the builder. It's a Claude Code session running on my desktop. Full access to the codebase, SSH to all hosts, MCP tools for everything. When I sit down to work, I start a session, Desk reads the context, and we build things together.

**FrontBot** is the project manager. It runs 24/7 on a separate VM. It doesn't write code. It reviews what Desk built (by reading git log), proposes what to work on next, maintains a prioritized backlog, and bridges context between sessions.

**I** decide what gets built, in what order, and when something is good enough. I resolve ambiguity. I say "no" to things that sound productive but aren't.

## The handoff

The critical moment is when I close a Desk session and FrontBot picks up.

When I close a session, Desk passes the git log and a brief narrative of what happened. FrontBot runs a postmortem: reads the commits, checks which items got resolved, extracts any decisions that were made, and writes a review.

The review proposes next sessions with reasoning: "Based on what was built, here's what makes sense to work on next." It also catches things Desk left incomplete or items that should be closed.

When I start the next session, Desk reads FrontBot's review and picks up with full context. No re-reading code. No "where was I?" The handoff is the product.

## What goes wrong

The handoff isn't perfect. FrontBot sometimes proposes sessions that duplicate existing backlog items. The dedup logic is improving but not bulletproof.

FrontBot's reviews are sometimes wrong about what happened in a session. It reads git log, which is a lossy representation of what actually occurred. Discussions, abandoned approaches, decisions made verbally — those don't show up in commits.

The postmortem runs asynchronously after session close. If I start a new session before it finishes, the review isn't available yet. This is a known race condition that I've mostly solved by not working at 2 AM.

## Why two agents instead of one

The builder and the reviewer shouldn't be the same entity. Desk is optimized for "get things done right now." FrontBot is optimized for "what should we do next and did we miss anything?"

Desk has full tool access and makes changes. FrontBot has read access and makes recommendations. The authority boundary prevents the project manager from accidentally deploying things, and prevents the builder from deciding its own priorities.

There's also a practical reason: Claude Code sessions are ephemeral. They start, they do work, they end. FrontBot is persistent. It wakes up, checks what changed, synthesizes. You need something that outlives individual work sessions.

## The coordination protocol

It's simpler than you'd expect:

1. FrontBot writes planned sessions to the database
2. At `/start`, Desk reads planned sessions and presents them to me
3. I pick one (or redirect)
4. Desk works, commits, closes session
5. FrontBot reviews, proposes next sessions
6. Repeat

The database (SQLite) is the coordination layer. No message passing, no event bus. FrontBot writes. Desk reads. I decide.

Items (tasks and decisions) live in the same database. Both agents can create items. Both can read them. Only I close decisions.

## Is this overkill?

Probably. A notebook and a to-do list would cover 80% of what this system does.

The 20% it handles that a notebook can't: automatic context bridging across sessions, git-log-based work verification, prioritized backlog grooming based on project state, and a morning brief that includes project status alongside infrastructure health.

If you're doing serious work across multiple projects with AI assistance, the context problem is real. Each session starts cold. The handoff mechanism is the thing that makes multi-session projects viable.
