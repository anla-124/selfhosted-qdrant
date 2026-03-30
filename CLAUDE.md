# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A Docker Compose deployment of Qdrant v1.14.1 for [Dokploy](https://dokploy.com). This is the vector database backend for [pdf-search](https://github.com/anla-124/pdf-search). There is no application code — just infrastructure config.

## Deployment Target

- **Platform**: Dokploy (manages Docker Compose services from GitHub repos)
- **Reverse proxy**: Traefik (injected by Dokploy via its Domains UI — no manual labels)
- **TLS**: Cloudflare Flexible SSL (CF terminates TLS; origin receives plain HTTP on port 80)
- **Network**: All containers must join `dokploy-network` (external Docker network)
- **Domain**: `qdrant.anduincapital.dev`

## Dokploy Compose Constraints

When editing `docker-compose.yml`, these rules are enforced by the Dokploy environment:

- **No `ports:`** — use `expose:` only; Traefik routes external traffic
- **No `container_name:`** — Dokploy names containers automatically
- **No manual Traefik labels** — Dokploy injects them when a domain is added in its UI
- **`env_file: .env`** — Dokploy creates `.env` from variables set in its UI
- After domain changes, the container must show **"Recreated"** (not "Started") in the deploy log

## Qdrant Image Constraints

- **No curl/wget** in `qdrant/qdrant` image — healthchecks must use `bash /dev/tcp`
- Volume must mount to `/qdrant/storage` exactly (not `/data` or `/qdrant/data`)
- API key env var uses double-underscore config mapping: `QDRANT__SERVICE__API_KEY`
- `QDRANT_ALLOW_RECOVERY_MODE` is a special-cased single-underscore variable (not a typo)
- API key header format is `api-key: <key>` (not `Authorization: Bearer`)

## Files

- `docker-compose.yml` — single-service Qdrant compose with healthcheck, volumes, network
- `.env.example` — template for `QDRANT_API_KEY` (generate with `openssl rand -hex 32`)
- `.gitignore` — excludes `.env`

## Verification

```bash
curl https://qdrant.anduincapital.dev/healthz -H "api-key: YOUR_KEY"
```

Dashboard: `https://qdrant.anduincapital.dev/dashboard`
