# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Deployment configuration — **not** application source code. It holds the Docker
Compose and environment config for self-hosting [OpenClaw](https://docs.openclaw.ai),
a multi-channel AI assistant, on a Hetzner (FSN1) host behind Dokploy + Traefik v3
with EU data residency. The OpenClaw application image (`coollabsio/openclaw`) is
pulled from a registry; its code does not live here.

There is no build, lint, or test step. Work here means editing compose files,
`.env` config, and docs.

## Commands

```
cp .env.example .env          # then fill in secrets (see below)
docker compose up -d          # full stack (openclaw + browser sidecar)
docker compose logs -f openclaw
docker compose pull && docker compose up -d   # update to a new image

# Validate a running instance:
docker compose exec openclaw openclaw doctor
```

## Architecture

- **`docker-compose.yaml`** — the whole stack. The `openclaw` service (persistent
  `openclaw-data` volume mounted at `/data`, joined to the external
  `dokploy-network`, public exposure entirely via Traefik labels — TLS via the
  `letsencrypt` certresolver, container port 8080; the router `Host()` rule
  contains a placeholder hostname `claw.example.es` that must be replaced per
  deployment) plus a `browser` service: a `kasmweb/chrome` sidecar exposing a CDP
  endpoint at `http://browser:9222` (in-network only), wired into openclaw via
  `BROWSER_CDP_URL`. The sidecar is a large image with meaningful RAM/CPU cost but
  ships enabled by default.
- **`.env`** (from `.env.example`) — all runtime config. Provider keys, web-UI
  auth, gateway token/bind, state and workspace dirs under `/data`, CORS origins,
  and per-channel settings. Git-ignored; only `.env.example` is committed.
- **`docs/deploy.md`** — step-by-step Dokploy deploy runbook and host
  prerequisites. Host must be **Ubuntu 24.04 LTS** (or 22.04); pre-release Ubuntu
  breaks Dokploy's Docker install.

## Security model — treat as load-bearing

OpenClaw is an agent with **shell and workspace access**; assume any instance can
run arbitrary commands on its host. Config changes here have real blast radius:

- Keep `OPENCLAW_GATEWAY_BIND=loopback` — never expose the gateway publicly.
- `AUTH_PASSWORD` is required; never run without one.
- Restrict every channel with an allowlist: set `*_DM_POLICY=allowlist` **and** a
  populated `*_ALLOW_FROM` (e.g. `TELEGRAM_DM_POLICY` / `TELEGRAM_ALLOW_FROM`).
- Deploy isolated from production systems.
- WhatsApp uses QR/pairing web-session auth (not the official Cloud API) and
  carries account-ban risk — it is not a BSP replacement. See the README caveat.

Generate secrets with `openssl rand` (`-base64 24` for `AUTH_PASSWORD`, `-hex 32`
for `OPENCLAW_GATEWAY_TOKEN`).
