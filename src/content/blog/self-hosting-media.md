---
title: "Replacing Netflix with Jellyfin"
description: "The media stack that got my wife to stop questioning the mini PC in the closet."
pubDate: 2026-03-24
tags: ["homelab", "media", "docker", "self-hosting"]
readTime: 5
---

The killer app for any homelab is media. Not because it's technically interesting — it's one of the simpler things to set up — but because it's the thing that makes your household actually benefit from the homelab existing.

My wife tolerated the mini PC in the closet. She appreciated it after Jellyfin replaced Netflix.

## The stack

All on one VM (media, 192.168.8.218):

- **Jellyfin** — the media server. Streams to any device on the network. Apps for Roku, iOS, Android, browser.
- **Radarr** — manages movies. You tell it what you want, it finds and grabs it.
- **Sonarr** — same thing, for TV shows. Monitors for new episodes and grabs them automatically.
- **Prowlarr** — indexer manager. Configures search sources for Radarr and Sonarr.
- **SABnzbd** — download client. Connects to Usenet via SSL.
- **Jellyseerr** — request UI. This is what non-technical household members actually use.

Jellyseerr is the user-facing piece. It looks like Netflix's browse page. You find something, click request, and it shows up in Jellyfin within minutes. My wife uses it without knowing what Radarr is.

## What it costs

Usenet access: $7/month. That's the only recurring cost. The mini PC was already bought for other reasons. Storage is internal to the VM — I'm not running a NAS for media because everything is re-downloadable if the disk fails.

Compare to: Netflix ($16) + Hulu ($13) + Disney+ ($8) = $37/month. And my library has everything, searchable, no ads, no "leaving in 3 days" notices.

## Setup gotchas

**Jellyseerr needs the setup wizard.** You can't configure it entirely via API or config files. The initial setup requires completing a web UI wizard. This broke my Ansible-everything approach — the playbook deploys the container but the first-run config is manual.

**SABnzbd direct SSL.** Use direct SSL connections to your Usenet provider (port 563), not the unencrypted port with STARTTLS. Direct SSL is more reliable and your ISP can't see the traffic.

**Stuck queue items.** SABnzbd occasionally gets items stuck in a "downloading" state with 0 speed. FrontBot checks for this every 6 hours and retries or deletes them. Before the automation, I'd discover stuck downloads days later.

**Don't back up media files.** My backup strategy explicitly excludes media content. It's re-downloadable. Backing up 2TB of movies to B2 would cost $10/month. Backing up the Radarr/Sonarr databases (which know what to re-download) costs nothing.

## The automation layer

FrontBot monitors the media stack as part of its health checks:

- Checks SABnzbd queue for stuck items
- Monitors disk usage on the media VM
- Reports new additions in the morning brief ("2 movies, 1 show grabbed overnight")
- Cleans up stale items from the download queue

The brief integration is nice. "Your show has 3 new episodes as of this morning" beats opening an app to check.

## Is this legal?

Usenet access is legal. What you download is your responsibility. I use it for content I would otherwise pay for through streaming services. Your ethical framework may differ.

## Would I recommend it?

If anyone in your household watches streaming content: yes. The time investment is a few hours for initial setup (including the inevitable "why isn't Prowlarr finding anything" debugging session). After that, it runs itself.

The moment my wife requested a movie through Jellyseerr and watched it 10 minutes later, the homelab justified its existence.
