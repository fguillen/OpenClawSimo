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

## Workflow

Always create a git commit after completing a task.

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
  `dokploy-network`, container port 8080). Routing and TLS are **not** in the
  compose file — the service carries no Traefik labels; configure the domain in
  Dokploy's **Domains** UI (service `openclaw`, port 8080, `letsencrypt`
  certresolver) and Dokploy generates the Traefik config. Env vars are injected via
  explicit `${VAR}` interpolation in the `environment:` block (no `env_file`); set
  them in Dokploy's **Environment Settings**, which Dokploy writes to a `.env`
  beside the compose file for Compose to interpolate. Required secrets
  (`AUTH_PASSWORD`, `OPENCLAW_GATEWAY_TOKEN`) use `${VAR:?}` and fail the deploy if
  unset. Plus a `browser` service: a `kasmweb/chrome` sidecar exposing a CDP
  endpoint at `http://browser:9222` (in-network only), wired into openclaw via
  `BROWSER_CDP_URL`. Pin a Kasm version tag — there is no `latest` tag — and pass
  `CHROME_ARGS=--remote-debugging-port=9222 --remote-debugging-address=0.0.0.0
  --remote-allow-origins=*`, without which Kasm's Chrome binds CDP to localhost and
  the openclaw container cannot reach it. The sidecar is a large image with
  meaningful RAM/CPU cost but ships enabled by default.
  Plus a `tailscale` service: a `tailscale/tailscale` sidecar that exposes the
  loopback-bound gateway (`127.0.0.1:18789`) to the tailnet over HTTPS via
  `tailscale serve`, so the remote **Gateway Dashboard** (Control UI) can reach it
  **without** any public route. It uses `network_mode: "service:openclaw"` to share
  openclaw's network namespace — so tailscaled sees the gateway on localhost and the
  tailnet interface lives in the same netns — which leaves the openclaw service
  itself untouched (still `loopback` bind, `8080` for Traefik). Serve (not funnel)
  keeps it tailnet-only. We run Serve from this sidecar rather than OpenClaw's
  built-in `gateway.tailscale.mode` so we don't depend on the `coollabsio/openclaw`
  image shipping the tailscale CLI. Requires (Tailscale admin console) MagicDNS +
  HTTPS certs enabled and a reusable non-ephemeral `TS_AUTHKEY` set in Dokploy; the
  `tailscale-state` volume persists node identity across redeploys, and
  `TS_USERSPACE=true` avoids needing `/dev/net/tun`/`NET_ADMIN`. The Serve config is
  the committed `ts-serve.example.json` (live copy in Dokploy's `files/config`,
  mounted at `/config`). Reach the dashboard at `wss://openclawsimo.<tailnet>.ts.net`.
- **`openclaw.json`** — declarative config for settings that have **no env-var
  equivalent** (env vars only cover keys, auth, paths, channels; things like
  `agents.defaults.memorySearch.enabled` are config-file only). Selected via
  `OPENCLAW_CONFIG_PATH` (default `/config/openclaw.json`). The live file is **not**
  committed — it lives in Dokploy's persistent `files/` dir (created via the Dokploy UI
  as a File Mount at `config/openclaw.json`); the committed **`openclaw.example.json`**
  is the reference for its contents. The compose file bind-mounts the **directory**
  `../files/config` at `/config` — not the single file — because `configure.js` reads
  the config and rewrites the normalized version back in-place at boot via an atomic
  `rename()`, which fails with `EBUSY` on a single-file bind mount (`:ro` fails earlier
  with `EROFS`). Mounting the parent directory lets the temp-file rename stay within one
  filesystem. Kept outside the `/data` volume so it is not shadowed by `openclaw-data`;
  the boot rewrite persists in Dokploy's `files/` dir across redeploys.
  Currently disables semantic memory search (no embedding provider is configured;
  OpenRouter cannot supply embeddings; keyword/FTS recall still works) and
  allowlists the tailnet Control-UI origin via `gateway.controlUi.allowedOrigins`
  (the Gateway Dashboard is served over the tailnet at
  `https://openclawsimo.<tailnet>.ts.net` — see the `tailscale` sidecar above — so
  that origin must be added or the WebSocket handshake is rejected with "origin not
  allowed"). Also sets `gateway.auth.allowTailscale: true` so tailnet identity
  satisfies Control-UI auth (the Gateway Token still works if omitted).
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
  Remote Control-UI access goes over the **tailnet only** (the `tailscale` sidecar
  running `tailscale serve`); never add a public Traefik route to port 18789 or
  switch the sidecar to `funnel`.
- `AUTH_PASSWORD` is required; never run without one.
- Restrict every channel with an allowlist: set `*_DM_POLICY=allowlist` **and** a
  populated `*_ALLOW_FROM` (e.g. `TELEGRAM_DM_POLICY` / `TELEGRAM_ALLOW_FROM`).
- Deploy isolated from production systems.
- WhatsApp uses QR/pairing web-session auth (not the official Cloud API) and
  carries account-ban risk — it is not a BSP replacement. See the README caveat.

Generate secrets with `openssl rand` (`-base64 24` for `AUTH_PASSWORD`, `-hex 32`
for `OPENCLAW_GATEWAY_TOKEN`).
