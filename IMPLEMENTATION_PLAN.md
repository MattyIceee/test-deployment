# Implementation Plan — Homelab Deployment Repo + Renovate Upgrade PR Demo

> Handoff doc. You are implementing this in a repo that is currently **empty** (git initialized, zero
> commits). Everything below is the full spec — no prior context needed.

## Context

`e:\GitHub\test-deployment` is an empty git repo wired to the public GitHub remote
`MattyIceee/test-deployment`. The goal is to build an example "homelab deployment database" — a
catalog of Docker Compose stacks for popular self-hosted apps — where every image is **deliberately
pinned to a stale 2022/2023 version that has a well-known breaking upgrade path**. Renovate is then
deployed against the repo and run to open a batch of upgrade PRs, so we can observe how it surfaces
major-version bumps that would genuinely break a real homelab.

The point is the *upgrade* signal, not running the services. Stacks are declarative only: valid,
parseable Compose that Renovate reads correctly. No volumes, healthchecks, secrets, or `.env` files,
and `docker compose up` is explicitly **not** a goal.

Confirmed available on this machine: `gh` 2.96.0 authed as `MattyIceee` with `repo` + `workflow`
scopes, Docker 28.3.3, Node 22.16, npx 11.4. Shell is PowerShell on Windows 11.

## Target repo layout

```
test-deployment/
├── .gitignore
├── README.md
├── renovate.json
├── .github/workflows/renovate.yml
└── stacks/
    ├── nextcloud/docker-compose.yml    # nextcloud:25.0.0, mariadb:10.6.11
    ├── monitoring/docker-compose.yml   # grafana/grafana:9.3.0
    ├── edge/docker-compose.yml         # traefik:v2.9.6
    ├── git/docker-compose.yml          # gitea/gitea:1.19.0
    └── data/docker-compose.yml         # postgres:14.5
```

Renovate's `docker-compose` manager default file pattern is
`/(^|/)(?:docker-)?compose[^/]*\.ya?ml$/`, which matches these nested paths with no custom
`managerFilePatterns` needed.

## The six pinned images and why each one breaks

Pins are full semver (not floating `10.6` / `v2.9`) so Renovate emits clean major-vs-minor PRs.

| Image | Pin | Known breaking change on upgrade |
|---|---|---|
| `nextcloud` | `25.0.0` | Nextcloud **refuses to skip major versions**. 25 → 27+ hard-fails; you must step 25→26→27→… one major at a time. Renovate proposes direct jumps that are guaranteed to break. |
| `mariadb` | `10.6.11` | 10.6 → 11.x drops the `mysql_*` wrapper scripts/symlinks (`mariadb-*` only), changes the `utf8` alias to `utf8mb3`, and shifts default collation. Data dir is not auto-upgraded — needs `mariadb-upgrade`. |
| `grafana/grafana` | `9.3.0` | 9 → 10 flips to unified alerting and deprecates AngularJS plugins; 10 → 11 **removes AngularJS support entirely** and drops the legacy alerting engine, breaking many panels and datasources. |
| `traefik` | `v2.9.6` | v2 → v3 is the canonical rewrite: router rule matcher syntax changed, `IPWhiteList` → `IPAllowList`, middleware and provider config keys renamed, `--experimental` flags removed. Requires the official migration tool. |
| `gitea/gitea` | `1.19.0` | 1.19 → 1.20 renames/removes `app.ini` keys (`[server]` SSH and reverse-proxy settings) and drops older DB support; 1.21/1.22 rename more keys. DB migrations are **one-way — no downgrade**. |
| `postgres` | `14.5` | The data directory is major-version-specific. The official image **refuses to start** on `PGDATA` initialized by an older major ("database files are incompatible with server"). Needs `pg_upgrade` or dump/restore. |

**Before committing, verify each tag still exists on the registry** (cheap and prevents a dead pin):

```powershell
'nextcloud:25.0.0','mariadb:10.6.11','grafana/grafana:9.3.0','traefik:v2.9.6','gitea/gitea:1.19.0','postgres:14.5' |
  ForEach-Object { docker manifest inspect $_ *> $null; "$_ -> $(if ($?) {'OK'} else {'MISSING'})" }
```

If a tag is MISSING, substitute the nearest existing patch release **in the same minor line** (e.g.
`mariadb:10.6.12`). Do not bump the minor or major — the staleness is the whole point.

## Compose file shape

Declarative and minimal. No `version:` key (obsolete in Compose v2). No `${VAR}` interpolation, so no
`.env` is required and `docker compose config` stays warning-free. Host ports are kept distinct
across stacks purely so the files read as a coherent catalog.

```yaml
# stacks/monitoring/docker-compose.yml
services:
  grafana:
    image: grafana/grafana:9.3.0
    container_name: grafana
    ports:
      - "3000:3000"
    restart: unless-stopped
```

Same pattern for every stack. `stacks/nextcloud/docker-compose.yml` holds two services (`nextcloud`,
`mariadb`) so one stack exercises a multi-image PR surface.

## `renovate.json`

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended", ":dependencyDashboard"],
  "schedule": ["at any time"],
  "prHourlyLimit": 0,
  "prConcurrentLimit": 0,
  "branchConcurrentLimit": 0,
  "separateMajorMinor": true,
  "separateMultipleMajor": true,
  "labels": ["renovate"],
  "packageRules": [
    {
      "matchUpdateTypes": ["major"],
      "addLabels": ["breaking-change"],
      "prBodyNotes": [
        ":warning: Major version bump — check this project's upgrade notes before merging. Homelab majors frequently require manual data or config migration."
      ]
    },
    {
      "matchUpdateTypes": ["minor", "patch"],
      "addLabels": ["non-breaking"]
    }
  ]
}
```

Notes:

- The top-level `prHourlyLimit` / `prConcurrentLimit` overrides are **required** — `config:recommended`
  pulls in `:prHourlyLimit2`, which would otherwise throttle to 2 PRs/hour and hide the demo.
- `separateMultipleMajor: true` opens a PR per intermediate major (nextcloud 25 → 26, → 27, → 28, …),
  which is the maximum-PR behavior we want here. Expect roughly **20–30 PRs total**. If that reads as
  too noisy, set it to `false` for exactly one major PR per image.
- Committing `renovate.json` on the default branch marks the repo as already onboarded, so Renovate
  will not open a "Configure Renovate" PR.
- Labels are auto-created by the GitHub API on first use; pre-creating them via `gh label create` is
  optional cosmetics only.

## `.github/workflows/renovate.yml`

Self-hosted Renovate with a manual trigger plus a weekly schedule.

```yaml
name: Renovate

on:
  workflow_dispatch:
    inputs:
      logLevel:
        description: Log level
        type: choice
        options: [info, debug]
        default: info
      dryRun:
        description: Dry run (no PRs created)
        type: boolean
        default: false
  schedule:
    - cron: "0 3 * * 1"

permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  renovate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: renovatebot/github-action@v43
        with:
          token: ${{ secrets.RENOVATE_TOKEN || secrets.GITHUB_TOKEN }}
        env:
          RENOVATE_PLATFORM: github
          RENOVATE_REPOSITORIES: ${{ github.repository }}
          RENOVATE_ONBOARDING: "false"
          RENOVATE_REQUIRE_CONFIG: optional
          LOG_LEVEL: ${{ inputs.logLevel || 'info' }}
          RENOVATE_DRY_RUN: ${{ inputs.dryRun && 'full' || '' }}
```

- Check the current major of `renovatebot/github-action` on the Marketplace rather than trusting
  `v43` blindly; same for `actions/checkout`.
- The `secrets.RENOVATE_TOKEN || secrets.GITHUB_TOKEN` fallback means the workflow runs with **zero
  secret setup**. The `permissions:` block above is what makes the `GITHUB_TOKEN` path work.
- Caveat worth putting in the README: PRs opened by `GITHUB_TOKEN` do not trigger other workflows,
  and Renovate cannot fetch cross-repo release notes without a real github.com token. For richer PR
  bodies, add a fine-grained PAT as the `RENOVATE_TOKEN` secret with Contents: RW, Pull requests: RW,
  Issues: RW, Metadata: R.

## `README.md`

This is the "database" surface — make it the primary artifact, not an afterthought:

- What the repo is and the explicit disclaimer that stacks are **declarative examples, not runnable**.
- The pinned-image table above, reproduced with the breaking-change rationale per image.
- How to trigger Renovate both ways (workflow dispatch and the local `npx` run below).
- A short "what to look for in the PRs" section: major vs minor separation, the `breaking-change`
  label, and the Dependency Dashboard issue as the index of everything pending.

## Execution order

1. Create all files listed in the layout above.
2. Verify the six image tags resolve (`docker manifest inspect` loop).
3. Sanity-check Compose syntax:
   ```powershell
   Get-ChildItem stacks -Recurse -Filter docker-compose.yml |
     ForEach-Object { docker compose -f $_.FullName config *> $null; "$($_.FullName) -> $(if ($?) {'OK'} else {'FAIL'})" }
   ```
4. `git add -A`, commit, `git push -u origin main` — this creates the default branch on the remote.
   The repo currently has **no commits**, so `renovate.json` must land on `main` before any Renovate
   run, or Renovate will treat the repo as un-onboarded.
5. Dry run to preview without creating anything:
   ```powershell
   $env:RENOVATE_TOKEN = gh auth token
   $env:RENOVATE_REPOSITORIES = "MattyIceee/test-deployment"
   $env:LOG_LEVEL = "debug"
   $env:RENOVATE_DRY_RUN = "full"
   npx --yes renovate@latest --platform=github --onboarding=false --require-config=optional
   ```
   Confirm the log lists all six images with detected upgrades before proceeding.
6. Real run — drop the dry-run flag and re-run the same command:
   ```powershell
   Remove-Item Env:\RENOVATE_DRY_RUN
   npx --yes renovate@latest --platform=github --onboarding=false --require-config=optional
   ```
   PRs get created.

   If the `gho_` OAuth token from `gh auth token` is rejected, fall back to a classic PAT with `repo`
   scope and set `RENOVATE_TOKEN` to that instead.

## Verification

- `gh pr list --repo MattyIceee/test-deployment --limit 50` — expect ~20–30 open PRs, each titled
  like `Update nextcloud Docker tag to v26` / `chore(deps): update grafana/grafana docker tag`.
- `gh issue list --repo MattyIceee/test-deployment` — expect one **Dependency Dashboard** issue
  indexing every pending update.
- Spot-check a major PR body: it should carry the `breaking-change` label and the `prBodyNotes`
  warning, and the diff should touch exactly one `image:` line.
- Confirm at least one intermediate-major PR exists for nextcloud (e.g. a 25 → 26 alongside a higher
  jump), proving `separateMultipleMajor` took effect.
- Test the committed workflow independently:
  `gh workflow run renovate.yml -f dryRun=true -f logLevel=debug` then `gh run watch`. A clean dry run
  confirms the Actions path works without needing a PAT.

## Out of scope

Running the stacks, volumes/networks/secrets/`.env`, Dockerfiles, GitHub Actions version pins, digest
pinning, and automerge. All six images stay tag-pinned in Compose files only.
