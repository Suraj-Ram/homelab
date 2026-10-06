# homelab

This repo is the source of intent for the UnRAID NAS. The previous Docker Compose lab is in `archived/` and stays there; DO NOT USE ANYTHING FROM `archived/`  as context or best practices. This is legacy stuff.

## NAS

Connect with `ssh nas`. That is root on `s-nas.local` (UnRAID 7.3.2, Intel i7-4790, 16 GB RAM, no VMs). Recheck `/etc/unraid-version` before assuming the version.

The host root filesystem, including `/root`, is a RAM disk. Files that must survive a reboot belong on a share:

- `/mnt/user/compose` — compose stacks for this setup
- `/mnt/user/appdata` — live container data

Leave every other share alone unless asked. Services run as Docker containers on this host. Before changing one, run `docker ps` over `ssh nas`.

## Repo checkout

Clone this repo on the host to `/mnt/cache/compose/homelab`. That directory is the source tree. See `docs/decisions/0001-nas-git-checkout.md`.

`/mnt` on this host contains `cache`, `disk1`, `user`, and `user0`. There is no `/mnt/compose`. The `compose` share is cache-only, so `/mnt/user/compose` and `/mnt/cache/compose` are the same files. Git uses the cache-disk path. Checkouts through `/mnt/user` go through the FUSE share layer, which mishandles how git reads and writes its own files.

Leave `/mnt/cache/compose` as the live stack directory. Stacks stay in `/mnt/cache/compose/<stack>/` (`glances` is there now). The repo root is `compose_stacks/`, `docs/`, and `archived/`, so the checkout is a subdirectory beside those stacks.

`/usr/bin/git` ships with Unraid and is present after reboot. The remote is public: `https://github.com/Suraj-Ram/homelab.git`. No deploy key.

Edit and commit on the workstation. The owner pushes. On the NAS, update with `git -C /mnt/cache/compose/homelab pull --ff-only`. Keep that pull out of `/boot/config/go`.

If `/mnt/cache/compose/homelab` is missing:

```bash
git clone https://github.com/Suraj-Ram/homelab.git /mnt/cache/compose/homelab
```

`.env` files stay gitignored beside the compose file that needs them. Appdata stays in `/mnt/user/appdata`.

## Host binaries

Unraid loads the OS into RAM, so anything installed under `/usr` or `/root` disappears on reboot. Since Unraid 6.8, files on the boot device cannot be given execute permission, so a binary on the flash drive will not run from `/boot`. See [Securing your boot device](https://docs.unraid.net/unraid-os/system-administration/secure-your-server/secure-your-boot-drive/).

Install a standalone binary in three steps:

1. Download it to `/boot/config/binaries/` on the flash drive.
2. Make `/boot/config/go` copy each file from that directory into `/usr/local/bin` and mark the copy executable. Keep `/usr/local/sbin/emhttp` as the last line, with no trailing `&`, and put the loop before it. `/etc/rc.d/rc.local` strips a trailing `&` from that line and then runs `go`. `emhttp` daemonizes, and `/usr/libexec/unraid/emhttpd` stays running under init for the life of the boot. Do not add `set -e`. A missing directory, a failed `cp`, or a failed `chmod` logs to stderr and continues, so `emhttp` still starts. On this NAS, `/boot/config/go` already restores `/boot/config/binaries/` and copies `docker-compose` into the Docker CLI plugin directory before `emhttp`. Docker Compose v5.6.0 is installed.

```bash
if [ -d /boot/config/binaries ]; then
  for binary in /boot/config/binaries/*; do
    [ -f "$binary" ] || continue
    name="$(basename "$binary")" || continue
    if cp -f "$binary" "/usr/local/bin/$name"; then
      chmod +x "/usr/local/bin/$name" || echo "go: chmod failed: $name" >&2
    else
      echo "go: copy failed: $name" >&2
    fi
  done
else
  echo "go: /boot/config/binaries is missing; skipping binary restore" >&2
fi
```

1. Run that same per-file copy immediately so the binary is on `PATH` before the next reboot. A failure for one file does not stop the remaining files.

Leave the flash copy non-executable. Keep logs and other frequently written files off `/boot/config/binaries/`.

## Working in this repo

- Keep secrets out of git. `.env` files stay untracked. Name variables, not values.
- Do not commit. The owner commits.
- Do not edit `archived/` except to read it.
- When a change chooses between real alternatives, add `docs/decisions/NNNN-title.md` with the context, the options, the choice, and the consequences.

