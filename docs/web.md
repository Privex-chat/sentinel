# Web Dashboard

The Sentinel web dashboard is a Next.js application that gives you full access to your selfbot's data from any browser — no Vencord, no Discord client required.

The hosted version is available at **[sentinel-panel.vercel.app](https://sentinel-panel.vercel.app)**.

---

## How It Works

The web app is a pure frontend — it has no backend of its own. It stores your selfbot URL and token in your browser's localStorage and makes all API calls directly from your browser to your selfbot. No data passes through Vercel or any third party.

This means:

- Your selfbot must be reachable from your browser. If it runs locally, you access the hosted app and point it at `http://localhost:48923`. If it runs on a VPS, you access the app and point it at your server's URL.
- Your token never leaves your browser (except in the requests you make to your own selfbot).

---

## Using the Hosted App

1. Go to [sentinel-panel.vercel.app](https://sentinel-panel.vercel.app)
2. Click **Settings** in the sidebar
3. Enter your **Sentinel API URL** (e.g., `http://localhost:48923` or `https://your-domain.com`)
4. Enter your **API Token** (the `API_AUTH_TOKEN` from your selfbot's `.env`)
5. Click **Save Configuration**

The dashboard will connect automatically and display your targets.

---

## Self-Hosting

If you want to run the web dashboard on your own server:

### Requirements

- Node.js 18 or newer
- pnpm (recommended) or npm

### Setup

```bash
git clone https://github.com/Privex-chat/sentinel-web.git
cd sentinel-web
pnpm install
pnpm build
pnpm start
```

The app runs on port `3000` by default.

### Docker

```bash
docker build -t sentinel-web .
docker run -p 3000:3000 sentinel-web
```

### Vercel (one-click)

Fork the repository and import it into Vercel. No environment variables are required — all configuration is done at runtime through the Settings page.

---

## Features

### Dashboard

The home view shows all tracked targets with their current status, what they're doing right now, and a live event feed as things happen in real time.

Global stats at the top show total event count, active targets, database size, and connection status.

### Target Detail

Click any target card to open their full profile. Tabs:

| Tab | Description |
|-----|-------------|
| **Overview** | Current status, live activity, today's stats, anomalies, recent events |
| **Timeline** | Gantt bar of today's sessions + filterable event history |
| **Analytics** | Presence distribution, gaming stats, message analysis, voice habits, music profile, social graph |
| **Messages** | Full message log, deleted messages, edited message history, ghost typing analytics |
| **Insights** | Sleep schedule estimate, weekly routine heatmap, anomaly detection |
| **Alerts** | Alert rule management and alert history |
| **Profile** | Avatar gallery, profile change timeline, connected accounts |

### Real-Time Updates

The web app maintains a persistent SSE connection to your selfbot. Events arrive in real time and automatically update the live feed and relevant analytics. On reconnect after a brief network blip, the panel passes `?since=<lastEventId>` so the selfbot replays buffered events from its 500-entry in-memory ring buffer — you don't miss events that fired during the dead window.

### Search and Filtering

The Targets page supports search by username, display name, or label. The Timeline tab supports filtering by event type. Server-side LIKE patterns (`%`, `_`) coming from the search box are escaped — a single `%` no longer triggers a full-table scan against the `data`/`content` columns.

### Per-Target Timezone

The Add Target form accepts an optional IANA timezone (defaults to the browser's detected zone, e.g. `America/New_York`). Sleep schedule, routine heatmap, and `UNUSUAL_HOUR` / `COMES_ONLINE after_hour` alert evaluation all run in that zone server-side. Edit it later from the target's settings or via the `$tz` self-command.

### Onboarding Bootstrap

Newly added targets show a small **"Bootstrapping…"** badge on the target card and an amber banner on the target detail page. While bootstrap is in progress (the initial `/users/{id}/profile` fetch hasn't landed yet), the selfbot server-side suppresses every alert evaluation and the `/insights/anomalies` endpoint returns an empty array for that target. This keeps the first wave of incomplete first-observation events out of the operator's feed.

Bootstrap normally completes within seconds of adding the target. Click **"Skip wait"** in the banner to force-complete immediately (calls `POST /api/targets/:userId/bootstrap/complete`) — useful if the immediate fetch failed (e.g. no mutual guilds AND the basic `/users/{id}` fallback also failing) and you don't want to wait for the 30-minute backstop sweep.

### Streaming Export

The full per-target export endpoint streams **NDJSON** (one row per line, framed by `_section` markers). The panel's `api.downloadExport(userId)` triggers a browser file-save without parsing megabytes of JSON on the main thread; `api.exportData(userId)` parses the stream into a `{ section: rows[] }` map for in-memory use. Multi-million-row exports complete without crashing the browser or the selfbot.

---

## Local Selfbot + Hosted Web App

This is the most common setup: selfbot running on your PC, using the hosted web app from any browser on the same machine.

Because the web app makes requests from your browser and your selfbot listens on `localhost`, this works without any special configuration as long as you access the web app from the **same machine** the selfbot runs on.

If you want to access the web app from your phone or another device on your network:

1. Find your PC's local IP address (e.g., `192.168.1.50`)
2. Make sure port `48923` is not blocked by your firewall
3. Set the Sentinel URL in web settings to `http://192.168.1.50:48923`

---

## Remote Selfbot + Any Device

If your selfbot runs on a VPS or cloud platform:

- Set the Sentinel URL to your server's public URL (e.g., `https://sentinel.yourdomain.com`)
- Make sure your server's firewall allows connections on port `48923`, or configure a reverse proxy (Nginx/Caddy) with HTTPS

This gives you access to your tracking data from any device, anywhere.

---

## Tech Stack

| Category | Technology |
|---|---|
| Framework | Next.js 15 (App Router) |
| Language | TypeScript 5 |
| Styling | Tailwind CSS 4 |
| Charts | Custom SVG + Recharts |
| Icons | Lucide React |

---

## Troubleshooting

**"Connection Failed" after entering URL and token**

- Confirm the selfbot is running (hit `GET /health` unauthenticated to check — should return `{"status":"ok",...}`)
- Confirm the URL is correct (include the port: `http://localhost:48923`)
- If using HTTPS, make sure your SSL certificate is valid
- Check that the selfbot's CORS allowlist includes your panel's origin. The selfbot ships with `https://sentinel-panel.vercel.app`, `http://localhost:5173`, and `http://localhost:3000` allowed by default. For any other origin set `API_CORS_ORIGINS` on the selfbot.

**Live feed not updating**

- Check that **Real-time Updates (SSE)** is enabled in Settings
- Some reverse proxies buffer SSE by default — add `X-Accel-Buffering: no` to your Nginx/Caddy config

**Data looks stale**

- Click the **Reconnect** button in Settings to force a fresh connection
- The SSE connection may have dropped — refreshing the page re-establishes it; the selfbot's `?since=<id>` replay catches anything that fired during the dead window (if within the 500-event buffer)

**HTTP 429 / rate-limited**

- The selfbot enforces 300 req/min/IP. Closing idle dashboard tabs that share the same egress IP usually clears it
- The 429 response carries a `Retry-After` header — the panel honours it automatically
- `/health` is allowlisted from the rate limit, so liveness probes don't burn the budget

**"Internal server error" with a `requestId`**

- The selfbot now returns generic `{ error: "Internal server error", requestId: "..." }` for unhandled errors instead of leaking the underlying message. The full detail (with stack) is in the selfbot's logs keyed by that `requestId` — grep your logs for it.

**Can't access local selfbot from another device**

- Your PC's firewall may be blocking incoming connections on port `48923`
- On Windows: open **Windows Defender Firewall → Allow an app or feature** and add an inbound rule for port `48923`
- Use your PC's local IP address, not `localhost`, in the Sentinel URL
