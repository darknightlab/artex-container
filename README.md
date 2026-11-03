# artex-container

Container image for [ARTEX](https://github.com/BaaRai/ARTEX) (AI autonomous penetration
testing system), built and published by GitHub Actions.

This repository carries **no sources**. `.github/workflows/docker-publish.yml` does the
following on every run:

1. checks out upstream `BaaRai/ARTEX` (default `main`) into `context/`,
2. builds the Next.js static export and `linux/amd64` Go binary with upstream's
   own `build.sh` — upstream's `Dockerfile` is runtime-only and expects a pre-built
   binary at `dist/amd64/artex`,
3. builds the amd64 image with the upstream `Dockerfile` and pushes it to GHCR.

Only `linux/amd64` is published, matching our onlinetorrent workflow. The first
multi-arch build failed on arm64 because BBOT's gowitness setup could not find its
Chrome executable; amd64 completed all Dockerfile steps. We do not suppress this
check to publish an incomplete arm64 image.

## Usage

```sh
docker pull ghcr.io/darknightlab/artex-container:latest
```

ARTEX needs a Postgres database; upstream ships a compose file at
<https://github.com/BaaRai/ARTEX/blob/main/docker-compose.yml>. Point the `artex`
service at this image instead of building it:

```yaml
services:
  artex:
    image: ghcr.io/darknightlab/artex-container:latest
    # build: ...   # drop the `build:` block from upstream's compose file
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      ARTEX_PG_DSN: postgres://${POSTGRES_USER:-artex}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB:-artex}?sslmode=disable
    ports:
      - "8787:8787"        # web UI
      - "127.0.0.1:8788:8788" # traffic recording proxy
    volumes:
      - ./data:/app/data
      - ./skills:/app/skills
    restart: unless-stopped
```

Data lives in the `/app/data` volume (SQLite + `jwt.key`). Web UI on `8787`, the
recording proxy on `8788`. See upstream's README for configuration and LLM provider
environment variables.

## Tags

| Tag | Meaning |
| --- | --- |
| `latest` | newest build run on this repository's `main`, including manual runs with a custom upstream ref |
| `sha-<upstream sha>` | identifies the upstream source commit; pin the image digest for immutable deployments |
| `<version>` | upstream release tag, present when the built ref is exactly a tagged release (e.g. `0.4.0`) |
| `edge-YYYYMMDD` | marker for the weekly scheduled rebuild |

Images are signed with [cosign](https://github.com/sigstore/cosign) (keyless, Sigstore).
Extra OCI labels record the upstream repository, commit and version.

## Triggers

- push to `main` (workflow changes)
- pull request (build only, nothing is pushed)
- weekly cron (`0 6 * * 1`) to track upstream `main` even when this repo is idle
- manual `workflow_dispatch` with `upstream_ref` (branch, tag or commit)

```sh
gh workflow run docker-publish.yml -f upstream_ref=v0.4.0
```

## Docker Hub (optional)

Add the `DOCKER_USERNAME` / `DOCKER_PASSWORD` repository secrets and the image is also
pushed to `docker.io/<DOCKER_USERNAME>/artex-container`. Without them only GHCR is used.

## Notes

- The binary is built from upstream `main` by default, or the ref selected manually.
- Source-SHA tags are not byte-for-byte reproducible or immutable: upstream's Dockerfile
  also fetches mutable dependencies. Use `image@sha256:...` to pin an image.
- New GHCR packages may default to private; make the package public in its settings
  if anonymous pulls are desired.
- `sha-<upstream sha>` tags make it possible to pin a build and to tell exactly which
  upstream commit an image contains (`docker inspect` → OCI labels).
- ARTEX is licensed AGPL-3.0; the published image redistributes upstream binaries. See
  upstream's [LICENSE](https://github.com/BaaRai/ARTEX/blob/main/LICENSE).
