# OpenClaw self-hosted

Docker deployment of [OpenClaw](https://docs.openclaw.ai), a self-hosted
multi-channel AI assistant. This repo tracks the compose and environment config
for running it on Hetzner (FSN1) behind Dokploy, with EU data residency. OpenClaw
is an agent with shell and workspace access, so treat any instance as if it can
run arbitrary commands on its host.

Runs the **official** image `ghcr.io/openclaw/openclaw` (not the community
`coollabsio/openclaw` build). The Control UI is served directly by the gateway and
reached over an SSH tunnel with token auth — no nginx front door, no basic auth.

## Quick start

```
cp .env.example .env
# fill in a provider key and OPENCLAW_GATEWAY_TOKEN
docker compose up -d
```

The Control UI is not public. It is reached over an SSH tunnel to the gateway on
`127.0.0.1:18789`, authenticated with `OPENCLAW_GATEWAY_TOKEN`. See
[docs/deploy.md](docs/deploy.md).

## Compose file

Everything runs from a single [docker-compose.yaml](docker-compose.yaml): just the
`openclawsimo` service. Config that has no env-var equivalent (model, channels,
tools, agent defaults) lives in `openclaw.json`; see
[openclaw.example.json](openclaw.example.json).

Browser automation is **disabled** in this deployment (`browser` is in
`tools.deny`). The old Kasm Chrome CDP sidecar is gone. To add browsing later, the
official image supports Playwright — install Chromium into the persisted volume.

## Hardening checklist

- Keep the gateway published to the **host loopback only** (`127.0.0.1:18789:18789`)
  and reach it over an SSH tunnel. No public Domain, no public route.
- Treat `OPENCLAW_GATEWAY_TOKEN` as the auth boundary — it is the only thing in
  front of the admin Control UI. Generate it with `openssl rand -hex 32` and keep it
  secret.
- Restrict every channel with an allowlist policy (`dmPolicy: allowlist` plus a
  populated `allowFrom` in `openclaw.json`).
- Run isolated from production. The agent has shell and workspace access.
- Run `openclaw doctor` after first boot to validate config.

## Remote access

The Control UI is admin only and stays private. Reach it over an SSH tunnel to the
gateway on `18789` and authenticate with the gateway token. Full runbook and
troubleshooting live in [docs/deploy.md](docs/deploy.md).

```
ssh -N -L 18789:127.0.0.1:18789 user@YOUR_HOST
# then open http://localhost:18789 and paste OPENCLAW_GATEWAY_TOKEN
```

## WhatsApp caveat

OpenClaw connects to WhatsApp through QR/pairing web-session auth, not the
official WhatsApp Cloud API. It is not a business-messaging or BSP replacement,
and web-session use carries account risk. See the
[docs](https://docs.openclaw.ai) before enabling it.
