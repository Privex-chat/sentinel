# Sentinel Selfbot — REST & SSE API Reference

The selfbot exposes a local HTTP API on port `48923` (configurable via `API_PORT`). All endpoints under `/api/` require authentication.

---

## Authentication

Every request must include the `Authorization` header:

```
Authorization: your_api_auth_token
```

The value must match `API_AUTH_TOKEN` in your `.env`. There is no `Bearer` prefix.

---

## Base URL

```
http://localhost:48923
```

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
    "active": 1
  }
]
```

---

### `POST /api/targets`
Add a new tracking target.

**Body:**
```json
{
  "userId": "123456789012345678",
  "label": "optional label",
  "notes": "optional notes",
  "priority": 0
}
```

- `userId` must be a valid Discord snowflake (17–20 digits)
- Rate limited to 1 new target per 15 minutes
- Returns `429` with `retryAfterMs` if rate limited

**Response:**
```json
{ "success": true, "userId": "123456789012345678" }
```

---

### `DELETE /api/targets/:userId`
Remove a target and all their tracking data.

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
  "active": false
}
```

Use `null` to clear `label` or `notes`. Setting `active: false` pauses tracking without deleting data.

---

## Current Status

### `GET /api/targets/:userId/status`
Get a target's current live state.

**Response:**
```json
{
  "presence": {
    "status": "dnd",
    "platform": "desktop",
    "clientStatus": { "desktop": "dnd" }
  },
  "activities": [
    { "name": "Valorant", "type": 0, "details": "Competitive", "state": "In Game" }
  ],
  "voiceState": null,
  "profileSnapshot": { "username": "...", "avatar_hash": "...", ... }
}
```

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
`PRESENCE_UPDATE`, `PLATFORM_SWITCH`, `INITIAL_PRESENCE`, `ACTIVITY_START`, `ACTIVITY_END`, `SPOTIFY_START`, `SPOTIFY_END`, `STREAMING_START`, `STREAMING_END`, `CUSTOM_STATUS_SET`, `CUSTOM_STATUS_CLEARED`, `VOICE_JOIN`, `VOICE_LEAVE`, `VOICE_MOVE`, `VOICE_STATE_CHANGE`, `MESSAGE_CREATE`, `MESSAGE_UPDATE`, `MESSAGE_DELETE`, `TYPING_START`, `GHOST_TYPE`, `PROFILE_UPDATE`, `AVATAR_CHANGE`, `USERNAME_CHANGE`, `REACTION_ADD`, `REACTION_REMOVE`, `NICKNAME_CHANGE`, `ROLE_ADD`, `ROLE_REMOVE`, `DM_CHANNEL_OPENED`

---

### `GET /api/events/stream`
Server-Sent Events stream for real-time event delivery.

**Query params:**
- `targetId` — optional, filter stream to a specific target

**Event format:**
```
data: {"target_id":"123...","event_type":"PRESENCE_UPDATE","timestamp":1714680000000,"data":{...}}
```

Keepalive `: ping` comments are sent every 25 seconds to prevent connection drops.

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

| Endpoint | Description |
|---|---|
| `GET /api/targets/:userId/insights` | All insights combined |
| `GET /api/targets/:userId/insights/sleep` | Inferred sleep schedule |
| `GET /api/targets/:userId/insights/routine` | Detected routine patterns |
| `GET /api/targets/:userId/insights/availability` | Predicted availability windows |
| `GET /api/targets/:userId/insights/anomalies` | Behavioral anomaly events |
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
  "condition": {},
  "digestMode": false,
  "fatigueThreshold": 20,
  "compositeCondition": null
}
```

**Available rule types:**
`COMES_ONLINE`, `GOES_OFFLINE`, `STARTS_ACTIVITY`, `STOPS_ACTIVITY`, `JOINS_VOICE`, `LEAVES_VOICE`, `SENDS_MESSAGE`, `DELETES_MESSAGE`, `GHOST_TYPES`, `STATUS_CHANGE`, `PROFILE_CHANGE`, `UNUSUAL_HOUR`, `NEW_GAME`, `KEYWORD_MENTION`

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

---

## Export

### `GET /api/export/:userId`
Export all data for a target as a single JSON object containing events, messages, presence/activity/voice sessions, profile snapshots, reactions, typing events, and daily summaries.

### `GET /api/export/:userId/csv`
Export events as a CSV file. Includes CSV-injection protection (formula characters prefixed with `'`).

---

## Common Patterns

### Timestamps
All timestamps are Unix milliseconds (`Date.now()` in JavaScript, or `EXTRACT(EPOCH FROM NOW()) * 1000` in SQL).

### Pagination
Most list endpoints accept `limit` and `offset`. Default limits vary by endpoint (typically 50–100). Hard caps noted above.

### Async operations
Backfill start, brief generation, and social graph analysis return `202 Accepted` immediately and run in the background. Poll the progress/result endpoints to check completion.
