# SABALIVE monorepo

Two independent apps on one Supabase project (ref `sfehzhtqtpuobnrvzvzp`):
`sabalive/` (Flutter consumer app, owns the database) and `sabaliveadmin/`
(React + Vite admin panel + `landing/` public site). See each folder's CLAUDE.md.

## Git
- Each app is its own repo, linked here as a **submodule**. Commit and push inside
  the app first, then `git add sabalive sabaliveadmin` and commit the pointer here.
- Always `cd` into an app before running its commands; nothing is shared between them.
- Never commit secrets. The panel only ever holds the anon key; `service_role` stays
  in Edge Functions.

## Database (lives in `sabalive/`)
- Migrations: `sabalive/supabase/migrations/` (timestamped, applied in order).
  Apply with `supabase db push` from `sabalive/` (CLI is linked to the project above).
  `--dry-run` first. The Supabase MCP in this account does NOT reach this project.
- Edge Functions: `sabalive/supabase/functions/` (`supabase functions deploy <name>`).
- Deploy order when a change touches both: migration -> functions -> panel/app.
  The panel can query columns the app doesn't need yet, so DB first.
- Validate migrations by replaying all of them on a scratch local Postgres with stubbed
  `auth`/`storage`/`cron`/`vault`/`net` schemas before pushing: `sabalive/scripts/db/replay_migrations.sh`.

## Production safety (read before touching the database, Edge Functions, or a release)
The one Supabase project is LIVE: the Play app (and old versions of it that stay in pockets for months),
the panel and real users all use it. Full steps: `sabalive/docs/DEPLOY_CHECKLIST.md`.
- **Never run anything against production without the user saying so in this conversation**: `supabase db push`,
  `functions deploy`, SQL through `db query`, restarting, maintenance mode, "log out all users". A past approval
  covers that one task only. Read-only checks (`scripts/prod/healthcheck.sh`) are fine.
- **Only add while old apps are live.** New table / column with a default / function with a NEW name: safe.
  Renaming or dropping anything the app uses, changing a function's arguments or return type, `not null`
  on an existing column, tightening a policy the app needs: never. Add `..._v2` and retire the old one later.
  Changing a function body, a trigger on a busy table, `alter column type`, or a bulk `update` is risky:
  quiet hour only (about 21:00-04:00 UTC), small batches, `set local lock_timeout`.
- **Every change is a migration in git.** No hand-run SQL on production. `supabase db push` applies EVERYTHING
  pending, including other sessions' unreviewed work: read `--dry-run`, and if it lists a migration you did not
  write or were not asked to ship, stop and ask.
- **Before a push:** `sabalive/scripts/db/replay_migrations.sh` (ALL_OK), a `supabase/rollbacks/<version>_down.sql`
  for each migration, `scripts/prod/healthcheck.sh` and `scripts/prod/smoke_play_app.sh` clean, and
  `scripts/prod/backup.sh` (BACKUP OK; the free plan has no backups). **After:** run the last two again.
- **If production misbehaves:** healthcheck first, smoke test second, then the down script of the suspect
  migration. Maintenance mode and "log out all users" are last resorts, not first reactions.
- **Don't change the version code or build a release unless asked.** Build Play bundles with
  `sabalive/scripts/release/build_play_bundle.sh`, which refuses a staging build.
- The database has only **60 connections** on this plan: don't add polling or per-client queries that multiply.

## Staging and branches
- **Staging** is a second Supabase project, `gsloixbaktefwgxzjtps` (production is `sfehzhtqtpuobnrvzvzp`), built from the same
  migrations with fake data. Use `sabalive/scripts/staging/*` (they use their own link folder, never production's) and read
  `sabalive/docs/STAGING.md`. Anything risky is rehearsed there first, then promoted to production with the same migration
  files via `docs/DEPLOY_CHECKLIST.md`. Never copy production data into staging.
- **Branches:** work on `develop` in each repo; `main` is production (the panel deploys from it). Move work to `main` only
  when the owner asks for a release. Don't commit straight to `main`.

## Roles
Ladder: Super > Master (`admin`) > Global > Country > Sub > Agency. Enforced in
Postgres (RLS + `SECURITY DEFINER` RPCs); the panel's capability list
(`sabaliveadmin/src/lib/capabilities.js`) mirrors `role_baseline()` in SQL and must
stay in sync. Staff accounts are refused by the consumer app.

## Ghost IDs (monitoring accounts)
Super Admin creates them (`ghost-admin` function); they sign in to the app and watch
any live invisibly and view-only. Super Admin and Master can also watch from the
panel (Live Monitor). Key design points, so you don't break them:
- `profiles.is_ghost` is set only via `app_metadata.ghost` (server-side). The
  profiles SELECT policy hides ghost rows from everyone but the ghost and Super Admin.
- `is_ghost()` is deliberately NOT executable by app users (it would reveal ghosts).
- `join_live_stream` leaves no viewer row / chat line for ghosts; `assert_not_ghost`
  triggers refuse their writes. New user-written tables should get that trigger.
- Panel counts/lists filter `is_ghost = false`.

## Panel-managed app features (keep app and panel in sync)
- Levels: `track_levels` (Wealth / Charm XP + join image) edited in Master -> Levels; the join row
  carries `level_image_url` and the app plays it full-screen.
- Coin sellers (`offline_coin_sellers`, linked to a user), Support chat (`support_threads/messages`),
  Lucky IDs (`lucky_ids`; owning one swaps `profiles.display_id`, expiry restores it), user Bag assign
  and profile edit (`admin_*` RPCs), Master seat handover (`master_handover_staff_seat`).
- Newer panel capability keys (`monitor_lives`, `manage_levels`, `manage_support`) are on by default for
  a Master and enforced in SQL by `staff_cap_on(key)` (explicit false blocks).
- Realtime cannot filter DELETE events: never put a `filter:` on a delete listener, check the old record.
