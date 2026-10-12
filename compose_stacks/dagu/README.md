# Dagu

[Dagu](https://docs.dagu.sh/) schedules YAML workflows. This stack runs the Web UI and scheduler in one container. The UI is `https://dagu.surajram.online`.

The first browser visit opens `/setup` to create the admin account. No license key is required for this community install.

Workflow files in `dags/` are mounted read-only. Edit them here, commit, and pull on the NAS. Run history, logs, and the auth secret stay in `/mnt/user/appdata/dagu`.

The host Docker socket is mounted. `dags/prune-unused-images.yaml` uses it to delete images that no container is using, every day at 02:00 America/New_York.

Before the first start, create that directory as Unraid's `nobody:users`:

```bash
mkdir -p /mnt/user/appdata/dagu
chown 99:100 /mnt/user/appdata/dagu
git -C /mnt/cache/compose/homelab pull --ff-only
docker compose -f /mnt/cache/compose/homelab/compose_stacks/dagu/docker-compose.yml up -d
```

`dagu.surajram.online` is covered by the existing wildcard DNS record. See `docs/decisions/0005-dagu-single-node.md`.
