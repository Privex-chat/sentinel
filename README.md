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

## 🧩 Deployment Modes

| Use Case         | Setup                          |
| ---------------- | ------------------------------ |
| Local monitoring | selfbot + plugin               |
| Remote tracking  | selfbot (VPS) + proxy + plugin |
| Web access       | selfbot + sentinel-web         |
| Full setup       | all components                 |

---

## ⚠️ Important

* Uses a Discord user account (selfbot) — against Discord ToS
* Use a separate account
* You are responsible for how you use this tool

---

## License

Same as before.

---

## Repositories

* sentinel (this repo)
* sentinel-selfbot
* sentinel-plugin
* sentinel-proxy
* sentinel-web
* sentinel-bot
