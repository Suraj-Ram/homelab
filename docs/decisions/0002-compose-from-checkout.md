# Compose from the checkout

## Context

Live stacks were a copied `docker-compose.yml` under `/mnt/cache/compose/<stack>/`, with the repo holding a second copy under `compose_stacks/`. The git checkout now lives at `/mnt/cache/compose/homelab`. Glances is running from the copy. Its compose file matches `compose_stacks/glances/docker-compose.yml`, and the container's Compose project name is `glances`.

## Options

1. Keep copying each stack's compose file to `/mnt/cache/compose/<stack>/` and run Compose from there.
2. Run Compose with `-f` pointed at the file inside the checkout.

## Choice

Option 2. After `git pull --ff-only`, run:

```bash
docker compose -f /mnt/cache/compose/homelab/compose_stacks/<stack>/docker-compose.yml up -d
```

## Consequences

The checkout is the file Compose reads. There is no second copy to drift. The project name stays the directory name of the compose file, so an existing container with that project name is the same project. Appdata stays in `/mnt/user/appdata`. A copied stack directory under `/mnt/cache/compose/<stack>/` is removed once that stack has been started from the checkout.
