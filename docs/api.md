# Sentinel Selfbot — REST & SSE API Reference

The selfbot exposes a local HTTP API on port `48923` (configurable via `API_PORT`). All endpoints under `/api/` require authentication.

---

## Authentication

Every `/api/*` request must include the `Authorization` header with a `Bearer` prefix:

```
Authorization: Bearer your_api_auth_token
```

The token must match `API_AUTH_TOKEN` in your `.env`. Comparison is constant-time (`crypto.timingSafeEqual`) to prevent token-prefix timing attacks. Missing header → `401`. Wrong token → `403`.

`/health` (unauthenticated liveness probe) and `OPTIONS` preflights are exempt.

---

## Base URL

```
http://localhost:48923
```

---

## Cross-cutting Behaviour

| Concern | Default |
|---|---|
| **CORS** | Allowlist via `API_CORS_ORIGINS` env (comma-separated). Unset → `https://sentinel-panel.vercel.app`, `http://localhost:5173`, `http://localhost:3000`. A literal `*` reflects any origin. |
| **Rate limit** | 300 req/min/IP via `@fastify/rate-limit`. `/health` is allowlisted. 429 carries `Retry-After`. |
| **Error responses** | Unhandled handler errors return `{ "error": "Internal server error", "requestId": "<id>" }`. Full detail (with stack) is in the selfbot's logs, keyed by `requestId`. Schema-validation errors return 400 with `{ "error": "Invalid request", "details", "requestId" }`. |
| **LIKE search escape** | `?search=` query params are escaped — `%` and `_` from the client are treated as literal substring characters, not wildcards. |
| **Body limit** | Fastify default (1 MB). |

---

## Liveness

### `GET /health`
**Unauthenticated.** Cheap probe for cloud platforms (Railway, Fly, uptime monitors).

**Response:**
```json
{ "status": "ok", "uptimeMs": 86423154, "gatewayConnected": true }
```

`gatewayConnected` is `null` if the gateway client hasn't been wired up yet (immediately after server start).

---

## Targets

### `GET /api/targets`
List all monitored targets (active and inactive).

**Response:** array of target objects
```json
[
  {
    "user_id": "123456789012345678",
    "added_at": 1714680000000,
    "label": "friend1",
    "notes": "[2025-05-01 14:23] went quiet",
    "priority": 0,
    "active": 1,
    "timezone": "America/New_York"
  }
]
```

`timezone` is an IANA identifier added in schema v7. Targets created before v7 (or POSTed without a `timezone` field) default to `"UTC"`. Drives every per-target hour/day analyser.

`bootstrap_completed_at` is an epoch-ms timestamp added in schema v8. `null` while the initial profile fetch is pending; a number once the target is operational. While `null`, the selfbot:
- suppresses `PROFILE_UPDATE` / `AVATAR_CHANGE` / `USERNAME_CHANGE` events
- early-returns from alert evaluation for **every** rule type
- returns an empty array from `detectAnomalies()`

Targets created before v8 are auto-migrated to `bootstrap_completed_at = added_at` so they start operational. Fresh targets get flipped on the first successful `/users/{id}/profile` fetch — usually within seconds of the immediate post-add bootstrap call. A 30-min stuck-bootstrap sweep is the backstop.

---

### `POST /api/targets`
Add a new tracking target.

**Body:**
```json
{
  "userId": "123456789012345678",
  "label": "optional label",
  "notes": "optional notes",
  "priority": 0,
  "timezone": "America/New_York"
}
```

- `userId` must be a valid Discord snowflake (17–20 digits)
- Rate limited to 1 new target per 15 minutes (returns `429` with `retryAfterMs`)
- `timezone` is optional. When provided it must be a valid IANA zone — invalid values return `400`. Omitted → defaults to `"UTC"`.

**Response:**
```json
{ "success": true, "userId": "123456789012345678" }
```

---

### `DELETE /api/targets/:userId`
Remove a target and all their tracking data.

Cascade: every per-target SQL row is removed via FK `ON DELETE CASCADE`. In-memory state — presence cache, voice state, typing pendings, guild-member cache, target-profile-poller failure tracking, alert composite tracker, NEW_GAME known-games cache, social-graph cache — is cleared. Any in-flight backfill is **hard-cancelled** at the next loop checkpoint (channel page, channel, or guild boundary).

**Response:** `{ "success": true }`

---

### `PATCH /api/targets/:userId`
Update target metadata.

**Body** (all fields optional):
```json
{
  "label": "new label",
  "notes": "new notes",
  "priority": 1,
  "active": false,
  "timezone": "Europe/London"
}
```

Use `null` to clear `label` or `notes`. Setting `active: false` pauses tracking without deleting data. Changing `timezone` triggers an immediate refresh of the selfbot's in-memory tz cache so the next analytics query sees the new value. Invalid IANA zones return `400`.

---

### `POST /api/targets/:userId/bootstrap/complete`
Force-complete a stuck onboarding bootstrap. Idempotent — already-operational targets return 200 with the existing timestamp.

Use this when the immediate post-add profile fetch failed (target has no mutual guilds AND the basic `/users/{id}` endpoint also failing) and you want alerts/anomalies to start flowing immediately rather than waiting for the 30-min sweep.

**Response:**
```json
{
  "success": true,
  "bootstrap_completed_at": 1714680000000,
  "wasAlreadyComplete": false
}
```

`wasAlreadyComplete` is `true` when the target was already operational before this call. Returns `404` for unknown user IDs.

---

## Current Status

### `GET /api/targets/:userId/status`
Get a target's current live state.

**Response:**
```json
{
  "target": { ... },
  "presence": {
    "status": "dnd",
    "platform": "desktop",
    "clientStatus": { "desktop": "dnd" }
  },
  "activities": [
    { "name": "Valorant", "type": 0, "details": "Competitive", "state": "In Game" }
  ],
  "voiceState": null,
  "profile": { "username": "...", "avatar_hash": "...", ... },
  "isBootstrapping": false
}
```

`isBootstrapping` mirrors `target.bootstrap_completed_at == null` — surfaced separately so UIs don't have to re-derive. While `true`, alerts and anomaly surfacing for this target are suppressed.

---

### `GET /api/status`
Server health check.

**Response:**
```json
{
  "uptime": 86423,
  "eventCount": 142831,
  "targetCount": 6,
  "activeTargetCount": 4,
  "dbSizeBytes": 24539136
}
```

---

## Events

### `GET /api/events`
Query the event log with filters.

**Query params:**
| Param | Description |
|---|---|
| `targetId` | Filter by user ID |
| `type` | Filter by event type (e.g. `PRESENCE_UPDATE`, `MESSAGE_CREATE`) |
| `since` | Start timestamp (ms) |
| `until` | End timestamp (ms) |
| `guildId` | Filter by guild |
| `channelId` | Filter by channel |
| `search` | Full-text search in event data |
| `limit` | Max results (1–1000, default 100) |
| `offset` | Pagination offset |

**Event types include:**
`PRESENCE_UPDATE`, `PLATFORM_SWITCH`, `INITIAL_PRESENCE`, `ACTIVITY_START`, `ACTIVITY_END`, `INITIAL_ACTIVITY`, `SPOTIFY_START`, `SPOTIFY_END`, `STREAMING_START`, `STREAMING_END`, `CUSTOM_STATUS_SET`, `CUSTOM_STATUS_CLEARED`, `VOICE_JOIN`, `VOICE_LEAVE`, `VOICE_MOVE`, `VOICE_STATE_CHANGE`, `MESSAGE_CREATE`, `MESSAGE_UPDATE`, `MESSAGE_DELETE`, `TYPING_START`, `GHOST_TYPE`, `PROFILE_UPDATE`, `AVATAR_CHANGE`, `USERNAME_CHANGE`, `REACTION_ADD`, `REACTION_REMOVE`, `NICKNAME_CHANGE`, `ROLE_ADD`, `ROLE_REMOVE`, `DM_CHANNEL_OPENED`, `SERVER_JOIN`, `SERVER_LEAVE`, `ACCOUNT_CONNECTED`, `ACCOUNT_DISCONNECTED`, `ALERT`, `ALERT_DIGEST`, `ALERT_SUPPRESSED`

---

### `GET /api/events/stream`
Server-Sent Events stream for real-time event delivery.

**Query params:**
- `targetId` — optional, filter stream to a specific target
- `since` — optional monotonic event id. On connect, the server replays buffered events with `id > since` before going live, so a brief reconnect doesn't drop events. The 500-entry in-memory buffer is process-local — a server restart resets the id counter (clients should treat post-restart as a fresh stream).

**Event format:**
```
id: 12345
data: {"target_id":"123...","event_type":"PRESENCE_UPDATE","timestamp":1714680000000,"data":{...}}
```

The `id:` line is the same `lastEventId` your `EventSource` consumer sees automatically — pass it back as `?since=` on reconnect.

The connection opens with a `{"type":"connected","lastEventId":N}` frame so the client knows where the live stream picks up. Keepalive `: ping` comments are sent every 25 seconds.

---

## Messages

### `GET /api/targets/:userId/messages`
Query collected messages.

**Query params:** `search`, `channelId`, `guildId`, `since`, `until`, `source`, `category`, `limit` (max 500), `offset`

### `GET /api/targets/:userId/messages/deleted`
Messages captured before deletion. Limit max 500.

### `GET /api/targets/:userId/messages/edited`
Messages that were edited (includes edit history). Limit max 500.

---

## Profile

### `GET /api/targets/:userId/profile/current`
Latest profile snapshot.

### `GET /api/targets/:userId/profile/history`
Profile snapshot history showing all detected changes. Limit max 200, default 50.

---

## Timeline

### `GET /api/targets/:userId/timeline`
Unified timeline combining events and sessions.

**Query params:** `type`, `event_types` (comma-separated), `since`, `until`, `search`, `limit` (max 1000), `offset`

### `GET /api/targets/:userId/timeline/day/:date`
All events and sessions for a single day. Date format: `YYYY-MM-DD`.

### `GET /api/targets/:userId/timeline/range`
Date-range Gantt view (max 30 days). Requires `from` and `to` query params (`YYYY-MM-DD`).

---

## Analytics

All analytics endpoints accept a `days` query param (1–365) unless noted otherwise.

| Endpoint | Description |
|---|---|
| `GET /api/targets/:userId/analytics/presence` | Status breakdown by time, platform split, total active minutes |
| `GET /api/targets/:userId/analytics/activities` | Gaming profile — top games, session durations, activity patterns |
| `GET /api/targets/:userId/analytics/messages` | Communication style — volume, length, deletion rate, ghost-type rate |
| `GET /api/targets/:userId/analytics/voice` | Voice habits — total time, top channels, co-participants |
| `GET /api/targets/:userId/analytics/social` | Social graph relationships |
| `GET /api/targets/:userId/analytics/heatmap` | Activity heatmap by hour/day of week. Uses `weeks` param (1–52) |
| `GET /api/targets/:userId/analytics/daily` | Daily summary rows |
| `GET /api/targets/:userId/analytics/music` | Spotify profile — top tracks, artists, listening hours |
| `GET /api/targets/:userId/analytics/categories` | Message category breakdown |
| `GET /api/targets/:userId/analytics/baselines` | Behavioral baselines for anomaly detection |
| `GET /api/targets/:userId/analytics/typing` | Typing statistics (ghost-type rate, delay distributions) |
| `POST /api/targets/:userId/analytics/baselines/recompute` | Trigger baseline recomputation |

### Target Config

`GET /api/targets/:userId/config` — get analytics weights and anomaly threshold
`PATCH /api/targets/:userId/config` — update `social_weight_messages`, `social_weight_reactions`, `social_weight_voice_hours`, `social_weight_mentions`, `anomaly_z_threshold`

---

## Insights

All insight endpoints run in the **target's own IANA timezone** (`targets.timezone`, default UTC). Set per target via `POST/PATCH /api/targets` `timezone` body field or the `$tz` self-command.

| Endpoint | Description |
|---|---|
| `GET /api/targets/:userId/insights` | All insights combined |
| `GET /api/targets/:userId/insights/sleep` | Inferred sleep schedule — response includes `timezone` so consumers can render local times |
| `GET /api/targets/:userId/insights/routine` | Detected routine patterns — 7×24 grid in local DOW/hour |
| `GET /api/targets/:userId/insights/availability` | Predicted availability windows |
| `GET /api/targets/:userId/insights/anomalies` | Behavioral anomaly events (SQL-aggregated; uses `json_extract` for new-game detection) |
| `GET /api/targets/:userId/insights/correlations` | Event co-occurrence correlations |

---

## Social Graph

| Endpoint | Description |
|---|---|
| `GET /api/targets/:userId/social/relationships` | All classified relationships (rule-based + AI) |
| `GET /api/targets/:userId/social/relationships/:otherId` | Deep-dive for a specific pair |
| `POST /api/targets/:userId/social/analyze` | Trigger AI re-analysis (returns 202, runs async) |
| `GET /api/targets/:userId/social/changes` | Relationship arc changes over time |

---

## Alerts

### `GET /api/alerts/rules`
List all alert rules.

### `POST /api/alerts/rules`
Create an alert rule.

**Body:**
```json
{
  "targetId": "123456789012345678",
  "ruleType": "COMES_ONLINE",
  "condition": { "after_hour": 22 },
  "digestMode": false,
  "fatigueThreshold": 20,
  "compositeCondition": null
}
```

**Validation (server-side):**
- `ruleType` must be one of the values below — unknown types return `400`.
- `compositeCondition`, when provided, must be `{ operator, window_ms, conditions: [{ rule_type, condition? }, ...] }`. Empty or missing `conditions`, non-string `rule_type`, or a sub-`rule_type` not in the enum below all return `400`.
- `fatigueThreshold` must be a positive integer.

**Available rule types:**
`COMES_ONLINE`, `GOES_OFFLINE`, `STARTS_ACTIVITY`, `STOPS_ACTIVITY`, `JOINS_VOICE`, `LEAVES_VOICE`, `SENDS_MESSAGE`, `DELETES_MESSAGE`, `GHOST_TYPES`, `STATUS_CHANGE`, `PROFILE_CHANGE`, `UNUSUAL_HOUR`, `NEW_GAME`, `KEYWORD_MENTION`

**Per-target timezone matters for:**
- `COMES_ONLINE` with `after_hour` — fires only when the target's local hour ≥ `after_hour`
- `UNUSUAL_HOUR` — `start_hour` / `end_hour` window evaluated in the target's local clock

**Digest mode** controls *SSE batching only*. Webhook delivery is always immediate per real event; `digest_mode=1` (or env `ALERT_DIGEST_MODE=true`) additionally emits an aggregated `ALERT_DIGEST` SSE event every `ALERT_DIGEST_INTERVAL_MS` for the dashboard live feed.

### `DELETE /api/alerts/rules/:id`
Delete a rule.

### `PATCH /api/alerts/rules/:id`
Toggle a rule enabled/disabled.

### `GET /api/alerts/history`
Alert history. Params: `limit` (max 500), `offset`, `targetId`.

### `PATCH /api/alerts/history/:id/ack`
Acknowledge an alert.

### `GET /api/alerts/rules/suppressed`
List auto-suppressed rules.

### `POST /api/alerts/rules/:id/unsuppress`
Manually unsuppress a rule and reset its fire count.

### `POST /api/alerts/test`
Send a test webhook notification.

---

## Backfill

| Endpoint | Description |
|---|---|
| `GET /api/targets/:userId/backfill/progress` | Progress summary + per-channel breakdown |
| `POST /api/targets/:userId/backfill/start` | Start or resume backfill (202 Accepted) |
| `POST /api/targets/:userId/backfill/custom` | Custom backfill with `mode`: `new_channels` or `full_reset` |
| `POST /api/targets/:userId/backfill/pause` | Pause an in-progress backfill |

---

## Daily Briefs

| Endpoint | Description |
|---|---|
| `GET /api/targets/:userId/briefs` | Brief history. `limit` max 365, default 30 |
| `GET /api/targets/:userId/briefs/:date` | Brief for a specific date (`YYYY-MM-DD`) |
| `POST /api/targets/:userId/briefs/generate` | Generate brief for today (or `?date=YYYY-MM-DD`) |

---

## Runtime Config

### `GET /api/config`
Get all runtime-reloadable configuration values. Sensitive values (tokens, keys) are masked.

### `PATCH /api/config`
Update a single config key at runtime without restarting.

**Body:**
```json
{ "key": "ALERT_DIGEST_MODE", "value": "true" }
```

Hot-reloadable keys include all polling intervals, AI settings, backfill settings, alert settings, and brief generation time. Core keys (`DISCORD_TOKEN`, `DB_MODE`, etc.) require a restart and cannot be changed via this endpoint.

**Validation (server-side):**
- `AI_PROVIDER` must be one of: `none`, `ollama`, `openai`, `anthropic`, `gemini`. Unknown values return `400`.
- `BRIEF_GENERATION_TIME` must be `HH:MM` (00:00–23:59).
- Interval keys (`*_INTERVAL_MS`, `*_POLL_*`) have minimum values (typically 60 s) — values below the floor return `400`.
- `BACKFILL_ENABLED`, `ALERT_DIGEST_MODE` parse as `"true"`/`"false"` exactly. Any other string is `false`.

Sensitive keys (`DISCORD_TOKEN`, `AI_API_KEY`, `SUPABASE_SERVICE_KEY`, `ALERT_WEBHOOK_URL`, `CRITICAL_WEBHOOK_URL`) are encrypted at rest with AES-256-GCM when `SENTINEL_DATA_KEY` is set (recommended in cloud mode). `GET /api/config` masks their values as `••••••••` in the response.

---

## Export

### `GET /api/export/:userId`
Streaming **NDJSON** (Content-Type `application/x-ndjson`). One JSON object per line. Sections are framed by marker rows so consumers can parse incrementally:

```
{"_section":"meta","userId":"123...","exportedAt":1714680000000}
{"_section":"events_start"}
{"_section":"events", ...row...}
{"_section":"events", ...row...}
{"_section":"events_end","rows":42}
{"_section":"messages_start"}
{"_section":"messages", ...row...}
...
{"_section":"complete","totalRows":N}
```

Tables exported in order: `events`, `messages`, `presence_sessions`, `activity_sessions`, `voice_sessions`, `profile_snapshots`, `reactions`, `typing_events`, `daily_summaries`.

The server uses `better-sqlite3.iterate()` so only one row is materialised in memory at a time — multi-million-row exports never OOM the process. Treat unknown `_section` values as opaque (forward-compatible).

If the stream is truncated mid-export, a `{"_section":"error","message":"..."}` line is emitted before the connection closes — consumers should detect truncation rather than treating a partial file as complete.

### `GET /api/export/:userId/csv`
Streaming CSV of the `events` table only. Row-by-row via the same cursor pattern. CSV-injection protection: cells starting with `=`/`+`/`-`/`@`/`\t`/`\r` are prefixed with `'` so Excel/Sheets can't interpret them as formulas.

---

## Common Patterns

### Timestamps
All timestamps are Unix milliseconds (`Date.now()` in JavaScript, or `EXTRACT(EPOCH FROM NOW()) * 1000` in SQL).

### Pagination
Most list endpoints accept `limit` and `offset`. Default limits vary by endpoint (typically 50–100). Hard caps noted above.

### Async operations
Backfill start, brief generation, and social graph analysis return `202 Accepted` immediately and run in the background. Poll the progress/result endpoints to check completion.
