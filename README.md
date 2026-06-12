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
3. Runs `docker compose build --no-cache` + `docker compose up -d`
4. Smoke-tests `127.0.0.1:$port` and the public URL
5. Writes a summary to the GitHub Actions step summary

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
across deploys.

## How to use this from your repo

Each app repo has its own `.github/workflows/deploy.yml` that
delegates to this reusable. Example:

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: camster91/ashbi-deploy/.github/workflows/deploy.yml@main
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

The composes + .envs are project-specific and live in each caller's
workflow env. This reusable workflow handles the ssh + docker
mechanics; callers provide the config.

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

Already migrated: `contraction-tracker`.
TODO: every other web app with a Dockerfile.

## Versioning

This workflow uses `@main` (floating ref) for now. When the workflow
matures, pin to tags like `@v1.0.0` and roll forward via PR.

## Layout

- `.github/workflows/deploy.yml` — the reusable workflow (`on:
  workflow_call`)
