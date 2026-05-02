# Sentinel Self-Command System

Sentinel includes a self-command system that lets you manage tracking directly from Discord, without touching the web UI or API. Type a command in any channel — the triggering message is deleted **immediately** before anyone else sees it, and the response appears as a temporary message that self-deletes after a few seconds. No permanent trace remains in the channel.

---

## How It Works

The selfbot watches every `MESSAGE_CREATE` event it receives from the gateway. When a message from **its own account** starts with the command prefix (`$`), it:

1. Deletes the command message immediately (before processing)
2. Executes the command
3. Sends a brief response that auto-deletes after its TTL expires

Works in **any channel** the selfbot can see: mutual server channels, your own private servers, or even DMs with yourself.

---

## Command Reference

All commands accept either `<@mention>` or a raw Discord user ID snowflake.

---

### Target Management

#### `$add <@user>`
Add a new tracking target.

- Validates the user ID format
- Enforces a 15-minute rate limit between additions to avoid Discord account flags
- Starts presence subscription within 5 seconds (op 14 + REQUEST_GUILD_MEMBERS)
- Starts historical message backfill if `BACKFILL_ENABLED=true`
- If the target was previously removed and re-added, re-activates their record

**Response TTL:** 4 s

---

#### `$remove <@user>`
Permanently remove a target and delete all their tracking history from the database.

> Use `$pause` instead if you want to stop tracking temporarily while preserving history.

**Response TTL:** 4 s

---

#### `$pause <@user>`
Suspend tracking for a target without deleting any data. The target's presence, messages, events, and sessions are all preserved. Use `$resume` to re-activate.

**Response TTL:** 4 s

---

#### `$resume <@user>`
Re-activate a paused target. Triggers an immediate presence subscription so their current status is updated within 5 seconds.

**Response TTL:** 4 s

---

#### `$label <@user> <text>`
Set a human-readable display label for a target (max 100 characters). Labels appear in `$list` output, the web dashboard, and the plugin UI.

**Example:**
```
$label @someuser best friend
```

**Response TTL:** 4 s

---

#### `$note <@user> <text>`
Append a timestamped note to a target's notes field. Each note is prefixed with `[YYYY-MM-DD HH:MM]` in local time. Multiple notes accumulate — existing notes are never overwritten.

**Example:**
```
$note @someuser went quiet around the same time as before
```

**Response TTL:** 4 s

---

### Intelligence

#### `$status <@user>`
Show the target's current live presence and all active activities.

**Example output:**
```
🔴 123456789 — dnd on desktop
🎮 Activities:
  Playing: Valorant — Competitive
  Listening: Spotify — Blinding Lights (The Weeknd)
```

**Response TTL:** 8 s

---

#### `$seen <@user>`
Show when the target was last seen online. If they're currently active, shows how long they've been in that status.

**Example output (currently online):**
```
🟢 123456789 is currently online on desktop (1h 23m so far).
```

**Example output (offline):**
```
⚫ 123456789 last seen dnd on mobile — 2h 14m ago (2025-05-02 11:43).
🔴 That session lasted 3h 7m.
```

**Response TTL:** 7–8 s

---

#### `$uptime <@user>`
Show the target's total active (online + idle + dnd) time for the current calendar day, with a visual progress bar.

**Example output:**
```
⏱ 123456789 — 2025-05-02 active time:
4h 12m of 9h 38m elapsed  (43.5%)
`████████████░░░░░░░░ 43.5%`
```

**Response TTL:** 8 s

---

#### `$streak <@user>`
Show how long the target has been in their current status without interruption. If offline, shows how long they've been offline.

**Example output (online):**
```
🟢 123456789 has been online on desktop for 2h 17m
(since 09:26)
```

**Example output (offline):**
```
⚫ 123456789 has been offline for 4h 51m
(since 06:52)
```

**Response TTL:** 7 s

---

#### `$history <@user> [count]`
Show the last N presence transitions in reverse chronological order. Default is 10, maximum is 20.

**Example:**
```
$history @someuser 8
```

**Example output:**
```
📜 Last 8 sessions for 123456789:
🟢 online/desktop   09:26 → now    (2h 17m)
⚫ offline           06:52 → 09:25  (2h 33m)
🔴 dnd/desktop      04:11 → 06:51  (2h 40m)
🌙 idle/desktop     03:58 → 04:10  (12m)
🟢 online/desktop   01:22 → 03:57  (2h 35m)
...
```

**Response TTL:** 12 s

---

#### `$pattern <@user>`
Show a 30-day hourly activity heatmap using Unicode block characters. Each column represents one hour of the day (local time); bar height reflects how much total active time falls in that hour across the last 30 days.

**Example output:**
```
📈 Hourly activity pattern for 123456789 (last 30 days, local time):
`00 01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16 17 18 19 20 21 22 23`
` ▁  ▁  ▁  ▁  ▁  ▂  ▃  ▄  ▅  ▇  █  ▇  ▆  ▅  ▅  ▄  ▄  ▅  ▅  ▄  ▃  ▂  ▂  ▁`
Peak: 10:00–11:00 (8h 23m total)
```

**Response TTL:** 14 s

---

#### `$list`
Show all active targets with their current live status, platform, and label (if set).

**Example output:**
```
📋 Active targets (3):
🟢 `123456789` **(friend1)** — online/desktop
🔴 `987654321` — dnd/mobile
⚫ `111222333` **(old friend)** — offline
```

**Response TTL:** 10 s

---

### System

#### `$ping`
Measure REST API latency and gateway heartbeat round-trip time. Sends a placeholder message, then edits it with the actual measurements once they're available.

**Example output:**
```
🏓 Pong!
REST API:          48 ms
Gateway heartbeat: 61 ms
Gateway status:    ✅ connected
```

**Response TTL:** 8 s

---

#### `$stats`
Show overall system statistics.

**Example output:**
```
📊 Sentinel Stats
Targets:    4 active / 6 total
Events:     142,831 recorded
Alert rules: 7 active
DB size:    23.4 MB
Process up: 14h 22m
Gateway:    ✅ `a3f8c1d2e0...` (61 ms)
```

**Response TTL:** 10 s

---

#### `$reload`
Reload alert rules and runtime configuration from the database without restarting the selfbot. Equivalent to calling `PATCH /api/config` for each key and then triggering a rule reload.

**Response TTL:** 5 s

---

#### `$help`
Display the full command reference. The message self-deletes after 15 seconds.

**Response TTL:** 15 s

---

## Error Handling

If a command throws an error, the selfbot sends a `⚠️ Command failed: ...` message (TTL 6 s) and logs the full error. The command message is always deleted regardless of whether the command succeeds or fails.

## Notes

- All commands work in **guild channels, DMs with yourself, or your own private servers**
- Rate limiting on `$add` mirrors the API route (15 minutes between new targets) to protect the account
- The `$remove` command deletes all database records for the target. Use `$pause` / `$resume` to preserve history
- `$reload` does not reconnect the gateway — use it only for config/rule changes, not for credential updates
