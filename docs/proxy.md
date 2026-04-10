# Proxy Setup

The Sentinel proxy is a lightweight tool for Windows users who run their selfbot on an external server (Railway, Fly.io, a VPS) and want to connect the Vencord plugin to it.

---

## Why the Proxy Exists

The Vencord plugin uses a `fetch()`-based SSE reader (not the browser's native `EventSource`) because it needs to attach a custom `Authorization: Bearer` header. This works fine for regular HTTP requests.

However, when your selfbot is on a remote HTTPS server, the browser enforces strict CORS and mixed-content rules that can interfere with long-lived connections. The proxy solves this by:

- Running locally on `http://localhost:42969` (no CORS issues from the plugin's perspective)
- Forwarding all requests — including SSE streams — to your configured remote selfbot URL
- Attaching the correct headers and logging everything for easy debugging

If your selfbot runs **locally** on the same machine as Discord, you don't need the proxy. Point the plugin directly at `http://localhost:48923`.

---

## Requirements

- Windows 10 or newer
- Node.js 18 or newer installed and in your PATH

---

## Installation

### Step 1 — Get the files

```bash
git clone https://github.com/Privex-chat/sentinel-proxy.git
```

Or download the ZIP from the GitHub releases page and extract it.

### Step 2 — Configure

Open `config.json` in a text editor:

```json
{
  "port": 42969,
  "target": "https://your-selfbot-url.com"
}
```

Replace `https://your-selfbot-url.com` with the full URL of your remote selfbot — for example:

- Railway: `https://sentinel-selfbot-production-xxxx.up.railway.app`
- Your own VPS: `https://sentinel.yourdomain.com`
- Bare IP: `http://123.45.67.89:48923` (HTTP, not HTTPS — only use this on a private network)

### Step 3 — Start manually (to test)

```bash
node sentinel-proxy.js
```

You should see output like:

```
╔══════════════════════════════════════════════╗
║       Sentinel Debug Proxy  RUNNING          ║
╚══════════════════════════════════════════════╝

  Local  :  http://localhost:42969
  Target :  https://your-selfbot-url.com

  Set plugin URL to:  http://localhost:42969
```

Test it by hitting the health endpoint:

```bash
curl -H "Authorization: Bearer YOUR_API_AUTH_TOKEN" http://localhost:42969/api/status
```

### Step 4 — Install to run on startup

Double-click `install-startup.bat` (or run it as Administrator if needed).

This script:
1. Creates a VBS launcher in your Windows Startup folder so the proxy starts silently on every boot
2. Starts the proxy immediately without waiting for a reboot

After running the script, the proxy will start automatically whenever you log in to Windows.

---

## Configuring the Plugin

Once the proxy is running, open your Vencord plugin settings and set:

- **Sentinel URL**: `http://localhost:42969`
- **Sentinel Token**: your `API_AUTH_TOKEN` (same as always — the proxy forwards it)

---

## Manual Start/Stop

To start without the installer:

```bash
node sentinel-proxy.js
```

To stop: close the terminal window, or if running hidden, use Task Manager to end the `node.exe` process associated with it.

---

## Troubleshooting

**Proxy starts but plugin shows "Disconnected"**

- Check that your selfbot is actually running and accessible at the configured target URL
- Verify your `API_AUTH_TOKEN` is correct
- Check the proxy console for errors — it logs every request and response status

**Proxy exits immediately after startup**

- Make sure Node.js is installed and in your PATH — open Command Prompt and run `node --version`
- Check that `config.json` is valid JSON (no trailing commas, correct quotes)

**`install-startup.bat` didn't work**

- Try running it as Administrator (right-click → Run as administrator)
- Alternatively, manually copy `run-hidden.vbs` and `sentinel-proxy.js` to your Startup folder and create a shortcut

**The proxy was installed but stopped running after a Windows update**

- Run `install-startup.bat` again — Windows updates occasionally clear startup entries

**I want to remove the autostart**

- Open Run (`Win+R`), type `shell:startup`, and delete the `SentinelProxy.vbs` file

---

## Security Note

The proxy runs on `127.0.0.1` only — it is not accessible from other machines on your network. Your API token is forwarded to the remote selfbot over the existing HTTPS connection. The proxy itself does not store or log tokens beyond the console output.
