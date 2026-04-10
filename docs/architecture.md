# Architecture

This document covers how the Sentinel ecosystem is structured, how data flows between components, and the reasoning behind key design decisions.

---

## System Overview

Sentinel is a two-layer system: a **data layer** (the selfbot) and one or more **UI layers** (plugin, web). They communicate over HTTP and Server-Sent Events.

```
Discord API (WebSocket + REST)
        │
        ▼
┌───────────────────────────────────────────────┐
│              sentinel-selfbot                 │
│                                               │
│  Gateway ──► Collectors ──► SQLite DB         │
│                    │                          │
│              Alert Engine                     │
│                    │                          │
│  Pollers ──────────┘                          │
│                    │                          │
│         Fastify HTTP API (:48923)             │
└───────────────┬───────────────────────────────┘
                │  REST + SSE
     ┌──────────┼──────────────────┐
     │          │                  │
     ▼          ▼                  ▼
sentinel-   sentinel-proxy    sentinel-web
 plugin     (Windows only)   (any browser)
(Vencord)        │
                 ▼
          sentinel-plugin
           (via proxy)
```

The selfbot is the only component that communicates with Discord. All UI layers talk exclusively to the selfbot's local API — they never touch Discord directly.

---

## Component Roles

### sentinel-selfbot

The engine. It:

- Maintains a persistent WebSocket connection to Discord's gateway
- Dispatches incoming events to collector functions
- Writes all observed data to a local SQLite database
- Runs periodic REST pollers for data not available via gateway
- Evaluates alert rules on every event
- Serves all collected data through a Fastify HTTP server
- Pushes live events to connected UI clients over SSE
- Optionally mirrors the local SQLite to a Supabase PostgreSQL instance

### sentinel-plugin

The in-Discord UI. It:

- Runs inside Vencord (a Discord client mod) as a settings panel
- Makes HTTP requests to the selfbot API
- Maintains a single SSE connection for live updates
- Adds a right-click context menu entry to quickly track users
- Requires no build step beyond Vencord's own build

### sentinel-proxy

A connectivity bridge for Windows users. It:

- Runs as a background Node.js process
- Starts silently on Windows boot via a startup VBS script
- Listens on `localhost:42969` and forwards all requests to a configured remote URL
- Handles SSE streaming correctly (no buffering)
- Logs all requests to the console for debugging
- Required only when the selfbot runs on a remote server (Railway, VPS, etc.) and the plugin needs to reach it — because the plugin cannot attach custom `Authorization` headers to `EventSource` connections, but it can to `fetch()`-based SSE, and the proxy forwards those headers

### sentinel-web

The browser dashboard. It:

- Is a Next.js 15 app that can be self-hosted or used from [sentinel-panel.vercel.app](https://sentinel-panel.vercel.app)
- Stores the selfbot URL and token in the browser's localStorage
- Makes all API calls directly from the browser to the selfbot URL
- Provides the same analytics, timelines, insights, and alerts as the plugin
- Works with any selfbot deployment — local or remote

### sentinel-bot (Planned)

A proper Discord bot (not a selfbot). It:

- Uses a bot token, not a user account token
- Is added to servers voluntarily by staff/admins
- Monitors public activity for flagged targets within those servers
- Contributes to and reads from a shared intelligence network across all participating servers
- Provides cross-server context — if a target is flagged in Server A, Server B's staff sees that automatically

---

## Data Flow

### Real-Time Events (Gateway)

```
Discord sends PRESENCE_UPDATE
        │
        ▼
GatewayClient.handleMessage()
        │
        ▼
setupGatewayHandlers() checks isTarget(userId)
        │
   ┌────┴────────────────────────────────┐
   │                                     │
   ▼                                     ▼
handlePresenceUpdate()            evaluateEvent()
writes to SQLite                  checks alert rules
   │                                     │
   ▼                                     ▼
pushSSEEvent()                  insertAlertHistory()
pushes to all                    fires alertCallback()
SSE clients                            │
                                       ▼
                                 pushSSEEvent()
                                 (ALERT event)
```

### Periodic Data (Pollers)

```
setInterval fires (every 5 min for profiles, 2 min for status)
        │
        ▼
discordFetch() — rate-limit aware REST call
        │
        ▼
diff against last snapshot in SQLite
        │
   if changed:
        ▼
insertSnapshot() + insertEvent()
```

### Analytics Requests

```
UI requests GET /api/targets/:userId/analytics/presence
        │
        ▼
Route handler calls analyzeGamingProfile() etc.
        │
        ▼
Analyzer reads SQLite via prepared statements
        │
        ▼
Returns computed JSON to UI
```

---

## Database

All data lives in a single SQLite file (`sentinel.db`). Key tables:

| Table | Purpose |
|-------|---------|
| `targets` | Tracked user IDs, labels, priority |
| `events` | Append-only log of every observed event |
| `presence_sessions` | Open/closed status sessions per user |
| `activity_sessions` | Game, Spotify, streaming sessions |
| `voice_sessions` | Voice channel sessions with co-participant tracking |
| `messages` | Message content, edit history, deletion timestamps |
| `typing_events` | Typing events with ghost detection |
| `reactions` | Reaction add/remove |
| `profile_snapshots` | Timestamped profile snapshots for diffing |
| `guild_member_events` | Nickname and role changes |
| `alert_rules` | Configured alert conditions |
| `alert_history` | Fired alerts |
| `daily_summaries` | Pre-computed daily stat rows |
| `sync_state` | Supabase sync watermarks |
| `heartbeat_log` | Process health timestamps for stale session recovery |

SQLite is used instead of Postgres because Sentinel is a single-operator tool. WAL mode with a 64MB cache handles tens of millions of rows comfortably with no separate database process, trivial backup (copy the file), and trivial deployment.

---

## API Communication

The selfbot exposes a Fastify HTTP server on port `48923` by default.

All endpoints require a `Bearer` token matching `API_AUTH_TOKEN` from `.env`. Requests without a valid token receive `401` or `403`.

Live events are delivered over SSE at `GET /api/events/stream`. The plugin and web dashboard maintain persistent connections to this endpoint and react to incoming events by invalidating their local caches and triggering UI refetches.

Full API reference: [api.md](api.md)

---

## Supabase Sync

When `DB_MODE=local+cloud` or `DB_MODE=cloud`, a background sync engine runs on a configurable interval (default 5 minutes) and pushes new/updated rows from SQLite to Supabase in batches.

Sync is always **SQLite → Supabase**, never the reverse (except on startup in `cloud` mode, where Supabase is used to hydrate a fresh SQLite before Discord connects).

Full Supabase setup: [supabase.md](supabase.md)

---

## Key Design Decisions

**Selfbot over bot** — Discord bots cannot read user presence or messages in guilds without explicit server-level permissions and privileged intents that are gatekept. A user account sees everything the logged-in user sees. For personal tracking purposes, this is the only viable approach.

**SQLite over Postgres** — Single operator, single machine, no concurrency requirements. SQLite in WAL mode is faster for this workload than Postgres with a network hop, and requires zero infrastructure.

**Prepared statements singleton** — All SQL runs through pre-compiled statements initialized once at startup. No per-query compilation overhead, no SQL injection surface, and a single auditable file for all queries.

**SSE over WebSockets for UI** — The UI only needs one-way push from the selfbot. SSE is simpler than WebSockets (no upgrade handshake), reconnects automatically via the browser's EventSource API, and works over standard HTTP with no additional infrastructure.

**fetch-based SSE in the plugin** — The browser's `EventSource` API doesn't support custom headers. Since the selfbot requires a `Bearer` token, the plugin uses a `fetch()`-based SSE reader instead, which supports arbitrary headers and is manually reconnected on disconnect.

**Fastify over Express** — Lower per-request overhead, built-in schema validation, better TypeScript types. Measurably faster for the high-frequency `/api/targets/:id/status` endpoint.
