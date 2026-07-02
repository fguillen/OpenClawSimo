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

The Control UI is not public. It is reached over an SSH tunnel to the nginx front
door on `127.0.0.1:8080`. See [docs/deploy.md](docs/deploy.md).

## Compose file

Everything runs from a single [docker-compose.yaml](docker-compose.yaml): the
`openclawsimo` service plus a `browser` sidecar (full Chrome via Kasm/noVNC, exposing
a CDP endpoint on `:9222`) for browser automation. The sidecar is heavy — a large
image with meaningful RAM and CPU cost — but is now included by default.

## Hardening checklist

- Keep `OPENCLAW_GATEWAY_BIND=loopback` so the gateway is never exposed publicly.
- Keep the Control UI private. Publish nginx to the host loopback only
  (`127.0.0.1:8080:8080`) and reach it over an SSH tunnel. No public Domain.
- Always set a strong `AUTH_PASSWORD`. Never run without one.
- Restrict every channel with an allowlist policy (`*_DM_POLICY=allowlist` plus a
  populated `*_ALLOW_FROM`).
- Run isolated from production. The agent has shell and workspace access.
- Run `openclaw doctor` after first boot to validate config.

## Remote access

The Control UI is admin only and stays private. Reach it over an SSH tunnel to the
nginx front door on `8080`, which keeps its basic auth. The gateway stays on
loopback `18789` behind nginx. Full runbook and troubleshooting live in
[docs/deploy.md](docs/deploy.md).

```
ssh -N -L 8080:127.0.0.1:8080 user@YOUR_HOST
# then open http://localhost:8080
```

## WhatsApp caveat

OpenClaw connects to WhatsApp through QR/pairing web-session auth, not the
official WhatsApp Cloud API. It is not a business-messaging or BSP replacement,
and web-session use carries account risk. See the
[docs](https://docs.openclaw.ai) before enabling it.
