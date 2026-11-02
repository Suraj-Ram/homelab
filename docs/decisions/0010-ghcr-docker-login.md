# GHCR login for private images

`docker login ghcr.io -u YOUR_GITHUB_USERNAME` was run as root with a GitHub PAT scoped to read, write, and delete packages. That added `ghcr.io` to `/boot/config/plugins/dockerMan/config.json`. `/root/.docker` is a symlink to that directory.
