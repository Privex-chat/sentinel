# API Reference

The selfbot exposes a Fastify HTTP API on port `48923` (configurable). All endpoints require a Bearer token matching `API_AUTH_TOKEN` from the selfbot's `.env`.

```
Authorization: Bearer <your_api_auth_token>
```

Base URL: `http://localhost:48923` (or your configured host/port)

All responses are JSON unless otherwise noted.

---

## System

### GET /api/status

Returns selfbot health and statistics.

```json
{
  "uptime": 86400000,
  "uptimeFormatted": "1d 0h 0m",
  "eventCount": 142837,
  "targetCount": 3,
  "activeTargets": 3,
  "dbSizeBytes": 52428800,
  "dbSizeMB": 50.0,
  "startedAt": 1712345678000
}
```

---

## Targets

### GET /api/targets

Returns all tracked users.

```json
[
  {
    "user_id": "123456789012345678",
    "added_at": 1712345678000,
    "label": "My Target",
    "notes": "Some notes",
    "priority": 1,
    "active": 1
  }
]
```

`priority`: `0` = normal, `1` = high, `2+` = critical.
`active`: `1` = tracking, `0` = paused.

### POST /api/targets

Add a new target.

**Body:**
```json
{
  "userId": "123456789012345678",
  "label": "Optional label",
  "notes": "Optional notes",
  "priority": 0
}
```

`userId` must be a valid Discord snowflake (17–20 digit numeric string). Returns `400` if invalid.

**Response:** `{ "success": true, "userId": "..." }`

### DELETE /api/targets/:userId

Remove a target. Stops tracking but does **not** delete historical data.

### PATCH /api/targets/:userId

Update target metadata. All fields optional.

**Body:**
```json
{
  "label": "New label",
  "notes": "Updated notes",
  "priority": 2,
  "active": false
}
```

---

## Target State

### GET /api/targets/:userId/status

Returns the target's current live state from in-memory tracking.

```json
{
  "target": { ... },
  "presence": {
    "status": "online",
    "platform": "desktop",
    "clientStatus": { "desktop": "online" }
  },
  "activities": [
    {
      "name": "Valorant",
      "type": 0,
      "details": "Competitive — In Game",
      "state": "5 Wins — Gold 2"
    }
  ],
  "voiceState": {
    "guildId": "...",
    "channelId": "...",
    "selfMute": false,
    "selfDeaf": false,
    "streaming": false
  },
  "profile": { ... }
}
```

Activity types: `0` = Playing, `1` = Streaming, `2` = Listening (Spotify), `4` = Custom Status.

---

## Events

### GET /api/events

Query the event log with optional filters.

**Query parameters:**

| Param | Description |
|-------|-------------|
| `targetId` | Filter to a specific user |
| `type` | Filter by event type (e.g., `PRESENCE_UPDATE`) |
| `since` | Unix timestamp ms — events after this time |
| `until` | Unix timestamp ms — events before this time |
| `guildId` | Filter to a specific guild |
| `channelId` | Filter to a specific channel |
| `limit` | Max results (default 100) |
| `offset` | Pagination offset (default 0) |

### GET /api/events/stream

Server-Sent Events stream. Pushes new events in real time.

**Query parameters:**
- `targetId` — filter to a specific user (optional)

**Response:** `Content-Type: text/event-stream`

Each event: `data: {"target_id":"...","event_type":"...","timestamp":...,"data":{...}}`

Initial message on connect: `data: {"type":"connected"}`

---

## Event Types

| Event Type | Trigger |
|------------|---------|
| `PRESENCE_UPDATE` | Status changed |
| `INITIAL_PRESENCE` | First presence observed this session |
| `PLATFORM_SWITCH` | Switched between desktop/mobile/web |
| `ACTIVITY_START` | Started a game or activity |
| `ACTIVITY_END` | Stopped a game or activity |
| `SPOTIFY_START` | Started Spotify |
| `SPOTIFY_END` | Stopped Spotify |
| `STREAMING_START` | Started streaming |
| `STREAMING_END` | Stopped streaming |
| `CUSTOM_STATUS_SET` | Set custom status |
| `CUSTOM_STATUS_CLEARED` | Cleared custom status |
| `MESSAGE_CREATE` | Sent a message |
| `MESSAGE_UPDATE` | Edited a message |
| `MESSAGE_DELETE` | Deleted a message |
| `TYPING_START` | Started typing |
| `GHOST_TYPE` | Typed but didn't send within 15 seconds |
| `VOICE_JOIN` | Joined a voice channel |
| `VOICE_LEAVE` | Left a voice channel |
| `VOICE_MOVE` | Moved between voice channels |
| `VOICE_STATE_CHANGE` | Mute/deafen/streaming state changed |
| `PROFILE_UPDATE` | Any profile field changed |
| `AVATAR_CHANGE` | Avatar changed |
| `USERNAME_CHANGE` | Username changed |
| `NICKNAME_CHANGE` | Server nickname changed |
| `ROLE_ADD` | Role added in a guild |
| `ROLE_REMOVE` | Role removed from a guild |
| `REACTION_ADD` | Added a reaction |
| `REACTION_REMOVE` | Removed a reaction |
| `SERVER_JOIN` | Joined a new server |
| `SERVER_LEAVE` | Left a server |
| `ACCOUNT_CONNECTED` | Connected an external account |
| `ACCOUNT_DISCONNECTED` | Disconnected an external account |
| `DM_CHANNEL_OPENED` | A DM channel was opened with the selfbot |
| `ALERT` | An alert rule was triggered |

---

## Timeline

### GET /api/targets/:userId/timeline

Returns recent events plus presence, activity, and voice sessions (Gantt chart data).

**Query parameters:** `limit`, `offset`, `type` (event type filter)

**Response:**
```json
{
  "events": [ ... ],
  "presenceSessions": [ ... ],
  "activitySessions": [ ... ],
  "voiceSessions": [ ... ]
}
```

### GET /api/targets/:userId/timeline/day/:date

All events and sessions for a specific day. Date format: `YYYY-MM-DD`.

---

## Analytics

All analytics endpoints accept a `days` query parameter (default varies).

### GET /api/targets/:userId/analytics/presence

`?days=30`

Presence time distribution by status and platform, plus `totalActiveMs` (online + idle + DND combined).

### GET /api/targets/:userId/analytics/activities

`?days=90`

Full gaming profile — playtime per game, session counts, peak hours, recently started, abandoned games.

### GET /api/targets/:userId/analytics/messages

`?days=30`

Communication stats — average length, edit/delete/ghost rates, vocabulary richness, messages by hour.

### GET /api/targets/:userId/analytics/voice

`?days=30`

Voice habits — total time, session count, mute ratio, preferred channels, top co-participants.

### GET /api/targets/:userId/analytics/social

`?days=30`

Social graph — scored connections ranked by interaction volume, with relationship classification.

### GET /api/targets/:userId/analytics/heatmap

Weekly activity heatmap — 7×24 grid of event counts.

### GET /api/targets/:userId/analytics/daily

`?days=30`

Array of pre-computed daily summaries (newest first). Each row includes `total_active_minutes` = `online + idle + dnd`.

### GET /api/targets/:userId/analytics/music

`?days=30`

Spotify profile — top artists, top songs, total listening time, recent track.

### GET /api/targets/:userId/analytics/typing

Ghost typing stats — total typing events, ghost count, ghost rate, average delay.

---

## Insights

### GET /api/targets/:userId/insights

Combined overview — all four analyzers in one response.

### GET /api/targets/:userId/insights/sleep

`?days=14`

Estimated bedtime and wake time from offline session patterns. Includes confidence score and weekday/weekend split.

### GET /api/targets/:userId/insights/routine

`?weeks=4`

Weekly routine heatmap with activity summary and current anomalies.

### GET /api/targets/:userId/insights/availability

`?weeks=4`

Four 7×24 probability matrices (online, messaging, voice, gaming). Values 0.0–1.0.

### GET /api/targets/:userId/insights/anomalies

`?days=7`

Detected behavioral deviations sorted newest first. Types: `UNUSUAL_HOUR`, `HIGH_MESSAGE_VOLUME`, `LOW_MESSAGE_VOLUME`, `NEW_GAME`, `PROFILE_CHANGE`, `GHOST_TYPE_SPIKE`.

---

## Messages

### GET /api/targets/:userId/messages

`?search=text&limit=100&offset=0`

All messages, with optional full-text search.

### GET /api/targets/:userId/messages/deleted

Messages where `deleted_at IS NOT NULL`.

### GET /api/targets/:userId/messages/edited

Messages where `edited_at IS NOT NULL`. Includes `edit_history` array.

---

## Profiles

### GET /api/targets/:userId/profile/current

Most recent profile snapshot.

### GET /api/targets/:userId/profile/history

`?limit=50`

All snapshots in reverse chronological order.

---

## Alerts

### GET /api/alerts/rules

All configured alert rules.

### POST /api/alerts/rules

Create a new alert rule.

**Body:**
```json
{
  "targetId": "123456789012345678",
  "ruleType": "COMES_ONLINE",
  "condition": {}
}
```

`targetId` is optional — omit for a global rule applying to all targets.

**Rule types and conditions:**

| Rule Type | Condition |
|-----------|-----------|
| `COMES_ONLINE` | `{ "after_hour": 22 }` (optional) |
| `GOES_OFFLINE` | `{}` |
| `STATUS_CHANGE` | `{ "field": "transition", "value": "online->dnd" }` (optional) |
| `STARTS_ACTIVITY` | `{ "value": "Valorant" }` (optional substring match) |
| `STOPS_ACTIVITY` | `{ "value": "Valorant" }` (optional) |
| `JOINS_VOICE` | `{ "field": "guildId", "value": "..." }` (optional) |
| `LEAVES_VOICE` | `{}` |
| `SENDS_MESSAGE` | `{ "field": "channelId", "value": "..." }` (optional) |
| `DELETES_MESSAGE` | `{}` |
| `GHOST_TYPES` | `{}` |
| `PROFILE_CHANGE` | `{}` |
| `UNUSUAL_HOUR` | `{ "start_hour": 2, "end_hour": 6 }` |
| `NEW_GAME` | `{}` |
| `KEYWORD_MENTION` | `{ "value": "keyword1,keyword2" }` |

### DELETE /api/alerts/rules/:id

Delete an alert rule.

### GET /api/alerts/history

`?targetId=...&limit=50&offset=0`

Fired alerts with acknowledgment status.

### PATCH /api/alerts/history/:id/ack

Mark an alert as acknowledged.

---

## Export

### GET /api/export/:userId

Full JSON export of all data for a target.

### GET /api/export/:userId/csv

CSV export of all events. Sets `Content-Disposition: attachment` header.
