# Deploy on Dokploy

Notes for deploying this stack as a Dokploy Compose application on a Hetzner
FSN1 host. Full product docs live at https://docs.openclaw.ai.

## Prerequisites

- Host OS **Ubuntu 24.04 LTS** (or 22.04). Do not use pre-release / non-LTS
  Ubuntu such as 26.04 "resolute" — Docker's apt repo has no packages for that
  codename, so the Dokploy Docker install fails with
  `'<version>' not found amongst apt-cache madison results`.
- A Dokploy host (Docker Swarm) with Traefik v3 configured and a `letsencrypt`
  certresolver.
- The `dokploy-network` overlay network already present. Dokploy creates it on
  install. Verify with `docker network ls`.
- A DNS A record pointing your host (e.g. `claw.example.es`) at the server.

## Steps

1. In Dokploy, create a new Compose application and point it at this repo.
2. Set the compose path to `docker-compose.yaml`.
3. Open [docker-compose.yaml](../docker-compose.yaml) and replace the placeholder
   host `claw.example.es` in the Traefik router label with your real hostname.
4. Add the environment variables from [.env.example](../.env.example) in the
   Dokploy UI. At minimum: one provider key, `AUTH_PASSWORD`, and
   `OPENCLAW_GATEWAY_TOKEN`. Generate secrets with `openssl rand`.
5. Deploy. Traefik issues the TLS cert on first request to the hostname.
6. After boot, exec into the container and run `openclaw doctor` to validate.

## Browser sidecar

The headless Chrome sidecar ships in the base `docker-compose.yaml` and starts
with the stack — no extra file or Dokploy config needed. It is heavy (large image,
meaningful RAM and CPU cost), so factor that into host sizing.

## Updating

Pull a new image and re-deploy:

```
docker compose pull
docker compose up -d
```
