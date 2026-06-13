# ashbi-deploy

Reusable deploy workflow for the shared Ashbi VPS (187.77.26.99 as of
2026-06). The default way to deploy a web app from `camster91/*` to
the VPS.

## What it does

`workflow_call` target. When invoked, it:

1. SSHes into the VPS as `$vps_user@$vps_host`
2. Writes a base64-encoded `docker-compose.yml` + `.env` to
   `/opt/projects/$project_name/` (and optionally a paired relay
   project at `/opt/projects/$relay_project_name/`)
3. If `image_tag` is set, rewrites the tag on the first `image:`
   line of the decoded compose (single-service composes only)
4. Runs `docker compose build --no-cache` + `docker compose up -d`
   (or `docker compose pull` + `up -d` in `deploy_mode=pull`)
5. Smoke-tests `127.0.0.1:$port` and the public URL
6. Writes a summary to the GitHub Actions step summary

The pattern matches the `deploy-to-vps.sh` script that ships in
`contraction-tracker/`.

## Why base64

`docker-compose.yml` and `.env` are multi-line. Inlining them in a
workflow's `env:` block means they get nested in a `<<EOF` heredoc
inside an `ssh` call. Earlier drafts of this workflow hit a
YAML/heredoc nesting bug: the inner `cat > .env <<ENV_EOF` heredoc
was treated as literal text by the outer `<<EOF` heredoc, which
corrupted the tracker's `.env` file with the relay's heredoc body.
base64 sidesteps the whole problem — single-line string, no
heredoc nesting.

## Why `--no-cache` but no `--pull`

The host's Docker build namespace can't resolve `registry-1.docker.io`
(worked around in 2026-06 by building from local context).
`--no-cache` forces a clean build so a bad layer doesn't get pinned
across deploys. In `deploy_mode=pull` (see below) `docker compose
pull` is used instead — and the host does have network access to
ghcr.io, so same-tag re-pushes are picked up.

## Inputs

| Input                 | Required | Default       | Purpose                                                                  |
|-----------------------|----------|---------------|--------------------------------------------------------------------------|
| `project_name`        | yes      | —             | Project dir on the VPS (`/opt/projects/$project_name/`)                  |
| `port`                | yes      | —             | Host-side port the container publishes; Caddy reverse-proxies to it      |
| `public_url`          | yes      | —             | Public HTTPS URL used for the post-deploy health check                   |
| `compose_b64`         | yes      | —             | Base64-encoded `docker-compose.yml`                                      |
| `env_b64`             | yes      | —             | Base64-encoded `.env`                                                    |
| `relay_project_name`  | no       | `''`          | Optional paired relay project dir                                        |
| `relay_port`          | no       | `''`          | Optional relay host-side port                                            |
| `relay_compose_b64`   | no       | `''`          | Base64-encoded relay `docker-compose.yml`                                |
| `relay_env_b64`       | no       | `''`          | Base64-encoded relay `.env`                                              |
| `environment_name`    | no       | `production`  | GitHub environment name (for approval gates)                             |
| `deploy_mode`         | no       | `build`       | `build` = `compose build --no-cache` + `up -d`; `pull` = `up -d` only   |
| `pull_always`         | no       | `true`        | In `deploy_mode=pull`, run `docker compose pull` before `up -d`          |
| `image_tag`           | no       | `''`          | Override the tag on the first `image:` line (single-service composes)    |

### When to use `deploy_mode=pull`

Use `pull` when your compose has no `build:` block — the image is
built and pushed to a registry (typically ghcr.io) by a separate
workflow like `build-and-push.yml`. The host only needs to pull
and run. Lull and lull-relay are the canonical examples.

`build` is the right default for apps that still build from source
on the host (contraction-tracker, the original use case).

### When to use `image_tag`

Use it when you want to pin a specific SHA or override the tag
baked into `compose_b64` for a particular deploy. The workflow
replaces the tag on the FIRST `image:` line in the decoded
compose and refuses to run on multi-service composes (for those,
bake the tag into `compose_b64`).

Canonical use case: animal-farts is currently deployed via
`docker run` with the image pinned to a SHA. To migrate to the
reusable without losing the SHA pin, the caller passes
`image_tag: ea83e36` (or whatever the current SHA is). The
compose in `compose_b64` has a placeholder tag like
`camster91/animal-farts:latest`, and the reusable rewrites it
to the SHA on every deploy.

`image_tag` works in both `deploy_mode=build` and
`deploy_mode=pull`. In `build` mode, the rewritten tag is what
gets `docker compose build`'d; in `pull` mode, it's what gets
pulled.

## How to use this from your repo

Each app repo has its own `.github/workflows/deploy.yml` that
delegates to this reusable. Minimal example (build-from-source on
the host, the original pattern):

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: camster91/ashbi-deploy/.github/workflows/deploy.yml@v0.2.0
    with:
      project_name: my-app
      port: "3050"
      public_url: https://my-app.ashbi.ca
      compose_b64: <base64 of docker-compose.yml>
      env_b64: <base64 of .env>
    secrets:
      vps_ssh_key: ${{ secrets.VPS_SSH_KEY }}
      vps_host:    ${{ secrets.VPS_HOST }}
      vps_user:    ${{ secrets.VPS_USER }}
```

Pull-only example (image is built and pushed by
`build-and-push.yml`, host just pulls and runs — the lull /
lull-relay / animal-farts pattern):

```yaml
    uses: camster91/ashbi-deploy/.github/workflows/deploy.yml@v0.2.0
    with:
      project_name: animal-farts
      port: "3015"
      public_url: https://animal-farts.ashbi.ca
      deploy_mode: pull
      pull_always: true
      image_tag: ea83e36        # pin to a specific build SHA
      compose_b64: <base64 of docker-compose.yml>
      env_b64: <base64 of .env>
    secrets:
      vps_ssh_key: ${{ secrets.VPS_SSH_KEY }}
      vps_host:    ${{ secrets.VPS_HOST }}
      vps_user:    ${{ secrets.VPS_USER }}
```

The composes + .envs are project-specific and live in each caller's
workflow env. This reusable workflow handles the ssh + docker
mechanics; callers provide the config.

## Pinning the reusable version

Callers should pin to a specific tag (e.g. `@v0.2.0`) rather than
floating `@main`. New tags are cut when the input shape or the
on-host behavior changes in a non-additive way. Minor additions
or bug fixes that don't change the input surface land on the
existing tag via a new commit.

## Required GitHub secrets (per calling repo)

| Secret        | What                                    |
|---------------|-----------------------------------------|
| `VPS_SSH_KEY` | SSH private key, root@$vps_host        |
| `VPS_HOST`    | VPS hostname or IP (e.g. "187.77.26.99") |
| `VPS_USER`    | SSH user (usually "root")               |

## Required GitHub environment

The reusable workflow's deploy job uses
`environment: ${{ inputs.environment_name }}` (default `production`).
Configure the `production` environment in the calling repo's
Settings → Environments to gate the deploy with manual approval if
desired.

## Migration path for existing apps

1. Add a tiny `deploy.yml` in the calling repo that calls this reusable
2. Base64-encode the existing `docker-compose.yml` + `.env` from the
   VPS and paste into the caller's `env:` block
3. Add the `VPS_*` secrets to the calling repo
4. Delete the old `deploy.yml` once the new one is verified

Already migrated: `contraction-tracker` (build mode), `lull` and
`lull-relay` (pull mode, pinned at v0.1.0).
TODO: every other web app with a Dockerfile.

## Versioning

This workflow is tagged. Pin callers to a specific tag (e.g.
`@v0.2.0`). Floating `@main` is supported for one-off tests but
not recommended for production deploys.

- `v0.1.0` — initial release. `docker compose build --no-cache` +
  `up -d`. Build-from-source-on-host pattern.
- `v0.2.0` — adds `deploy_mode` (build/pull), `pull_always`, and
  `image_tag` inputs. Additive: existing callers continue to work
  unchanged. New pull-only callers (lull, lull-relay,
  animal-farts) can skip the build step.

## Layout

- `.github/workflows/deploy.yml` — the reusable workflow (`on:
  workflow_call`)
