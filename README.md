# OpenClaw self-hosted

Docker deployment of [OpenClaw](https://docs.openclaw.ai), a self-hosted
multi-channel AI assistant. This repo tracks the compose and environment config
for running it on Hetzner (FSN1) behind Dokploy and Traefik v3, with EU data
residency. OpenClaw is an agent with shell and workspace access, so treat any
instance as if it can run arbitrary commands on its host.

## Quick start

```
cp .env.example .env
# fill in a provider key, AUTH_PASSWORD, and OPENCLAW_GATEWAY_TOKEN
docker compose up -d
```

Set the real hostname in the Traefik router label in
[docker-compose.yaml](docker-compose.yaml) before deploying.

## Compose file

Everything runs from a single [docker-compose.yaml](docker-compose.yaml): the
`openclaw` service plus a `browser` sidecar (full Chrome via Kasm/noVNC, exposing
a CDP endpoint on `:9222`) for browser automation. The sidecar is heavy — a large
image with meaningful RAM and CPU cost — but is now included by default.

## Hardening checklist

- Keep `OPENCLAW_GATEWAY_BIND=loopback` so the gateway is never exposed publicly.
- Always set a strong `AUTH_PASSWORD`. Never run without one.
- Restrict every channel with an allowlist policy (`*_DM_POLICY=allowlist` plus a
  populated `*_ALLOW_FROM`).
- Run isolated from production. The agent has shell and workspace access.
- Run `openclaw doctor` after first boot to validate config.

## WhatsApp caveat

OpenClaw connects to WhatsApp through QR/pairing web-session auth, not the
official WhatsApp Cloud API. It is not a business-messaging or BSP replacement,
and web-session use carries account risk. See the
[docs](https://docs.openclaw.ai) before enabling it.
