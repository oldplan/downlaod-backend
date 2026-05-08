# Fetch. — Backend

Node 20 / Express / TypeScript service that wraps yt-dlp behind a small REST API and a Bull worker.

---

## Deploy on Railway (recommended)

`nixpacks.toml`, `railway.json` and a `Procfile` are already in this folder, so a fresh Railway project will Just Work.

1. **New project → Deploy from GitHub** (or `railway up` from this folder).
2. **Add the Redis plugin** in the same project. Railway auto-injects `REDIS_URL` into your service.
3. **Set the env vars** (Settings → Variables). Only one is *required* — `REDIS_URL` is auto-set by the Redis plugin. Everything else has sensible defaults:

   ```
   NODE_ENV=production
   YTDLP_PATH=yt-dlp           # Nixpacks puts yt-dlp on $PATH
   TEMP_DIR=/tmp
   FILE_TTL_MINUTES=10
   RATE_LIMIT_MAX=20
   MAX_CONCURRENT_DOWNLOADS=5
   ```

4. Railway exposes the service over HTTPS at `https://<service>.up.railway.app`. `GET /health` is wired as the healthcheck and returns `{"success":true,"data":{"status":"ok"}}`.

> ⚠️ **Vercel will not work** for this backend. Vercel's serverless functions have 10–60 s timeouts, no persistent `/tmp`, and you can't bundle `yt-dlp` + `ffmpeg` into the function image. Use Railway, Render, Fly.io, or any container/VM host with a long-running process.

### How the build works

`nixpacks.toml` tells Railway to install `nodejs_20`, `python3`, `ffmpeg` and `yt-dlp` from Nixpkgs, then runs `npm ci` → `npm run build` → `node dist/index.js`.

---

## Deploy on Render / Fly.io / a plain VM

Same recipe — install the system deps and run the same start command.

```bash
# system deps
sudo apt-get install -y python3 ffmpeg
sudo curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp \
  -o /usr/local/bin/yt-dlp && sudo chmod +x /usr/local/bin/yt-dlp

# app
cp .env.example .env
npm ci
npm run build
node dist/index.js
```

Don't forget a Redis instance (Upstash, Render Redis, Railway Redis, etc.) and set `REDIS_URL`.

---

## Run locally

```bash
# pre-reqs: Node 20+, redis-server, ffmpeg, yt-dlp on $PATH
cp .env.example .env
npm install
npm run dev      # tsx watcher
```

Or with Docker if you want everything in one shot (`docker-compose.yml` is included for local dev):

```bash
cp .env.example .env
docker compose up --build
```

---

## API

```
POST /api/fetch-info       { url, platform }
POST /api/download         { url, platform, formatId? }   → 202 { jobId }
GET  /api/progress/:jobId                                 → { status, progress, … }
GET  /api/file/:jobId                                     → streamed mp4 (Range)
GET  /health                                              → liveness probe
```

All endpoints return `{ success: true, data }` on success and `{ success: false, error, message }` on failure. Error codes the frontend acts on:

* `INVALID_URL`, `UNSUPPORTED_PLATFORM`, `PLATFORM_MISMATCH`
* `PRIVATE_VIDEO`, `GEO_BLOCKED`, `NOT_FOUND`
* `RATE_LIMITED`, `JOB_NOT_FOUND`, `NOT_READY`, `INTERNAL_ERROR`

## yt-dlp invocation

```
common: --no-warnings --no-call-home --no-playlist --ignore-config --no-color
fetch:  --dump-single-json
download:
  --newline --progress --progress-template 'PRG %(progress._percent_str)s|…'
  -f <selector> -o /tmp/fetch_<jobId>.%(ext)s --no-part --no-mtime
```

Format selectors:

| Platform | `-f` | Extra |
| --- | --- | --- |
| YouTube | `bestvideo[ext=mp4]+bestaudio[ext=m4a]/best[ext=mp4]/best` | `--merge-output-format mp4` |
| Instagram | `best[ext=mp4]/best` | — |
| Snapchat | `best[ext=mp4]/best` | — |

## Cleanup

* `cleanup.service.ts` runs every 60 s and deletes any `/tmp/fetch_*` file older than `FILE_TTL_MINUTES` (default 10).
* `scheduleFileDeletion()` is also called the first time `/api/file/:jobId` is hit, so the typical lifetime is `now + 10 min`.

## Rate limiting

`express-rate-limit` runs in `draft-7` mode with a 60 s window and `RATE_LIMIT_MAX` (20 by default) on `/api/*`. The file route gets 2× that to allow large downloads + range resumes.

---

## Wiring the mobile app

After Railway gives you a public URL (e.g. `https://fetch-backend.up.railway.app`), edit `frontend/app.json`:

```json
"extra": {
  "apiBaseUrl": "https://fetch-backend.up.railway.app"
}
```

Reload the Expo app and downloads will route through your hosted backend.
