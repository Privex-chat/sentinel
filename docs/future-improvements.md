# Future Improvements & Roadmap

This document covers planned features and directions for Sentinel. Items are grouped by theme. Everything here is planned, not built — unless the codebase says otherwise.

---

## 1. AI-Powered Social Graph Engine

This is the biggest planned addition and the one that changes Sentinel from a data logger into something genuinely intelligent.

### The Problem with the Current Social Graph

The current `social-graph.ts` analyzer is rule-based: it counts replies, reactions, mentions, and shared voice time, weights them, and classifies the result as "acquaintance", "voice buddy", or "frequent chat partner". That works as a starting point, but it misses the actual texture of relationships.

Two people who reply to each other 10 times could be close friends, or they could be two strangers who had one argument. The counts don't tell you that. Basic math can't distinguish flirting from friendly banter, or a moderator doing their job from someone who actually knows the target personally.

### What the AI Social Graph Does

The AI Social Graph Engine builds a full pipeline:

**1. Data Collection Layer (already exists)**
Messages, replies, reactions, mentions, voice co-presence, and shared channel activity are all logged. This is the raw material.

**2. Context Extraction Layer (new)**
For each interaction between the target and another user, extract features beyond counts:
- Message sentiment direction (is the target being warm, neutral, or hostile?)
- Response latency (how quickly does the target reply to this specific person vs. others?)
- Initiation ratio (who starts the conversations more often?)
- Topic clustering (what subjects come up with this person vs. others?)
- Conversation length (does the conversation die in 2 messages or go for 50+?)
- Private channel activity (voice channels vs. large public servers — tighter spaces indicate closer relationships)
- Time-of-day correlation (do they interact late at night? that's a different signal than daytime public chat)

**3. LLM Analysis Layer (new)**
A configurable LLM (local via Ollama, or cloud via OpenAI/Anthropic API) receives batched interaction summaries for a given pair of users and classifies the relationship. The prompt frames this as a behavioral analysis task, not a content-reading task — the model is analyzing patterns, not reproducing private messages.

Example classification outputs:
- `casual acquaintance` — periodic replies in public channels, no clear rapport
- `regular friend` — frequent mutual engagement, balanced initiation, warm tone
- `close friend` — high message volume, late-night conversations, lots of shared voice time, consistent topic depth
- `potential romantic interest` — elevated response speed to each other, higher emoji/reaction rate, private voice sessions, flirty tone patterns
- `group friend` — they only interact in group contexts, rarely one-on-one
- `server contact` — purely functional/topical interaction (gaming callouts, question-answering)
- `conflict relationship` — negative sentiment, argument patterns, mutual ignoring periods

**4. Confidence Scoring**
Each classification comes with a confidence score based on data volume. Less than 7 days of interactions → low confidence. 30+ days with consistent patterns → high confidence. The UI shows confidence so you know when to trust the output.

**5. Relationship Timeline**
Track how relationships evolve over time. If two people were "casual acquaintances" 3 months ago and are now "close friends," that shift is surfaced as a notable change. Relationship arc tracking over weeks and months.

### Implementation Plan

**Backend:**
- New `relationship_analysis` table: `(target_id, other_user_id, classification, confidence, reasoning, analyzed_at, data_window_start, data_window_end)`
- `src/analyzers/social-graph-ai.ts` — the pipeline coordinator
- `src/ai/provider.ts` — abstraction over LLM providers (Ollama, OpenAI, Anthropic)
- `src/ai/prompts.ts` — prompt templates for relationship classification
- New API endpoints:
  - `GET /api/targets/:userId/social/relationships` — all classified relationships
  - `GET /api/targets/:userId/social/relationships/:otherId` — specific pair deep-dive
  - `POST /api/targets/:userId/social/analyze` — trigger re-analysis
  - `GET /api/targets/:userId/social/changes` — relationship arc changes over time

**Config:**
```env
AI_PROVIDER=ollama         # ollama | openai | anthropic
AI_MODEL=llama3.2          # model name
AI_API_KEY=                # leave empty for ollama
AI_ANALYSIS_INTERVAL_MS=86400000   # re-analyze daily
```

**Plugin UI changes:**
- Social tab redesigned: relationship cards with classification badge, confidence meter, and key evidence
- Relationship history graph showing classification changes over time
- "Why?" expandable section showing the evidence that drove the classification

---

## 2. Automated Daily Intelligence Brief

**What it is:** Every morning, Sentinel generates a structured plaintext summary for each active target answering: what did they do, when were they active, anything unusual, what changed.

**What it looks like:**
```
DAILY BRIEF — Alex — Monday April 7

PRESENCE: Online 5h 40min. First seen 11:23am, last seen 11:58pm. Desktop only.

ACTIVITY: Played Valorant 3h 14min (2 sessions). Listened to Spotify 1h 22min.

MESSAGES: 47 messages across 4 channels. 3 deleted. 6 ghost typing events.

VOICE: 2h 18min. Joined #Gaming at 9:14pm with 3 others.

PROFILE: No changes.

ANOMALIES: None.
```

**Implementation:** Cron job at a configurable time. Stores briefs in a `daily_briefs` table. Exposed via `GET /api/targets/:id/briefs`. Displayed in a new "Briefs" tab in the plugin.

---

## 3. Historical Message Backfill

**What it is:** When you add a new target, Sentinel only starts collecting from that moment forward. Backfill walks backwards through message history in every shared channel to build a historical dataset.

**How it works:**
1. On target add, enumerate mutual guilds.
2. For each guild, enumerate readable channels.
3. Paginate backwards through each channel using the Discord API's `before` parameter.
4. Filter by author ID and insert as backfilled messages.
5. Track progress per channel in a `backfill_progress` table so interrupted backfills can resume.

**Config options:** max days to backfill, max messages per channel. Rate-limited to stay within API limits.

**UI:** Progress indicator in the target detail view — "Backfill: 73% (23 channels remaining)".

---

## 4. Message Content Categorization

**What it is:** Automatically tag messages into broad topic categories without reading the content yourself.

Categories: Gaming talk, Music, Venting/emotional, Humor/memes, Planning/logistics, Questions, General conversation.

**How it works:** An LLM processes recent messages in batches and assigns category tags. Tags are stored alongside message records. Analytics can then show "this person talks about gaming 40% of the time and venting 20% of the time."

**Note:** This analyzes the target's public messages — content they've chosen to post in shared spaces.

---

## 5. Alert Improvements

### Digest Mode
Instead of an alert per event, batch alerts into a digest notification every N minutes. Reduces noise for high-activity targets.

### Composite Alerts
"Notify me when this person comes online AND it's after midnight." Current system handles individual conditions; composite conditions require multiple rules or custom logic.

### Alert Fatigue Detection
If a rule has fired more than N times in the past 24 hours, automatically suppress further notifications and flag the rule as too noisy.

### Desktop Notifications
Pass alerts through to the OS notification system so you don't need Discord open to receive them.

---

## 6. Timeline Improvements

### Better Gantt View
Current Gantt bar shows today only. Extend it to allow browsing any date by picking from a calendar. Support zoom levels: 1 day, 3 days, 1 week.

### Searchable Event Log
Full-text search across the event log with type filters and date range. Currently only messages are searchable.

### Event Correlation
Highlight correlations visually — e.g., "every time this person goes offline late, they were playing Valorant beforehand." Automated correlation detection between behavioral patterns.

---

## 7. Weekly Automated Report

On a configurable schedule (weekly/monthly), generate a full structured report for a target:

- Executive summary (1 paragraph overview of the week)
- Behavioral stats vs. previous period (more/less active? different sleep pattern?)
- Notable changes: new games, profile updates, server joins
- Relationship changes: any new frequent contacts?
- Anomaly log for the period
- Charts as embedded images

Export as Markdown or PDF. Store in a `reports` table for historical reference.

---

## 8. Stale Session Recovery

**Problem:** If the selfbot crashes or is force-killed, open presence/activity/voice sessions are never closed. The next startup handles this by closing them with the current timestamp — but the duration is approximate.

**Improvement:** Log heartbeat timestamps periodically. On restart, close stale sessions at the last heartbeat timestamp rather than the restart timestamp. This produces more accurate session durations when the process exits uncleanly.

---

## 9. Data Retention Controls

**What it is:** A configurable policy for how long event data is kept.

- `DATA_RETENTION_DAYS=180` — events older than this are deleted or archived.
- Per-target overrides for targets that need longer/shorter retention.
- `DELETE /api/targets/:id?purge=true` — hard delete that removes all associated data, not just stops tracking.
- Auto-export before purge so nothing is lost permanently.

**Why this matters:** Long-running instances accumulate millions of rows. Retention policies keep the database size manageable and limit exposure if the machine is ever compromised.

---

## 10. OPSEC Mode

**What it is:** A plugin mode that hides Sentinel's presence from casual inspection.

- Removes the "Track with Sentinel" right-click menu item.
- Renames the settings panel to something generic.
- Suppresses any visual indicators that monitoring is active.

**Selfbot side:** Adds random jitter (±20%) to all polling intervals so they don't create a recognizable request pattern. Randomizes the `browser` and `os` fields in the gateway IDENTIFY payload.

---

## 11. Multi-Account Gateway Support

**What it is:** Run multiple selfbot accounts in parallel, each responsible for different targets or servers.

**Why:** A single account only sees events from servers it's a member of. Two accounts can cover twice the surface.

**Config:**
```env
DISCORD_TOKENS=token1,token2,token3
```

**Implementation:** One gateway client instance per token. Targets assigned to the account with the most mutual servers. All accounts write to the same SQLite database. API remains unchanged.

---

## 12. Materialized View Performance

**What it is:** As the events table grows into millions of rows, analytics queries slow down. Materialized views are pre-computed summaries that are updated incrementally rather than recomputed from scratch.

**In practice:** Add SQLite triggers that update `daily_summaries` in real time as events are inserted, rather than computing them in hourly batch jobs. Analytics queries read from these maintained summaries instead of scanning raw events.

**Result:** Sub-100ms analytics queries even with years of data.

---

## Out of Scope (Removed)

The following ideas from earlier roadmap drafts were removed because they don't fit the realistic use case:

- **Life event detection** (job changes, relationship status, moving cities) — too speculative, relies on inferring things from Discord behavior that can't be reliably inferred. Produces more noise than signal.
- **Multi-operator role system** — Sentinel is a single-user tool. Adding user accounts, roles, and audit logs adds significant complexity for a feature almost no one needs.
- **OSINT cross-platform integration** — scraping external platforms is legally and technically complex and outside what this tool is built for.
- **Predictive models for future behavior** — predicting what someone will do next based on 3 weeks of Discord data is not reliable enough to be useful.

---

*Last updated: April 2026*
