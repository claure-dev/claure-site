---
title: "Homelab Backup Strategy That Actually Works"
description: "Three layers: VM snapshots, local vzdump, offsite restic to B2. What I back up, what I don't, and what I learned the hard way."
pubDate: 2026-03-28
tags: ["homelab", "backup", "proxmox"]
readTime: 5
---

My backup strategy has three layers because one layer failed and I learned my lesson.

## Layer 1: Proxmox VM snapshots

Every Sunday at 3 AM, Proxmox takes snapshots of all three VMs. Staggered: VM 100 at 3:00, 101 at 3:15, 102 at 3:30. Keeps 3 snapshots.

Snapshots are fast (seconds) and let you roll back instantly. But they live on the same disk as the VM. If the disk dies, the snapshots die with it.

Use case: "I'm about to do something risky, let me snapshot first." Rollback takes 10 seconds.

## Layer 2: Local vzdump to NFS

Also Sunday, at 2 AM. Proxmox dumps full VM backups over NFS to my desktop. Keeps 3.

This protects against disk failure on the Proxmox host. The backups live on a separate physical machine. Restore takes 15-30 minutes depending on VM size.

The NFS mount was surprisingly tricky to get stable. The initial setup worked, then failed silently when the desktop went to sleep. Wake-on-LAN from the Proxmox host before the backup job runs solved it, but it took two failed backups before I noticed.

## Layer 3: Offsite restic to Backblaze B2

Every day at 5 AM UTC. Each VM runs restic targeting a B2 bucket. Retention: 7 daily, 4 weekly, 3 monthly.

This is the nuclear option — protects against house fire, theft, or me doing something catastrophic to the entire Proxmox host. Restore means provisioning a new host and pulling data from B2, which takes hours, but it's possible.

Cost: about $0.50/month. I back up configs and databases, not media files. Jellyfin's movies and shows are re-downloadable; the *arr apps know what to re-grab if you restore their databases.

## What I actually back up

Config directories, databases, and state files. Specifically:

- Docker compose files and environment configs
- PostgreSQL dumps (Synapse/Matrix)
- SQLite databases (FrontBot traces.db, state.json)
- Ansible playbooks and inventory
- SSL certificates and proxy configs
- Email OAuth tokens and bridge configs

What I don't back up: Docker images (re-pullable), media files (re-downloadable), build artifacts, logs, caches.

The total backup size across all three VMs is about 2 GB. That's why B2 costs cents.

## The restic setup

Each VM has a `~/.restic-env` file with the B2 credentials and repository path:

```bash
export RESTIC_REPOSITORY=b2:bucket-name:vm-name
export RESTIC_PASSWORD=<from password manager>
export B2_ACCOUNT_ID=<key id>
export B2_ACCOUNT_KEY=<application key>
```

The backup script sources this and runs:

```bash
source ~/.restic-env
restic backup /home/adam/docker /home/adam/.frontbot --exclude='*.log'
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 3 --prune
```

Forget + prune after every backup keeps the repository lean. Without it, B2 costs creep up as old snapshots accumulate.

## What I learned

**Test restores.** I verified my backups worked by actually restoring a VM from vzdump. The restore worked but some file permissions were wrong because I'd backed up as a different user. Fixed the backup script, re-tested. Don't assume backups work — prove it.

**Monitor the backups.** For two months, the B2 backup on brain was silently failing because the restic password had been rotated in the password manager but not on the VM. I now have FrontBot check backup freshness as part of the health sweep.

**Back up the backup config.** The restic passwords and B2 keys live in my password manager, not just on the VMs. If I lose all three VMs simultaneously, I can still access B2 to restore.

**Stagger everything.** Running three VM snapshots at the same time tanks I/O on the Proxmox host. Staggering by 15 minutes keeps the system responsive during backups.
