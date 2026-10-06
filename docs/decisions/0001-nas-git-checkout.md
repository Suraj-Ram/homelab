# NAS git checkout

## Context

This repo is the source of intent for the UnRAID host. The host root filesystem, including `/root`, is a RAM disk. The `compose` share is cache-only on the `cache` pool. Live stacks already live under `/mnt/cache/compose/<stack>/`. `/usr/bin/git` ships with Unraid 7.3.2. The GitHub remote is public HTTPS.

## Options

1. `/mnt/cache/compose/homelab`, talking to the cache disk directly.
2. `/mnt/user/compose/homelab`, the same share through the FUSE share layer.
3. `/mnt/cache/compose` as the git root, so the repo and the live stacks share one directory.
4. `/mnt/compose`, `/root`, or `/boot`.

## Choice

Option 1. Clone to `/mnt/cache/compose/homelab`.

## Consequences

The checkout survives reboot because the share is cache-only, and git bypasses the share layer. Live stacks stay in sibling directories and are not the repo root. `/mnt/compose` does not exist on this host. `/root` is wiped on reboot. `/boot` is the flash drive and is the wrong place for a `.git` directory. Updates on the host are `git pull --ff-only` after the workstation pushes. The pull stays out of `/boot/config/go`.
