<p align="center">
  <img src="demo.gif" alt="Sentinel Demo" width="720">
</p>

<h1 align="center">🛡️ Sentinel</h1>
<h3 align="center"><em>Know everything. Miss nothing.</em></h3>
<p align="center">
  Real‑time Discord intelligence with AI‑powered analysis.
</p>

<p align="center">
  <!-- Project badges -->
  <a href="https://github.com/Privex-chat/sentinel"><img src="https://img.shields.io/github/stars/Privex-chat/sentinel?style=social" alt="GitHub stars"></a>
  <a href="https://github.com/Privex-chat/sentinel"><img src="https://img.shields.io/github/forks/Privex-chat/sentinel?style=social" alt="GitHub forks"></a>
  <br>
  <a href="https://polyformproject.org/licenses/noncommercial/1.0.0"><img src="https://img.shields.io/badge/License-PolyForm%20Noncommercial%201.0.0-blue" alt="License"></a>
  <img src="https://img.shields.io/badge/status-active-brightgreen" alt="Project Status">
  <img src="https://img.shields.io/badge/self‑hosted-yes-green" alt="Self-Hosted">
</p>

---

## 🧠 Why Sentinel?

Sentinel is a **self‑hosted Discord intelligence system** that turns raw activity into meaningful insight. It watches everything — presence, messages, edits, deletions, interactions — and then uses AI to **categorise, map, and surface what actually matters.**

> 🔍 **From “what happened?” to “what does it mean?”** — automatically.

---

## ⚡ Core Capabilities

| Capability | What it does |
|------------|--------------|
| 🧭 **Full Behavioural Tracking** | Presence, activities, messages, edits, deletions, and interaction timelines |
| 🏷️ **AI Message Categorisation** | Instantly classifies messages into context‑rich categories |
| 🌐 **AI Social Graph Analysis** | Maps who talks to whom, how often, and how relationships evolve |
| 📈 **Pattern & Behaviour Insights** | Detects trends, anomalies, and sudden shifts in behaviour |
| 🔔 **Instant Discord Webhook Alerts** | Real‑time notifications the moment something important happens |
| 📡 **Live Monitoring (REST + SSE)** | Stream events as they happen, plain HTTP or Server‑Sent Events |
| 🔒 **Local‑First + Private** | SQLite by default, optional cloud sync — nothing leaves your system unless you allow it |

---

## 🖼️ See It In Action

### 🏷️ AI Labeling Messages

<p align="center">
  <img src="ai-label.png" alt="AI message categorisation" width="600">
</p>

Each message is automatically tagged with categories like *question*, *gratitude*, *disagreement*, *urgent*, etc. — giving you instant context, at scale.

### 🌐 Social Graph Visualization

<p align="center">
  <img src="social-graph.png" alt="Sentinel social graph" width="600">
</p>

Sentinel builds a dynamic map of communication patterns. See clusters, bridges, and influencers emerge over time.

---

## 💥 One “Wow” Example

> **Behavioural shift detected for `@cryptofox#1234`**  
> Previously: 80% of messages in `#general`, mostly neutral.  
> Today: 90% in `#announcements`, sentiment shifted sharply negative.  
> → **Alert pushed to your Discord webhook in real time.**

This isn’t just logging — it’s understanding that something **changed**, and telling you before you’d notice.

---

## 🧱 The Ecosystem

| Component | Description | Status |
|-----------|-------------|--------|
| [sentinel-selfbot](https://github.com/Privex-chat/sentinel-selfbot) | Core intelligence engine. Collects data, runs AI categorisation + social graph analysis, powers the system. | Stable |
| [sentinel-plugin](https://github.com/Privex-chat/sentinel-plugin) | In‑Discord dashboard via Vencord. Real‑time insights without leaving Discord. | Stable |
| [sentinel-proxy](https://github.com/Privex-chat/sentinel-proxy) | Seamless remote access bridge for externally hosted setups. | Stable |
| [sentinel-web](https://github.com/Privex-chat/sentinel-web) | Full‑featured web dashboard. Monitor from anywhere. | Stable |
| [sentinel-bot](https://github.com/Privex-chat/sentinel-bot) | Multi‑server intelligence network (in development). | In Development |

---

## 🔧 How It Works

```
Selfbot → Data Collection → AI Processing → Storage → Real‑Time Interfaces → Alerts
```

- **Collection** — Continuous event ingestion from Discord  
- **Processing** — AI categorisation + relationship mapping  
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
```

Then connect the plugin to:
```
http://localhost:48923
```

---

## 🗺️ Deployment Options

| Scenario | What to use |
|----------|-------------|
| Everything local (your PC) | selfbot + plugin. No proxy needed. |
| Selfbot on VPS / Railway, plugin on local Discord | selfbot (remote) + proxy (local Windows) + plugin |
| Access from any device / browser | selfbot (anywhere) + sentinel-web |
| All three | selfbot + proxy + plugin + web |

**Recommended:** Deploy the selfbot on Railway for 24/7 uptime and access the data via the web panel at [Sentinel Panel Website](https://sentinel-panel.vercel.app).

---

## ⚠️ Important Notes

- **Selfbot usage** — Running automated code on a regular Discord user account violates Discord's Terms of Service. Use a dedicated, separate account. Understand the risks.
- **Data stays local** — Nothing is sent to external servers unless you explicitly configure Supabase sync or webhook integrations.
- **Ethical boundary** — Only track people you have a legitimate reason to monitor. This tool is for personal use and research. Using it to stalk, harass, or harm anyone is entirely your responsibility.

---

## 📜 License

Licensing varies by component:

| Component | License |
|-----------|---------|
| sentinel-selfbot | [PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0) — free for personal and non‑commercial use |
| sentinel-plugin | [PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0) |
| sentinel-proxy | [PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0) |
| sentinel-web | [PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0) |
| sentinel-bot | **Proprietary** — Source visible for transparency. No rights to use, copy, modify, distribute, or fork. See [LICENSE](https://github.com/Privex-chat/sentinel/blob/main/sentinel-bot-LICENSE). |

Copyright © 2026–present Hemansh ([privexchat@gmail.com](mailto:privexchat@gmail.com))

---

## 📂 Repository Index

- [sentinel](https://github.com/Privex-chat/sentinel) — This repo. Umbrella docs and overview.
- [sentinel-selfbot](https://github.com/Privex-chat/sentinel-selfbot) — Data collection engine.
- [sentinel-plugin](https://github.com/Privex-chat/sentinel-plugin) — Vencord plugin UI.
- [sentinel-proxy](https://github.com/Privex-chat/sentinel-proxy) — Windows local proxy.
- [sentinel-web](https://github.com/Privex-chat/sentinel-web) — Browser‑based dashboard.
- [sentinel-bot](https://github.com/Privex-chat/sentinel-bot) — Community intelligence bot (in development, proprietary).