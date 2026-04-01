---
title: Check for duplicate data across mount points after home directory migration
tags: [disk-space, mount-points, migration, duplicates]
verified: 2026-04-01
platform: linux
---

## Problem
Disk at 90% capacity after migrating home directories to a new partition. Old data left behind on the original mount.

## Solution
Compare device IDs and inodes to confirm duplicates, then delete the stale copies:

```bash
# Check if two paths are the same data or copies on different devices
stat -c "%d %i %n" /home/user/dir /mnt/oldhome/user/dir

# Different device numbers = separate copies (safe to delete one)
# Same device + inode = same data (hardlink or bind mount)

# Verify the copy you're keeping is intact before deleting
du -sh /home/user/dir
ls /home/user/dir/important_file

# Then remove the stale copy
sudo rm -rf /mnt/oldhome/user/
```

## What didn't work
Assuming `df` or `du` alone would reveal duplicates. You need `stat` to compare device IDs since `du` just shows sizes which might coincidentally match.

## Context
- Common after migrating `/home` from HDD/RAID to NVMe
- In one case, 444GB of duplicates were silently consuming a 916GB RAID1
- Always check `/mnt/*/`, `/media/*/`, and old mount points for orphaned data
- The `/archive` or backup volumes may be intentional copies -- ask before deleting
