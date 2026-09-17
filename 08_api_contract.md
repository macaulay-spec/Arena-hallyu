# Hallyu — API Contract

> **Rewritten 2026-09-17 for Supabase.** This document previously described
> Firestore collections, Security Rules and Cloud Functions. The backend is
> Supabase (Postgres + PostgREST + Auth + Realtime + Storage + Edge Functions),
> and that changes the shape of the contract: instead of Rules guarding document
> paths, **RLS policies guard tables and columns**, and instead of Cloud
> Functions, most server-side logic is a **Postgres trigger or function** that
> runs inside the same transaction as the write.

## What the client actually talks to

There is no custom REST layer. The client uses `@supabase/supabase-js`, which
speaks **PostgREST** — so this *is* effectively a REST API, auto-derived from
the schema. That means the contract has three parts, and all three are
version-controlled SQL in `supabase/migrations/`:

1. **Schema** — tables, columns, types, constraints
2. **Grants** — what the `authenticated` and `anon` roles may even see
3. **RLS policies** — which rows those roles may touch

**Rule: if it isn't in a migration, it isn't the contract.** Do not encode access
rules in client code or Edge Functions.

## Table surface (client read/write)

"Read"/"Write" below describe the **RLS-enforced** surface, i.e. what a normal
`authenticated` user can do with the publishable key.

| Table | Read | Write | Notes |
|---|---|---|---|
| `profiles` | Own: full. Others: public columns only | Own row, public columns only | `follower_count` / `following_count` are **trigger-maintained** — a direct write is denied |
| `dramas` | All (public) | None | Catalog data, seeded from TVmaze. Service-role writes only |
| `actors` | All (public) | None | Same as `dramas` |
| `drama_cast` | All (public) | None | Join table |
| `episodes` | All (public) | None | `air_date` is `timestamptz` |
| `posts` | Public rows; community-gated and DM-scoped rows filtered by policy | Create: own. Update/delete: own (soft delete via `deleted_at`) | `like_count` / `comment_count` trigger-maintained |
| `media` | Via parent post | Create: own, referencing own upload | Stores `r2_key`, `duration_s`, `poster_key` |
| `comments` | Same visibility as parent post | Create: authenticated. Delete: own comment **or** post author | Two-branch policy — the author-deletes-any-comment case is easy to forget |
| `likes` | Counts only (via post) | Own rows; insert + delete | Unique constraint `(user_id, post_id)` makes double-tap idempotent |
| `follows` | Own rows + counts | Own rows | Unique `(follower_id, followee_id)`. Counts updated by trigger **in the same transaction** — no race, no client increment |
| `communities` | Public: all. Private: members only | Create: authenticated (topic communities). Drama communities are auto-created by trigger | |
| `community_members` | Own memberships; counts public | Own row (join/leave) | Drives `member_count` via trigger |
| `conversations` | Members only | Create: authenticated | 1:1 and group |
| `conversation_members` | Members only | Via conversation creation | Membership is the gate for both tables |
| `messages` | Members only | Members only | Realtime-subscribed, scoped per conversation |
| `channels` | All (public metadata) | None in MVP — creator program is gated | **Deliberately distinct from `conversations`.** Do not unify |
| `channel_subscriptions` | Own | Own | Subscription existence gates broadcast reads |
| `channel_broadcasts` | Subscribers only | Owner only | Policy checks an active row in `channel_subscriptions` |
| `notifications` | Own only | Update `read_at` only | Insert is **service-role/trigger only** — a user must not be able to fabricate a notification |
| `push_tokens` | None | Own rows | Write-only from the client's perspective |
| `reports` | None | Create: authenticated | Never readable back by the reporter — avoids revealing moderation decisions |
| `moderation_actions` | None | None | Service role only. Audit log |

**Columns that must never be client-writable:** every `*_count`, every
`moderation_status`, `is_banned` / `is_verified`, `email`, and `r2_key` on a row
belonging to someone else. Express this as column-level `GRANT`s *plus* RLS —
column grants alone are not sufficient, and RLS alone is easy to get wrong.

## Server-side operations

In Supabase these split three ways, and picking the wrong one is the main design
decision here:

### A. Postgres triggers / functions — same transaction, no HTTP

Use for anything that must be **atomically consistent** with the write.

| Function | Trigger | Purpose |
|---|---|---|
| `handle_new_user()` | `auth.users` AFTER INSERT | Create the `profiles` row. Without this, signup produces an auth user with no profile and every downstream read fails |
| `sync_follow_counts()` | `follows` AFTER INSERT/DELETE | Update both `profiles.follower_count` and `following_count` |
| `sync_post_counts()` | `likes`, `comments` AFTER INSERT/DELETE | Update `posts.like_count` / `comment_count` |
| `sync_community_members()` | `community_members` AFTER INSERT/DELETE | Update `communities.member_count` |
| `sync_profile_display()` | `profiles` AFTER UPDATE | Propagate denormalized display name/avatar to cached rows, if you denormalize at all (prefer a view over denormalizing — see below) |
| `auto_create_drama_community()` | `dramas` AFTER INSERT | Seed the matching community row |
| `fan_out_post_notifications()` | `posts` AFTER INSERT | Insert `notifications` rows for followers |

> **Prefer a view to a trigger where you can.** `follower_count` maintained by a
> trigger is a denormalization you must keep correct forever. A materialized
> view refreshed on a schedule, or a plain aggregate on read for small counts,
> is less code and cannot drift. Use triggers only where the count is read on
> every feed render and the aggregate would be too expensive.

### B. `pg_cron` + `pg_net` — scheduled, in-database

| Job | Schedule | Purpose |
|---|---|---|
| `notify_new_episodes()` | hourly | Find `episodes` whose `air_date` crossed into the past hour; insert notifications for followers of that drama. **Compare in UTC, display in the user's zone** — K-dramas air on KST |
| `refresh_recommendations()` | nightly | Recompute the rule-based ranking table |
| `expire_pending_moderation()` | every 15 min | Anything stuck in `pending` beyond the SLA gets escalated, not silently published |
| `purge_soft_deleted()` | weekly | Hard-delete rows past retention |

Both extensions are available on Supabase. Prefer `pg_cron` over an external
scheduler — it's free and it's already in the database.

### C. Edge Functions (Deno) — anything needing external HTTP or long work

| Function | Purpose |
|---|---|
| `moderate-content` | Call the moderation provider on text/image, set `moderation_status`. Triggered from the insert path via `pg_net` or invoked by the client after upload |
| `moderate-video` | Frame sampling (1 frame / 5 s → ~60 frames for a 5-minute clip) + per-frame moderation. **This is the long one** — must be async, must not block the upload response |
| `push-fanout` | Send push via Expo Push Service for a batch of `notifications` rows |
| `catalog-sync` | TVmaze ingest/refresh. Service-role, cron-invoked |
| `r2-signed-upload` | Issue the upload URL for video, and **enforce the 5-minute cap server-side** by checking the row's `duration_s` before it becomes visible |

**Every Edge Function must verify the incoming JWT itself.** PostgREST does this
for you; an Edge Function that forgets is an unauthenticated endpoint.

## Realtime

| Channel | Table | Scope |
|---|---|---|
| `dm:{conversation_id}` | `messages` | Members only — Realtime applies RLS, so membership is the filter |
| `channel:{channel_id}` | `channel_broadcasts` | Subscribers only |
| `thread:{post_id}` | `comments` | Public posts |

Supabase Realtime applies RLS to `postgres_changes`, which is the reason this
works at all — but it also means **a policy bug is a Realtime leak**, not just a
REST one. Test it (`11_test_strategy.md` §4).

Subscribe narrowly. One channel per open conversation, torn down on unmount;
a global "all my messages" channel does not scale and does not respect the
membership boundary cleanly.

## Storage

**Do not use Supabase Storage for video at MVP.** Free tier is 1 GB and uploads
are capped at 50 MB — a 5-minute clip can exceed both. Video goes to
**Cloudflare R2** ($0 egress, 10 GB free storage). Supabase Storage is fine for
**avatars and post images** (small, and it keeps RLS in one place).

That split means `media` holds two kinds of reference: a Supabase Storage path
or an R2 key. Keep them distinguishable in the column (`storage_provider`) rather
than guessing from a prefix.

Bucket policies are RLS: a user may write only under their own
`{user_id}/` prefix, may not overwrite another user's object, and public buckets
must be public-read *only* for image MIME types you actually serve.

## Versioning

No URL versioning, because PostgREST derives its surface from the schema. So:

1. **Additive-only by default** — new nullable columns, new tables. Never rename
   or drop a column while an old client build is in the wild.
2. **This matters more than usual here.** With link-distributed APKs and no store
   review, users run *old* versions indefinitely. An EAS Update fixes the JS but
   not an installed-but-never-updated APK. A dropped column breaks those users
   silently and permanently.
3. Where a breaking change is unavoidable, add the new column, dual-write,
   migrate readers, then drop in a later release — with a `schema_version` on the
   app bundle checked against a server-side minimum at startup.

## Before this can be implemented

**`09_database_schema.md` does not exist.** This document describes the *surface*;
the actual DDL, the RLS policy text, the trigger bodies and the migration
ordering are all still unwritten, and they are the deliverable that
`04_ai_studio_build_prompt.md` needs. Write `09` next.
