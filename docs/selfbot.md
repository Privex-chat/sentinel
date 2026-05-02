# sentinel-selfbot — Setup & Configuration Guide

The selfbot is the core data-collection engine of the Sentinel ecosystem. It connects to Discord as a real user account, tracks behavioral data on specified targets, and exposes everything through a local REST/SSE API.

---

## Prerequisites

- **Node.js 18+** (LTS recommended)
- **A Discord user account token** — not a bot token. See the [token extraction guide](#getting-your-discord-token) below.
- **At least one mutual Discord server** with each target you want to track (required for presence data)
- *(Optional)* A Supabase project for cloud sync / cloud deployments
- *(Optional)* An AI API key (Google Gemini free tier recommended)

---

## Installation

```bash
git clone https://github.com/Privex-chat/sentinel-selfbot.git
cd sentinel-selfbot
npm install
cp .env.example .env
```

Edit `.env` with your values, then:

```bash
npm run build
npm start
```

The API server starts on port `48923` by default.

---

## Getting Your Discord Token

> **Warning:** Your Discord token is equivalent to your account password. Never share it.

1. Open Discord in a browser (not the desktop app)
2. Open DevTools → **Network** tab
3. Refresh the page and find any request to `discord.com/api`
4. Look in the request headers for `Authorization` — the value is your token
5. Copy the full token (no `Bearer` prefix needed)

---

## Environment Variables

### Core (requires restart to change)

| Variable | Default | Description |
|---|---|---|
| `DISCORD_TOKEN` | *(required)* | Your Discord account token |
| `API_PORT` | `48923` | Port the REST API listens on |
| `API_AUTH_TOKEN` | *(required)* | Secret token for API authentication (use a random 32+ char string) |
| `DB_PATH` | `./data/sentinel.db` | SQLite database file path |
| `LOG_LEVEL` | `info` | Logging verbosity: `debug`, `info`, `warn`, `error` |
| `RANDOM_JITTER` | `false` | Add ±20% jitter to polling intervals; randomise gateway fingerprint |
| `DB_MODE` | `local` | Database mode: `local`, `local+cloud`, or `cloud` |

### Runtime-reloadable (hot-reloadable via `PATCH /api/config` or `$reload`)

| Variable | Default | Description |
|---|---|---|
| `PROFILE_POLL_INTERVAL_MS` | `300000` | How often to fetch full profile data (5 min) |
| `STATUS_POLL_INTERVAL_MS` | `120000` | How often to run presence confirmation polls (2 min) |
| `DAILY_SUMMARY_INTERVAL_MS` | `3600000` | How often to recompute daily summaries (1 hour) |
| `AI_PROVIDER` | `none` | AI provider: `none`, `gemini`, `openai`, `anthropic`, `ollama` |
| `AI_MODEL` | `gemini-2.5-flash` | Model name for the selected provider |
| `AI_API_KEY` | — | API key for the AI provider |
| `AI_BASE_URL` | `http://localhost:11434/v1` | Base URL (used for Ollama and OpenAI-compatible endpoints) |
| `AI_ANALYSIS_INTERVAL_MS` | `86400000` | How often to run AI analysis cycles (24 hours) |
| `AI_CATEGORIZATION_BATCH_SIZE` | `50` | Messages processed per AI categorization batch |
| `BACKFILL_ENABLED` | `true` | Whether to backfill message history for new targets |
| `BACKFILL_MAX_DAYS` | `90` | How far back to backfill messages |
| `BACKFILL_MAX_MESSAGES_PER_CHANNEL` | `5000` | Cap per channel during backfill |
| `ALERT_WEBHOOK_URL` | — | Discord webhook URL for alert notifications |
| `CRITICAL_WEBHOOK_URL` | — | Separate webhook for critical system errors only |
| `ALERT_DIGEST_MODE` | `false` | Batch alerts into periodic summaries instead of firing per-event |
| `ALERT_DIGEST_INTERVAL_MS` | `900000` | Digest flush interval (15 min) |
| `ALERT_FATIGUE_THRESHOLD` | `20` | Auto-suppress a rule after this many fires in 24 hours |
| `BRIEF_GENERATION_TIME` | `07:00` | Time (UTC, HH:MM) to generate daily AI briefs |

---

## Database Modes

### `local` (default)

SQLite only. All data stays on disk at `DB_PATH`. Use this for home servers, VPS instances, and any machine with a persistent filesystem.

```env
DB_MODE=local
```

### `local+cloud`

SQLite is the live database; a copy is pushed to Supabase on the sync interval. Good for self-hosted setups that want a cloud backup or cross-device access.

```env
DB_MODE=local+cloud
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_KEY=your_service_role_key
SUPABASE_SYNC_INTERVAL_MS=300000
```

### `cloud`

For ephemeral hosts (Railway, Render, Fly.io) where the filesystem is wiped on redeploy. On startup, Sentinel pulls all data from Supabase into a fresh local SQLite before connecting to Discord. Set the sync interval low so minimal data is lost if the container is killed.

```env
DB_MODE=cloud
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_KEY=your_service_role_key
SUPABASE_SYNC_INTERVAL_MS=30000
```

You must run `supabase-schema.sql` against your Supabase project before first use in cloud mode.

---

## AI Setup

AI is fully optional. When disabled (`AI_PROVIDER=none`), all core tracking still works — only social graph analysis, message categorization, and daily briefs are unavailable.

### Google Gemini (recommended — free)

Free tier gives 15 requests/minute and 1 million tokens/day.

```env
AI_PROVIDER=gemini
AI_MODEL=gemini-2.5-flash
AI_API_KEY=your_key_from_aistudio.google.com
```

Get a key at [aistudio.google.com](https://aistudio.google.com).

### Ollama (local, private)

Runs entirely on your machine. Requires Ollama installed and a model pulled (`ollama pull llama3.2`). To reach it from a cloud selfbot, expose it with ngrok:

```bash
ngrok http 11434 --host-header="localhost:11434"
```

```env
AI_PROVIDER=ollama
AI_MODEL=llama3.2
AI_BASE_URL=https://your-ngrok-url.ngrok-free.app/v1
AI_API_KEY=  # leave blank
```

### OpenAI

```env
AI_PROVIDER=openai
AI_MODEL=gpt-4o-mini
AI_API_KEY=sk-...
AI_BASE_URL=https://api.openai.com/v1
```

### Anthropic

```env
AI_PROVIDER=anthropic
AI_MODEL=claude-3-5-haiku-20241022
AI_API_KEY=sk-ant-...
```

---

## Deployment

### Local / VPS

```bash
npm run build && npm start
```

With PM2 for auto-restart:

```bash
npm install -g pm2
pm2 start npm --name sentinel-selfbot -- start
pm2 save && pm2 startup
```

### Docker

```bash
docker build -t sentinel-selfbot .
docker run -d \
  --name sentinel-selfbot \
  -p 48923:48923 \
  -v $(pwd)/data:/app/data \
  --env-file .env \
  sentinel-selfbot
```

### Railway (one-click)

[![Deploy on Railway](https://railway.app/button.svg)](https://railway.com/deploy/sentinel-selfbot?referralCode=zpvHsG&utm_medium=integration&utm_source=template&utm_campaign=generic)

Set these environment variables in the Railway dashboard:
- `DISCORD_TOKEN`, `API_AUTH_TOKEN`
- `DB_MODE=cloud`
- `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`
- `SUPABASE_SYNC_INTERVAL_MS=30000`
- `RANDOM_JITTER=true`

### Fly.io

```bash
fly launch   # uses fly.toml
fly secrets set DISCORD_TOKEN=... API_AUTH_TOKEN=...
fly deploy
```

Use `DB_MODE=cloud` with Supabase for persistent storage.

---

## Adding Your First Target

**Via self-command** (recommended — no trace):
```
$add @username
```
Type this in any mutual Discord channel. The message deletes instantly; tracking starts within 5 seconds.

**Via REST API:**
```bash
curl -X POST http://localhost:48923/api/targets \
  -H "Authorization: your_api_auth_token" \
  -H "Content-Type: application/json" \
  -d '{"userId": "123456789012345678"}'
```

**Via the dashboard** — use sentinel-web or the Vencord plugin UI.

---

## OPSEC Recommendations

- Set `RANDOM_JITTER=true` — randomises polling intervals and the browser/OS fingerprint in Discord gateway payloads and REST headers, making traffic patterns less predictable
- Avoid adding more than one target every 15 minutes (the selfbot enforces this automatically)
- Use `CRITICAL_WEBHOOK_URL` pointing to a private channel to be notified immediately if your token is invalidated
- Self-commands delete instantly — use them in any channel rather than the API when you need to act quickly and leave no trace
- Do not run the selfbot from a residential IP that you also use to log into the Discord account normally; use a VPS

---

## Upgrading

```bash
git pull
npm install
npm run build
pm2 restart sentinel-selfbot   # or restart however you run it
```

Database migrations run automatically on startup — no manual steps needed.
