# Deploy on Dokploy

Notes for deploying this stack as a Dokploy Compose application on a Hetzner
FSN1 host. Full product docs live at https://docs.openclaw.ai.

## Prerequisites

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

## Browser overlay

To add the headless Chrome sidecar, deploy with both files:

```
docker compose -f docker-compose.yaml -f docker-compose.browser.yaml up -d
```

In Dokploy, set the compose path to include both files, or add
`docker-compose.browser.yaml` as an additional compose file on the app. This is
heavy and opt-in. Skip it unless you need browser automation.

## Updating

Pull a new image and re-deploy:

```
docker compose pull
docker compose up -d
```
