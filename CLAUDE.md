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
docker compose up -d          # full stack (openclawsimo + browser sidecar)
docker compose logs -f openclawsimo
docker compose pull && docker compose up -d   # update to a new image

# Validate a running instance:
docker compose exec openclawsimo openclaw doctor
```

## Architecture

- **`docker-compose.yaml`** — the whole stack. The `openclawsimo` service (persistent
  `openclaw-data` volume mounted at `/data`, joined to the external
  `dokploy-network`, container port 8080). The Control UI is private. There is no
  public route and no Traefik Domain. nginx (port 8080, basic auth) is published to
  the host loopback only via `ports: 127.0.0.1:8080:8080`, reached over an SSH tunnel
  (see docs/deploy.md), and the gateway stays on loopback 18789 behind nginx. The
  service stays on `dokploy-network` only to reach the browser sidecar. Env vars are injected via
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
  allowlists the Control-UI origin via `gateway.controlUi.allowedOrigins`
  (the Control UI is loaded over the SSH tunnel at `http://localhost:8080`, so that
  origin must be added or the WebSocket handshake is rejected with "origin not
  allowed"). Also enables **Anthropic prompt caching** via
  `agents.defaults.models["anthropic/claude-sonnet-5"].params.cacheRetention: "long"`.
  Every request carries a large fixed prefix (system prompt + all tool schemas — the
  browser/CDP tools are big — + skills + bootstrapped workspace files), so a trivial
  "hi" still costs ~40k input tokens; caching bills that stable prefix as cache reads
  (~10% of input price) instead of full input on repeat messages. Caching is
  **Anthropic-API-only**, which is why `OPENCLAW_PRIMARY_MODEL` uses the native
  `anthropic/claude-sonnet-5` form (not `openrouter/…`); routed via OpenRouter the
  prefix gets no cache discount. Caching cuts cost, not the reported token *count*.
  To shrink the count itself, the config also prunes the tool surface via a top-level
  `tools.deny` list (deny wins; a denied tool's JSON schema is not sent to the model
  that turn) — dropping tools this deployment does not use (`qqbot_remind` is QQ-only,
  plus `video_generate`/`music_generate`, the multi-agent `subagents`/`sessions_spawn`/
  `sessions_yield`/`sessions_send`, and the `*_goal` tools). Kept: cron, tts,
  image_generate, skill_workshop, nodes, canvas, browser, and the file/exec/message/web
  core. Further count levers, driven by `/context detail`: trim the injected skills list
  via `agents.defaults.skills` (an allowlist — 17 skills ≈ 1.5k tok by default), cap
  bootstrap file injection (`agents.defaults.bootstrapMaxChars` /
  `bootstrapTotalMaxChars`), and slim the largest bootstrapped workspace file
  (`AGENTS.md` ≈ 2.1k tok, injected on every message). Run `/context detail` for the
  exact per-section breakdown before cutting.
- **`.env`** (from `.env.example`) — all runtime config. Provider keys, nginx basic
  auth, gateway token/bind, state and workspace dirs under `/data`, CORS origins,
  and per-channel settings. Git-ignored; only `.env.example` is committed.
- **`docs/deploy.md`** — step-by-step Dokploy deploy runbook and host
  prerequisites. Host must be **Ubuntu 24.04 LTS** (or 22.04); pre-release Ubuntu
  breaks Dokploy's Docker install.

## Security model — treat as load-bearing

OpenClaw is an agent with **shell and workspace access**; assume any instance can
run arbitrary commands on its host. Config changes here have real blast radius:

- Keep `OPENCLAW_GATEWAY_BIND=loopback` — never expose the gateway publicly.
- The Control UI is an admin surface (chat, config, exec approvals). Never expose it
  publicly. Publish nginx to the host loopback only and reach it over an SSH tunnel.
- `AUTH_PASSWORD` is required; never run without one.
- Restrict every channel with an allowlist: set `*_DM_POLICY=allowlist` **and** a
  populated `*_ALLOW_FROM` (e.g. `TELEGRAM_DM_POLICY` / `TELEGRAM_ALLOW_FROM`).
- Deploy isolated from production systems.
- WhatsApp uses QR/pairing web-session auth (not the official Cloud API) and
  carries account-ban risk — it is not a BSP replacement. See the README caveat.

Generate secrets with `openssl rand` (`-base64 24` for `AUTH_PASSWORD`, `-hex 32`
for `OPENCLAW_GATEWAY_TOKEN`).
