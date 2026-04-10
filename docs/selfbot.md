# Selfbot Setup

The selfbot is the core of Sentinel. It connects to Discord as a regular user account, collects behavioral data on your tracked targets, and exposes everything through a local API.

---

## Before You Start

You need:

1. **A dedicated Discord account** — Do not use your main account. Create a fresh one. Running automated code on a user account violates Discord's Terms of Service.
2. **Node.js 18 or newer** — Check with `node --version`. Download from [nodejs.org](https://nodejs.org) if needed.
3. **Your Discord token** — The token of the dedicated account, not a bot token.

### How to Get Your Discord Token

1. Open Discord in a browser (not the desktop app)
2. Press `F12` to open DevTools
3. Go to the **Network** tab
4. Send any message or change any setting to generate a request
5. Find a request to `discord.com/api`
6. Look for the `Authorization` header in the request headers
7. That value is your token — keep it private

---

## Installation

### Step 1 — Get the code

```bash
git clone https://github.com/Privex-chat/sentinel-selfbot.git
cd sentinel-selfbot
```

Or download the ZIP from the GitHub releases page.

### Step 2 — Install dependencies

```bash
npm install
```

### Step 3 — Create your config

```bash
cp .env.example .env
```

Open `.env` in any text editor and fill in the values:

```env
DISCORD_TOKEN=your_dedicated_account_token_here
API_PORT=48923
API_AUTH_TOKEN=generate_a_random_string_here
DB_PATH=./data/sentinel.db
LOG_LEVEL=info
PROFILE_POLL_INTERVAL_MS=300000
STATUS_POLL_INTERVAL_MS=120000
DAILY_SUMMARY_INTERVAL_MS=3600000
```

To generate a secure random token for `API_AUTH_TOKEN`:

```bash
# On Linux/macOS/WSL
openssl rand -hex 32

# On Windows PowerShell
-join ((65..90 + 97..122 + 48..57) | Get-Random -Count 48 | ForEach-Object {[char]$_})
```

### Step 4 — Build and start

```bash
npm run build
npm start
```

You should see:

```
[INFO] [Sentinel] === Sentinel Starting ===
[INFO] [Database] Database initialized with WAL mode
[INFO] [API] API server listening on port 48923
[INFO] [Gateway] Connecting to gateway...
[INFO] [Gateway] READY! Logged in as YourUsername#0 | 50 guilds
[INFO] [Sentinel] === Sentinel Fully Operational ===
```

### Step 5 — Verify the API

```bash
curl -H "Authorization: Bearer YOUR_API_AUTH_TOKEN" http://localhost:48923/api/status
```

You should receive a JSON response with uptime, event count, and database size.

---

## Configuration Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `DISCORD_TOKEN` | — | User account token for the dedicated Discord account |
| `API_PORT` | `48923` | Port the selfbot API listens on |
| `API_AUTH_TOKEN` | — | Bearer token required by all API clients (plugin, web) |
| `DB_PATH` | `./data/sentinel.db` | Path to the SQLite database file |
| `LOG_LEVEL` | `info` | Logging verbosity: `debug`, `info`, `warn`, `error` |
| `PROFILE_POLL_INTERVAL_MS` | `300000` | How often to re-fetch user profiles (5 minutes) |
| `STATUS_POLL_INTERVAL_MS` | `120000` | How often to re-poll presence via guild member chunk (2 minutes) |
| `DAILY_SUMMARY_INTERVAL_MS` | `3600000` | How often to compute daily stat rows (1 hour) |
| `RANDOM_JITTER` | `false` | Adds ±20% random jitter to polling intervals and randomises gateway IDENTIFY fingerprint |
| `DB_MODE` | `local` | Database mode: `local`, `local+cloud`, or `cloud` |
| `SUPABASE_URL` | — | Required when `DB_MODE` is not `local` |
| `SUPABASE_SERVICE_KEY` | — | Required when `DB_MODE` is not `local` |
| `SUPABASE_SYNC_INTERVAL_MS` | `300000` | How often to push to Supabase (default 5 min; recommend 30 s for `cloud` mode) |

---

## Adding Your First Target

### Via the API

```bash
curl -X POST \
  -H "Authorization: Bearer YOUR_API_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"userId":"123456789012345678","label":"Optional label"}' \
  http://localhost:48923/api/targets
```

### Via the Plugin or Web Dashboard

Use the **+ Add Target** button in the dashboard and enter the user's Discord ID. You can also right-click any user in Discord (with the plugin installed) and select **Track with Sentinel**.

### Finding a Discord User ID

Enable Developer Mode in Discord: **Settings → Advanced → Developer Mode**. Then right-click any user and select **Copy User ID**.

---

## Keeping It Running

### Using PM2 (recommended for VPS / always-on)

```bash
npm install -g pm2
pm2 start dist/index.js --name sentinel
pm2 save
pm2 startup   # follow the printed instruction to enable autostart
```

Useful commands:

```bash
pm2 logs sentinel      # live logs
pm2 restart sentinel   # restart
pm2 stop sentinel      # stop
pm2 status             # check if running
```

### Docker

A `Dockerfile` is included. Build and run:

```bash
docker build -t sentinel-selfbot .
docker run -d \
  -p 48923:48923 \
  -v /path/to/data:/data \
  -e DISCORD_TOKEN=your_token \
  -e API_AUTH_TOKEN=your_api_token \
  -e DB_PATH=/data/sentinel.db \
  --name sentinel \
  sentinel-selfbot
```

### Railway / Render / Fly.io

A `railway.toml` and `fly.toml` are included in the repo. Set your environment variables (`DISCORD_TOKEN`, `API_AUTH_TOKEN`, `DB_MODE=cloud`, `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`) in the platform's dashboard, then deploy.

**Important:** Use `DB_MODE=cloud` on ephemeral platforms so data survives redeployments. See [supabase.md](supabase.md).

---

## Running on a Remote Server

If the selfbot runs on a VPS or cloud platform and you want to connect to it from the Vencord plugin:

- The plugin can reach the API over the network, but it needs the proxy if you're on Windows (see [proxy.md](proxy.md))
- Configure the plugin's **Sentinel URL** to your server's IP or domain (e.g., `http://123.45.67.89:48923` or `https://sentinel.yourdomain.com`)
- Make sure port `48923` is open in your firewall / security group
- For HTTPS, put Nginx or Caddy in front of the selfbot proxying to `localhost:48923`

Alternatively, use [sentinel-web](https://github.com/Privex-chat/sentinel-web) to access your data from any browser without needing the plugin.

---

## Troubleshooting

**Selfbot won't connect to Discord**

- Make sure your token is a user account token, not a bot token. Bot tokens have three dot-separated sections.
- Error code `4004` → token is wrong or expired.
- Error code `4013` or `4014` → intents issue. Confirm you're using a user token, not a bot token.

**No presence data for a tracked target**

- Sentinel can only observe events from servers that both the selfbot account and the target share. If there are no mutual servers, presence won't update.
- Add the dedicated account to a server that the target is also in.

**Database growing large**

- Each event is a row. Active targets with lots of messages generate thousands of rows per day.
- SQLite handles hundreds of millions of rows without issue, but if disk is a concern, increase `PROFILE_POLL_INTERVAL_MS` and `STATUS_POLL_INTERVAL_MS`.

**High memory usage**

- The 64MB SQLite cache is expected. The selfbot itself is lightweight. Check Node.js version and look for memory leaks if usage is unexpectedly high.

**Starting fresh**

- Stop the selfbot, delete `./data/sentinel.db`, and restart. All data is lost but the process starts clean.
