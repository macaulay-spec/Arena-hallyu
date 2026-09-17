# Hallyu — Architecture & Plan Review

*Reviewed 2026-09-16 against the 11 documents currently in this repository.*

> ## ⚠️ The stack assumption in this review is wrong — read this first
>
> **Hallyu is React Native + Expo, not Kotlin Multiplatform.** That was confirmed
> 2026-09-17, after this review was written. The pack's own documents said KMP,
> so the review assessed the KMP plan on its own terms — competently, but against
> a stack that isn't being built.
>
> **Everything below about the backend (Supabase, RLS, R2, egress, moderation,
> TVmaze, budgets, roadmap realism) still stands.** Everything about the *client*
> is superseded. Specifically:
>
> | This review says | Now |
> |---|---|
> | `supabase-kt` is community-maintained, effectively one maintainer — a load-bearing risk (§1, §9) | **Void.** `@supabase/supabase-js` is Supabase's own first-party client with a documented React Native quickstart. Risk gone. |
> | Compose/Wasm renders to a `<canvas>`, so **zero SEO** — build a separate prerendering system | **Void, and better than fixed.** Expo Router does static rendering (build-time HTML) and `generateMetadata` for per-route SEO. It's a framework feature, not a project. |
> | A `<video>` element can't render inside a Skia canvas — needs `wasmJsMain` DOM interop, position-synced to a swipe gesture (§6, §8) | **Void.** `react-native-web` renders real DOM; `expo-video` maps to a real `<video>`. This was the hardest problem in the plan and it no longer exists. |
> | Client-side video compression is a platform project — `Media3 Transformer` in `androidMain`, `AVAssetExportSession` in `iosMain` | **Void.** It's one prop: `expo-image-picker`'s `videoQuality`. Which also re-encodes iPhone HEVC → H.264 for free. |
> | Offline-first needs SQLDelight `.sqm` migrations; SQLDelight on Wasm ships a second WASM artifact | **Replaced.** `expo-sqlite` on native; a two-tier storage interface on web (§4b of `16`). |
> | macOS CI minutes are critical — a KMP project must compile an iOS target per run | **Reduced.** Expo's free tier gives 15 iOS builds/month and local builds are free. PR pipeline is Linux-only. |
> | 16 KB page-size alignment is a known KMP/Kotlin-Native runtime gotcha | **Moot** for Expo; still worth verifying on a real 16 KB emulator image. |
> | Ktor, kotlinx.serialization, Koin, Coil 3, Voyager, Coroutines/Flow (§9 library choices) | **Replaced.** TanStack Query, Zustand, NativeWind, FlashList v4, Reanimated v4, expo-image, expo-router. |
> | Video cap is 60 seconds | **Wrong regardless of stack — it's 5 minutes.** Storage, not bandwidth, becomes the binding constraint (§2.1 of `16`). |
>
> **Also void from the earlier banner:** `04_ai_studio_build_prompt.md` has been
> rewritten. It instructed an AI to scaffold a Kotlin/Gradle project and would
> have generated the wrong app. That was the single most consequential defect in
> the pack and it is fixed.
>
> **Read this review for the backend and product analysis. For the client, read
> `03_technical_architecture_expo.md` and §3–§4 of `16_zero_budget_stack.md`.**

> **Superseded in part by `16_zero_budget_stack.md`** (same date), which was
> written after a "must run on zero budget" constraint was added. Three
> conclusions below are revised there:
>
> - **§4 (TMDB licensing)** — TVmaze is a free, no-API-key alternative with
>   episode air dates, schedules, and alternate titles. TMDB's ~$149/month
>   becomes a later gap-filler, not a launch blocker. See `16` §5.
> - **§6 (video pipeline)** — Cloudflare R2 has **$0 egress, uncapped, forever**,
>   and Cloudflare's terms explicitly permit serving video from it via their
>   CDN. This turns video from the largest cost risk into a near-zero one, and
>   makes the "defer the transcode pipeline" instinct correct after all — via
>   client-side compression rather than Supabase Storage. See `16` §3.
> - **§8 ("first iOS build installable via TestFlight" in Phase 0)** — **void.**
>   The free Apple ID tier does not grant push or Sign in with Apple
>   entitlements, so two features `06` requires cannot even be *developed* at
>   $0. MVP becomes Android + Web; keep `iosMain` compiling in CI. See `16` §4.
>
> Everything else in this review stands, and `16` §12 lists the concrete edits
> both documents imply for the rest of the pack.

## Verdict

The product thinking is strong. `01_vision_and_case.md` identifies a real gap
(fandom-native social home vs. five half-fitting generic tools), and the
"weekly episode cadence as a structural retention hook" argument is the best
single insight in the pack — it's a genuine, defensible wedge that generic
platforms aren't built to exploit. The design system brief in
`05_stitch_uiux_prompt.md` is unusually mature: it covers empty/error/offline
states, spoiler treatment, and imagery normalization for inconsistent source
art, which is where most product design packs fall down.

The **engineering documentation is not yet buildable**, and the blocker is not
ambition — it's internal contradiction and missing foundations. Specifically:

1. The pack is **split-brained on backend**: 2 documents say Supabase, 6 still
   describe Firebase/Firestore in implementation detail.
2. **4 documents that the others depend on do not exist**, including the
   database schema.
3. The **entire product premise depends on a drama/episode catalog whose data
   source is never named** — and the obvious source (TMDB) has a licensing
   clause that this product likely trips.
4. The **named content-moderation vendor is being shut down at the end of this
   year.**

None of these are fatal. All of them are cheap to fix *now* and expensive to
fix after code exists. Details below.

---

## 1. The pack contradicts itself on backend (P0) — **RESOLVED 2026-09-17**

You asked about migrating to Supabase. The important finding was that **you
already had — on paper, in two documents — and never finished.** The result was
a spec that could not be handed to an engineer or to an AI build agent without
producing two different apps.

**This is now fixed.** The table below is kept as the record of what was wrong;
the "Now" column is current.

| Document | Backend it *specified* | Now |
|---|---|---|
| `03_technical_architecture_expo.md` *(was `..._kmp.md`)* | **Supabase** (Postgres + Auth + Realtime + Storage) | ✅ Supabase, and rewritten for Expo. Storage split: **video → Cloudflare R2**, avatars/post images → Supabase Storage |
| `04_ai_studio_build_prompt.md` | **Supabase** via Ktor | ✅ Rewritten — `@supabase/supabase-js`, no Ktor |
| `06_prd_and_roadmap.md` | **Firebase** — 6 refs: "Firebase Auth", "Cloud Function", "Firestore listeners", "FCM" | ✅ All six replaced: Supabase Auth, Postgres triggers, Supabase Realtime, Expo Push + Web Push |
| `08_api_contract.md` | **Firebase** — 17 refs; entire doc was Firestore collections, Cloud Functions, Security Rules | ✅ **Fully rewritten** as a Supabase contract: RLS-gated table surface, trigger/`pg_cron`/Edge-Function split, Realtime topology |
| `11_test_strategy.md` | **Firebase** — 8 refs; Firebase Emulator Suite, `@firebase/rules-unit-testing`, "Firebase autoscaling" | ✅ **Fully rewritten**: local Supabase + **RLS policy tests as the non-negotiable layer** |
| `12_content_moderation_policy.md` | **Firebase** — Cloud Function trigger, "reading directly from Firestore" | ✅ Fixed — service-role Postgres read |
| `15_privacy_policy.md` | **Firebase** — 5 refs incl. "Firebase Analytics" and Firebase as a named processor | ✅ Fixed — PostHog for analytics; processors now Supabase, Cloudflare, Expo, PostHog, TVmaze |

The pack now says Supabase consistently. **The remaining gap is not a
contradiction but an absence: `09_database_schema.md` still does not exist**, so
there is RLS prose in `08` and RLS test cases in `11` with no policies between
them. That is the critical path.

Concrete collisions, as they were:

- `06` §1 acceptance criteria said *"Sign up via email or Google/Apple sign-in
  **(Firebase Auth)**"* while `03` said Supabase Auth. → Fixed; Apple sign-in
  dropped entirely (no native iOS build, no Apple account).
- `06` §7 said *"DMs: real-time via **Firestore listeners**"* while `03` said
  Supabase Realtime. → Fixed.
- `08`'s entire premise — *"Firebase/Firestore doesn't use a REST/OpenAPI
  contract the way a custom backend would"* — **was already false when written.**
  Supabase exposes an auto-generated REST surface via PostgREST plus Edge
  Functions with ordinary HTTP contracts. That document needed a rewrite, not a
  patch. → Rewritten.
- `15` named **Firebase (Google)** as a service provider and a data processor.
  → Fixed.
  Shipping that privacy policy against a Supabase backend would be a
  materially inaccurate disclosure to users and regulators. This one has legal
  teeth, not just technical ones.
- `11` calls Firestore Security Rules testing "**non-negotiable**" and names a
  Firebase-only toolchain. Under Supabase the equivalent is Row Level Security,
  and it's *also* non-negotiable — but the tooling is completely different
  (`pgTAP`, or integration tests against a local Supabase stack via the
  Supabase CLI). The test strategy as written is unexecutable.

**Recommendation: commit to Supabase and finish the migration.** Reasons it's
the right call for *this* project specifically:

- **Relational fit.** The core domain — dramas ↔ cast ↔ episodes ↔ discussion
  threads ↔ communities ↔ follows — is a graph of many-to-many relationships
  with strong integrity needs. That is Postgres's home turf and Firestore's
  worst case. The schema sketched in `03` is already fighting the doc model
  (see §3 below).
- **KMP support.** `supabase-kt` is a genuine Kotlin Multiplatform client
  (Android, JVM, Kotlin/Native, JS, **and wasm-JS from 3.0.0+**) covering
  `auth-kt`, `postgrest-kt`, `realtime-kt`, `storage-kt`, `functions-kt`.
  Firebase has **no official KMP SDK** — you'd be writing `expect`/`actual`
  shims around three platform-native SDKs, including a web target that has no
  Firebase-friendly story at all. For a KMP-first project this is close to
  disqualifying.
- **Query power.** Feed ranking, "next episode airing soon", search across five
  entity types, and spoiler-gated thread queries are all trivial SQL and all
  awkward-to-impossible in Firestore without precomputed index collections.
- **Exit path.** `03` says "swappable later for a custom Ktor server." That's
  far more true of Postgres + RLS than of Firestore Security Rules.

**Two honest caveats to record in `03` so nobody is surprised later:**

- `supabase-kt` is **community-maintained, not official** (Supabase's own docs
  say so), and effectively has one primary maintainer. That's a bus-factor risk
  for a product whose whole backend depends on it. Mitigation: it's a thin
  wrapper over PostgREST/Realtime/Auth HTTP APIs on Ktor, so worst case you can
  drop to raw Ktor calls per-endpoint. Keep your repository interfaces (already
  planned in `03`) as the seam so that swap is contained.
- **You lose Firestore's free offline persistence.** This is the single largest
  hidden cost of the migration and *no document mentions it*. Firestore gives
  you an offline cache and local writes for free. Supabase gives you neither —
  `SQLDelight` in `03` is listed as "offline-first support," but SQLDelight is
  just a local database; **the sync layer, outbox, and conflict resolution are
  yours to build.** For a social feed app a pragmatic MVP design is: cache last
  N feed pages read-only + a local write outbox for posts/likes/follows +
  reconcile-on-reconnect, and *no* offline DM composition. That must be a
  written decision, not an assumption baked into a table cell.

**Action items:**
- Rewrite `08_api_contract.md` as a real Supabase contract: PostgREST resource
  map, an **RLS policy matrix** (role × table × operation), Edge Function
  endpoints with request/response schemas, and the Realtime channel topology.
- Rewrite `11_test_strategy.md` §2 (Integration) around Supabase: RLS policy
  tests with `pgTAP` against a local Supabase stack, Edge Function tests, and
  Realtime reconnect/backfill tests. Delete the Firebase Emulator references.
- Update `06` §1, §7, §10, §11 and Phase 0 to name Supabase primitives.
- Update `15` §1, §3, §7, §8 to name Supabase as the processor, and fix §1's
  "Firebase Analytics" line to whatever `07_analytics_tracking_plan.md`
  actually chooses.
- Update `12` to say "Supabase Edge Function" / "Postgres".

---

## 2. Four referenced documents don't exist (P0)

Grep for cross-references finds these cited but absent:

| Missing file | Cited by | Consequence |
|---|---|---|
| **`09_database_schema.md`** | `14_terms_of_service.md` | **Most consequential gap in the pack.** No schema exists anywhere; `03` has only an 8-bullet "sketch." Everything in `06`, `08`, `11` and `12` silently depends on it. |
| **`13_data_retention_and_privacy_handling.md`** | `12`, `14` §8, `15` §1/§5/§7 (6 citations) | `15` §5 promises users a **30-day purge window** but nothing defines what's retained, what's purged, or how. `12` §4 promises removed content is retained "per `13_...`" — a policy that points at a document that doesn't exist. |
| **`07_analytics_tracking_plan.md`** | `06` ("Analytics note"), `11` ("what NOT to test") | `06` explicitly warns "instrumenting late is expensive to retrofit" and then references a plan that isn't written. Also: `02`'s success signals (episode-day WAU spikes, community join behavior, engagement ratio) are **unmeasurable without it.** |
| **`10_content_moderation_policy.md`** | `08` Cloud Functions table | Stale reference — the real file is `12_content_moderation_policy.md`. Renumber or fix the link. |

`09` should be written **before** the AI Studio prompt in `04` is run, because
`04` currently tells the build agent to paste in "the Data model sketch section
from `03`" — i.e. it will scaffold code from an 8-line sketch and invent the
rest. That invented schema then becomes expensive to change.

**New documents I'd also add**, in priority order:

- ~~`16_content_catalog_sourcing.md`~~ → **written, as `16_zero_budget_stack.md`
  §5.** Catalog sourcing is answered by TVmaze (free, no API key); TMDB is
  demoted to a later gap-filler. Still worth extracting into its own doc if the
  ingestion pipeline grows.
- ~~`17_video_pipeline_and_cost_model.md`~~ → **written, as
  `16_zero_budget_stack.md` §3.** R2 + client-side compression.
- **`18_security_and_abuse_controls.md`** — rate limiting, scraping, spam,
  account-creation abuse. `14` §5 *prohibits* scraping and automated access but
  nothing describes a technical control enforcing it. **Still outstanding**,
  and more urgent now: PostgREST exposes your tables as REST to any
  authenticated client, which makes scraping *easier* than it was on Firestore.
  Interim guidance in `16` §8.
- **`07_analytics_tracking_plan.md`** and **`13_data_retention_and_privacy_handling.md`**
  — both still outstanding; vendor/parameters now chosen in `16` §1–2 and §6.

---

## 3. The data model is Firestore-shaped and won't survive Postgres (P0)

`03`'s sketch carries NoSQL habits that are actively wrong for the target DB:

- **`User.followedDramaIds`, `User.followedActorIds`** — arrays of IDs on the
  user row. In Firestore this is a known anti-pattern (1 MB doc limit, no
  efficient reverse lookup). In Postgres it should be a **`follows` join table**.
  `08` already implies one exists (`follows/{id}`), so the two documents
  disagree with each other. One table, polymorphic or split:
  `follows(follower_id, target_type, target_id, created_at)` with a unique
  index — or separate `user_follows` / `drama_follows` / `actor_follows` tables
  for clean FK integrity. I'd take the split tables; the polymorphic version
  gives up foreign keys.
- **`Drama.castIds`, `Actor.dramaIds`** — same problem, plus you lose the cast
  *metadata* the Drama Hub Cast tab needs (role name, billing order, character
  photo). Needs `drama_cast(drama_id, actor_id, character_name, billing_order)`.
  Storing the relationship in both directions guarantees drift.
- **No `Episode` entity at all** — despite `06` §5 requiring an episode list
  with air dates, a per-episode discussion thread, and a "currently airing"
  next-episode countdown. **This is the entity the entire retention thesis in
  `01` is built on, and it isn't in the model.** Needs at minimum:
  `episodes(id, drama_id, number, title, air_date_timestamptz, runtime_min,
  status)` — and `air_date` must be **`timestamptz`**, because K-dramas air on
  KST and a global audience sees the countdown in their own zone. A naive
  timestamp here produces wrong countdowns for most of your users.
- **No `Comment` entity** — `08` has `posts/{postId}/comments/{commentId}`, and
  `06` §3 ranks by comment counts, but `03` doesn't model it.
- **No `Like`/`Reaction` entity** — yet `likeCount` is denormalized in `08` and
  "reactions" on channel broadcasts are a distinct product behavior in `02`.
- **No `Genre` entity** — yet `06` §2 requires "≥3 genres" at onboarding and
  §3 ranks by genre. Genres currently have no home.
- **No `ChannelSubscription` entity** — yet `08` gates broadcast *reads* on "an
  active subscription doc existing." The gate references a table that isn't
  modeled.
- **No `Report` entity** — `12` §2 specifies `reports/{reportId}` with content
  reference, reporter, reason, status. Absent from `03`.
- **No `moderationStatus` or `spoiler` flag on `Post`** — `12` §1 requires
  `moderationStatus: pending|visible|removed`, and `12`'s spoiler section plus
  `05` screen #38 ("tagging screen … spoiler toggle") require a spoiler flag.
  Neither is in the model.
- **No soft-delete / `deleted_at`** — `12` §4 explicitly says removal is "not
  hard-deleted," and `13` (missing) is supposed to define purge windows.
- **No conversation membership model** — `02` §7 and `06` §7 require *group*
  DMs; `03` has only `Message`. Needs `conversations` +
  `conversation_members(conversation_id, user_id, role, last_read_at)`.

**On denormalized counts:** `08` says *"No client ever writes a denormalized
count field directly — always via Cloud Function using the Admin SDK,"* and
`06` §11 justifies it as *"not client-side increment, to avoid race
conditions."* **That rationale does not transfer to Postgres.** In Postgres,
`UPDATE users SET follower_count = follower_count + 1 WHERE id = $1` is atomic
by definition; a row trigger on `follows` does it transactionally with no race
and no serverless function. Even better for a young product: **don't
denormalize at all** — `SELECT count(*)` over an indexed `follows` table is
fast well into millions of rows, and you delete an entire class of
count-drift bugs. Add the counter column later if measurement says you need it.
This is a place where migrating to Supabase makes the design *simpler*, and the
docs currently carry the more complex version forward for a reason that no
longer applies.

**On feed fan-out:** `08`'s `onPostCreate` "fan out to tagged drama/community
feed indexes" is write-fan-out — the pattern that breaks at scale and is
unnecessary here. On Postgres, do **read fan-out at MVP**:

```sql
select p.* from posts p
where p.author_id in (select followee_id from user_follows where follower_id = $me)
   or p.tagged_drama_id in (select drama_id from drama_follows where user_id = $me)
   or p.tagged_community_id in (select community_id from community_members where user_id = $me)
order by p.rank_score desc, p.id desc
limit 20;
```

That's one indexed query, no triggers, no fan-out table, no eventual-consistency
window — and it directly satisfies `06` §8's "appears within 5 seconds" because
there's nothing to propagate.

**Also unspecified: pagination strategy.** `06` §3 says "infinite scroll,
paginated (20 items/page)" and "pull-to-refresh re-ranks rather than just
re-fetching page 1." With a mutable ranking, OFFSET pagination produces
duplicates and skips. Specify **keyset pagination on `(rank_score, id)`**. This
is a one-line decision that prevents a class of "why did I see this post twice"
bugs.

---

## 4. The catalog data source is unnamed — and it's a licensing problem (P0)

`02` §5, `06` §5, `05` screens 21–25 and `01`'s whole retention argument depend
on: drama synopses, posters, cast lists with photos, episode lists, and **air
dates accurate enough to drive a countdown**. `08` says `dramas` and `actors`
are *"Seeded/managed by content team, not user-writable"* — which means someone
is hand-populating a catalog of thousands of titles with per-episode air dates
that shift constantly due to delays and hiatuses. **Nobody will do that by
hand, and no document says how it happens.**

The obvious answer is TMDB. And TMDB's API Terms of Use define **commercial
use** to include, verbatim:

> *"Using TMDB, the TMDB APIs, or TMDB Content on or in connection with a
> **'destination' website**, search engine, or interactive query-response
> system … **or for driving traffic or generating revenue for a website**"*
> and *"Operating a website that generates revenue through charging users for
> access to content, or **through recommend content, such as movies, television
> shows and music**, in connection with … Your use of TMDB"*

Hallyu is a destination app whose core feature is recommending TV content, and
`01` §4 states the explicit long-term plan is monetization via "ads, creator
tools, merch/ticketing." TMDB also *"reserves the right to determine whether
Your use is commercial"* in its sole discretion. Publicly reported commercial
pricing is roughly **US$149/month** under $1M revenue, custom above that.

Note the trap: `01` and `02` lean on "no monetization yet" as a reason to defer
decisions. **That framing does not protect you here** — "driving traffic" and
"destination website" are commercial triggers independent of revenue.

This is also the concrete answer to the IP worry flagged in `14` §7 and its
closing note, which correctly says posters/cast photos need rights clearance
but offers no path. There are three, and the pack should pick one:

1. **License TMDB commercially** (~$149/mo). Cheapest, fastest, gets you
   metadata + hosted images. Requires the mandated attribution string in your
   About/Credits screen: *"This [app] uses TMDB and the TMDB APIs but is not
   endorsed, certified, or otherwise approved by TMDB."* `05` screen #50
   ("About / legal") is the natural home — add it to that brief.
2. **License from a commercial metadata provider** (Gracenote, etc.) — far more
   expensive, cleaner rights.
3. **Build the catalog from user/community contribution** — avoids the metadata
   license but *not* the underlying poster/still copyright, and adds a
   moderation + data-quality burden you have no team for.

Whichever is chosen, `16_content_catalog_sourcing.md` must also specify: the
ingestion job, refresh cadence (air dates change weekly), how you handle a
delayed or hiatused episode without sending a wrong push notification, TMDB ID ↔
Hallyu ID mapping, and **what happens to episode-thread notifications when the
source date is wrong.** A false "Episode 7 is live!" push is a direct hit on the
retention hook that is your core thesis.

---

## 5. The named moderation vendor shuts down at the end of this year (P0, time-boxed)

`12` §1 says: *"Text: run through a moderation API (**Google Cloud's
Perspective API** or similar)."*

Google Jigsaw has announced that **Perspective API will no longer be in service
after 2026.** New usage requests stopped being accepted after February 2026,
and Jigsaw states it will not offer migration support. As of today
(2026-09-16) that's roughly **3.5 months of runway**, and your Phase 3
"moderation tooling hardened" milestone is scheduled for week 14+.

Do not name Perspective as the primary vendor. Pick a live one — options worth
evaluating: OpenAI's moderation endpoint, Hive, Webpurify, ActiveFence,
Amazon Comprehend/Rekognition, or Google Cloud Vision SafeSearch (still active
for image/video, which `12` also names). Whichever you choose, **put it behind
a `ContentModerationService` interface in the domain layer** so the vendor is a
config swap, not a refactor. That's exactly the seam `03`'s clean architecture
already provides — use it.

Two further moderation issues:

- **`12` §1 and `06` §8 directly conflict.** `12` says flagged content is set
  to `pending` and *not* immediately visible; `06` §8 requires tagged content
  to appear in Hub/Community feeds *within 5 seconds*. You must choose a
  posture: **publish-optimistically + async moderate + retroactively remove**
  (fast, standard for high-volume UGC, some exposure window), or **hold +
  review + publish** (safe, slow, and it breaks `06` §8 for every flagged
  item). My recommendation for this product: publish optimistically for
  text-only and low-risk content; pre-screen **only** media (image/video),
  where harm is concentrated and review latency is expected anyway. Write that
  decision down explicitly — right now the two documents each assert the
  opposite.
- **`12`'s non-negotiable baseline needs a technical path, not just a policy
  line.** "No tolerance for CSAM … immediate removal and reporting to relevant
  authorities" requires, in practice: hash-matching against a known-material
  database (NCMEC/PhotoDNA-class service, or IWF), a legally-reviewed reporting
  workflow, and an evidence-retention carve-out from `13`'s 30-day purge. None
  of that exists yet. This is the single highest-liability item in the pack and
  it's currently one bullet long.

---

## 6. Video is under-specified and is the likeliest way to blow the budget (P1)

`03` says *"Supabase Storage → migrate to Cloudflare Stream/Mux if video volume
grows"* and *"Keep MVP simple, avoid premature infra investment."* For a
TikTok-style autoplay feed, this is the one place where "keep it simple" is the
expensive choice, because **Supabase Storage is not a video platform**: no
transcoding, no adaptive bitrate (HLS/DASH), no generated renditions, no
per-timestamp thumbnails.

Consequences on the Explore feed (`02` §4, `06` §4: vertical swipe, autoplay on
focus):

- Without HLS + multiple renditions, every viewer pulls the full original file.
  On mobile networks that means buffering, which for a swipe feed means
  churn — the feature either feels instant or it fails.
- Supabase's current published rates: Pro is $25/mo with **250 GB origin egress
  + 250 GB cached egress** included, then **$0.09/GB origin, $0.03/GB cached**.
  Rough math on a 60-second clip: at ~15 MB per full view, **the included
  quota covers only ~16,000 video views/month.** A million views/month is
  ~15 TB → on cached egress ≈ **$440/mo**, on origin egress ≈ **$1,330/mo**,
  and that's before storage, image transforms, database, or auth. (Verify
  against current pricing before committing; these are 2026 published rates.)
  `01` §4 talks about the fandom graph as an asset; nobody has written down what
  serving it costs.
- Also unmodeled: `06` §4's 60-second upload cap needs **server-side
  enforcement**, not just a client-side indicator (`05` screen #37). Client
  caps get bypassed; a 10-minute upload to a bucket designed for 60-second
  clips changes your cost curve entirely.

**Recommendation:** budget a real video pipeline for MVP, not for "later."
Either Mux or Cloudflare Stream (upload → auto-transcode → HLS delivery → CDN →
playback metrics) removes transcoding, ABR, thumbnails, and most of the egress
risk for a per-minute-plus-delivery fee that is almost certainly cheaper than
DIY-at-scale. If you must stay on Supabase Storage for MVP, then at minimum add
an Edge Function/worker that runs `ffmpeg` on upload to produce HLS renditions
+ a poster frame, and serve through a CDN in front. Write
`17_video_pipeline_and_cost_model.md` with the actual numbers and a
"kill-switch" traffic threshold.

Related, and also unaddressed: **`05` screen #37 is a video trim UI.** Trimming
in Compose Multiplatform across Android + iOS + Wasm is a substantial
platform-interop project (Media3/Muxer on Android, AVFoundation on iOS, nothing
on web). Either scope it as a real work item or cut trimming from MVP and
enforce the cap at upload.

---

## 7. Platform risk is concentrated in Web, and the roadmap hides it (P1)

`03` targets Android + iOS + **Web (Compose Multiplatform, Wasm)**. Current
state of that stack: Compose Multiplatform for iOS went **stable** in 1.8.0
(May 2025), and KMP itself has been stable since 2023 — so the mobile bet is
sound and well-proven (Netflix, Cash App, McDonald's). **The Web target is
Beta, not stable**, and JetBrains itself notes browser adaptation of most
components is incomplete. Two specific risks:

- **WASM-GC browser support.** ~~Kotlin/Wasm depends on WASM-GC, which has
  historically been the blocker for Safari/WebKit~~ → **RESOLVED, and I was
  working from stale sources.** WasmGC shipped in Chrome 119, Firefox 120, and
  **Safari 18.2 (December 2024)** — cross-browser baseline since 11 Dec 2024.
  JetBrains confirms Kotlin/Wasm apps "can now run across all modern major
  browsers." With iOS 26 current, the iOS install base that can't run it is
  small. **This risk is closed** — and closing it is what makes the web version
  viable as the iOS delivery vehicle (see the revision note below and
  `16_zero_budget_stack.md` §4).
- **Video on Wasm.** `03` flags this correctly and proposes an HTML5 `<video>`
  interop fallback. That's the right instinct, but it's larger than a footnote:
  hosting a DOM video element beneath a Compose canvas requires `wasmJs`-only
  DOM interop, breaks the "UI written once" promise for your single most
  complex screen, and interacts badly with the swipe/autoplay gesture layer.
  Also note **SQLDelight on Wasm** runs via a WASM SQLite build — it works, but
  it's a second WASM artifact to ship and debug.

**Recommendation: cut Web from MVP.** Ship Android + iOS in Phase 1–2, and move
Web to Phase 3 as an explicitly separate bet with its own spike. This is not a
retreat from the KMP thesis — the shared `shared/` and `composeApp/` code is
still written once, and adding `wasmJsMain` later is cheap *once the Web target
matures.* What's not cheap is doing it in weeks 10–13 alongside short-form
video, DMs, channels, search, notifications, and moderation.

> **⚠ This recommendation is INVERTED by the zero-budget constraint — see
> `16_zero_budget_stack.md` §4.** Under a $0 budget the cut is **iOS**, not
> Web, for a reason unrelated to maturity: the free Apple ID tier does not
> grant the **push notification** or **Sign in with Apple** entitlements, so
> two features `06` §1 and §10 *require* cannot be developed at all without
> $99/year. Meanwhile Web hosting is genuinely free with unlimited bandwidth on
> Cloudflare Pages.
>
> The revised plan is **MVP = Android native + Web PWA**, and Web is not a
> secondary target — **it is how iOS users get the product.** The WASM-GC/Safari
> question is resolved (above); what remains genuinely unproven is narrower, so
> the week-1 spike should be narrower too:
>
> *"Can a canvas-rendered Compose/Wasm screen host a DOM `<video>` overlay that
> plays an R2-hosted MP4 on iOS Safari, inside a Home-Screen-installed
> standalone PWA?"*
>
> Test it in standalone mode specifically — installed-web-app behavior differs
> from a Safari tab on push, storage eviction, and video autoplay. If it fails,
> the fallback is **tap-to-play video on web** (a poster grid, one `<video>` at
> a time) rather than dropping Web, because Web still carries feed, drama hubs,
> episode threads, communities, search, DMs and push. Keep `iosMain` compiling
> and `iosTest` running in CI so native iOS stays a distribution switch, not a
> rewrite. Full capability matrix in `16_zero_budget_stack.md` §4.

Which leads to:

---

## 8. The roadmap is not credible at the stated scope (P1)

`06` proposes **13 weeks to closed beta** covering: new KMP project scaffold,
CI, a 55-screen design system implemented as tokens, auth + profile,
onboarding, ranked feed, per-episode discussion threads, communities, a
short-form video feed with upload/transcode/playback, 1:1 + group DMs,
broadcast channels, five-way search, push notifications, follow system, content
moderation with a human review queue, **and three platforms.**

Things not budgeted anywhere:

- **iOS CI and release ops.** Phase 0 (weeks 1–3) says "CI pipeline." macOS
  runners, certificates, provisioning profiles, and TestFlight distribution for
  a KMP framework are a real setup cost — typically days to a week before the
  first iOS build is even installable.
- **Apple Sign-In.** `06` §1 requires Google/Apple sign-in. If you offer any
  third-party login on iOS, App Store Review Guideline 4.8 effectively requires
  a privacy-preserving option, so Sign in with Apple is mandatory, not optional
  — and it needs an Apple Developer service ID, domain verification, and
  handling of Apple's private-relay email addresses.
- **App Store review latency** vs. `08`'s `schemaVersion` compat strategy. Your
  iOS users will be running older client versions for weeks after release. That
  makes backwards-compatible API/schema changes a hard requirement, not a
  nicety — and `08`'s Firestore-shaped "additive-only fields + a
  `schemaVersion` on documents" answer is weaker than what Postgres gives you
  for free: **SQLDelight `.sqm` migration files** plus a versioned API surface
  and Postgres views for compatibility shims. Rewrite that section.
- **The moderation team.** `12` §3 assumes 1–2 human reviewers at MVP with a
  24-hour SLA. That's a headcount and a tooling cost (an internal admin
  dashboard — which `12` calls "a simple internal tool," i.e. a 12th app
  surface nobody has scoped).
- **The missing catalog ingestion pipeline** (§4), which is a standing
  operational job, not a one-time seed.

**Recommendation:** re-cut the roadmap to **~20–24 weeks to closed beta on
Android + Web only** *(revised from "Android + iOS" — see the note in §7 and
`16_zero_budget_stack.md` §4)*, or hold 13 weeks and cut features. Add a hard
**~3-week Google Play closed-test lead time** that cannot be compressed and
cannot start until an installable build exists (§7.2 of `16_...`): personal Play
accounts created after 13 Nov 2023 need **12 testers opted in continuously for
14 days** before they may even *apply* for production access. If you cut, the
highest-value cut is **Channels** (a distinct data model, distinct moderation
surface, distinct notification path, and gated to a creator program that `08`
says doesn't exist in MVP anyway — *"Create: none in MVP (creator program
gated)"*). You are specifying, designing (`05` screens 31–32), and testing a
feature that literally cannot be created by any user in the MVP. Move it to
Phase 3 wholesale. Second-best cut: group DMs (keep 1:1) — which the zero-budget
constraint independently forces, via Realtime's 200-concurrent-connection free
cap.

One more roadmap note: `06` Phase 1 promises "Android + iOS parity" by week 9
but Phase 0 only lands "Auth + profile working end-to-end **on Android**."
Originally I recommended moving *"first iOS build runs and is installable via
TestFlight"* into Phase 0, on the principle that iOS integration pain is much
cheaper to discover in week 3 than in week 6.

> **⚠ Revised — that deliverable is impossible at $0** (`16_zero_budget_stack.md`
> §4: free Apple ID = no TestFlight, no external testers, 7-day build expiry,
> and no push / Sign in with Apple entitlements). The underlying principle
> still holds, so substitute the free version of it: **"iOS target compiles and
> `iosTest` passes in CI"** as an explicit Phase 0 deliverable. On a public
> GitHub repo that costs nothing — GitHub Actions is unmetered on public repos
> *including macOS runners*, whereas a private repo's 2,000 free minutes become
> only ~200 macOS minutes at the 10x multiplier. That catches KMP/iOS linkage
> pain in week 3 without spending $99, and it's the single highest-value free
> decision in the whole plan. Phase 1's "Android + iOS parity" should become
> "Android + Web parity; iOS compiles and is tested but not distributed."

---

## 9. Smaller but real gaps

**Korean-language search is unsolved.** `06` §9 wants tabbed search over
Dramas/Actors/Users/Communities/Posts with 300 ms debounce. For a K-drama app,
users will search in Hangul (*"사랑의 불시착"*), in official English titles
(*"Crash Landing on You"*), and in fan abbreviations (*"CLOY"*). Postgres full-text
search has **no built-in Korean configuration** — no Korean stemmer ships with
it. Plan for: an `aliases`/`akas` table holding native title + romanization +
common abbreviations per drama and actor; `pg_trgm` GIN indexes for fuzzy
Latin matching; and **verify whether a CJK-capable extension (`pg_bigm`,
`pgroonga`) is available on Supabase** before designing around it. If it isn't,
alias-table matching plus trigram is your MVP and that's fine — but it must be
decided, because "search" as currently written will silently fail on the core
subject matter of the product. Relatedly: `02` defers localization past MVP,
which is reasonable **for UI strings** but must not be allowed to mean the
*schema* lacks Korean-language fields.

**Channel visibility contradicts itself.** `08` says broadcast content is
readable by *"Subscribers only … Enforced via Security Rules checking
subscription doc."* But `02` §7 and `05` screens 31–32 describe channels on the
WhatsApp Channels model, and WhatsApp Channels are **publicly discoverable and
previewable** — that's the point. If only subscribers can read, channels can't
be discovered, and `05` screen 31 ("Channels list") has nothing to list for a
new user. Decide: public-readable channel metadata + previews, subscriber-only
for the full archive — or genuinely private channels, in which case the
discovery UX in `05` needs redesigning.

**Push notifications have no owner after the migration.** `06` §10 says FCM.
Supabase doesn't do push. You now own: FCM HTTP v1 integration (an Edge
Function can do it), APNs registration *through* FCM for iOS, **device-token
storage and lifecycle** (invalidation, reinstall, multi-device — nobody models
a `push_tokens` table), and **Web Push with VAPID keys** if Web survives §7.
Also worth stating: FCM delivery is unreliable or blocked in some markets, and
your audience is explicitly global (`01`). `06` §10's "new episode alerts for
followed dramas" is your core retention mechanic riding on that delivery path —
it deserves a reliability note and an in-app fallback.

**Age gating isn't wired to signup.** `14` §2 sets a minimum age (13/16, TBD)
and `15` §6 says the service isn't directed at children. But `06` §1's signup
criteria collect **no date of birth**, and `05`'s auth screens (#4–#10) have no
age gate. Given `01` says the audience "skews Gen Z / younger Millennial," this
is real COPPA/GDPR-K exposure and it's a Phase 0 fix — you cannot retrofit an
age gate onto an existing user base without re-consenting everyone.

**DM scanning isn't disclosed.** `12`'s closing section says harassment policies
"apply equally to DMs/channels, not just public posts." But `15` §1 lists
messages merely as collected content and §2 never says private messages are
automatically scanned. Scanning private communications is a disclosure
obligation in several jurisdictions (and touches ePrivacy/DSA territory in the
EU). `15` §2 needs an explicit line, and `12` needs to state the actual
mechanism (hash-matching and report-triggered review vs. blanket scanning —
they have very different privacy profiles).

**No abuse/rate-limiting controls.** `14` §5 prohibits spam, scraping, and
automated access. PostgREST exposes your tables as a REST API to any
authenticated client, which makes scraping *easier* than Firestore, not harder.
Supabase gives you some protection but you should specify: per-IP and per-user
rate limits at the edge, row-count caps on unbounded selects, and
`anon`-key vs. `authenticated`-key exposure review. Also verify the `anon` key
never ships in the Wasm bundle with broader access than intended — a client-side
key plus a permissive RLS policy is the classic Supabase breach pattern.

**`11`'s "What NOT to test" needs one edit.** *"Firebase autoscaling handles
MVP-stage load"* is no longer true and also wasn't quite true — Supabase on the
Pro plan is a **single Postgres instance with a fixed compute tier**, not
autoscaling. Load capacity is now something you actively manage (connection
pooling via Supavisor, read replicas later). That flips `11`'s advice: you
*do* need a basic load test on the feed query and the RLS policies before
closed beta, because RLS predicates evaluated per-row across a join can turn a
fast query slow in a way that never shows up in unit tests.

---

## 10. Recommended actions, prioritized

**P0 — before any code is generated (fix these first, they change everything downstream)**
1. Decide Supabase definitively; rewrite `08` and `11`; patch `06`, `12`, `15`.
2. Write `09_database_schema.md` — real Postgres DDL with joins, `Episode`,
   `Comment`, `Like`, `Genre`, `Follow` (tables not arrays), `ChannelSubscription`,
   `Report`, `PushToken`, `moderationStatus`, `spoiler`, `deleted_at`, RLS
   policies, and SQLDelight migration strategy.
3. Write `13_data_retention_and_privacy_handling.md` — `15` already promises
   users a 30-day purge it can't currently honor.
4. Resolve the catalog data source and **get a license position in
   writing.** This can block launch and has legal lead time. → *Revised: now
   covered by `16_zero_budget_stack.md` §5 — TVmaze (free, no API key,
   CC BY-SA + attribution) instead of TMDB (~$149/mo commercial). Confirm
   TVmaze's commercial terms in writing and have counsel look at the
   share-alike clause.*
5. Replace Perspective API in `12` (dead after 2026) and resolve the
   publish-vs-hold contradiction with `06` §8.
6. Fix `08`'s stale `10_content_moderation_policy.md` reference → `12_...`.

**P1 — before Phase 0 starts**
7. Re-cut platform scope and the roadmap. → *Revised twice. Final position:
   **cut native iOS, not Web — and Web is promoted to a primary delivery
   channel, because it's how iOS users get the product.** WasmGC shipped in
   Safari 18.2 (Dec 2024), so the browser-support blocker is closed; iOS users
   also get full-screen home-screen install by default since iOS 26 and **$0 Web
   Push via VAPID with no Apple account**. The residual spike is the narrow
   DOM-`<video>`-in-canvas question on iOS Safari. **Video posting is restricted
   to Android** (no viable browser-side transcode) — Android is the creator
   client, Web is the reader/discusser client. See `16_zero_budget_stack.md` §4.
   Roadmap still needs re-cutting to ~20–24 weeks, plus a hard ~3-week Google
   Play closed-test lead time.*
8. Move Channels out of MVP (it's already un-creatable per `08`); reconsider
   group DMs. → *Both confirmed as cuts in `16` §11: group DMs are forced out by
   Realtime's 200-concurrent-connection free cap.*
9. Write the video pipeline + cost model; pick the transcode path now, not at
   "if volume grows." → *Done: `16_zero_budget_stack.md` §3. Answer is
   **Cloudflare R2 ($0 uncapped egress) + client-side compression to a single
   720p rendition**, with an HLS ladder deferred until buffering shows up in
   playback metrics. Supabase Storage is dropped entirely — its 1 GB free cap
   is ~100 videos.*
10. Write `07_analytics_tracking_plan.md` — `02`'s success signals are
    currently unmeasurable. → *Vendor chosen: PostHog free (1M events/mo, plus
    free feature flags). Still needs writing; must include playback buffering
    metrics, which are the trigger for the HLS upgrade.*
11. Add DOB/age gate to `06` §1 and `05` screens #4–#10.
12. Decide the Korean/Hangul search strategy and add `aliases` to the schema.
    → *Partly solved for free: TVmaze exposes alternate titles (AKAs), which
    populate the `aliases` table. See `16` §5.*
13. Specify the offline/sync strategy that Firestore used to give you for free.
14. ~~Add "first iOS build installable via TestFlight" to Phase 0.~~ **Void** —
    impossible at $0 (`16` §4). Replace with: public-repo CI decision
    (unlimited free macOS Actions minutes), Supabase keep-alive cron,
    `pg_dump`→R2 backup cron, custom SMTP via Brevo, and recruiting 12 Play
    closed testers. All five are Phase 0 and all are free. See `16` §7–§8.

**P2 — before closed beta**
15. Write `18_security_and_abuse_controls.md`: rate limits, RLS exposure review,
    server-side 60 s video cap enforcement, `anon`-key audit.
16. Add the TMDB attribution string to `05` screen #50's brief.
17. Add DM-scanning disclosure to `15` §2; align `12`'s mechanism to it.
18. Add a basic load test on the feed query + RLS to `11` (reversing its
    current "don't test load" advice).
19. Specify keyset pagination on `(rank_score, id)` in `06` §3.
20. Define the CSAM hash-matching + legal reporting workflow behind `12`'s
    non-negotiable baseline.

---

## What I'd leave alone

- The **product thesis** in `01` and the persona/IA structure in `02`. The
  episode-cadence retention hook and the drama-hub-centric information
  architecture are the right shape.
- **KMP + Compose Multiplatform for Android and iOS.** Stable, proven, and the
  correct cost lever for an unfunded team. Only the Web target is premature.
  → *Revised under zero budget: the **Web** target stays (hosting is genuinely
  $0 with unlimited bandwidth on Cloudflare Pages) and **iOS** is what gets
  deferred, for an entitlement reason rather than a maturity one. Crucially the
  KMP thesis still holds — iOS becomes a distribution switch you flip later,
  not a rewrite, as long as `iosMain` stays compiling and tested in CI from
  day one. See `16_zero_budget_stack.md` §4.*
- **The library choices in `03`**: Ktor, kotlinx.serialization, Koin, Coil 3,
  SQLDelight, Coroutines/Flow, Voyager. All multiplatform-appropriate. (Minor
  note: `supabase-kt` requires Ktor 3.x — pin the version catalog accordingly,
  and remember min Android SDK 26 or enable core library desugaring.)
- **`05`'s design-system brief.** Genuinely excellent. Its only gaps are the
  TMDB attribution placement and the DOB/age-gate screen.
- **`12`'s spoiler design.** Treating episode threads as implicitly
  spoiler-permissive while the Home Feed is not is exactly right, and it's a
  fandom-specific insight generic moderation policies don't have. Keep it —
  just make sure `Post` actually has the flag (§3).
- **The legal drafts' posture.** `14` and `15` are appropriately humble about
  needing counsel, and `14`'s closing IP note is the most important sentence in
  the pack. They just need `13` to exist and the Firebase references removed.
