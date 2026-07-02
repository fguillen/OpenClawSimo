# Deploy on Dokploy

Notes for deploying this stack as a Dokploy Compose application on a Hetzner
FSN1 host. Full product docs live at https://docs.openclaw.ai.

## Prerequisites

- Host OS **Ubuntu 24.04 LTS** (or 22.04). Do not use pre-release / non-LTS
  Ubuntu such as 26.04 "resolute" — Docker's apt repo has no packages for that
  codename, so the Dokploy Docker install fails with
  `'<version>' not found amongst apt-cache madison results`.
- A Dokploy host (Docker Swarm). Traefik and a public certresolver are not needed
  for the Control UI, which stays private.
- The `dokploy-network` overlay network already present. Dokploy creates it on
  install. Verify with `docker network ls`.
- No public DNS record or Domain is needed. The Control UI is reached over an SSH
  tunnel. See Access below.

## Steps

1. In Dokploy, create a new Compose application and point it at this repo.
2. Set the compose path to `docker-compose.yaml`.
3. Do **not** add a public Domain. The Control UI is admin only. If this app
   already has a Domain in the **Domains** tab, remove it. Compose publishes nginx
   to the host loopback (`127.0.0.1:8080`), reached over an SSH tunnel. See Access.
4. In the app's **Environment Settings**, add the variables from
   [.env.example](../.env.example). `AUTH_PASSWORD` and `OPENCLAW_GATEWAY_TOKEN`
   are required — the compose file fails the deploy if either is unset — plus at
   least one provider key. Generate secrets with `openssl rand`.
5. Create the declarative config file. In the app's **Advanced → Volumes / Mounts**,
   add a **File Mount** at mount path `config/openclaw.json` (Dokploy writes it to
   `../files/config/openclaw.json`, which the compose file bind-mounts as the directory
   `/config`). Paste the contents of [openclaw.example.json](../openclaw.example.json).
   A **directory** is mounted (not the single file) because OpenClaw rewrites this file
   in place at boot via an atomic `rename()`, which fails with `EBUSY` on a single-file
   bind mount. Do this **before deploying** — if `/config` is empty the config path won't
   exist. Unlike the old repo-clone approach, this file persists across redeploys.
6. Deploy. The stack starts with no public route.
7. After boot, validate with `docker compose exec openclawsimo openclaw doctor`.

## Access (SSH tunnel)

The Control UI is an admin surface (chat, config, exec approvals). It is never
public. There is one UI, served by the gateway and fronted by nginx.

| Port  | Role                                                        | Exposure                                              |
|-------|-------------------------------------------------------------|-------------------------------------------------------|
| 8080  | nginx front door, adds basic auth, proxies to the Control UI | published to host `127.0.0.1` only, reach via SSH tunnel |
| 18789 | gateway, serves the Control UI over WebSocket                | loopback inside the container, never published        |

Runbook:

```
ssh -N -L 8080:127.0.0.1:8080 user@YOUR_HOST
# open http://localhost:8080, pass nginx basic auth
# paste OPENCLAW_GATEWAY_TOKEN into the Control UI settings, then approve the device
```

Approve the device from inside the container:

```
docker compose exec openclawsimo openclaw devices list
docker compose exec openclawsimo openclaw devices approve <id>
```

If the Control UI shows "could not connect", check these in order:

- Non-secure context. Open `http://localhost:8080`, not a raw LAN or tailnet IP
  over plain HTTP. Browsers treat non-localhost plain HTTP as insecure and block
  the WebSocket the Control UI needs.
- Device pairing. A new browser is closed with code 1008 until you approve it with
  `openclaw devices approve <id>`.
- Token. Paste `OPENCLAW_GATEWAY_TOKEN` into the Control UI settings.
- Origin. `OPENCLAW_ALLOWED_ORIGINS` must include `http://localhost:8080` (match
  the port you forward).

Later: Tailscale Serve is the upgrade path for access without a tunnel. It keeps
the gateway loopback only and puts the Control UI on a private tailnet URL.

Security note: the Control UI is an admin surface. Never expose it publicly. See
https://docs.openclaw.ai/web/dashboard

## Browser sidecar

The full Chrome (Kasm/noVNC) sidecar ships in the base `docker-compose.yaml` and
starts with the stack — no extra file or Dokploy config needed. It exposes a CDP
endpoint on `:9222` (in-network) that OpenClaw drives, and a web desktop on `:6901`
for manual login to auth-gated sites. It is heavy (large image, meaningful RAM and
CPU cost), so factor that into host sizing.

## Updating

Pull a new image and re-deploy:

```
docker compose pull
docker compose up -d
```
