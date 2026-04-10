# Plugin Setup

The Sentinel plugin is a Vencord plugin that adds a full intelligence dashboard directly inside Discord's settings panel. It talks to your selfbot API in real time.

---

## Requirements

- **Vencord** installed in your Discord client — [vencord.dev](https://vencord.dev)
- A running `sentinel-selfbot` instance (see [selfbot.md](selfbot.md))
- The selfbot's API URL and token

---

## Installation

### Step 1 — Find your Vencord plugins folder

The location depends on how you installed Vencord:

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
cp -r sentinel-plugin/plugins/sentinel-ui /path/to/Vencord/src/plugins/sentinel-ui
```

On Windows (PowerShell):

```powershell
Copy-Item -Recurse sentinel-plugin\plugins\sentinel-ui "$env:APPDATA\Vencord\src\plugins\sentinel-ui"
```

### Step 3 — Rebuild Vencord

```bash
cd /path/to/Vencord
pnpm build
```

If you're running Vencord in dev mode (`pnpm dev`), it rebuilds automatically when you copy files.

### Step 4 — Enable the plugin

1. Open Discord
2. Go to **Settings → Plugins**
3. Search for **SentinelUI**
4. Toggle it on

### Step 5 — Configure the plugin

1. Click the **SentinelUI** plugin name to open its settings
2. Set **Sentinel URL** to the address where your selfbot is running:
   - Local selfbot: `http://localhost:48923`
   - Remote selfbot (via proxy): `http://localhost:42969`
   - Remote selfbot (direct): `http://your-server-ip:48923` or `https://your-domain.com`
3. Set **Sentinel Token** to the same `API_AUTH_TOKEN` value from your selfbot's `.env`
4. Keep **Enable SSE** on for real-time updates

### Step 6 — Open the dashboard

The Sentinel dashboard appears in the plugin's settings section. Click **SentinelUI** in the settings list to open the full panel.

---

## Features

### Dashboard

The main view shows all your tracked targets with live status indicators, what they're doing right now, and a live event feed as things happen.

### Per-Target Views

Click any target to open their full profile. Available tabs:

| Tab | What's in it |
|-----|-------------|
| **Overview** | Current status, activity, today's stats, anomalies, recent events |
| **Timeline** | Gantt bar of today's sessions + paginated event log with filtering |
| **Analytics** | Presence distribution, gaming profile, messages, voice, music, social graph |
| **Messages** | All messages, deleted messages, edited messages, ghost typing stats |
| **Profile** | Avatar history, profile change timeline, connected accounts |
| **Insights** | Sleep schedule estimate, weekly routine heatmap, anomaly feed |
| **Alerts** | Per-target alert rule configuration and alert history |

### Context Menu

Right-click any user in Discord and select **Track with Sentinel** to immediately add them as a target, or **Stop Tracking (Sentinel)** to remove them.

### Real-Time Updates

The plugin maintains a single SSE connection to the selfbot. As events arrive, they appear in the live feed and automatically update the relevant analytics views. No manual refresh required.

---

## Updating the Plugin

Pull the latest plugin code and re-copy the folder:

```bash
cd sentinel-plugin
git pull
cp -r plugins/sentinel-ui /path/to/Vencord/src/plugins/sentinel-ui
cd /path/to/Vencord && pnpm build
```

---

## Troubleshooting

**Plugin shows "Disconnected"**

- Make sure the selfbot is running (`pm2 status` or check your terminal)
- Confirm the URL and port match — default is `http://localhost:48923`
- If using the proxy, confirm it's running and the URL is `http://localhost:42969`
- Check that the token in plugin settings matches `API_AUTH_TOKEN` in the selfbot's `.env`
- Open DevTools in Discord (`Ctrl+Shift+I`) and check the console for error messages

**Plugin doesn't appear in Settings → Plugins**

- Make sure you ran `pnpm build` after copying the plugin
- Check that the folder is named exactly `sentinel-ui` inside Vencord's `userplugins` directory
- Restart Discord after building

**Real-time feed not updating**

- Check that **Enable SSE** is toggled on in plugin settings
- If the selfbot is behind a reverse proxy (Nginx, Caddy), make sure it's configured to not buffer SSE responses (add `X-Accel-Buffering: no` header)

**No data for a target after adding them**

- The first data arrives when the selfbot finishes mapping mutual servers and requesting their presence — wait 1–2 minutes
- If you share no servers with the target, presence data won't arrive via gateway. Only profile data from polling will be collected.
