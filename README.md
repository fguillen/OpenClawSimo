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

## Remote access to the Gateway Dashboard (SSH tunnel)

The gateway (which serves the Control UI / **Gateway Dashboard** on port `18789`)
binds to loopback and is never routed through Traefik, so it is **not** reachable
from a browser over the internet — by design. To reach it from your own machine,
tunnel to it over SSH.

Because the gateway lives on the *container's* loopback, a plain `ssh -L` to the
host can't see it until the port is reachable on the host. Publish it to the
host's loopback **only** (never `0.0.0.0`):

1. In the `openclaw` service in [docker-compose.yaml](docker-compose.yaml), make the
   gateway listen on the container's network interface and publish it to host
   loopback:

   ```yaml
       environment:
         # was loopback; a network-accessible bind is required so Docker can reach
         # and publish the port. Verify accepted values with `openclaw config schema`.
         - OPENCLAW_GATEWAY_BIND=lan
       ports:
         - "127.0.0.1:18789:18789"   # host loopback ONLY — never 0.0.0.0:18789
   ```

   Binding to `127.0.0.1` on the **host** is what keeps this private: only the host
   itself — and therefore your SSH tunnel — can reach `18789`. It is never on the
   public internet. Redeploy for the change to take effect.

2. From your machine, open the tunnel and leave the shell running:

   ```
   ssh -N -L 18789:127.0.0.1:18789 user@YOUR_HOST
   ```

3. Point the dashboard at the tunnel: open `http://127.0.0.1:18789`, or in the
   Gateway Dashboard set the WebSocket URL to `ws://127.0.0.1:18789`, and
   authenticate with `OPENCLAW_GATEWAY_TOKEN`. No `controlUi.allowedOrigins` entry
   is needed — the gateway auto-allows `localhost`/`127.0.0.1` origins.

Close the SSH session to revoke access. This deliberately relaxes the strict
`loopback` bind from the hardening checklist; the `127.0.0.1`-scoped port publish
is what preserves the security posture, so **never** publish `18789` on `0.0.0.0`
or add a Traefik route to it.

## WhatsApp caveat

OpenClaw connects to WhatsApp through QR/pairing web-session auth, not the
official WhatsApp Cloud API. It is not a business-messaging or BSP replacement,
and web-session use carries account risk. See the
[docs](https://docs.openclaw.ai) before enabling it.
