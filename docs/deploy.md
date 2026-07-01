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
3. Open the app's **Domains** tab and add a domain: service `openclaw`, container
   port `8080`, HTTPS enabled with the `letsencrypt` certresolver. Dokploy
   generates the Traefik routing config — the compose file carries no Traefik
   labels.
4. In the app's **Environment Settings**, add the variables from
   [.env.example](../.env.example). `AUTH_PASSWORD` and `OPENCLAW_GATEWAY_TOKEN`
   are required — the compose file fails the deploy if either is unset — plus at
   least one provider key. Generate secrets with `openssl rand`.
5. Deploy. Traefik issues the TLS cert on first request to the hostname.
6. After boot, exec into the container and run `openclaw doctor` to validate.

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
