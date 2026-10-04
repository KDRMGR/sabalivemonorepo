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
  `auth`/`storage`/`cron`/`vault`/`net` schemas before pushing.

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
