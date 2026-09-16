# SETUP — Whited Consulting fork

Fork of [`mutonby/openshorts`](https://github.com/mutonby/openshorts) (MIT), kept at
[`cwhited26/openshorts`](https://github.com/cwhited26/openshorts). This file covers
running it locally on Chase's machine. For the product background, see the upstream
`README.md`. For production hosting options, see `deploy/README.md`.

## Remotes

```bash
origin    https://github.com/cwhited26/openshorts.git   # our fork, push here
upstream  https://github.com/mutonby/openshorts.git     # pull fixes from here
```

To sync upstream:

```bash
git fetch upstream
git merge upstream/main
git push origin main
```

## Prerequisites

- Docker Desktop (or OrbStack) running: `docker --version && docker compose version`
- About 10 GB of free disk for the images (Python + ML deps, Node, Remotion renderer)

## 1. Server-side config: `.env`

`.env` is gitignored. Create it from the example:

```bash
cp .env.example .env
```

| Variable | Required | Notes |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | No | Leave empty to disable S3. A placeholder string makes boto3 try to upload and fail. |
| `AWS_SECRET_ACCESS_KEY` | No | Same as above |
| `AWS_REGION` | No | `us-east-1` |
| `AWS_S3_BUCKET` | No | Private bucket for clip backup |
| `AWS_S3_PUBLIC_BUCKET` | No | Public bucket for the gallery and avatars |
| `MAX_CONCURRENT_JOBS` | No | Defaults to `5` |

The backend reads `.env` with `python-dotenv` from the bind-mounted `/app` directory,
so changes take effect after `docker compose restart backend`.

## 2. Launch

```bash
docker compose up --build -d
docker compose ps
curl -sf http://localhost:5175 > /dev/null && echo OK
```

| Service | Container | Port |
|---|---|---|
| `frontend` | `openshorts-frontend` | http://localhost:5175 (Vite dev server with HMR) |
| `backend` | `openshorts-backend` | http://localhost:8000 (FastAPI) |
| `renderer` | `openshorts-renderer` | http://localhost:3100 (Remotion) |

Logs: `docker compose logs -f backend`. Stop: `docker compose down`.

## 3. Client-side API keys: enter them in the dashboard, not in `.env`

Open http://localhost:5175, go to **Settings**, and paste each key. The keys are
encrypted in the browser's localStorage and sent with each request. They are never
written to the server or committed.

| Key | Needed for | Get it |
|---|---|---|
| `GEMINI_API_KEY` | Every AI feature (moment detection, scripts, thumbnails) | https://aistudio.google.com/app/apikey (free) |
| `FAL_KEY` | AI Shorts (actor, video, lip-sync, b-roll) | https://fal.ai (pay per use) |
| `ELEVENLABS_API_KEY` | Voiceover and dubbing | https://elevenlabs.io (free tier) |
| `UPLOAD_POST_API_KEY` | Auto-publishing to TikTok, Reels, and YouTube | https://upload-post.com (free tier) |

Keys live in the browser, so a different browser or profile needs them entered again.

## 4. Smoke test (after the keys are in)

1. Clip Generator: upload a short WC webinar clip and confirm the 9:16 shorts render.
2. AI Shorts: generate one short from a product description (costs about $0.50 to $1.50 on fal.ai).
3. Publish one clip through Upload-Post to a test account.

## Status

- 2026-09-16: forked and cloned, `.env` scaffolded. Local launch is blocked until Docker is installed on this machine.
