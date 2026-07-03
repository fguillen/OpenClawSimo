# Deploy on Dokploy

Notes for deploying this stack as a Dokploy Compose application on a Hetzner
FSN1 host, using the official `ghcr.io/openclaw/openclaw` image. Full product docs
live at https://docs.openclaw.ai.

## Prerequisites

- Host OS **Ubuntu 24.04 LTS** (or 22.04). Do not use pre-release / non-LTS
  Ubuntu such as 26.04 "resolute" — Docker's apt repo has no packages for that
  codename, so the Dokploy Docker install fails with
  `'<version>' not found amongst apt-cache madison results`.
- A Dokploy host (Docker Swarm). No public Traefik route is needed — the Control
  UI stays private.
- The `dokploy-network` overlay network already present. Dokploy creates it on
  install. Verify with `docker network ls`.
- No public DNS record or Domain is needed. The Control UI is reached over an SSH
  tunnel. See Access below.

## Steps

1. In Dokploy, create a new Compose application and point it at this repo.
2. Set the compose path to `docker-compose.yaml`. Pin the image tag in
   `docker-compose.yaml` to the current stable release (do not track `latest`).
3. Do **not** add a public Domain. The Control UI is admin only. If this app
   already has a Domain in the **Domains** tab, remove it. Compose publishes the
   gateway to the host loopback (`127.0.0.1:18789`), reached over an SSH tunnel.
   See Access.
4. In the app's **Environment Settings**, add the variables from
   [.env.example](../.env.example). `OPENCLAW_GATEWAY_TOKEN` is required — the
   compose file fails the deploy if it is unset — plus at least one provider key.
   Generate the token with `openssl rand -hex 32`. There is no `AUTH_PASSWORD`
   anymore: the token is the only auth boundary.
5. Create the declarative config file. In the app's **Advanced → Volumes / Mounts**,
   add a **File Mount** at mount path `config/openclaw.json` (Dokploy writes it to
   `../files/config/openclaw.json`, which the compose file bind-mounts as the directory
   `/config`). Paste the contents of [openclaw.example.json](../openclaw.example.json)
   and fill in the real values (model, the Telegram bot token, and `allowFrom` — the
   live file is not committed, so the token is safe there). A **directory** is mounted
   (not the single file) because OpenClaw rewrites this file in place at boot via an
   atomic `rename()`, which fails with `EBUSY` on a single-file bind mount. Do this
   **before deploying** — if `/config` is empty the config path won't exist. This file
   persists across redeploys. Set `gateway.bind: "lan"` in it (see Access) so the
   gateway is reachable through the published port.
   > Confirm the exact `openclaw.json` schema keys for the default model and the
   > Telegram channel against the official config reference; the example uses a
   > best-effort structure.
6. Permissions: the official image runs as non-root `node` (UID 1000), but a fresh
   named volume and the Dokploy `files/config` bind mount are created root-owned, so
   `node` gets `EACCES` and the config never loads. The `init-perms` service in the
   compose file chowns both mounts to UID 1000 on every deploy before openclawsimo
   starts — no manual step needed. If the `openclaw-data` volume already exists with
   unrelated root-owned content from an earlier attempt, delete it (Dokploy → the app's
   **Volumes**, or `docker volume rm <project>_openclaw-data`) so it starts clean.
7. Deploy. The stack starts with no public route.
8. After boot, validate with `docker compose exec openclawsimo openclaw doctor`.

## Access (SSH tunnel)

The Control UI is an admin surface (chat, config, exec approvals). It is never
public. It is served **directly by the gateway** on port 18789 — there is no nginx
front door.

| Port  | Role                                          | Exposure                                                 |
|-------|-----------------------------------------------|----------------------------------------------------------|
| 18789 | gateway, serves the Control UI over WebSocket | published to host `127.0.0.1` only, reach via SSH tunnel |

Set `gateway.bind: "lan"` in `openclaw.json` so the gateway listens on the
container's network interface (not just in-container loopback); Docker then forwards
the host-loopback-published port to it. Do **not** use `0.0.0.0` — the gateway
rejects raw addresses; the accepted modes are `loopback` / `lan` / `tailnet` / `auto`
/ `custom`. Because the publish is host-loopback only (`127.0.0.1:18789`), it stays
reachable solely over an SSH tunnel, and the gateway token authenticates the session.
A non-loopback bind also makes the gateway enforce the Control-UI origin check, which
is why `gateway.controlUi.allowedOrigins` must list `http://localhost:18789`.

Runbook:

```
ssh -N -L 18789:127.0.0.1:18789 user@YOUR_HOST
# open http://localhost:18789
# paste OPENCLAW_GATEWAY_TOKEN into the Control UI settings, then approve the device
```

Approve the device from inside the container:

```
docker compose exec openclawsimo openclaw devices list
docker compose exec openclawsimo openclaw devices approve <id>
```

If the Control UI shows "could not connect", check these in order:

- Non-secure context. Open `http://localhost:18789`, not a raw LAN or tailnet IP
  over plain HTTP. Browsers treat non-localhost plain HTTP as insecure and block
  the WebSocket the Control UI needs.
- Device pairing. A new browser is closed with code 1008 until you approve it with
  `openclaw devices approve <id>`.
- Token. Paste `OPENCLAW_GATEWAY_TOKEN` into the Control UI settings.
- Origin. `gateway.controlUi.allowedOrigins` in `openclaw.json` must include
  `http://localhost:18789` (match the port you forward).

Later: Tailscale Serve is the upgrade path for access without a tunnel. It keeps
the gateway loopback only and puts the Control UI on a private tailnet URL.

Security note: the Control UI is an admin surface. Never expose it publicly. See
https://docs.openclaw.ai/web/dashboard

## Browser automation (disabled)

This deployment ships with browser automation **off** — `browser` is in the
`tools.deny` list in `openclaw.json`, and there is no Chrome/CDP sidecar. The
official image does not bundle a browser.

To enable browsing later, remove `browser` from `tools.deny` and give the image a
Playwright Chromium: either bake it at build time (`OPENCLAW_INSTALL_BROWSER=1`) or
install it into the persisted volume so it survives redeploys. Allow extra shared
memory (e.g. `shm_size: 2gb`) when running Chromium in-container. Confirm the exact
steps against https://docs.openclaw.ai/install/docker.

## Updating

Bump the pinned image tag in `docker-compose.yaml`, then pull and re-deploy:

```
docker compose pull
docker compose up -d
```
