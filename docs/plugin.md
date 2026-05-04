# Plugin Setup

The Sentinel plugin is a Vencord plugin that adds a full intelligence dashboard directly inside Discord's settings panel. It talks to your selfbot API in real time via REST and SSE.

---

## Requirements

- **Vencord** installed in your Discord client — [vencord.dev](https://vencord.dev)
- A running `sentinel-selfbot` instance (see [selfbot.md](selfbot.md))
- The selfbot's API URL and auth token

---

## Installation

### Step 1 — Find your Vencord plugins folder

| Install type | Path |
|---|---|
| Standard (Windows) | `%APPDATA%\Vencord\src\plugins\` |
| Standard (Linux/macOS) | `~/.config/Vencord/src/plugins/` |
| Dev install (cloned repo) | `Vencord/src/plugins/` |

### Step 2 — Copy the plugin

```bash
# Clone the plugin repo
git clone https://github.com/Privex-chat/sentinel-plugin.git

# Copy the plugin folder into Vencord's plugins directory
cp -r sentinel-plugin/Vencord/src/plugins/sentinel-ui /path/to/Vencord/src/plugins/sentinel-ui
```

On Windows (PowerShell):

```powershell
Copy-Item -Recurse sentinel-plugin\Vencord\src\plugins\sentinel-ui "$env:APPDATA\Vencord\src\plugins\sentinel-ui"
```

### Step 3 — Rebuild Vencord

```bash
cd /path/to/Vencord
pnpm build
pnpm inject
```

If you're running Vencord in dev mode (`pnpm dev`), it rebuilds automatically when you copy files.

### Step 4 — Enable the plugin

1. Open Discord
2. Go to **Settings → Plugins**
3. Search for **SentinelUI**
4. Toggle it on

### Step 5 — Configure the plugin

Click the **SentinelUI** plugin name to open its settings, then configure:

| Setting | Description |
|---------|-------------|
| **Sentinel URL** | Address where your selfbot is running — default `http://localhost:48923`. Use `http://localhost:42969` if using the proxy. |
| **Sentinel Token** | The `API_AUTH_TOKEN` value from your selfbot's `.env` |
| **Enable SSE** | Keep on for real-time event streaming |
| **Dashboard Refresh Interval** | Auto-refresh interval in seconds (default 30) |
| **OPSEC Mode** | Hide all Sentinel traces from Discord's UI (see OPSEC Mode below) |
| **Disguise Name** | Name shown instead of "Sentinel" when OPSEC Mode is active (default: "Discord Utilities") |

### Step 6 — Open the dashboard

The Sentinel panel appears in the plugin settings section. Click **Open Sentinel Dashboard** to open the full interface in a large modal.

---

## Features

### Dashboard

The main view shows all tracked targets with live status indicators, current activity, platform, and a live event feed that updates in real time as events arrive.

### Runtime Config

The **Runtime Config** tab lets you hot-swap selfbot settings without restarting. Changes take effect immediately — useful for toggling AI features, adjusting polling intervals, or updating webhook URLs on the fly.

### Per-Target Views

Click any target to open their full profile. Available tabs:

| Tab | What's in it |
|-----|-------------|
| **Overview** | Current status, activity, today's stats, anomaly flags, recent events |
| **Timeline** | Gantt chart of today's sessions + paginated, filterable event log |
| **Analytics** | Presence distribution, gaming profile, messages, voice, music, social graph |
| **Messages** | All messages, deleted messages, edited messages, ghost typing stats |
| **Profile** | Avatar history, profile change timeline, connected accounts |
| **Insights** | Sleep schedule estimate, weekly routine heatmap, anomaly feed |
| **Alerts** | Per-target alert rule configuration and alert history |
| **Briefs** | AI-generated daily intelligence summaries — requires `AI_PROVIDER` set in selfbot |
| **Backfill** | Trigger historical channel backfill and monitor per-channel progress |
| **Config** | Per-target tracking configuration |

### Context Menu

Right-click any user in Discord to instantly add or remove them as a tracking target:

- **Track User** — adds the user and starts tracking immediately
- **Stop Tracking** — removes the user from tracking

> Context menu items are hidden when OPSEC Mode is active.

### Real-Time Updates

The plugin maintains a single SSE connection to the selfbot. As events arrive, they appear in the live feed and automatically update the relevant analytics views. No manual refresh required.

---

## OPSEC Mode

OPSEC Mode removes all visible traces of Sentinel from Discord's UI — useful if you want the tool to go unnoticed.

**What it does:**

- Hides "Track User" and "Stop Tracking" from all right-click context menus
- Replaces "Sentinel" with a configurable disguise name (default: "Discord Utilities") in the settings panel and modal header
- Adds a panic key — press **`Ctrl+Shift+.`** at any time to instantly close the dashboard
- Replaces `[Sentinel]` with `[Plugin]` in all console output

**Enabling OPSEC Mode:**

1. Go to **Settings → Plugins → SentinelUI**
2. Toggle **OPSEC Mode** on
3. Optionally change the **Disguise Name** to whatever you want the panel to appear as
4. Reload Discord for context menu changes to take effect

The panic key is always active regardless of OPSEC Mode setting — it works any time the dashboard is open.

---

## Updating the Plugin

Pull the latest plugin code and re-copy the folder:

```bash
cd sentinel-plugin
git pull
cp -r Vencord/src/plugins/sentinel-ui /path/to/Vencord/src/plugins/sentinel-ui
cd /path/to/Vencord && pnpm build && pnpm inject
```

---

## Troubleshooting

**Plugin shows "Connection failed" or "Disconnected"**

- Make sure the selfbot is running (`pm2 status` or check your terminal)
- Confirm the URL and port match — default is `http://localhost:48923`
- If using the proxy, confirm it is running and the URL is `http://localhost:42969`
- Check that the token in plugin settings matches `API_AUTH_TOKEN` in the selfbot's `.env`
- Open DevTools in Discord (`Ctrl+Shift+I`) and check the console for error messages

**Plugin doesn't appear in Settings → Plugins**

- Make sure you ran `pnpm build` and `pnpm inject` after copying the plugin
- Check that the folder is named exactly `sentinel-ui` inside Vencord's `plugins` directory
- Restart Discord after building

**Real-time feed not updating**

- Check that **Enable SSE** is toggled on in plugin settings
- If the selfbot is behind a reverse proxy (Nginx, Caddy), make sure it is configured to not buffer SSE responses — add `X-Accel-Buffering: no` to the response headers

**No data for a target after adding them**

- The first data arrives when the selfbot finishes mapping mutual servers and subscribing to their presence — wait 1-2 minutes
- If you share no servers with the target, presence data will not arrive via gateway. Only profile data from polling will be collected.

**Briefs tab shows no briefs**

- Daily briefs require `AI_PROVIDER` to be set in the selfbot's `.env` (see [selfbot.md](selfbot.md))
- Briefs generate once per day at the time set in `BRIEF_GENERATION_TIME`

**Context menu items not appearing**

- Make sure **OPSEC Mode** is not enabled — it suppresses the context menu entries
- Restart Discord after enabling or disabling the plugin

**Text appears the wrong color (white on white / dark on dark)**

- This is a known issue if Vencord's CSS variable resolution differs from expectations
- The plugin uses DOM-based theme detection (`theme-light`/`theme-dark` class on `<html>`) to set text color correctly across all Discord themes (Light, Ash, Dark, Onyx)
- If colors still look wrong, check your Discord theme in **Settings → Appearance** and report the issue
