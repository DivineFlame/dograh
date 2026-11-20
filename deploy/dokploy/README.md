# Deploying Dograh on Dokploy

This folder holds a Dokploy-ready version of the Dograh stack. It uses the
prebuilt `dograhai/*` images and Dokploy's built-in Traefik for HTTPS.

| File | Purpose |
|---|---|
| `docker-compose.yaml` | The stack: postgres, redis, minio, coturn, api, ui, with Traefik labels for routing and TLS |
| `.env.example` | Variables to paste into Dokploy's **Environment** tab |

Routing on one domain (`PUBLIC_HOST`):

| Path | Service |
|---|---|
| `/api/v1/*` (REST + signaling WebSocket) | `api:8000` |
| `/voice-audio/*` | `minio:9000` |
| everything else | `ui:3010` |

HTTP is redirected to HTTPS; certificates come from Let's Encrypt.

## Before you deploy

1. **DNS:** create an A record for your domain (e.g. `voice.octave-consultancy.com`)
   pointing at the Dokploy server's public IP.
2. **Firewall:** open these on the server (and in your cloud provider's
   security group). Calls connect without them but have **no audio**.
   - TCP 80, 443 (Traefik — normally already open for Dokploy)
   - UDP + TCP 3478 and 5349 (TURN)
   - UDP 49152–49200 (TURN relay range)
3. **Size:** 2 vCPU / 4 GB RAM is a sensible minimum (`api` runs the voice
   pipeline, migrations and background workers in one container).

## Set up in Dokploy

1. **Create a project**, then **Create Service → Compose** (type: Docker Compose).
2. **General → Provider:** GitHub (or Git) →
   repository `DivineFlame/dograh`, branch `main`,
   **Compose Path** `./deploy/dokploy/docker-compose.yaml`.
3. **Environment:** paste `.env.example`, then fill in `PUBLIC_HOST`,
   `TURN_HOST` and every `change-me` with a fresh secret
   (`openssl rand -hex 32`).
4. **Domains tab:** leave it empty. Routing is already defined by the labels
   in the compose file; adding a domain there too creates duplicate routers.
5. **Advanced:** leave *Isolated Deployments* off (the compose file joins
   `dokploy-network` itself).
6. Click **Deploy**. First boot pulls images and runs DB migrations; `ui`
   waits until `api` is healthy (allow 1–3 minutes).
7. Open `https://<PUBLIC_HOST>` and create the first account. Then consider
   setting `ENABLE_SIGNUP=false` and redeploying.

Enable **Autodeploy** in Dokploy (or add the GitHub webhook) if you want each
push to `main` to redeploy.

## Notes

- **Upgrades:** set `DOGRAH_VERSION` to a release tag and redeploy. With
  `latest`, a redeploy only picks up new images if Dokploy pulls them —
  pinning a tag is more predictable.
- **Postgres password** is fixed at the first deploy. Changing
  `POSTGRES_PASSWORD` later breaks the API's DB connection unless you also
  change it inside Postgres (`ALTER USER postgres PASSWORD '...'`).
- **Backups:** the data lives in the `postgres_data` and `minio-data`
  volumes. Use Dokploy's volume backups or your own `pg_dump` schedule.
- **MinIO console** is not exposed. Reach it with an SSH tunnel to the minio
  container's port 9001 if needed.
- **Single worker:** `FASTAPI_WORKERS` is pinned to 1 (see the comment in the
  compose file). Scale the server up rather than adding replicas.
- **Two instances on one server** (e.g. staging): give each a different
  `PUBLIC_HOST` and `ROUTER_PREFIX`. coturn's host ports can only be bound
  once per server, so the second instance needs its coturn ports changed or
  removed.

## Troubleshooting

| Symptom | Check |
|---|---|
| Certificate error / Traefik default cert | DNS not pointing at the server yet, or port 80 blocked (HTTP-01 challenge). Redeploy after fixing. |
| 404 from Traefik | `PUBLIC_HOST` mismatch, or a duplicate domain added in the Domains tab. |
| UI loads, calls connect, no audio | TURN ports closed, or `TURN_HOST` isn't the public IP. |
| Deploy fails with "… is required" | A required variable is missing from the Environment tab. |
| API restarting | `docker logs` on the api container — usually DB auth (see Postgres password note). |
