# Sentinel Bot

> **Status: Planned — not yet released.**

Sentinel Bot is the next evolution of the Sentinel ecosystem. Where the selfbot is a personal tool for individual operators, Sentinel Bot is designed for communities — server staff who need better tools to monitor suspected bad actors across the servers they manage.

---

## The Problem

Discord's native moderation tooling is inadequate for dealing with bad actors who move between servers. A server mod who identifies a groomer or serial harasser has no way to automatically warn other communities that the same person has appeared there. Every server starts from zero. Manual cross-server coordination — reaching out to other server staff, sharing ban lists — is time-consuming and inconsistent.

Bots like MEE6 and Dyno handle routine moderation (auto-mod, logging, role management) but none of them build a shared behavioral intelligence layer across communities. Sentinel Bot does.

---

## How It Works

### Basic Operation

A server admin adds Sentinel Bot to their server. The bot then monitors public activity within that server — presence events (via gateway), messages, voice joins, reactions — for any users that have been flagged as targets by that server's staff.

Flagging is done through a dashboard (web-based, separate from sentinel-web) or via Discord slash commands.

### The Shared Intelligence Network

This is the core innovation. Every server that adds the bot contributes to and benefits from a collective dataset:

- Server A's staff flags a user for suspected predatory behavior
- Server B also has the bot and that user is a member
- Server B's staff sees the flag automatically, including the context from Server A (when flagged, what category, any notes)

The network grows its coverage organically. The more servers that participate, the better the cross-server visibility.

### Privacy and Consent Model

- Only **public activity** is monitored — messages sent in public channels, public voice channel presence, public reactions
- Server staff opt in by adding the bot
- Users are not individually notified they are being monitored (standard for moderation bots)
- Server staff retain control over which flags they contribute to the network and which they keep internal

---

## Intended Users

Server staff — moderators, admins — who:
- Manage communities with recurring bad actor problems
- Currently lack tools to see if a suspected bad actor has a history in other communities
- Want real-time alerts when a flagged user becomes active in their server
- Need behavioral analytics (activity patterns, message volume changes, unusual hours) to build better cases for bans or escalation

---

## Planned Feature Set

### Core

- [ ] Multi-server monitoring via a single bot token
- [ ] Staff dashboard (web-based) for managing targets and viewing analytics
- [ ] Slash commands for flagging users, viewing their status, and managing alerts
- [ ] Per-server configuration (which events to log, which alerts to fire, data retention)

### Intelligence

- [ ] Shared flag network — cross-server target visibility
- [ ] Behavioral analytics matching sentinel-selfbot's analyzer suite
- [ ] Anomaly detection (behavioral changes, unusual activity hours)
- [ ] Activity timeline with Gantt visualization

### Alerts

- [ ] Real-time notifications when a flagged user joins a participating server
- [ ] Webhook delivery to moderation channels
- [ ] Discord DM delivery to designated staff
- [ ] Configurable alert rules per server

### Privacy and Control

- [ ] Per-server data isolation (each server only sees what they flagged, plus cross-network flags)
- [ ] Network opt-in/opt-out per server
- [ ] Data retention configuration

---

## Technical Approach

Unlike the selfbot, Sentinel Bot uses a proper Discord bot token. It does not violate Discord's Terms of Service.

The tradeoff is reduced visibility — a bot cannot see user presence without privileged gateway intents (which require verification and approval for large bots), and cannot read messages in servers where it hasn't been granted access. The design accounts for this by working within what a verified bot can see.

The backend will share architectural patterns with sentinel-selfbot (SQLite as the primary store, Fastify API, SSE for real-time updates) but will be redesigned for multi-tenant operation — multiple servers, multiple staff members, shared data with per-server access controls.

---

## Timeline

No release date is set. The selfbot ecosystem (selfbot, plugin, proxy, web) will be stable and well-documented before bot development begins in earnest.

If you want to contribute to or follow development, watch the [sentinel-bot repository](https://github.com/Privex-chat/sentinel-bot).

---

## Contact

If you run a community that would benefit from early access to Sentinel Bot, or if you have specific use cases or requirements you'd like considered in the design, open an issue on the sentinel-bot repository.
