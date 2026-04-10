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

You'll see a success message. All 13 tables are created with the correct indexes and RLS disabled.

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
| `targets` | Full sync every cycle (small table) |
| `alert_rules` | Full sync every cycle (small table) |
| `events` | New rows by ID, up to 5,000 rows per cycle |
| `presence_sessions` | New rows + re-sync rows opened in last 1 hour |
| `activity_sessions` | Same as presence_sessions |
| `voice_sessions` | Same as presence_sessions |
| `messages` | All messages created/edited/deleted since last sync |
| `typing_events` | New rows + re-sync recent rows |
| `reactions` | New rows + re-sync recent rows |
| `profile_snapshots` | New rows + re-sync recent rows |
| `guild_member_events` | New rows only |
| `alert_history` | New rows + re-sync recent rows |
| `daily_summaries` | Last 7 days always re-synced (updated hourly in-place) |

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
