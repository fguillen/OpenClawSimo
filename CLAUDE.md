# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Deployment configuration — **not** application source code. It holds the Docker
Compose and environment config for self-hosting [OpenClaw](https://docs.openclaw.ai),
a multi-channel AI assistant, on a Hetzner (FSN1) host behind Dokploy with EU data
residency. The **official** OpenClaw image (`ghcr.io/openclaw/openclaw`; the Docker
Hub mirror is `openclaw/openclaw`) is pulled from a registry; its code does not live
here. This deployment previously used the community `coollabsio/openclaw` image,
which bundled an nginx basic-auth front door, env-var → config conversion, and a
Chrome sidecar — none of which the official image provides. The migration dropped all
three (see below).

There is no build, lint, or test step. Work here means editing compose files,
`.env` config, and docs.

## Workflow

Always create a git commit after completing a task.

## Commands

```
cp .env.example .env          # then fill in secrets (see below)
docker compose up -d          # single service (openclawsimo)
docker compose logs -f openclawsimo
docker compose pull && docker compose up -d   # update to a new image (bump the pinned tag)

# Validate a running instance:
docker compose exec openclawsimo openclaw doctor
```

## Architecture

- **`docker-compose.yaml`** — the whole stack: a single `openclawsimo` service on the
  official `ghcr.io/openclaw/openclaw` image (pin a dated tag; do not track `latest`),
  with a persistent `openclaw-data` volume mounted at `/home/node/.openclaw` (official
  layout; the image runs as non-root `node` / UID 1000, so that volume and the config
  dir must be writable by 1000). There is no nginx, no `browser` sidecar, no public
  route, and no Domain. The **gateway serves the Control UI directly on port 18789**,
  published to the host loopback only via `ports: 127.0.0.1:18789:18789` and reached
  over an SSH tunnel (see docs/deploy.md). Because there is no nginx in front, the
  gateway must listen beyond the in-container loopback for Docker to forward the
  published port — this is **not** an env var: it is set in `openclaw.json` as
  `gateway.bind: "lan"` (accepted modes are `loopback`/`lan`/`tailnet`/`auto`/`custom`;
  a raw `0.0.0.0` is rejected). Effective exposure still stays loopback-on-host; the
  **gateway token is the auth boundary**. A one-shot **`init-perms`** service (busybox,
  runs as root, `depends_on … service_completed_successfully`) chowns the data volume
  and the `/config` bind mount to UID 1000 before openclawsimo starts, because the image
  runs as non-root `node` (1000) and fresh mounts are created root-owned (otherwise:
  `EACCES` on `/home/node/.openclaw` and the config never loads). The service stays on
  `dokploy-network` for Dokploy compatibility (the token, not the network, is what
  protects it; it may be moved to a private network now that the sidecar is gone). Env
  vars are injected via explicit `${VAR}` interpolation in the `environment:` block (no
  `env_file`); set them in Dokploy's **Environment Settings**, which Dokploy writes to a
  `.env` beside the compose file. The official image is config-file-driven, so the env
  block is slim — provider keys (read from env; confirmed), the required
  `OPENCLAW_GATEWAY_TOKEN` (`${VAR:?}`, fails the deploy if unset), and the config path
  `OPENCLAW_CONFIG_PATH` (honored — the gateway watches that exact path); model,
  channels, tools and `gateway.bind` live in `openclaw.json`. **Still verify against the
  actual image:** the exact `openclaw.json` schema keys for the default model and the
  Telegram channel.
- **`openclaw.json`** — declarative config. Under the official image this holds the bulk
  of configuration (the community image's env-var → config conversion is gone): the
  default model, channels (Telegram), `tools.deny`, and agent defaults, plus settings
  with no env-var equivalent like `agents.defaults.memorySearch.enabled`. Selected via
  `OPENCLAW_CONFIG_PATH` (default `/config/openclaw.json`). The live file is **not**
  committed — it lives in Dokploy's persistent `files/` dir (created via the Dokploy UI
  as a File Mount at `config/openclaw.json`) and, because it is uncommitted, safely holds
  the real Telegram bot token; the committed **`openclaw.example.json`** is the reference
  (placeholders for secrets). Confirm the exact schema keys for the default model and the
  Telegram channel against the official config reference. The compose file bind-mounts the
  **directory** `../files/config` at `/config` — not the single file — because openclaw
  reads the config and rewrites the normalized version back in-place at boot via an atomic
  `rename()`, which fails with `EBUSY` on a single-file bind mount (`:ro` fails earlier
  with `EROFS`). Mounting the parent directory lets the temp-file rename stay within one
  filesystem, and keeping it out of the data volume means the boot rewrite persists in
  Dokploy's `files/` dir across redeploys.
  Currently disables semantic memory search (no embedding provider is configured;
  OpenRouter cannot supply embeddings; keyword/FTS recall still works) and
  allowlists the Control-UI origin via `gateway.controlUi.allowedOrigins`
  (the Control UI is loaded over the SSH tunnel at `http://localhost:18789`, so that
  origin must be added or the WebSocket handshake is rejected with "origin not
  allowed"). Also enables **Anthropic prompt caching** via
  `agents.defaults.models["anthropic/claude-sonnet-5"].params.cacheRetention: "long"`.
  Every request carries a large fixed prefix (system prompt + all tool schemas + skills
  + bootstrapped workspace files), so a trivial "hi" still costs tens of thousands of
  input tokens; caching bills that stable prefix as cache reads (~10% of input price)
  instead of full input on repeat messages. Caching is **Anthropic-API-only**, which is
  why the default model uses the native `anthropic/claude-sonnet-5` form (not
  `openrouter/…`); routed via OpenRouter the prefix gets no cache discount. Caching cuts
  cost, not the reported token *count*. To shrink the count itself, the config also prunes
  the tool surface via a top-level `tools.deny` list (deny wins; a denied tool's JSON
  schema is not sent to the model that turn) — dropping tools this deployment does not use:
  `browser` (browser automation is disabled — dropping it also removes its large schema
  from the prefix), `qqbot_remind` (QQ-only), `video_generate`/`music_generate`, the
  multi-agent `subagents`/`sessions_spawn`/`sessions_yield`/`sessions_send`, and the
  `*_goal` tools. Kept: cron, tts, image_generate, skill_workshop, nodes, canvas, and the
  file/exec/message/web core. Further count levers, driven by `/context detail`: trim the
  injected skills list via `agents.defaults.skills` (an allowlist), cap bootstrap file
  injection (`agents.defaults.bootstrapMaxChars` / `bootstrapTotalMaxChars`), and slim the
  largest bootstrapped workspace files. Run `/context detail` for the exact per-section
  breakdown before cutting.
- **`.env`** (from `.env.example`) — slim runtime config for the official image: provider
  keys, the gateway token/bind, and the config path. Model, channels, and tool settings
  moved to `openclaw.json`. Git-ignored; only `.env.example` is committed
  (`.env.production` is also git-ignored and holds the real secrets locally).
- **`docs/deploy.md`** — step-by-step Dokploy deploy runbook and host
  prerequisites. Host must be **Ubuntu 24.04 LTS** (or 22.04); pre-release Ubuntu
  breaks Dokploy's Docker install.

## Security model — treat as load-bearing

OpenClaw is an agent with **shell and workspace access**; assume any instance can
run arbitrary commands on its host. Config changes here have real blast radius:

- Never expose the gateway publicly. Publish it to the **host loopback only**
  (`127.0.0.1:18789:18789`) and reach it over an SSH tunnel. Do not add a Dokploy
  Domain. Note the gateway binds to the container's external interface (so Docker can
  forward the loopback-published port) — the loopback-only *publish* plus the token are
  what keep it private, not an in-container loopback bind.
- The Control UI is an admin surface (chat, config, exec approvals). There is no nginx
  and no basic auth: `OPENCLAW_GATEWAY_TOKEN` is the **only** auth boundary in front of
  it. Treat it as a high-value secret; rotate it if leaked.
- Restrict every channel with an allowlist: set `dmPolicy: allowlist` **and** a
  populated `allowFrom` in `openclaw.json` (e.g. the `channels.telegram` block).
- Deploy isolated from production systems.
- WhatsApp uses QR/pairing web-session auth (not the official Cloud API) and
  carries account-ban risk — it is not a BSP replacement. See the README caveat.

Generate the gateway token with `openssl rand -hex 32`.
