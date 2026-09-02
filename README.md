# Mattermost + Traefik + Let's Encrypt — Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/mattermost-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/mattermost-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Contents

- [Why this stack?](#why-this-stack)
- [Prerequisites](#prerequisites)
- [Getting started](#getting-started)
- [Features](#features)
- [Supply chain trust](#supply-chain-trust)
- [Production checklist](#production-checklist)
- [Backups](#backups)
- [Testing](#testing)
- [Security Notes](#security-notes)
- [About the maintainer](#about-the-maintainer)

This repository deploys **Mattermost Team Edition** behind **Traefik** with automatic **Let's Encrypt TLS**, backed by **PostgreSQL**, with scheduled **backups** (database + file uploads) and companion **restore scripts**. One `docker compose up` away from self-hosted team chat at `https://your-domain`.

📙 Full narrative installation guide on the blog: [heyvaldemar.com/install-mattermost-using-docker-compose/](https://www.heyvaldemar.com/install-mattermost-using-docker-compose/).

## Why this stack?

| Need | This stack | Manual install | Kubernetes | Other compose examples |
|------|-----------|----------------|------------|------------------------|
| Ready to deploy in <10 min | ✅ | ❌ | ✅ if K8s is already running | Often |
| TLS via Let's Encrypt, auto-renewed | ✅ Traefik ACME built-in | Manual certbot | Via cert-manager | Rare |
| PostgreSQL wired with healthchecks | ✅ | Separate install | ✅ | Varies |
| Scheduled DB + uploads backups + pruning | ✅ | Manual cron | External | Rare |
| Restore scripts included | ✅ two scripts | Manual | Manual | Rare |
| Upstream images pinned by `sha256` digest | ✅ | N/A | Depends | Rare |
| Weekly pin-freshness check in CI | ✅ | N/A | Depends | Rare |
| CI-verified deployment on every push | ✅ ping answers | N/A | Varies | Rare |
| Credentials via env (never committed) | ✅ | N/A | K8s Secrets | Often committed plaintext |

Four moving parts (Traefik + Mattermost + Postgres + backups). No Kubernetes prerequisites, no manual certificate management.

## Prerequisites

- **A Linux server** with a public IP. Tested on Ubuntu 22.04 LTS+ and Debian 12+.
- **Docker Engine 24+ and Docker Compose 2.20+.**
- **A domain you control,** with two `A` records pointing at your server's public IP — one for Mattermost, one for the Traefik dashboard. DNS must propagate before deploy.
- **Ports 80 and 443 open** on the server's firewall.
- **~1 GB free RAM and 1 free CPU** for a small team, plus disk for uploads and backups.

## Getting started

```bash
# 1. Clone
git clone https://github.com/heyvaldemar/mattermost-traefik-letsencrypt-docker-compose
cd mattermost-traefik-letsencrypt-docker-compose

# 2. Create the two Docker networks the stack expects
docker network create traefik-network
docker network create mattermost-network

# 3. Copy the environment template and fill in required values
cp .env.example .env
$EDITOR .env
# ^ Required: MATTERMOST_DB_PASSWORD, MATTERMOST_HOSTNAME, MATTERMOST_URL,
#   TRAEFIK_HOSTNAME, TRAEFIK_ACME_EMAIL, TRAEFIK_BASIC_AUTH.

# 4. Deploy
docker compose -f mattermost-traefik-letsencrypt-docker-compose.yml -p mattermost up -d
```

Within a minute or two `https://${MATTERMOST_HOSTNAME}` serves Mattermost with a fresh Let's Encrypt certificate. The first account created becomes the system admin — do it before sharing the URL.

### What success looks like

```bash
# All services healthy:
docker compose -f mattermost-traefik-letsencrypt-docker-compose.yml -p mattermost ps

# The system ping answers:
curl -fsS "https://${MATTERMOST_HOSTNAME}/api/v4/system/ping"
# Expected: {"ActiveSearchBackend":"...","status":"OK"}

# Traefik issued a certificate:
docker compose -p mattermost logs traefik | grep -i "adding certificate"

# First backup lands after BACKUP_INIT_SLEEP (default 30m):
docker compose -p mattermost logs backups | tail -3
```

### Common first-deploy issues

- **Cert issuance fails.** DNS hasn't propagated or port 80 isn't reachable from the internet.
- **`docker compose up` fails with `set in .env`.** A required variable is empty; the error names it.
- **`network mattermost-network not found`.** Step 2 was skipped.
- **Websocket errors in the client.** `MATTERMOST_URL` must exactly match the public URL.

### Apply `.env` or compose-file changes

```bash
docker compose -f mattermost-traefik-letsencrypt-docker-compose.yml -p mattermost up -d --force-recreate
```

## Features

- **Mattermost Team Edition** (11.10 line) — channels, DMs, calls, playbooks-lite, integrations.
- **PostgreSQL** backing store with healthcheck and start-order dependency.
- **Traefik v3** with automatic HTTP→HTTPS redirect and Let's Encrypt TLS-ALPN certificate issuance (websockets included).
- **Basic-auth protected Traefik dashboard** on a separate hostname.
- **Scheduled backups** of the database (`pg_dump | gzip`) and file uploads (`tar.gz`) with retention pruning, plus restore scripts for both.
- **Credentials required at deploy time** — compose fails fast if `.env` is incomplete.

## Supply chain trust

This repository is a **deployment template**, not a custom Docker image. It orchestrates three upstream images:

- [`traefik`](https://hub.docker.com/_/traefik) — reverse proxy, Docker Hub official image
- [`mattermost/mattermost-team-edition`](https://hub.docker.com/r/mattermost/mattermost-team-edition) — Mattermost upstream
- [`postgres`](https://hub.docker.com/_/postgres) — PostgreSQL, Docker Hub official image

All three are pinned to `tag@sha256:<digest>` as interpolation defaults in the compose file's `x-images` block — `git pull` alone delivers the version combination this repository has tested; an `*_IMAGE_TAG` variable in `.env` overrides deliberately.

The weekly `check-pin-freshness` CI job re-resolves each pinned tag against its registry and compares the pinned Mattermost and Traefik versions against the latest upstream releases. CI runs on every push, pull request, and every Monday at 06:00 UTC. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

- [ ] **Create the admin account immediately after deploy** — first sign-up wins.
- [ ] **Strong secrets.** `MATTERMOST_DB_PASSWORD` at 24+ random characters; regenerate the Traefik dashboard BCrypt hash per deployment.
- [ ] **Review sign-up settings** (System Console → Authentication) unless the workspace is meant to be open.
- [ ] **Host-mount the backup volumes** for disaster recovery.
- [ ] **Verify Let's Encrypt cert issuance** in the Traefik logs on first start.
- [ ] **Back up before upgrades** and consult Mattermost's upgrade documentation for your path (this template tracks the latest release line).

## Backups

The `backups` container performs a dump → archive → prune → sleep loop: `pg_dump | gzip` of the database, `tar.gz` of the uploads directory, pruning by retention windows, then sleeping `BACKUP_INTERVAL` (default 24h).

Each cycle logs `Database backup OK: <file> (<bytes> bytes)` or `Database backup FAILED` (the same for the data archive where there is one). A failed dump is kept as `<file>.failed` for diagnosis and never overwrites a good backup — grep the log for `FAILED` from your monitoring.

**Verify backups are running:**

```bash
docker compose -p mattermost logs backups | tail -5
```

**Restore** with the interactive scripts (`chmod +x *.sh` once): `./mattermost-restore-database.sh`, then `./mattermost-restore-application-data.sh`.

## Resource limits

Every service carries memory and CPU limits plus reservations as compose-level defaults — the same values CI boots the stack under. Override any of them in `.env` (the knobs and their defaults are listed in `.env.example`, e.g. `TRAEFIK_MEMORY_LIMIT=512m`) and the override survives every `git pull`. If a service is OOM-killed under real load, `docker inspect <container> --format '{{.State.OOMKilled}}'` says so; raise its `_MEMORY_LIMIT` and recreate.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/mattermost-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every Monday at 06:00 UTC:

1. **Lint** — shellcheck on both restore scripts, actionlint on the workflow.
2. **Trivy scans** of all three pinned images (CRITICAL/HIGH, SARIF to the Security tab).
3. **Pin freshness** (weekly/manual) — digest drift plus release-lag checks for Mattermost and Traefik.
4. **Deploy-and-test** — boots the full stack with ephemeral credentials and requires `/api/v4/system/ping` to answer OK through Traefik.

A green run is the authoritative proof that the template deploys end-to-end and that its backups restore.

### Backup and restore, proven

`tests/e2e-backup-restore.sh` runs against the live stack and is what CI executes after the HTTPS smoke. The scenario that matters most is the restore roundtrip: insert a marker row, restore the earliest backup, assert the marker is gone — a backup that cannot be restored fails the build. Run it yourself against a running deployment with short intervals in `.env` (`BACKUP_INIT_SLEEP=15s`, `BACKUP_INTERVAL=60s`):

```bash
chmod +x tests/e2e-backup-restore.sh
./tests/e2e-backup-restore.sh
```

It stops the database container briefly to prove failure detection — run it on a staging copy, not on production.

## Security Notes

- Credentials are read from `.env` at deploy time; `.env` is gitignored and compose fails fast on missing required variables.
- **Pre-rotation advisory.** Releases before v1.0.0 (2026-08-31) shipped a tracked `.env` with a generated-looking database password. Rotate `MATTERMOST_DB_PASSWORD` if your deployment reused it.
- The database listens only on the internal network.
- Upstream image digests are pinned; the weekly freshness job flags drift loudly.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** — Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
