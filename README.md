# Sentinel

> Know everything. Miss nothing.  
> Real-time Discord intelligence with AI-powered analysis.

---

## What is Sentinel?

Sentinel is a self-hosted Discord intelligence system built for deep visibility.

Track activity. Analyze behavior. Surface patterns.

From presence changes and deleted messages to interaction networks and behavioral shifts — Sentinel turns raw Discord activity into structured, actionable insight.

Now powered by AI, Sentinel doesn’t just log data — it interprets it.

Everything runs on your infrastructure. You stay in control.

---

## ⚡ Core Capabilities

- **Full Behavioral Tracking**  
  Presence, activities, messages, edits, deletions, and interaction timelines

- **AI Message Categorization**  
  Automatically classifies messages into meaningful categories — instantly understand context at scale

- **AI Social Graph Analysis**  
  See who talks to who, how often, and how relationships evolve over time

- **Pattern & Behavior Insights**  
  Identify trends, anomalies, and shifts in user behavior

- **Instant Discord Webhook Alerts**  
  Get real-time notifications the moment something important happens

- **Live Monitoring (REST + SSE)**  
  Stream events as they happen

- **Local-First + Private**  
  SQLite by default, optional cloud sync — nothing leaves your system unless you allow it

---

## The Ecosystem

| Component | Description | Status |
|-----------|-------------|--------|
| [sentinel-selfbot](https://github.com/Privex-chat/sentinel-selfbot) | Core intelligence engine. Collects data, runs AI categorization + social graph analysis, and powers the entire system. | Stable |
| [sentinel-plugin](https://github.com/Privex-chat/sentinel-plugin) | In-Discord dashboard via Vencord. Real-time insights without leaving Discord. | Stable |
| [sentinel-proxy](https://github.com/Privex-chat/sentinel-proxy) | Seamless remote access bridge for externally hosted setups. | Stable |
| [sentinel-web](https://github.com/Privex-chat/sentinel-web) | Full-featured web dashboard. Monitor from anywhere. | Stable |
| [sentinel-bot](https://github.com/Privex-chat/sentinel-bot) | Multi-server intelligence network (in development). | In Development |

---

## 🧠 How It Works

```

Selfbot → Data Collection → AI Processing → Storage → Real-Time Interfaces → Alerts

````

- **Collection** — Continuous event ingestion from Discord  
- **Processing** — AI categorization + relationship mapping  
- **Storage** — Local database (SQLite / optional cloud sync)  
- **Interface** — Plugin + Web dashboard  
- **Alerts** — Instant webhook notifications  

---

## 🚀 Quick Start

```bash
git clone https://github.com/Privex-chat/sentinel-selfbot.git
cd sentinel-selfbot

npm install

cp .env.example .env
# Configure tokens + optional AI / webhook settings

npm run build && npm start
````

Then connect the plugin to:

```
http://localhost:48923
```

---

## Deployment Options

| Scenario                                          | What to use                                       |
| ------------------------------------------------- | ------------------------------------------------- |
| Everything local (your PC)                        | selfbot + plugin. No proxy needed.                |
| Selfbot on VPS / Railway, plugin on local Discord | selfbot (remote) + proxy (local Windows) + plugin |
| Access from any device / browser                  | selfbot (anywhere) + sentinel-web                 |
| All three                                         | selfbot + proxy + plugin + web                    |

Recommended : Deploy selfbot on railway to run 24/7 and access the data from a panel provided by sentinel-web at [Sentinel Panel Website](https://sentinel-panel.vercel.app)
---

## Important Notes

**This project uses a selfbot.** Running automated code on a regular Discord user account violates Discord's Terms of Service. Use a dedicated, separate account. Understand the risks before proceeding.

**Data is stored locally.** Nothing is sent to any external server unless you configure Supabase sync or external webhook integrations.

**Only track people you have a legitimate reason to monitor.** This tool is built for personal use and research. Using it to stalk, harass, or harm anyone is entirely your responsibility.

---

## License

Licensing varies by component:

| Component        | License                                                                                                                                                                                                                                                                                                                                                                                   |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sentinel-selfbot | [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0) — free for personal and non-commercial use                                                                                                                                                                                                                                               |
| sentinel-plugin  | [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0) — free for personal and non-commercial use                                                                                                                                                                                                                                               |
| sentinel-proxy   | [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0) — free for personal and non-commercial use                                                                                                                                                                                                                                               |
| sentinel-web     | [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0) — free for personal and non-commercial use                                                                                                                                                                                                                                               |
| sentinel-bot     | **Proprietary — Source Visibility Only.** The source code is publicly visible for transparency and community supervision purposes only. No rights to use, copy, modify, distribute, or fork are granted. See the [sentinel-bot LICENSE](https://github.com/Privex-chat/sentinel/blob/main/sentinel-bot-LICENSE) for full terms. Commercial rights are retained exclusively by the author. |

Copyright (c) 2026–present Hemansh ([privexchat@gmail.com](mailto:privexchat@gmail.com))

---

## Repository Index

* [github.com/Privex-chat/sentinel](https://github.com/Privex-chat/sentinel) — This repo. Umbrella docs and overview.
* [github.com/Privex-chat/sentinel-selfbot](https://github.com/Privex-chat/sentinel-selfbot) — The data collection engine.
* [github.com/Privex-chat/sentinel-plugin](https://github.com/Privex-chat/sentinel-plugin) — Vencord plugin UI.
* [github.com/Privex-chat/sentinel-proxy](https://github.com/Privex-chat/sentinel-proxy) — Windows local proxy.
* [github.com/Privex-chat/sentinel-web](https://github.com/Privex-chat/sentinel-web) — Browser-based dashboard.
* [github.com/Privex-chat/sentinel-bot](https://github.com/Privex-chat/sentinel-bot) — Community intelligence bot (in development, proprietary).

