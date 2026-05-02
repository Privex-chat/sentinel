# Presence Tracking — Technical Overview

This document explains how Sentinel tracks Discord presence in real time, why it works the way it does, and what its limitations are.

---

## The Problem With Polling

A naive approach to presence tracking would be to periodically ask Discord "is this user online?" But Discord's API doesn't expose presence for arbitrary users — it only surfaces presence for users you're connected to via friends, shared guilds, or active subscriptions. Even when presence is available, polling introduces a lag equal to the poll interval: if you poll every 2 minutes, you can miss short online sessions entirely and always know offline status up to 2 minutes late.

Sentinel solves this with an event-driven approach using Discord's gateway.

---

## Op 14 — GUILD_SUBSCRIBE with Member Targeting

Discord's gateway uses **opcodes** (numeric message types) for different operations. Op 14 is `GUILD_SUBSCRIBE`, an undocumented opcode used by Discord's own client to receive presence updates for a guild.

The key insight is the `members` field. Without it, op 14 only delivers presence updates for users currently visible in the member list sidebar — which covers a tiny fraction of a large guild and never reliably delivers offline transitions. With `members: ["userId1", "userId2", ...]`, Discord registers those specific user IDs for tracking in that guild and pushes **every status change** for those users, including:

- online → idle → dnd → offline transitions
- Platform changes (desktop ↔ mobile ↔ web) even without a status change
- Activity start/stop (games, Spotify, custom status, streaming)

This is what Sentinel sends for each mutual guild when you add a tracked target:

```json
{
  "op": 14,
  "d": {
    "guild_id": "the_mutual_guild_id",
    "typing": false,
    "activities": true,
    "threads": false,
    "members": ["target_user_id"]
  }
}
```

---

## What Triggers a PRESENCE_UPDATE

Once subscribed, Discord pushes a `PRESENCE_UPDATE` dispatch event (op 0) to the gateway connection any time the target's status or activities change. Sentinel's handler processes this immediately:

- Status changes (`online` → `dnd` → `offline` etc.) create a new presence session in the database
- Platform changes (`desktop` → `mobile`) emit a `PLATFORM_SWITCH` event, even if status stays the same
- Activity changes trigger `ACTIVITY_START` / `ACTIVITY_END` events, with type-specific handling for Spotify, streaming, and custom status

The key correctness property: **offline detection fires when the target actually goes offline**, not the next time a poll cycle runs. There is no polling lag.

---

## Subscription Lifecycle

Op 14 subscriptions are not permanent. Discord can silently drop them — after a reconnect, after a session resume, or after they age out. Sentinel renews subscriptions aggressively:

| Event | What happens |
|---|---|
| Gateway `READY` | Subscribe to all mutual guilds with member IDs, 2 s delay |
| Session `RESUMED` | Re-subscribe to all guilds, 3 s delay |
| Every 4 minutes | Periodic renewal across all active targets and guilds |
| New target added | Immediate subscription within 5 s of the `$add` or API call |

For the periodic renewal, Sentinel also sends `REQUEST_GUILD_MEMBERS` (op 8) to get the current snapshot — useful for confirming online status after a reconnect where events may have been missed.

---

## Initial Presence Discovery

When the gateway first connects or after a resume, Sentinel sends `REQUEST_GUILD_MEMBERS` for all tracked targets across all mutual guilds. Discord responds with `GUILD_MEMBERS_CHUNK` events containing presence data for users who are currently active (non-offline).

**Important:** absence from a chunk does not mean offline. A user can be visible in guild A's presence stream but absent from guild B's due to per-guild visibility rules, privacy settings, or guild size. Treating chunk absence as "offline" causes false oscillation. Sentinel therefore:

- **Uses chunks only for initial discovery** (first time a target is seen, no existing state)
- **Confirms online status** when a user IS in a chunk's presences
- **Never sets a user offline** based on chunk absence — only real `PRESENCE_UPDATE` events do that

---

## Why Chunks Can't Detect Offline

Discord's `GUILD_MEMBERS_CHUNK` presences list only includes users who are actively online, idle, or DND **and** visible to the requesting account in that guild. Multiple factors can exclude a user from the list even when they're online:

- The user is in a large guild where presence is only streamed for visible sidebar members
- The user has presence hidden in that specific guild
- The user is in a different privacy tier

Relying on chunk absence as an offline signal causes rapid false transitions when you query multiple guilds and the user appears in some but not others. The correct approach — which Sentinel uses — is to let op 14 member subscriptions deliver offline transitions directly.

---

## Presence State Machine

Sentinel maintains an in-memory `Map<userId, PresenceState>` alongside the database. This avoids unnecessary database writes when nothing changes.

```
unknown  ─── first PRESENCE_UPDATE ──▶  online / idle / dnd / offline
   │
   └── first chunk with user absent ──▶  offline  (initial only, no session opened)

online ─── PRESENCE_UPDATE ──▶  idle / dnd / offline
idle   ─── PRESENCE_UPDATE ──▶  online / dnd / offline
dnd    ─── PRESENCE_UPDATE ──▶  online / idle / offline
offline ── PRESENCE_UPDATE ──▶  online / idle / dnd
```

Rules:
- Presence sessions only open for non-offline states
- The first observation of an already-offline user skips session creation (no meaningful start time exists)
- `PRESENCE_UPDATE` events fire to the database, SSE stream, and alert engine **only when status changes** — platform-only changes emit `PLATFORM_SWITCH` separately without a spurious `PRESENCE_UPDATE`
- All open sessions for a target are closed atomically on any transition (prevents orphaned sessions)

---

## Limitations

### Mutual server required

Discord only delivers presence for users connected to the selfbot account. The only covert connection available is a shared guild. If a target shares no server with the selfbot account, their presence is invisible to the gateway — there is no workaround.

Friendship would solve this (friends get presence without guild context) but is visible to the target and defeats the covert use case.

### Activities require guild presence

Activity data (games, Spotify, etc.) flows through the same op 14 stream. No mutual server = no activity data.

### Offline to offline

If a target was offline before the selfbot connected and never comes online, the selfbot may not have a presence record for them until their first online transition is observed. The initial `REQUEST_GUILD_MEMBERS` call covers this for users who are already online at connect time.

### Gateway reconnects

During a gateway reconnect or resume, any status changes that happened while the connection was down are missed. The reconnect flow re-requests all presences immediately, so the gap is bounded by the reconnect duration (typically seconds for resumes, up to a minute for full reconnects).

---

## Data Flow Summary

```
Discord gateway
      │
      │  PRESENCE_UPDATE (op 0)
      ▼
handlePresenceUpdate(userId, data)
      │
      ├─▶ Close all open presence sessions for userId
      ├─▶ Open new session for new status (if not offline after unknown)
      ├─▶ Insert PRESENCE_UPDATE event (if status changed)
      ├─▶ Insert PLATFORM_SWITCH event (if platform changed)
      ├─▶ Evaluate alert rules
      ├─▶ Push to SSE stream
      └─▶ Update in-memory currentPresence map

handleActivityUpdate(userId, activities)
      │
      ├─▶ Diff old vs new activity list
      ├─▶ Close ended activity sessions
      ├─▶ Open new activity sessions
      ├─▶ Insert typed events (ACTIVITY_START / SPOTIFY_START / etc.)
      ├─▶ Evaluate alert rules
      └─▶ Push to SSE stream
```
