# KMA Windmill — Fly.io deploy

KMA-hosted [Windmill](https://www.windmill.dev) on Fly.io (org `kma`, region `fra`). This directory holds the deploy config; the rest of the repo is a fork of `windmill-labs/windmill` for future patching.

- **Live URL:** https://kma-windmill.fly.dev
- **Default login:** `admin@windmill.dev` / `changeme` — **rotate immediately on first access**

## Topology

| Fly app | Role | Image | Size |
|---|---|---|---|
| `kma-windmill` | HTTP UI + API (port 8000), runs migrations | `ghcr.io/windmill-labs/windmill:main` | shared-cpu-2x / 2 GB |
| `kma-windmill-worker` | Job runner (default group, all langs except docker) | `ghcr.io/windmill-labs/windmill:main` | shared-cpu-2x / 2 GB |
| `kma-windmill-db` | Postgres 17 metadata + queue | `postgres:17` | shared-cpu-2x / 2 GB + 10 GB vol |

All three live in Fly org `kma`, region `fra`. Server and worker share one DSN pointed at the DB on the 6PN private network.

## Wire layout

```
internet → kma-windmill.fly.dev (Fly proxy → server :8000)
                                       │
                                       ▼
                              kma-windmill-db.internal:5432
                                       ▲
                                       │
                              kma-windmill-worker (job poll)
```

DSN form (both apps):

```
postgres://windmill:<pw>@kma-windmill-db.internal:5432/windmill?sslmode=disable
```

`kma-windmill-db.internal` is Fly's 6PN DNS — internal-only, never reachable from the public internet.

## Files

- [`fly.server.toml`](fly.server.toml) — server (HTTP-routed, public)
- [`fly.worker.toml`](fly.worker.toml) — worker (no public service, restart policy `always`)
- [`fly.db.toml`](fly.db.toml) — plain `postgres:17` machine + volume

We point at upstream `ghcr.io/windmill-labs/windmill:main` for now. To run patched code, replace the image refs with `ghcr.io/kma-core/windmill:<tag>` once we build (see "Patching" below).

## First-time deploy (reproduce from zero)

```sh
cd deploy/fly

# 1. Postgres — plain image + volume
flyctl apps create kma-windmill-db --org kma
PG_PASS=$(openssl rand -hex 24)
flyctl secrets set POSTGRES_PASSWORD="$PG_PASS" -a kma-windmill-db --stage
flyctl volumes create pgdata --app kma-windmill-db --region fra --size 10 --yes
flyctl deploy -c fly.db.toml --ha=false

# 2. Server + worker
flyctl apps create kma-windmill --org kma
flyctl apps create kma-windmill-worker --org kma

DSN="postgres://windmill:${PG_PASS}@kma-windmill-db.internal:5432/windmill?sslmode=disable"
flyctl secrets set DATABASE_URL="$DSN" -a kma-windmill --stage
flyctl secrets set DATABASE_URL="$DSN" -a kma-windmill-worker --stage

flyctl deploy -c fly.server.toml --image ghcr.io/windmill-labs/windmill:main --ha=false
flyctl deploy -c fly.worker.toml --image ghcr.io/windmill-labs/windmill:main --ha=false
```

## Update Windmill to latest upstream

```sh
cd deploy/fly
flyctl deploy -c fly.server.toml --image ghcr.io/windmill-labs/windmill:main
flyctl deploy -c fly.worker.toml --image ghcr.io/windmill-labs/windmill:main
```

Server applies migrations on boot; the worker waits for the server to finish before picking up jobs.

## Backups

Fly takes daily volume snapshots (5 retained by default). Ad-hoc:

```sh
flyctl volumes list -a kma-windmill-db        # get vol id (e.g. vol_xxxx)
flyctl volumes snapshots create vol_xxxx
flyctl volumes snapshots list vol_xxxx
```

Restore: create a new volume `--snapshot-id <snap>`, then redeploy `fly.db.toml` pointing at it.

## Patching (when we need to)

This repo is a fork of `windmill-labs/windmill`. Patch flow:

1. Make changes on a branch (e.g. `kma-patches`).
2. The upstream GitHub Actions workflow `.github/workflows/docker-image.yml` builds and pushes to `ghcr.io/<owner>/windmill:<tag>` on tag pushes. Configure the fork's GHA secrets accordingly (or strip the workflow down to push only to `ghcr.io/kma-core/windmill`).
3. Update `fly.server.toml` and `fly.worker.toml` to reference `ghcr.io/kma-core/windmill:<tag>`.
4. `flyctl deploy` as above.

Sync with upstream periodically:

```sh
git fetch upstream
git rebase upstream/main kma-patches    # or merge, your call
git push --force-with-lease origin kma-patches
```

## Known limits

- **No Docker-in-Docker.** Upstream worker runs user containers via a DinD sidecar. Fly machines don't run privileged DinD by default, so this deploy supports Python / TypeScript / Bash / Go / Java / Ruby / Rust / PHP / Powershell / DuckDB scripts only — no `docker run` from job code.
- **No dedicated native-jobs worker.** Add a second `kma-windmill-worker-native` app with `WORKER_TAGS=native` when light-jobs load justifies it.
- **No LSP container.** In-browser code intelligence is reduced; add later as a separate Fly app on port 3001 if needed.
- **Postgres is single-node.** Backups via Fly volume snapshots. For HA, add a replica via streaming replication (manual setup, not automated here).
- **Default admin still in place** — `admin@windmill.dev` / `changeme`. Rotate.
- **SSO is capped at 10 users in CE.** Either keep under, patch in this fork, or front Windmill with `oauth2-proxy`.
