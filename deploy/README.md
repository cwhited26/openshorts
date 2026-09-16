# Production deploy options

Nothing is deployed yet. Chase picks the production host after the local smoke test
passes (see `SETUP.md`). The target hostname is `shorts.buildoutstudios.com`, which
matches the BOS Custom pattern.

## What production has to run

| Service | Build | Port | Notes |
|---|---|---|---|
| backend | `Dockerfile` (repo root) | 8000 | FastAPI + Whisper + YOLO + FFmpeg. CPU-heavy: an 8-minute video takes 5 to 8 minutes on CPU. Needs persistent disk for `/app/output`. Healthcheck: `/health/ready`. |
| frontend | `dashboard/Dockerfile`, target `prod` | 80 | Static Vite build served by nginx. `VITE_API_URL` is baked in at **build** time; leave it empty for same-origin. |
| renderer | `render-service/Dockerfile` | 3100 | Remotion renderer. Shares the `output` volume with the backend. |

Before either deploy path, two changes are needed:

- **`dashboard/nginx.conf` is written for openshorts.app.** It redirects the apex to
  `www.openshorts.app` and sends `/gallery` and `/video/` to `api.openshorts.app`.
  Strip those blocks, or route those paths to the backend before the request reaches nginx.
- **`docker-compose.yml` is dev-only.** It bind-mounts the source tree and runs the Vite
  dev server. Production needs a `docker-compose.prod.yml` that uses the `prod` frontend
  target, drops the source bind mounts, and keeps only the `output` volume.

Set `FORWARDED_ALLOW_IPS` to the proxy's address, never `*` (see `.env.example`). Keep
`RATE_LIMIT_ENABLED` on for anything reachable from the internet.

---

## Option A (recommended): DigitalOcean droplet + `docker compose` + Caddy

This is the closest match to local: the same compose file, one box, one bill.

1. **Droplet.** CPU-Optimized, 4 vCPU / 8 GB at minimum; Whisper and YOLO are the
   bottleneck. Ubuntu LTS, Docker Engine plus the compose plugin. Attach a block-storage
   volume for `output/`.
2. **DNS.** Point an `A` record for `shorts.buildoutstudios.com` at the droplet.
3. **App.** Clone `cwhited26/openshorts`, write `.env` on the box (never commit it), then run
   `docker compose -f docker-compose.prod.yml up -d --build`. Bind the service ports to
   `127.0.0.1` only.
4. **Caddy** on the host handles TLS automatically and routes by path on one origin:

   ```caddyfile
   shorts.buildoutstudios.com {
       encode gzip
       @api path /api/* /videos/* /thumbnails/* /gallery* /video/* /health/*
       reverse_proxy @api 127.0.0.1:8000
       handle_path /render/* {
           reverse_proxy 127.0.0.1:3100
       }
       reverse_proxy 127.0.0.1:8080   # frontend nginx (map container :80 -> host :8080)
   }
   ```

   Check the `/render` prefix against `dashboard/vite.config.js` before going live. The dev
   proxy forwards `/render` to the renderer unchanged, so `handle` without prefix stripping
   may be the right choice there.
5. **Ops.** `ufw` should allow only 22, 80, and 443. Turn on unattended-upgrades, weekly
   `docker system prune`, and DigitalOcean volume snapshots for `output/`.

**Pros:** no platform translation layer, predictable cost, easy to SSH in and debug
FFmpeg. **Cons:** the box is ours to patch, and there's no autoscaling.

---

## Option B: Fly.io + Docker

Fly doesn't run compose files, so each service becomes its own Fly app on the private
`.internal` network.

1. **Three apps:** `openshorts-api` (root `Dockerfile`), `openshorts-web` (`dashboard/Dockerfile`,
   `prod` target), and `openshorts-render` (`render-service/Dockerfile`). Each gets its own
   `fly.toml` under `deploy/fly/`.
2. **Storage.** Fly volumes attach to one machine and can't be shared. Backend and renderer
   both write to `output/`, so either run them in one machine (a process group, or a combined
   image) or turn on S3 (`AWS_S3_BUCKET`) and treat local disk as scratch space.
3. **Sizing.** `performance-4x` / 8 GB for the API. Fly GPU machines (`a10`, `l40s`) are an
   option later if CPU clip times hurt. They pair with the upstream `GPU: "1"` build arg.
4. **Secrets.** Use `fly secrets set AWS_ACCESS_KEY_ID=... -a openshorts-api` and never
   commit `.env`.
5. **Routing.** Put `shorts.buildoutstudios.com` on `openshorts-web`. Build it with
   `VITE_API_URL=https://api.shorts.buildoutstudios.com`, or have nginx proxy
   `/api` to `openshorts-api.internal:8000`. Update `ALLOWED_ORIGINS` to match.
6. **Machines.** Set `auto_stop_machines = "off"` on the API. Long clip jobs run in-process,
   and a scale-to-zero stop kills them mid-render.

**Pros:** managed TLS, global anycast, easy GPU upgrade path. **Cons:** three apps to keep in
sync, volume limits push S3 from optional to required, and more moving parts than the droplet.

---

## Decision checklist (Chase)

- [ ] Pick a host: droplet (A) or Fly (B)
- [ ] Confirm the subdomain `shorts.buildoutstudios.com`
- [ ] Decide whether S3 is on in prod (required for Fly, optional on a droplet)
- [ ] Decide on access control. The base BYOK stack has no login, so anyone who reaches
      the URL can use it. Put it behind Cloudflare Access or Caddy `basicauth` until
      the cloud/billing mode (`docker-compose.cloud.yml`) is evaluated.
