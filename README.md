# Sentinel

> Self-hosted Discord intelligence. Behavioral tracking, analytics, and real-time monitoring — built for people who need to know more than Discord tells them.

---

## What is Sentinel ?

Sentinel is an ecosystem of tools that work together to collect, store, and surface behavioral data on Discord users. You choose who to track. Sentinel handles everything else — logging when they come online, what they play, who they talk to, what they delete, and how their patterns change over time.

Everything runs on your own infrastructure. No data leaves your setup unless you configure it to.

---

## The Ecosystem

| Component | Description | Status |
|-----------|-------------|--------|
| [sentinel-selfbot](https://github.com/Privex-chat/sentinel-selfbot) | The data collector. Runs as a background process using a Discord user account token. Stores everything locally in SQLite and exposes a REST/SSE API. | Stable |
| [sentinel-plugin](https://github.com/Privex-chat/sentinel-plugin) | Vencord plugin that renders the full dashboard inside Discord's settings panel. Talks to the selfbot API in real time. | Stable |
| [sentinel-proxy](https://github.com/Privex-chat/sentinel-proxy) | Lightweight Windows proxy for users running the selfbot on an external server (Railway, Fly.io, etc.). Runs silently on boot and forwards requests from the plugin. | Stable |
| [sentinel-web](https://github.com/Privex-chat/sentinel-web) | Next.js web dashboard. Same analytics and monitoring as the plugin, accessible from any browser. Hosted at [sentinel-panel.vercel.app](https://sentinel-panel.vercel.app). | Stable |
| [sentinel-bot](https://github.com/Privex-chat/sentinel-bot) | Proper Discord bot for server staff. Multi-server shared intelligence network — no user token required. | In Development |

---

## How They Fit Together

```
┌─────────────────────────────────────────────────────────────────┐
│                        Your Infrastructure                      │
│                                                                 │
│   ┌──────────────────┐        ┌──────────────────────────────┐  │
│   │ sentinel-selfbot │◄──────►│         SQLite DB            │  │
│   │  (Node.js proc)  │        │   (local or + Supabase)      │  │
│   └────────┬─────────┘        └──────────────────────────────┘  │
│            │  REST + SSE API (:48923)                           │
│            │                                                    │
│     ┌──────┴──────────────────────────────┐                     │
│     │                                     │                     │
│  ┌──▼────────────┐            ┌───────────▼──────────────────┐  │
│  │sentinel-proxy │            │         sentinel-web         │  │
│  │ (Windows only)│            │  (Next.js, Vercel or self-   │  │
│  └──────┬────────┘            │   hosted, any browser)       │  │
│         │                     └──────────────────────────────┘  │
│  ┌──────▼────────┐                                              │
│  │sentinel-plugin│                                              │
│  │  (Vencord)    │                                              │
│  └───────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘
```

The selfbot is the core. Everything else is a UI layer or a connectivity helper that talks to its API.

---

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture](docs/architecture.md) | How the full system is structured and how components communicate |
| [Selfbot Setup](docs/selfbot.md) | Installing and configuring the selfbot |
| [Plugin Setup](docs/plugin.md) | Installing the Vencord plugin |
| [Proxy Setup](docs/proxy.md) | Setting up the Windows proxy for remote selfbot access |
| [Web Dashboard](docs/web.md) | Using and self-hosting the web frontend |
| [API Reference](docs/api.md) | Every selfbot API endpoint |
| [Supabase / Cloud](docs/supabase.md) | Configuring cloud backup and sync |
| [Sentinel Bot](docs/bot.md) | The community intelligence bot (in development) |
| [Future Improvements](docs/future-improvements.md) | Planned features and roadmap |

---

## Quick Start

If you just want the selfbot + plugin running locally in five minutes:

```bash
# 1. Clone the selfbot
git clone https://github.com/Privex-chat/sentinel-selfbot.git
cd sentinel-selfbot

# 2. Install dependencies
npm install

# 3. Configure
cp .env.example .env
# Edit .env — set DISCORD_TOKEN and API_AUTH_TOKEN

# 4. Build and run
npm run build && npm start
```

Then install the Vencord plugin from `sentinel-plugin/plugins/sentinel-ui/` and point it at `http://localhost:48923`.

Full instructions: [docs/selfbot.md](docs/selfbot.md) and [docs/plugin.md](docs/plugin.md)

---

## Deployment Options

| Scenario | What to use |
|----------|-------------|
| Everything local (your PC) | selfbot + plugin. No proxy needed. |
| Selfbot on VPS / Railway, plugin on local Discord | selfbot (remote) + proxy (local Windows) + plugin |
| Access from any device / browser | selfbot (anywhere) + sentinel-web |
| All three | selfbot + proxy + plugin + web |

---

## Important Notes

**This project uses a selfbot.** Running automated code on a regular Discord user account violates Discord's Terms of Service. Use a dedicated, separate account. Understand the risks before proceeding.

**Data is stored locally.** Nothing is sent to any external server unless you configure Supabase sync. Your database lives on whatever machine runs the selfbot.

**Only track people you have a legitimate reason to monitor.** This tool is built for personal use and research. Using it to stalk, harass, or harm anyone is entirely your responsibility.

---

## License

Licensing varies by component:

| Component | License |
|-----------|---------|
| sentinel-selfbot | [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0) — free for personal and non-commercial use |
| sentinel-plugin | [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0) — free for personal and non-commercial use |
| sentinel-proxy | [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0) — free for personal and non-commercial use |
| sentinel-web | [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0) — free for personal and non-commercial use |
| sentinel-bot | **Proprietary — Source Visibility Only.** The source code is publicly visible for transparency and community supervision purposes only. No rights to use, copy, modify, distribute, or fork are granted. See the [sentinel-bot LICENSE](https://github.com/Privex-chat/sentinel-bot/blob/main/LICENSE) for full terms. Commercial rights are retained exclusively by the author. |

Copyright (c) 2026–present Hemansh (privexchat@gmail.com)

---

## Repository Index

- [github.com/Privex-chat/sentinel](https://github.com/Privex-chat/sentinel) — This repo. Umbrella docs and overview.
- [github.com/Privex-chat/sentinel-selfbot](https://github.com/Privex-chat/sentinel-selfbot) — The data collection engine.
- [github.com/Privex-chat/sentinel-plugin](https://github.com/Privex-chat/sentinel-plugin) — Vencord plugin UI.
- [github.com/Privex-chat/sentinel-proxy](https://github.com/Privex-chat/sentinel-proxy) — Windows local proxy.
- [github.com/Privex-chat/sentinel-web](https://github.com/Privex-chat/sentinel-web) — Browser-based dashboard.
- [github.com/Privex-chat/sentinel-bot](https://github.com/Privex-chat/sentinel-bot) — Community intelligence bot (in development, proprietary).
