# Homelab Deployment Database (Renovate Upgrade PR Demo)

A catalog of Docker Compose stacks for popular self-hosted apps, where every image is
**deliberately pinned to a stale 2022/2023 version** with a well-known breaking upgrade path.
This repo exists to demonstrate how [Renovate](https://docs.renovatebot.com/) surfaces major-version
bumps that would genuinely break a real homelab.

> **Disclaimer:** The stacks are **declarative examples, not runnable**. There are no volumes,
> healthchecks, secrets, or `.env` files. `docker compose up` is explicitly **not** a goal — the
> point is the upgrade signal, not running the services.

## Stacks

| Stack | Compose file | Images (pinned) |
|---|---|---|
| nextcloud | `stacks/nextcloud/docker-compose.yml` | `nextcloud:25.0.0`, `mariadb:10.6.11` |
| monitoring | `stacks/monitoring/docker-compose.yml` | `grafana/grafana:9.3.0` |
| edge | `stacks/edge/docker-compose.yml` | `traefik:v2.9.6` |
| git | `stacks/git/docker-compose.yml` | `gitea/gitea:1.19.0` |
| data | `stacks/data/docker-compose.yml` | `postgres:14.5` |

## The pinned images and why each upgrade breaks

Pins are full semver (not floating `10.6` / `v2.9`) so Renovate emits clean major-vs-minor PRs.

| Image | Pin | Known breaking change on upgrade |
|---|---|---|
| `nextcloud` | `25.0.0` | Nextcloud **refuses to skip major versions**. 25 → 27+ hard-fails; you must step 25→26→27→… one major at a time. Renovate proposes direct jumps that are guaranteed to break. |
| `mariadb` | `10.6.11` | 10.6 → 11.x drops the `mysql_*` wrapper scripts/symlinks (`mariadb-*` only), changes the `utf8` alias to `utf8mb3`, and shifts default collation. Data dir is not auto-upgraded — needs `mariadb-upgrade`. |
| `grafana/grafana` | `9.3.0` | 9 → 10 flips to unified alerting and deprecates AngularJS plugins; 10 → 11 **removes AngularJS support entirely** and drops the legacy alerting engine, breaking many panels and datasources. |
| `traefik` | `v2.9.6` | v2 → v3 is the canonical rewrite: router rule matcher syntax changed, `IPWhiteList` → `IPAllowList`, middleware and provider config keys renamed, `--experimental` flags removed. Requires the official migration tool. |
| `gitea/gitea` | `1.19.0` | 1.19 → 1.20 renames/removes `app.ini` keys (`[server]` SSH and reverse-proxy settings) and drops older DB support; 1.21/1.22 rename more keys. DB migrations are **one-way — no downgrade**. |
| `postgres` | `14.5` | The data directory is major-version-specific. The official image **refuses to start** on `PGDATA` initialized by an older major ("database files are incompatible with server"). Needs `pg_upgrade` or dump/restore. |

## Running Renovate

### Via GitHub Actions (no secret setup)

Commit this repo to GitHub, then trigger the workflow:

```bash
gh workflow run renovate.yml -f dryRun=true -f logLevel=debug
gh run watch
```

The workflow in `.github/workflows/renovate.yml` uses
`secrets.RENOVATE_TOKEN || secrets.GITHUB_TOKEN` as its fallback, so it runs with **zero secret
setup**. It also runs weekly on Mondays at 03:00 UTC.

### Via local `npx` run

```powershell
$env:RENOVATE_TOKEN = gh auth token
$env:RENOVATE_REPOSITORIES = "MattyIceee/test-deployment"
$env:LOG_LEVEL = "debug"
npx --yes renovate@latest --platform=github --onboarding=false --require-config=optional
```

Add `$env:RENOVATE_DRY_RUN = "full"` for a preview that creates nothing.

## What to look for in the PRs

- **Major vs minor separation.** `separateMajorMinor` and `separateMultipleMajor` are enabled, so
  each image gets a PR per major bump (e.g. nextcloud 25 → 26, → 27, → 28, …) plus separate minor/patch
  PRs. Expect roughly 20–30 open PRs.
- **The `breaking-change` label.** Any major-version PR carries this label plus a body note warning to
  check upgrade notes before merging.
- **The Dependency Dashboard.** An issue titled *Dependency Dashboard* indexes every pending update —
  it's the single overview of the whole upgrade backlog.

## Caveats

- PRs opened by `GITHUB_TOKEN` do not trigger other workflows, and Renovate cannot fetch cross-repo
  release notes without a real github.com token. For richer PR bodies, add a fine-grained PAT as the
  `RENOVATE_TOKEN` secret with **Contents: RW, Pull requests: RW, Issues: RW, Metadata: R**.
