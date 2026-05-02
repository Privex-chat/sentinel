# Architecture

This document covers how the Sentinel ecosystem is structured, how data flows between components, and the reasoning behind key design decisions.

---

## System Overview

Sentinel is a two-layer system: a **data layer** (the selfbot) and one or more **UI layers** (plugin, web). They communicate over HTTP and Server-Sent Events.

```
Discord API (WebSocket + REST)
        │
        ├── Gateway events (PRESENCE_UPDATE, MESSAGE_CREATE, VOICE_STATE_UPDATE, ...)
        │
        ▼
┌────────────────────────────────────────────────────┐
│                 sentinel-selfbot                   │
│                                                    │
│  Gateway ──► Self-command handler                  │
│      │                                             │
│      └──► Collectors ──► SQLite DB                 │
│                │                                   │
│          Alert Engine                              │
│                │                                   │
│  Pollers ──────┘                                   │
│                │                                   │
│        AI Analysis Engine                          │
│                │                                   │
│        Fastify HTTP API (:48923)                   │
└────────────────┬───────────────────────────────────┘
                 │  REST + SSE
      ┌──────────┼──────────────────┐
      │          │                  │
      ▼          ▼                  ▼
 sentinel-  sentinel-proxy    sentinel-web
  plugin    (Windows only)   (any browser)
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
- Sends op 14 (GUILD_SUBSCRIBE) with `members` arrays to subscribe to real-time presence for tracked users in each mutual guild
- Dispatches incoming events to collector functions (presence, activity, message, voice, profile, etc.)
- Handles self-commands from the selfbot's own account — deletes command messages instantly, executes actions, sends self-deleting responses
- Writes all observed data to a local SQLite database
- Runs periodic REST pollers for data not available via gateway (full profiles, connected accounts, mutual server lists)
- Evaluates alert rules on every event
- Optionally runs AI analysis: message categorisation, social graph classification, daily briefs
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
- Required when the selfbot runs on a remote server and the plugin needs to reach it — because the browser's `EventSource` API doesn't support custom headers, but the proxy can forward `Authorization` headers on behalf of the plugin's `fetch`-based SSE reader

### sentinel-web

The browser dashboard. It:

- Is a Next.js 15 app that can be self-hosted or used from [sentinel-panel.vercel.app](https://sentinel-panel.vercel.app)
- Stores the selfbot URL and token in the browser's localStorage
- Makes all API calls directly from the browser to the selfbot URL
- Provides analytics, timelines, insights, alerts, social graph, and AI briefs
- Works with any selfbot deployment — local or remote

### sentinel-bot (In Development)

A proper Discord bot (not a selfbot). It:

- Uses a bot token, not a user account token
- Is added to servers voluntarily by staff/admins
- Monitors public activity for flagged targets within those servers
- Contributes to a shared intelligence network across participating servers

### sentinel-desktop (In Development)

An Electron desktop app that:

- Bundles both the selfbot backend and the web dashboard into a single Windows installer
- Runs everything locally with no manual setup required

---

## Data Flow

### Real-Time Presence (Gateway Op 14)

Presence tracking is **event-driven**, not polled. On connect and periodically, the selfbot sends op 14 subscriptions to Discord for each mutual guild, including a `members` array of tracked user IDs. Discord then pushes `PRESENCE_UPDATE` events for those users whenever their status or activities change.

```
On READY / RESUMED / every 4 min:
selfbot sends op 14 to Discord
  { guild_id, members: [userId, ...], activities: true }
        │
        ▼
Discord tracks those users, pushes PRESENCE_UPDATE on any change
        │
        ▼
GatewayClient receives PRESENCE_UPDATE
        │
        ▼
setupGatewayHandlers() checks isTarget(userId)
        │
   ┌────┴────────────────────────────────────────┐
   │                                             │
   ▼                                             ▼
handlePresenceUpdate()                  handleActivityUpdate()
  - close old presence sessions           - diff old vs new activities
  - open new session for new status       - close ended activity sessions
  - emit PRESENCE_UPDATE event            - open new activity sessions
  - emit PLATFORM_SWITCH if               - emit ACTIVITY_START / SPOTIFY_START
    platform changed independently          / STREAMING_START / CUSTOM_STATUS_SET
        │                                        │
        ▼                                        ▼
  pushSSEEvent()                           pushSSEEvent()
  evaluateEvent() → alert rules            evaluateEvent() → alert rules
```

**Key property:** Offline detection fires when Discord pushes the PRESENCE_UPDATE, not when a poll cycle next runs. The gap is under a second.

### Self-Commands

```
User types "$add @target" in any Discord channel
        │
        ▼
Gateway receives MESSAGE_CREATE
        │
        ▼
handleSelfCommand() — checks message.author.id === selfbot user ID
        │
        ├── deleteMessage() called immediately (before processing)
        │
        ├── parse command + args
        │
        ├── execute command (DB write, requestPresenceForUser, etc.)
        │
        └── sendTempMessage() — response auto-deletes after TTL
```

### Periodic Data (Pollers)

```
setInterval fires (every 5 min for profiles, 2 min for status confirmation)
        │
        ▼
discordFetch() — rate-limit aware REST call with browser headers
        │
        ▼
diff against last snapshot in SQLite
        │
   if changed:
        ▼
insertSnapshot() + insertEvent() + pushSSEEvent()
```

### Analytics Requests

```
UI requests GET /api/targets/:userId/analytics/presence
        │
        ▼
Route handler calls presence analyzer
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
|---|---|
| `targets` | Tracked user IDs, labels, notes, priority, active flag |
| `events` | Append-only log of every observed event with typed data |
| `presence_sessions` | Open/closed status sessions with platform and duration |
| `activity_sessions` | Game, Spotify, streaming, custom status sessions |
| `voice_sessions` | Voice channel sessions with co-participant tracking |
| `messages` | Message content, edit history, deletion timestamps |
| `typing_events` | Typing start events with ghost detection |
| `reactions` | Reaction add/remove with emoji metadata |
| `profile_snapshots` | Timestamped profile snapshots for change diffing |
| `guild_member_events` | Nickname and role changes |
| `alert_rules` | Configured alert conditions with fatigue tracking |
| `alert_history` | Fired alerts with acknowledgement state |
| `daily_summaries` | Pre-computed daily stat rows per target |
| `daily_briefs` | AI-generated daily intelligence briefs |
| `relationship_analysis` | AI-classified relationship pairs with confidence scores |
| `behavioral_baselines` | Per-target metric baselines for anomaly detection |
| `message_categories` | AI-assigned message category tags |
| `backfill_progress` | Per-channel backfill state tracking |
| `sync_state` | Supabase sync watermarks |
| `heartbeat_log` | Process health timestamps for stale session recovery |

SQLite is used instead of Postgres because Sentinel is a single-operator tool. WAL mode with a 64MB cache handles tens of millions of rows comfortably with no separate database process, trivial backup (copy the file), and trivial deployment.

---

## API Communication

The selfbot exposes a Fastify HTTP server on port `48923` by default.

All endpoints require an `Authorization` header matching `API_AUTH_TOKEN` from `.env`. Requests without a valid token receive `401`.

Live events are delivered over SSE at `GET /api/events/stream`. The plugin and web dashboard maintain persistent connections to this endpoint and react to incoming events by invalidating local caches and triggering UI refetches.

Full API reference: [api.md](api.md)

---

## Supabase Sync

When `DB_MODE=local+cloud` or `DB_MODE=cloud`, a background sync engine runs on a configurable interval (default 5 minutes) and pushes new/updated rows from SQLite to Supabase in batches.

Sync is always **SQLite → Supabase**, never the reverse (except on startup in `cloud` mode, where Supabase hydrates a fresh SQLite before Discord connects).

Full Supabase setup: [supabase.md](supabase.md)

---

## Key Design Decisions

**Op 14 over REQUEST_GUILD_MEMBERS for presence** — `REQUEST_GUILD_MEMBERS` (op 8) returns a snapshot of who is currently online in a guild. It can't push offline transitions. Op 14 with a `members` array tells Discord to track specific user IDs and push every status change including offline in real time. Absence from an op 8 chunk does not mean offline (visibility varies per guild), so sentinel never uses chunk absence as an offline signal.

**Selfbot over bot** — Discord bots cannot read user presence or messages in guilds without explicit server-level permissions and privileged intents that are gatekept by Discord. A user account sees everything the logged-in user sees. For personal tracking purposes, this is the only viable approach.

**SQLite over Postgres** — Single operator, single machine, no concurrency requirements. SQLite in WAL mode is faster for this workload than Postgres with a network hop, and requires zero infrastructure.

**Prepared statements singleton** — All SQL runs through pre-compiled statements initialized once at startup. No per-query compilation overhead, no SQL injection surface, and a single auditable file for all queries.

**SSE over WebSockets for UI** — The UI only needs one-way push from the selfbot. SSE is simpler (no upgrade handshake), reconnects automatically, and works over standard HTTP.

**fetch-based SSE in the plugin** — The browser's `EventSource` API doesn't support custom headers. Since the selfbot requires an `Authorization` header, the plugin uses a `fetch()`-based SSE reader instead, which supports arbitrary headers and is manually reconnected on disconnect.

**Fastify over Express** — Lower per-request overhead, built-in schema validation, better TypeScript types. Measurably faster for the high-frequency status and timeline endpoints.

**Browser fingerprint consistency** — The gateway IDENTIFY payload and REST headers (`User-Agent`, `X-Super-Properties`, `X-Discord-Locale`) use identical browser profile data, chosen once per process startup. This makes the selfbot's traffic indistinguishable from a real browser session. With `RANDOM_JITTER=true`, the profile is randomised per process restart across a pool of real browser fingerprints.
