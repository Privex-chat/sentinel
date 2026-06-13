# Supabase / Cloud Backup

Sentinel's primary database is SQLite. Supabase is an optional cloud mirror — useful for backup, cross-device access, or running the selfbot on ephemeral platforms like Railway or Render where the local filesystem is wiped on redeploy.

---

## How It Works

Sentinel always uses SQLite as the live working database. When Supabase is enabled, a background sync engine periodically pushes new and updated rows to your Supabase project in batches. All reads still come from SQLite — Supabase is write-only from the selfbot's perspective (except on startup in `cloud` mode).

### Database Modes

| `DB_MODE` | SQLite | Supabase |
|-----------|--------|----------|
| `local` (default) | Primary store — all reads and writes | Not used |
| `local+cloud` | Primary store — all reads and writes | Async mirror, synced on interval |
| `cloud` | Hydrated from Supabase on startup; used as working store | Permanent store; data persists across redeploys |

**Note on `cloud` mode:** On startup, Sentinel pulls all data from Supabase into a fresh local SQLite before connecting to Discord. This means all historical data is immediately available. The local file still exists during the run — it's the working database. On the next cold start (e.g., after a Railway redeploy), the process repeats.

---

## Step 1 — Create a Supabase Project

1. Go to [supabase.com](https://supabase.com) and sign up for a free account
2. Click **New Project**
3. Choose a name, set a database password, and pick a region close to where your selfbot runs
4. Wait for provisioning (~1 minute)

---

## Step 2 — Run the Schema

1. In your Supabase dashboard, go to **SQL Editor → New Query**
2. Open `supabase-schema.sql` from the `sentinel-selfbot` repository
3. Paste the entire contents into the SQL editor
4. Click **Run**

You'll see a success message. All tables are created with the correct indexes and RLS disabled. The schema is idempotent — re-run it whenever you upgrade the selfbot to pick up new columns (e.g. v7 added `targets.timezone`).

---

## Step 3 — Get Your Credentials

### Project URL

**Settings → API → Project URL**

Looks like: `https://abcdefghijklmnop.supabase.co`

### Service Role Key

**Settings → API → Project API Keys → `service_role`**

> ⚠️ Use the `service_role` key, not the `anon` key. The service role key bypasses Row Level Security, which is required for Sentinel to write data. Keep this key private — it has full database access.

---

## Step 4 — Configure Your `.env`

```env
# Choose: local | local+cloud | cloud
DB_MODE=local+cloud

SUPABASE_URL=https://abcdefghijklmnop.supabase.co
SUPABASE_SERVICE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# How often to push to Supabase (milliseconds)
# local+cloud default: 300000 (5 min)
# cloud recommended:    30000 (30 sec — limits data loss on crash)
SUPABASE_SYNC_INTERVAL_MS=300000
```

---

## Step 5 — Start the Selfbot

```bash
npm start
```

On startup you'll see:

```
[INFO] [SupabaseSync] Supabase sync enabled (DB_MODE=local+cloud)
[INFO] [SupabaseSync] Supabase connection OK
[INFO] [SupabaseSync] Supabase sync will run every 300s
```

The first sync starts 60 seconds after startup. For existing databases with lots of data, the first few sync cycles will backfill historical rows in batches.

---

## Verifying Data

In your Supabase dashboard, go to **Table Editor**. You should see rows in `events`, `targets`, `presence_sessions`, etc.

You can also query directly in the SQL Editor:

```sql
SELECT COUNT(*) FROM events;
SELECT * FROM targets;
SELECT * FROM daily_summaries ORDER BY date DESC LIMIT 10;
```

---

## Sync Behavior by Table

| Table | Strategy |
|-------|----------|
| `targets` | Full sync every cycle (small table) — includes `timezone` column (v7+) |
| `alert_rules` | Full sync every cycle (small table) |
| `events` | New rows by ID, up to 5,000 rows per cycle |
| `presence_sessions` | New rows + re-sync rows opened in last 1 hour |
| `activity_sessions` | Same as presence_sessions |
| `voice_sessions` | Same as presence_sessions |
| `messages` | Created/edited/deleted within the last 7 days since last sync watermark |
| `typing_events` | New rows + re-sync recent rows |
| `reactions` | New rows + re-sync recent rows |
| `profile_snapshots` | New rows + re-sync recent rows |
| `guild_member_events` | New rows only |
| `alert_history` | New rows + re-sync recent rows |
| `daily_summaries` | **Today + yesterday only** (tightened from 7 days). `computeDailySummaries` only writes today's row; the 2-day window catches the final pre-midnight value if the process restarted overnight. |
| `relationship_analysis` | New rows by ID |
| `relationship_history` | New rows by ID |
| `daily_briefs` | Watermarked by `generated_at` |
| `backfill_progress` | Watermarked by `COALESCE(completed_at, started_at)` |
| `behavioral_baselines` | Watermarked by `computed_at` |
| `target_config` | Watermarked by `updated_at` |
| `message_categories` | Watermarked by `categorized_at` |
| `runtime_config` | Watermarked by `updated_at`; per-instance `_internal_*` keys are filtered out so multiple bots sharing one Supabase project don't cross-contaminate. Sensitive values arrive encrypted (`enc:v1:` envelope) when `SENTINEL_DATA_KEY` is set — Supabase only ever sees ciphertext. |

---

## Encrypting Secrets at Rest

Sensitive `runtime_config` values — `DISCORD_TOKEN`, `AI_API_KEY`, `SUPABASE_SERVICE_KEY`, `ALERT_WEBHOOK_URL`, `CRITICAL_WEBHOOK_URL` — are stored in the `runtime_config` table whenever you update them through the dashboard (`PATCH /api/config`) or `$reload`. In `local+cloud` / `cloud` mode they're then mirrored to Supabase, which means a leaked Supabase service key would also leak your Discord token unless you encrypt those values at rest.

Set a 32-byte base64 key in `.env`:

```env
SENTINEL_DATA_KEY=$(node -e "console.log(require('crypto').randomBytes(32).toString('base64'))")
```

Once set, all writes through `setRuntimeConfig` are AES-256-GCM encrypted before they hit SQLite (and therefore before they reach Supabase). The selfbot logs a loud warning at startup if `DB_MODE=local+cloud` / `cloud` and the key is missing — values would otherwise sync in plaintext.

Treat the key like any other secret. Losing it makes existing `enc:v1:`-prefixed rows unreadable; rotating it requires re-writing every encrypted row through `PATCH /api/config` so the new key takes effect. Legacy plaintext rows continue to work — encryption is upgraded lazily on next write.

---

## Restoring from Supabase

If your local `sentinel.db` is lost:

**Option 1 — Use `cloud` mode (automatic)**

Set `DB_MODE=cloud` and restart. Sentinel will pull everything from Supabase before connecting to Discord.

**Option 2 — Export and reimport**

Use the Supabase CLI:

```bash
supabase db dump \
  --db-url "postgresql://postgres:<password>@db.<project-ref>.supabase.co:5432/postgres" \
  > backup.sql
```

Or use the **Table Editor → Export** feature for individual tables.

---

## Troubleshooting

**`Supabase connection test failed`**

- Double-check `SUPABASE_URL` and `SUPABASE_SERVICE_KEY` in `.env`
- Make sure you ran `supabase-schema.sql` — the tables must exist before the first sync

**`Supabase upsert error on "events": ...`**

- The schema wasn't applied correctly. Re-run `supabase-schema.sql` (all statements use `CREATE TABLE IF NOT EXISTS` — safe to re-run)

**Sync is slow on first run**

- Expected. Large existing databases sync in batches of 5,000 rows per cycle. To speed up, temporarily lower `SUPABASE_SYNC_INTERVAL_MS` to `60000`, then restore it.

**Tables missing in Supabase**

- Re-run the schema SQL. `IF NOT EXISTS` makes it safe to run multiple times.
