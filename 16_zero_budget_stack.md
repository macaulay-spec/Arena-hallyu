# Hallyu — Zero-Budget Stack & Cost Model

*Researched 2026-09-16. All free-tier limits verified against current published
pricing; items marked **[verify]** could not be confirmed from a primary source
and should be checked before you build on them.*

This document absorbs and replaces the proposed `17_video_pipeline_and_cost_model.md`
from `00_review_and_recommendations.md`, and answers the catalog-sourcing
question that `16_content_catalog_sourcing.md` was going to cover (§5 below).

---

## 0. Bottom line

**You can build and launch Hallyu for $0/month, and it will work well — up to
roughly 1,000–2,000 daily active users.** That is not a hedge; it's the actual
ceiling I can compute from published free-tier limits, and it is *exactly* the
range in which you're still validating the thesis in `01_vision_and_case.md`.
Free tiers are sized for validation, not for scale.

The right framing is therefore not "free forever." It's:

> **Zero budget means free until you have evidence. Architect so that every
> paid upgrade is a config change, not a rewrite.**

`03_technical_architecture_expo.md` already has the seam that makes this true —
per-feature `api.ts` modules with TanStack Query hooks in front of them, so no
screen ever touches Supabase directly. Hold that line religiously and the
$0→$25→$99 ladder in §9 becomes a series of small diffs.

> **Stack note (2026-09-17).** This document was drafted against a Kotlin
> Multiplatform plan and has been revised for **React Native + Expo**, which is
> what Hallyu is actually being built on. Several risks the KMP plan carried
> simply don't exist here, and one section (§4's video-on-web question) went from
> "highest-risk unknown" to "solved by the framework." Those are called out
> inline rather than silently deleted, because the reasoning still matters.

**Two things genuinely cannot be free:**

1. **A *native* iOS app.** Not "hard" — impossible at $0, because the free
   Apple ID tier withholds the push and Sign in with Apple entitlements. But
   this is a much smaller problem than it first looks: **iOS users are served by
   the web version, which gets full-screen home-screen install on iOS 26 and
   $0 Web Push with no Apple account at all.** See §4 — this is the load-bearing
   section of the whole plan.
2. **A domain name**, ~$10/year, if you want uncapped video delivery (§3).
   There is a $0 workaround with a lower ceiling.

Everything else on this list is $0.

---

## 1. The zero-budget stack

| Layer | Choice | Free allowance | Binding constraint | Mitigation |
|---|---|---|---|---|
| Database + Auth + Realtime + Edge Functions | **Supabase Free** | 500 MB DB, 5 GB API egress + 5 GB cached, 50k MAU, 500k Edge Function invocations, 2 projects | **5 GB API egress/month** ≈ 1.25M responses at 4 KB → ~41k requests/day | `?select=` explicit columns only (never `select=*`); `expo-sqlite` client cache; gzip; keyset pagination |
| Client SDK | **`@supabase/supabase-js`** — Supabase's own reference client, with a documented React Native setup | $0, MIT | None material. **The KMP-era `supabase-kt` single-maintainer risk is gone** | `react-native-url-polyfill/auto` + SecureStore session + one AppState auto-refresh listener, wired exactly once (`03_...expo.md`) |
| App framework | **Expo SDK 57** (`expo-router`, `expo-video`, `expo-image-picker`, `expo-updates`) | Expo SDK/CLI **free forever**; local builds free | New Architecture mandatory since SDK 55 — no Old Arch fallback | Start on SDK 57; `npx expo install` for every package, never hand-pinned versions |
| **All user media** (avatars, post images, video) | **Cloudflare R2** | **10 GB storage, 1M write ops, 10M read ops/month, and $0 egress forever — uncapped** | 10 GB stored; **R2 activation may require a card on file [verify]** | Custom domain for edge caching (§3); if card is a blocker, Worker-on-`workers.dev` path |
| Video encoding | **Client-side, on device — via `expo-image-picker`'s `videoQuality` option** | $0 | No adaptive-bitrate ladder | Re-encode to a single 720p H.264 rendition at pick time (§3); this also **kills the iPhone HEVC incompatibility** for free. Add a server HLS ladder later |
| Web app hosting | **Cloudflare Pages** | 500 builds/month, **unlimited static requests and bandwidth** | Pages *Functions* draw on Workers' 100k req/day | Export with **Expo Router static rendering** — build-time HTML, not a client-only bundle. That's what makes drama/episode routes crawlable (§4) |
| Edge glue (signed URLs, webhooks, catalog refresh) | **Cloudflare Workers Free** | 100k requests/day, 10 ms CPU/request, 50 subrequests, **5 cron triggers** | 10 ms CPU rules out any transcoding; 5 crons is exactly enough (§6) | Workers orchestrate only; heavy work goes to Supabase Edge Functions |
| Push notifications | **FCM** (Android) + **Web Push / VAPID** (web, **incl. iOS**) | $0, unlimited — **no Apple account needed** | iOS web push requires a **Home Screen install** + `display: standalone` manifest; Safari-only install step | See §4's requirements table; make `05` screen #14 an "Add to Home Screen" explainer on iOS |
| Transactional email | **Brevo** via Supabase custom SMTP | 300/day (~9k/month), no card | Supabase caps custom-SMTP sends at **30/hour** | Raise the cap in Auth → Rate Limits (free); batch non-urgent mail |
| Analytics | **PostHog Free** | 1M events/month, 5k session recordings, **1M feature-flag requests**, 1-yr retention, unlimited seats, no card | 1 project | Do **not** put events in Postgres — 1M events ≈ 200 MB of your 500 MB (§2) |
| CI/CD | **GitHub Actions on a public repo** | **Unlimited minutes, including macOS runners** | Only *standard* 2-core runners; larger runners always bill | Make the repo public — this is worth ~$60+/month of macOS CI (§7) |
| Text + image moderation | **OpenAI Moderation API** (`omni-moderation-latest`) | **$0, no per-request charge**, 100+ languages, text *and* image | Image covers only sexual/violence/self-harm; no video | Frame-sample videos client-side (`expo-video` thumbnails) → image moderation (§6) |
| CSAM detection | **Cloudflare CSAM Scanning Tool**, **Meta PDQ/TMK** (open source), **Microsoft PhotoDNA** (free for qualified orgs), Thorn's free hashing toolkit | $0 | Application/vetting lead time | Start the application in week 1 (§6) |
| Drama/episode catalog | **TVmaze public API** | **Free, no API key, no stated call limit**, CC BY-SA 4.0 + attribution | ~20 calls/10s per IP; SA clause needs a legal look | Nightly cron + local Postgres cache (§5) |
| Design | **Stitch** (per `05_stitch_uiux_prompt.md`) | $0 | — | Already planned |
| DB backups | **GitHub Actions cron → `pg_dump` → gzip → R2** | $0 | Free tier has **no automatic backups at all** | 30-day rolling window in R2 (§8) |
| Android distribution | **Google Play** | **$25 one-time, no renewal** | Personal accounts created after 13 Nov 2023: **closed test with 12 testers opted in for 14 continuous days** before production access | Start the closed test the day the first build is installable (§7) |
| Legal review | — | **Not free** | Real cost, real lead time | See §10 |

---

## 2. Three constraints that decide the design

Everything else in this document follows from these three numbers.

### 2.1 Supabase's 1 GB file storage makes R2 non-negotiable

The free tier gives you **1 GB of file storage**. At a **5-minute** cap
re-encoded to 720p H.264 (~45 MB/clip, §3) that is **about 20 videos, total,
forever.** You cannot run a short-form-video product on it. Avatars and post
images would consume it alone.

So: **video never touches Supabase Storage.** All video goes to R2. Supabase
Storage stays fine for **avatars and post images** (small, and it keeps their
access control inside the same RLS system as everything else) — but its 1 GB is
a budget to watch, not a free-for-all.

Supabase becomes database + auth + realtime + edge functions + small images, and
its 5 GB egress quota then carries mostly JSON — which is what makes 5 GB
survivable.

This also removes the upload-size problem: Supabase's free tier caps uploads at
**50 MB**, which a 5-minute clip can exceed. R2 has no such cap and multipart
upload handles large files.

> **⚠️ The 5-minute cap changed the binding constraint, and it's worth reading
> twice.** At 60 seconds the scarce resource was *bandwidth*. At 5 minutes it is
> *storage*:
>
> | | 60 s clip (~10 MB) | **5 min clip (~45 MB)** |
> |---|---|---|
> | R2 free storage (10 GB) | ~1,000 videos | **~220 videos** |
> | R2 free Class A ops (1M writes) | not binding | not binding |
> | Egress | $0 uncapped | **$0 uncapped — unchanged** |
>
> Bandwidth stays free forever, which is the important half. But ~220 videos is
> a real ceiling for a *content* product. Mitigations, in order of preference:
>
> 1. **Compress harder at pick time.** `videoQuality` on `expo-image-picker` at
>    720p/~1 Mbps gets ~45 MB; at 480p you're near ~20 MB → **~500 videos**.
>    For talking-head drama-reaction clips, 480p is genuinely watchable.
> 2. **Accept ~220 as the seed-content budget.** An MVP that hasn't launched yet
>    does not need more than 220 videos. This may be a non-problem in practice.
> 3. **R2 storage beyond free is $0.015/GB-month** — 100 GB is **$1.50/month**,
>    and egress stays $0 forever. That is the cheapest possible overflow valve,
>    and it breaks the strict $0 rule by an amount that is easy to defend.
> 4. **Hard-delete on retention policy** — `13_data_retention_and_privacy_handling.md`
>    (unwritten) should specify what happens to videos from deleted accounts.
>    That reclaims storage and is required for GDPR anyway.
>
> What you should **not** do is quietly revert to a 60-second cap to dodge this.
> The cap is a product decision, and 5 minutes was chosen deliberately.

### 2.2 500 MB of Postgres is a real budget — spend it deliberately

Rough planning figures (verify against your actual row widths):

| Table | Row size | Rows in 500 MB (with indexes) |
|---|---|---|
| `posts` (500-char caption) | ~800 B | ~400k |
| `comments` | ~400 B | ~800k |
| `follows` | ~60 B | ~5M |
| `episodes` (catalog) | ~300 B | ~1M |
| `analytics_events` | ~200 B | **~1.5M ← eats the entire database** |

Two conclusions:

- **Analytics must not live in Postgres.** 1M events would consume ~40% of your
  total database. PostHog's free 1M events/month is the answer, and it also
  gives you free feature flags — use those to gate the risky surfaces (video,
  web) rather than shipping them to everyone.
- The catalog (`dramas`, `actors`, `episodes`) is small and cheap. A complete
  K-drama catalog with per-episode rows is on the order of tens of thousands of
  rows — single-digit megabytes. **The catalog is not your DB problem; your
  problem would be event and notification tables.** Add a retention job for
  `notifications` (they're transient) and roll up or truncate aggressively.

Also note: free-tier **log retention is short** (published figures conflict
between 1 and 7 days — **[verify]**), so don't rely on logs for forensics. Ship
what you need to retain into your own table or to PostHog.

### 2.3 Realtime's 200 concurrent connections caps your chat design

Supabase free gives **200 peak concurrent Realtime connections, 100
messages/second, 2M messages/month.** Published guidance is explicit that
leaked channels — subscriptions created on navigation and never disposed — are
the number-one way teams hit this wall, and that once you do, **new
subscriptions fail silently.**

Design rules that follow:

- **Never hold a Realtime connection open globally.** Open one only while a DM
  thread or channel detail screen is actually on screen, and dispose it in a
  `useEffect` cleanup / on navigation away. Make this a lint-level rule in
  `src/features/messages`.
- **The conversation list polls** (e.g. every 30 s, or on foreground). It does
  not subscribe.
- **Feed, Explore, Drama Hub, Communities: no Realtime at all.** They're
  pull-to-refresh + client cache. `06` §8's "appears within 5 seconds" is
  satisfied by read-fan-out (see `00_review_and_recommendations.md` §3), not by
  a socket.
- At 200 concurrent connections and, say, 8% of active users sitting in a DM
  thread at any moment, this supports roughly **2,500 concurrent users** — well
  above your egress ceiling, so egress breaks first. Fine.
- **Budget for the reconnect path.** Supabase Realtime does not guarantee
  delivery; on reconnect the client must backfill from Postgres by
  `(conversation_id, created_at > last_seen)`. `11_test_strategy.md` already
  asks for "real-time listener behavior under flaky network conditions" — make
  that a named test, because it's the one that fails in production.

---

## 3. Video at $0 — the finding that changes the economics

This is the single most important section. My earlier review said video was the
likeliest way to blow the budget. On the *default* path (Supabase Storage) it
certainly was — and moving the cap from 60 s to **5 minutes** made it worse by
~4.5x:

| | 1M views of **5-minute** clips (~45 TB delivered) |
|---|---|
| Supabase Storage, cached egress | ~$1,350/mo **and** you'd have blown the 1 GB storage cap ~45x over |
| Supabase Storage, origin egress | ~$4,050/mo |
| Cloudflare Stream ($1/1,000 min delivered → 5M minutes) | **~$5,000/mo** |
| Bunny Stream (~$0.005–0.01/GB) | ~$225–450/mo |
| **R2 + Cloudflare CDN** | **~$0–40/mo** (storage + ops only; egress is $0) |

*Order-of-magnitude figures, scaled from the 60 s model in §2.1 — treat as
"which option is free," not as a quote.*

**R2 charges $0 for egress. Always. Uncapped. No ratio limit, no fair-use cap,
on every storage class.** It bills only storage ($0.015/GB-month beyond the free
10 GB) and operations (Class A writes $4.50/M, Class B reads $0.36/M beyond
1M/10M free). And Cloudflare's terms **explicitly permit serving video through
their CDN when the content is hosted in R2** — that permission is the whole
point of the 2023 §2.8 rewrite, and Cloudflare staff have confirmed R2 counts as
the qualifying Developer Platform service.

### The MVP pipeline, at $0

```
capture (device)
  → expo-image-picker, `videoQuality` set → 720×1280 H.264, ~1.2 Mbps, ≤5 min  (≈45 MB)
  → duration read; reject anything over the cap
  → expo-video native thumbnail generation → poster frame                      (≈50 KB WebP)
  → upload both to R2 (presigned URL from a Worker)
  → row in Postgres: r2_key, duration_s, poster_key, moderation_status
  → serve via R2 custom domain, edge-cached, $0 egress
  → play with expo-video (progressive MP4, byte-range)
```

**No transcoding. No ffmpeg. No HLS ladder. No server-side video processing at
all.** The user was right that this was being over-complicated — under Expo the
entire pipeline is: pick, check duration, upload, play.

**Why compress on the client instead of transcoding on a server:** it removes an
entire machine you'd have to provision, monitor, and keep alive; it's free at
any volume; and it side-steps every upload-size cap. And under Expo it is a
**one-line config option**, not a platform-specific subsystem: `videoQuality` on
`expo-image-picker` re-encodes during pick, identically on Android and iOS. The
KMP plan needed `Media3 Transformer` in `androidMain` and
`AVAssetExportSession` in `iosMain` — two separate implementations to write and
maintain. That work is now a prop.

**This also fixes the HEVC problem for free**, which is worth stating because it
is otherwise a silent failure: iPhones record H.265/HEVC by default. Safari
plays it, but **Chrome and Edge on Windows need a hardware decoder plus a paid
Microsoft extension, and Firefox desktop had it disabled through v136** (partial
from v137). So raw iPhone clips would play on your Mac and refuse to play for a
chunk of your web audience, with no error — just a black rectangle. Setting
`videoQuality` re-encodes to H.264, which plays everywhere. Poster frames come
from `expo-video`'s native thumbnail generation, so no server work there either.

The honest tradeoff: **one rendition, no adaptive bitrate.** At 720p/~1.2 Mbps
that plays fine on 4G and most Wi-Fi, and it's a phone-screen vertical feed —
but users on congested networks will buffer, and a 5-minute clip is 5x more
buffering exposure than a 60-second one. That is the one real quality concession
the zero-budget plan asks for. The upgrade path is clean: add an ffmpeg worker
later that produces an HLS ladder, keep the same R2 bucket and the same URLs, and
switch the player to HLS. `expo-video` supports HLS; nothing about the client
changes except the source URL.

### What it actually costs at volume

| Scale | R2 storage | R2 read ops | Egress | **Total** |
|---|---|---|---|---|
| 1,000 clips, 100k views/mo | free (10 GB) | free | $0 | **$0** |
| 10k clips, 1M views/mo | ~$1.35 | ~$0 (≈10M reads ≈ free cap) | $0 | **~$1.35** |
| 100k clips, 10M views/mo | ~$15 | ~$32 | $0 | **~$47** |

Compare that against `01_vision_and_case.md`'s premise that the fandom graph is
the long-term asset. **Video delivery is effectively free at this scale.** That
is a genuinely unusual position for a short-form-video product and it is the
strongest argument for the zero-budget plan being viable rather than merely
cheap.

### Two caveats

- **`r2.dev` is not the production path.** The auto-assigned development
  hostname is explicitly rate-limited and **not edge-cached**. You must attach a
  **custom domain**, which requires a domain in your Cloudflare zone — hence the
  ~$10/year in §0. Without it you pay origin reads on every single view and hit
  rate limits.
- **$0-domain fallback:** serve R2 objects through a **Worker on a free
  `*.workers.dev` subdomain** with `Cache-Control` headers so the edge caches
  the response. Content is still in R2, so it's the same permitted pattern, and
  it costs nothing. The cap is Workers' **100k requests/day**; with ~10
  byte-range requests per video view that's roughly **10k video views/day**
  before you need the domain. Perfectly adequate for a closed beta, and it
  means you can start at literally $0 and add the domain when traffic justifies
  it.
- **[verify]** whether R2 activation requires a payment method on file even
  though free-tier usage bills $0. If it does and that's a hard blocker, the
  fallbacks are all *worse* (Bunny Stream ~$50–100/mo at 1M views; Cloudflare
  Stream ~$1,000/mo), so it's worth checking early — this decision is worth
  more than every other line in this document combined.

### Options I'd reject, and why

- **YouTube/Vimeo as the video host.** $0 and tempting. Rejected: it puts your
  core content on a competitor's platform under terms that restrict embedding
  in third-party players and downloading, it inserts their ads and their
  recommendations into your feed, and you cannot moderate, trim, or control
  playback latency. For a product whose entire thesis is *owning the fandom
  home*, this is self-defeating.
- **Server-side ffmpeg on Oracle Cloud Always Free.** Genuinely $0 (2 OCPU /
  12 GB ARM after the mid-2026 halving, 200 GB block storage, 10 TB/month
  egress) and it would give you a real HLS ladder. Rejected *for MVP* because it
  adds a machine to operate, free-tier ARM capacity is frequently "out of
  capacity" in popular regions, and idle instances get reclaimed. This is the
  right Phase-2 upgrade once you need ABR — not a week-1 dependency.
- **AWS Rekognition for moderation.** Note: it **closed video and batch image
  moderation to new customers on 30 April 2026.** Don't plan around it.

---

## 4. iOS at $0: the web version *is* the iOS version

**This section was revised.** My first pass concluded "defer iOS, ship Android +
Web" and treated Web as a risky secondary target. That was too pessimistic, and
the correction matters: **Web isn't the fallback for iOS users — it's the
delivery vehicle for them, deliberately, and it works.** Two facts I'd
under-weighted:

**1. The web target is boring now — and that's the point.** The KMP plan rendered
to WebAssembly through Skia into a `<canvas>`, which meant browser-compat
roulette (WasmGC shipped in Safari 18.2, Dec 2024 — fine, but it was a real
question for a long time), a canvas that no crawler could read, and video that
couldn't live inside the render surface. **Expo web is `react-native-web`:
ordinary JavaScript rendering to ordinary DOM elements.** No Wasm, no canvas, no
new browser-compiler dependency. Whatever the Safari version, this is the same
class of thing that already powers X/Twitter's web client. The entire category of
"will the web runtime work" risk is deleted rather than mitigated.

**2. iOS users get push notifications for $0, with no Apple account at all.**
This is the big one, because `06` §10's "new episode alerts for followed
dramas" is the entire retention thesis in `01` §2.

- Web Push works on iOS since **16.4**, via **VAPID keys** — *"you don't need
  to register at apple.com"*; the subscription endpoint is simply
  `https://web.push.apple.com/...`.
- Since **Safari 18.4 (March 2025)** there's also **Declarative Web Push**,
  which works *without a service worker* and has a built-in fallback if the JS
  step fails — simpler and more reliable.
- Since **iOS 26 (autumn 2025)**, *every* website added to the Home Screen opens
  as a **full-screen web app by default, even without a manifest.** Apple now
  describes **zero requirements for installability in Safari.** The historical
  "PWAs feel second-class on iOS" objection has largely evaporated.

### The hard requirements for iOS push (get these wrong and it silently fails)

| Requirement | Detail |
|---|---|
| **Must be a Home Screen web app** | Push does **not** work in a Safari tab. `pushManager` is `undefined` until installed. |
| **`manifest.json` with `display: standalone`** | Apple calls this a "Home Screen web app." **Without it, Web Push is not enabled at all.** |
| **Safari only** | Chrome/Firefox on iOS are WebKit underneath and **cannot** do the install step. Your in-app prompt must say "Open in Safari." |
| **HTTPS + service worker** | (Declarative Web Push relaxes the SW requirement.) |
| **Permission from a click handler** | iOS Safari silently ignores `Notification.requestPermission()` not called directly from a user gesture. |
| **Detect before prompting** | `display-mode: standalone` media query — if iOS and *not* standalone, show an "Add to Home Screen" explainer instead of requesting permission. |
| **No silent push / no background execution** | Notification-only. Fine for episode alerts; rules out background sync. |

The honest caveat: **conversion is the real limit, not technology.** Users must
install to Home Screen *and then* grant permission, so only a fraction complete
it. That's an argument for making `05` screen #14 (notification permission
prompt) do double duty as an **"Add to Home Screen" pre-permission explainer on
iOS** — it's already in the design inventory, and this is exactly what it should
say on that platform. Add it to the brief.

### The capability split — and why restricting video posting is correct

Your instinct to restrict video posting isn't just a cost lever; **it's the
technically correct call.** Client-side compression is a one-line native option
on Android (`expo-image-picker`'s `videoQuality`) and effectively unsolved on the
web:

- **WebCodecs** exists (Chrome/Edge 94+, Safari 16.4+, Firefox partial) but
  building a real transcode pipeline on it is substantial work.
- **ffmpeg.wasm** is ~25–30 MB and the *threaded* build needs
  `SharedArrayBuffer`, which requires cross-origin isolation (`COOP`/`COEP`
  headers) that CDNs frequently strip — a known production trap.
- **`MediaRecorder`** can't cleanly re-encode a picked file without realtime
  canvas playback, which is lossy and unseekable.

None of this is an Expo limitation — it's the browser platform. It was equally
true under the KMP plan (where it was *worse*, because Compose/Wasm also lacked
file-picker and camera parity outright). Under Expo the native side is trivial
and the web side is unchanged, so the split still holds for the same reason.

So the platform matrix becomes a coherent product story rather than a list of
compromises:

| Capability | **Android (native)** | **Web / PWA (incl. iOS)** |
|---|---|---|
| Home feed, Drama Hub, episode threads, Communities, Search, Profile | ✅ | ✅ |
| Follow graph, onboarding | ✅ | ✅ |
| Post **text / image** | ✅ | ✅ (client-side resize → WebP) |
| **Post video** | ✅ **`expo-image-picker` `videoQuality` → single 720p H.264 rendition → R2** | ❌ **restricted** |
| **Watch video** | ✅ vertical autoplay swipe | ✅ **tap-to-play** at MVP (§ below) |
| DMs (1:1) | ✅ Supabase Realtime | ✅ Realtime over WebSocket |
| **Push** | ✅ FCM via Expo Push Service | ✅ **Web Push (VAPID), $0, no Apple account** |
| **OTA updates** | ✅ **EAS Update — free to 1,000 MAU; ship fixes with no store** | n/a — every deploy is instant |
| Sign-in | Google + email | **Google + email** — see note |
| Distribution | **Direct APK link, $0** (Play Store optional later, $25 once) | Cloudflare Pages, **$0, unlimited bandwidth** |

> **Note on Sign in with Apple:** configuring it requires an Apple Developer
> account (a Services ID), so it's unavailable at $0 — but you don't need it.
> **Guideline 4.8 binds native iOS apps distributed via the App Store, not web
> apps.** A web client offering Google + email is compliant. This dissolves the
> objection that made me write "you can't even develop the required features."
> The features are required *of an App Store app*; you're not shipping one.

**The framing that makes this a product decision rather than a cut: Android is
the creator client, Web is the reader-and-discusser client.** That maps onto
actual behavior — short-form video *creation* is a mobile-native act, while
reading episode threads and arguing about a finale is something people happily
do in a browser or an installed web app. And it happens to match your audience:
Android is the majority platform globally, and dominant in the
Africa/Southeast-Asia/LatAm markets where K-drama fandom is growing fastest.

Three cost/abuse benefits come free with the restriction: R2's 10 GB free tier
lasts far longer, video moderation load drops (recall it's ~60 sampled frames per
upload at 1 frame/5 s over a 5-minute cap — see §6), and the highest-abuse upload
surface is confined to the platform where you can enforce a server-side
**5-minute** cap most reliably.

### The one real risk left: the Explore feed on iOS Safari

**The canvas-interop risk is gone.** Under the KMP plan, `<video>` couldn't
render inside a Skia `<canvas>` and needed a `wasmJsMain` DOM-interop overlay
position-synced to a swipe gesture. Under Expo, `react-native-web` renders
`expo-video` as a real `<video>` element in real DOM. There is nothing to
overlay and nothing to sync. That was the hardest problem in the previous plan
and it no longer exists.

**What replaces it is smaller but still real: iOS Safari's autoplay and decoder
policy.** Muted `playsinline` autoplay only; a hard limit on how many `<video>`
elements can be live at once before the oldest gets torn down; and aggressive
memory reclaim in an installed-PWA context. A vertical swipe feed that keeps
three players alive for smooth preloading will hit that ceiling on an iPhone.

**Recommendation: still ship tap-to-play on web for MVP** — a poster grid/list
where one `<video>` element is live at a time — but for *these* reasons rather
than the canvas ones. It's simpler, it's robust on Safari, and it costs $0 egress
either way since the bytes come from R2. Build the full vertical swipe feed
(Reanimated paging, player reuse via a shared `VideoPlayer` instance) on
Android, and treat web-video-swipe as a later upgrade once you know people watch
on web at all.

The week-1 spike is correspondingly cheaper. It is no longer "can a canvas host a
DOM video overlay in a standalone PWA" — it's **"does `expo-video` preloading and
reuse behave on iOS Safari in Home-Screen standalone mode?"** Test it in
standalone mode specifically, because installed-web-app behavior differs from a
Safari tab. That's an afternoon, not a week.

### Two consequences nobody has written down

**(a) SEO was going to be zero — Expo Router solves it for free.** A Skia-canvas
app exposes no crawlable HTML, and for a K-drama app organic search is a real
acquisition channel: people literally Google *"crash landing on you episode 5
ending explained."* The KMP-era fix was to hand-build a separate prerendering
template system. **You don't need to.** Expo Router does **static rendering —
build-time HTML — on web today**, and **`generateMetadata` for per-route SEO
since SDK 56.** So Drama Hub and episode-thread routes ship as real, indexable
HTML pages (title, synopsis, cast, episode list, `og:image` from the TVmaze
poster) straight to **Cloudflare Pages** — unlimited static requests and
bandwidth on the free plan. This is now a *feature of the framework you already
chose*, not a project. It may be your cheapest meaningful growth channel, and it
also gives you the public URL you'd need for any store listing's privacy policy.

**(b) iOS web storage is disposable.** The `expo-sqlite` cache (IndexedDB/OPFS-backed
on web) sits in storage Safari evicts after ~7 days of non-use, with a quota
around 1 GB. Home-screen apps get a more durable bucket but are still evictable.
So the offline cache on web must be designed as a **performance optimization that
can vanish at any time**, never as a source of truth — no unsynced local-only
writes on web. This sharpens the "you-lost-Firestore's-offline-persistence" point
from `00_review_and_recommendations.md` §1: on Android you can afford a real
write outbox; **on web, don't.** Optimistic UI only, with server confirmation
required before the local row is treated as durable.

Put the two storage tiers behind one interface in `lib/storage.ts` and let the
platform pick the implementation. That's also what keeps the eventual
"add a real offline mode" change to one file.

### What still costs $99, and when to buy it

Native iOS remains blocked on: Sign in with Apple, APNs native push (richer than
web push, no install step, better conversion), Background Sync, deep links /
Universal Links, App Store discovery, and widgets. Buy the $99 when either
(a) web analytics show meaningful iOS traffic and low Home-Screen install
conversion — i.e. the install step is costing you retention — or (b) there's
first revenue or funding. Until then, **keep `iosMain` compiling and `iosTest`
running in CI** (free on a public repo, §7.1) so native iOS stays a
distribution switch rather than a rewrite.

Two edits to `06_prd_and_roadmap.md` follow: Phase 1's "Android + iOS parity"
becomes **"Android native + Web parity; iOS target compiles and is tested in CI
but is not distributed"**, and Phase 2's "Web target brought online" moves
**earlier** — it's no longer a stretch goal, it's how half your audience gets
the product.

---

## 5. Catalog data at $0 — this reverses my earlier TMDB warning

My review flagged TMDB's commercial-use clause as a launch blocker, with public
pricing around **$149/month**. Under a zero-budget constraint that is
unaffordable *and* the licensing analysis gets worse, not better, the moment you
have any traffic.

**The answer is TVmaze.** This is the best single finding in this document
after R2:

- **Free. No API key. No account. No stated limit on call volume.**
- TV-focused, which is exactly your domain: shows, **seasons, episodes with air
  dates and runtimes**, cast and crew, and a **`/schedule?country=KR&date=`**
  endpoint returning everything airing on a given date — which is precisely
  what drives `06` §5's "currently airing" countdown and `12`/`08`'s
  new-episode notification trigger.
- **80,000+ shows**, community-maintained, images included (posters), plus
  IMDb and TheTVDB IDs for cross-referencing.
- `?embed=episodes` and `?embed=cast` fold related data into one request —
  important given the ~20 calls/10s per-IP rate limit.
- **Supports alternate titles (AKAs)** — which directly solves the
  Hangul / romanization / fan-abbreviation search problem I flagged in
  `00_review_and_recommendations.md` §9. Populate your `aliases` table from
  TVmaze AKAs and you get multi-script search for free.
- License is **CC BY-SA 4.0 with attribution**. Most endpoints are edge-cached
  60 minutes.

**Two honest caveats:**

1. **CC BY-SA is share-alike.** For an app that *serves* catalog data to its own
   clients this is normally fine, but the adaptation/share-alike question on a
   derived database deserves a real legal look — it's a different risk profile
   from TMDB's commercial ban, not a risk-free one. It's also the reason to keep
   the catalog in its own tables with a clean provenance column, so you can
   re-source it without touching user-generated data.
2. **Coverage is community-maintained and TV-only.** K-drama coverage is good
   but will have gaps and stale air dates. There is a corroborating real-world
   datapoint: a developer who hit TMDB's $149/month wall on a TV-tracking app
   pivoted to TVmaze and reported that TVmaze confirmed commercial use was fine,
   with licensing only discussed above a revenue threshold. That's encouraging
   but it's one anecdote — **get it in writing from TVmaze before launch.**

**Design the catalog layer to be swappable from day one.** One
`CatalogDataSource` interface in `shared/data`, one `external_ids` table mapping
`hallyu_id ↔ tvmaze_id ↔ tmdb_id ↔ imdb_id ↔ tvdb_id`. Then adding TMDB later
as a gap-filler (or switching entirely) is a new implementation, not a
migration. **Store the IDs even for sources you don't use yet** — that's the
cheap part, and retrofitting identity mapping across a live catalog is not.

**Ingestion job** (one of your 5 free Workers crons, or a Supabase scheduled
function):

- Nightly: pull `/schedule?country=KR` for the next 14 days → upsert air dates.
- Nightly: re-poll episodes for every `status = 'airing'` drama.
- On demand: fetch a full show when a user follows a drama not yet in your
  catalog, then queue it.
- **Never push a "new episode is live" notification from a single source read.**
  Air dates shift constantly for delays and hiatuses, and a false push is a
  direct hit on the retention hook that is your core thesis (`01` §2). Require
  the air date to be confirmed on two consecutive polls, or within N hours of
  airtime, before enqueueing.

**Attribution:** TVmaze requires attribution to tvmaze.com. Add it to
`05_stitch_uiux_prompt.md` screen #50 ("About / legal") alongside the TMDB
string, in case you add TMDB as a gap-filler later:
*"This product uses the TVmaze API but is not endorsed or certified by TVmaze."*

---

## 6. Moderation at $0 (and the CSAM baseline)

`12_content_moderation_policy.md` currently names **Perspective API, which
shuts down after 2026.** The zero-budget replacement is better and free:

**Primary: OpenAI Moderation API** — `omni-moderation-latest`. **$0, no
per-request charge**, covers **text and images** across 13 categories
(sexual, hate, harassment, self-harm, violence, illicit, and graphic variants),
**100+ languages**. Standard rate limits apply. This is the direct Perspective
substitute and it costs nothing.

Known limits to design around: image coverage is a **subset** of the text
categories (sexual / violence / self-harm only — no hate or harassment from
images), and there is **no native video or audio moderation.**

**Video moderation at $0 — frame sampling.** No vendor gives you free video
moderation. So: extract N frames plus the client-generated poster frame, and run
each through the free image moderator. Take the max severity across frames.

At a **5-minute** cap the sampling rate matters for cost-of-attention, not
dollars (the calls are $0 either way):

| Sampling | Frames per 5-min upload | Notes |
|---|---|---|
| 1 per 2 s | **150** | Overkill — adjacent frames are near-identical |
| **1 per 5 s** | **60** | **Recommended.** Catches anything lasting >5 s |
| 1 per 10 s | 30 | Minimum defensible; a 6-second flash slips through |

**Do the extraction client-side.** `expo-video` already generates the poster
natively; sample the extra frames during pick and upload them alongside the
video. That needs **no server, no ffmpeg, no Oracle box** and it puts the frames
in R2 before the video row goes visible. The server-side ffmpeg path (§6's
Oracle option) stays as the fallback for anything a client fails to sample.

One consequence of client-side sampling worth writing down: **a modified client
can upload a video with clean frames and dirty content.** So treat client frames
as a *first pass*, and keep a server-side re-sample in the escalation path for
anything reported. Don't let the cheap path become the only path.

**Free fallbacks / second opinion** (useful because single-vendor moderation is
a real risk when the vendor is free):
- **Azure AI Content Safety** — 5,000 images + 5,000 text records/month free (card required).
- **Google Cloud Vision SafeSearch** — 1,000 units/month free (card required).
- **Sightengine** — 2,000 ops/month free, 500/day cap, no card.
- **Self-hosted open source** on the Oracle free ARM box: **NudeNet** (CPU-friendly nudity detection), **Falconsai/nsfw_image_detection**, **NSFW.js** (Inception V3, runs in-browser via TF.js — could do a *client-side* pre-filter before upload, which is $0 and reduces what you must store).

**CSAM — free tools genuinely exist, and this is your highest-liability item.**
`12`'s "non-negotiable baseline" is currently one bullet long. At $0 you can
actually implement it:

| Tool | Cost | Covers |
|---|---|---|
| **Cloudflare CSAM Scanning Tool** | Free | Known-content hash scanning; recommended baseline for smaller platforms |
| **Meta PDQ / TMK+PDQF** (ThreatExchange) | Free, open source | Photo + video perceptual hashing, self-hostable |
| **Microsoft PhotoDNA** | Free for qualified organizations (vetted) | Still images, industry standard, NCMEC-vetted |
| **Google CSAI Match** | Free for qualifying partners | **Known video** matching |
| **Thorn hashing toolkit / SSVH** (locally installed) | Free for approved companies | Scene-sensitive **video** hashing |
| **Project Arachnid Shield** | Free | Known-content detection |

Thorn also supports **pHash as an open-source alternative** for companies
without a PhotoDNA license, so you can start hashing on day one and upgrade to
PhotoDNA once vetted.

Three things to know:
- **Build the reporting workflow, not just detection.** 18 U.S.C. §2258A
  requires US companies to report suspected CSAM to **NCMEC's CyberTipline**
  when they become aware of it. Registration is free but is a process. Note also
  that no US law currently *requires* proactive detection — but the EU's DSA
  creates obligations, and `01` explicitly targets a global audience.
- **Only PhotoDNA, Google CSAI Match, and AWS's offering are formally
  NCMEC-vetted.** Rolling your own perceptual hash and claiming equivalence
  will not survive scrutiny. Use a vetted tool.
- **Lead time is the real cost.** Published practitioner guidance puts
  decision-to-live at **4–6 months**, with vendor vetting alone at 2–4 weeks.
  **Start the applications in week 1**, in parallel with development. This is
  the only item in this document where the constraint is calendar time rather
  than money — and it's the one that can stop a launch.
- **Evidence retention must be carved out of `13`'s purge window.** `15` §5
  promises a 30-day purge; removed CSAM and its report metadata must be retained
  on a different schedule, with restricted access. Write that into the missing
  `13_data_retention_and_privacy_handling.md`.

Also resolve the contradiction I flagged earlier: `12` §1 (flagged content held
as `pending`) vs `06` §8 (visible within 5 seconds). At zero budget with no
moderation staff, the only workable posture is **publish optimistically,
moderate asynchronously, remove retroactively** — with pre-screening
concentrated on **media**, where harm is concentrated and users already expect
upload latency. Text-only posts publish immediately and are scanned in the
background.

**Human review at $0:** `12` §3 assumes 1–2 paid moderators and an internal
admin dashboard. At zero budget: (a) you are the moderator, (b) the "dashboard"
is a **Supabase SQL view + a table in the Supabase dashboard**, or a free
Retool/Appsmith-tier tool — not a 12th app surface you build, and (c) add
**community moderation** (report + trusted-user hide thresholds) to `02`'s MVP
scope, because it's cheap to build and it's the only moderation that scales
without money. Be honest in `12` that the 24-hour SLA is a founder-availability
SLA at MVP.

---

## 7. Free infrastructure that requires a decision, not money

Three items here are worth more than any vendor choice above, because they're
free but only if you decide early.

### 7.1 Make the repository public → unlimited free CI

GitHub Actions is **free and unmetered on public repositories, including macOS
runners**, on standard 2-core runners. On a private repo, macOS minutes consume
your allowance at a **10x multiplier** — the Free plan's 2,000 Linux minutes
become **~200 macOS minutes**, which is exhausted almost immediately by anything
that builds iOS natively. Confirmed still true after the 1 January 2026
repricing: *"Usage of standard GitHub-hosted runners in public repositories will
remain free."*

**Under Expo this matters less than it did — but for a better reason.** The KMP
plan needed macOS runners on every CI run to compile and test an iOS target.
Expo's free tier gives you **15 iOS + 15 Android EAS builds per month** and
**local builds cost nothing at all**, so iOS native compilation is no longer a
per-PR gate. Your PR pipeline is Linux-only: typecheck → lint → Jest → local
Supabase + RLS tests. Fast and free either way.

Still make the repo public. It buys unlimited Linux minutes (so the nightly E2E,
Realtime and emulator jobs never have to be rationed), and it's the right call
for an unfunded project whose moat is the fandom graph and the product, not the
code. Signing certificates and credentials live in encrypted secrets either way,
so visibility doesn't expose them. **Decide this before the first commit**,
because it changes your repo hygiene from day one.

### 7.2 Distribution is a link, not a store — and what that costs you

**Decision made: no Play Store at MVP.** Ship a **direct APK download link**
plus the Cloudflare Pages web URL. $0, no review queue, no 12-tester closed-test
gate, and it pairs with EAS Update so fixes reach installed users without anyone
re-downloading anything.

Know what you're trading away, because these are real and none of them are
obvious until a user hits them:

| | Direct APK link | Play Store |
|---|---|---|
| Cost | **$0** | $25 once |
| Time to first install | minutes | **~3 weeks minimum** (see below) |
| Install friction | **Play Protect "unknown developer" warning** on every install | none |
| Updates | EAS Update for JS; **users re-download for native changes** | automatic |
| Trust signal | weak — sideloading reads as sketchy to non-technical users | strong |
| Discoverability | none | store search |

The Play Protect warning is the one that actually costs you users. Mitigations,
all free: an install page with screenshots of exactly what the warning looks like
and what to tap; keep the APK on a domain you own (not a file host); publish the
SHA-256 next to the link so it's verifiable.

**If you add Play later, the lead time is not compressible.** Personal developer
accounts created **after 13 November 2023** must run a **closed test with at
least 12 testers opted in continuously for the last 14 days** before they can
even *apply* for production access. "Opted in" means they **accepted the invite
and installed** — invited-but-not-installed doesn't count, and a tester who drops
out on day 10 resets their clock. Internal testing does not substitute. Budget
~3 weeks between "first installable build" and "live in production," and start
recruiting from the K-drama communities `01` identifies (Discord, Reddit,
DramaBeans) before you need it.

Two build requirements that are cheap to satisfy now and expensive to retrofit:
new Play submissions must **target API 36** — **Expo already does** (SDK 54+
targets Android 16 / API 36) — and apps shipping native code must be **64-bit
and 16 KB page-size compatible**. That last one was a genuine KMP/Kotlin-Native
gotcha; on Expo it's handled by the React Native build, but **verify it on an
actual 16 KB emulator image**, not just a clean upload, because it fails at
runtime rather than at build time.

### 7.3 Fix auth email before you have a single real user

This is the landmine most likely to silently break your launch, and nothing in
the current pack mentions it.

Supabase's **built-in email provider is capped at 2 messages per hour,
project-wide** — and, worse, it **only sends to pre-authorized addresses** (your
project team). Your signup confirmation and password-reset emails **will not
reach real users at all** on the default configuration. Upgrading to Pro does
**not** change this; custom SMTP is the only documented fix.

The fix is free and takes about ten minutes:
1. Create a **Brevo** account (300 emails/day ≈ 9k/month free, no card) — the
   most generous free option. Alternatives: Resend (3k/mo, 100/day), Mailjet
   (6k/mo, 200/day), SendGrid (100/day).
2. Supabase → Authentication → Emails → SMTP Settings: host
   `smtp-relay.brevo.com`, port 587, your verified sender.
3. Raise the Supabase-side cap under Authentication → Rate Limits. **Custom
   SMTP starts you at 30/hour**, which caps you at 30 new signups/hour until you
   raise it.
4. Set **SPF, DKIM, and DMARC** DNS records or your mail lands in spam.

Note also that as of **June 2026, new free-tier projects using the default email
provider cannot customize auth templates at all** — another reason this isn't
optional. And `06` §1's signup flow is the top of your funnel: a silent 2/hour
cap there is indistinguishable from "signup is broken."

---

## 8. Free-tier operational risks you must design for

Zero budget buys you money and spends reliability. Be clear-eyed about the
trade:

| Risk | Reality | $0 mitigation |
|---|---|---|
| **Supabase pauses your project after 7 days of DB inactivity** | ~30s cold start; app is offline until then. Tracked against *database* activity, not dashboard visits. Has taken down staging environments over quiet holiday weeks. | **GitHub Actions scheduled workflow** hitting a `ping` table every 3 days. Free on a public repo. Set this up in Phase 0, not after the first outage. |
| **No backups at all on the free tier** | No daily backups, no PITR. One bad migration or one deleted table is terminal. | **GitHub Actions cron → `pg_dump` → gzip → R2**, 30-day rolling window. ~10 lines of YAML, $0. This is non-negotiable — treat it as a Phase 0 deliverable alongside the scaffold. |
| **No SLA anywhere** | Supabase free is community support only. TVmaze has no SLA. Oracle free instances get reclaimed for idleness. Free tiers get reduced with little notice (Oracle halved its ARM allowance in mid-2026; Mixpanel cut its free tier 20M→1M in late 2025). | Assume any free tier can change or vanish on 30 days' notice. Keep the repository-interface seam. Keep the `external_ids` mapping. Budget a "vendor changed under us" contingency each quarter. |
| **~~Single-maintainer dependency~~ — RESOLVED by the stack change** | This row said: `supabase-kt` is community-maintained, *not* official, with effectively one primary maintainer, and your entire backend client depends on it. **That risk is gone.** Expo uses `@supabase/supabase-js`, Supabase's own first-party reference client with a documented React Native quickstart. | Nothing to mitigate. Keep the feature `api.ts` modules as the seam anyway — it's free discipline and it's what makes the eventual "swap a vendor" change a one-file diff. |
| **Expo SDK churn** | Expo ships a new SDK roughly quarterly (57 current, Sept 2026). Breaking changes land in `expo-av`-style removals — `expo-av` was **deleted outright in SDK 55**. | Pin the SDK in `package.json`; upgrade deliberately, not automatically. Run `npx expo install --check` after every upgrade. Never `npm install` a React Native package directly. |
| **EAS Update's 1,000-MAU hard cap** | Past 1,000 monthly active updaters the free tier **stops sending updates** (apps keep running, they just stop updating). It is a hard cap, not an overage bill — which is good for the budget and bad for silently-stale clients. | 1,000 MAU is a real milestone, not a rounding error. When you hit it: $5 per 1,000 updaters, **or self-host** — the Updates protocol is an open spec. Add an MAU alert at 700 so this is never a surprise. |
| **Free moderation vendor risk** | OpenAI's free moderation endpoint has no contractual commitment. Perspective's shutdown is the precedent: free, widely depended on, then withdrawn with no migration support and ~10 months' notice. | Put it behind a `ContentModerationService` interface (your clean architecture already provides this). Configure at least two providers so switching is a config change. |
| **No load headroom** | Supabase free is **shared compute, ~500 MB RAM, 50 direct / 200 pooled connections**. Not autoscaling. (The original `11` said "Firebase autoscaling handles MVP-stage load" — that line is gone, but the *posture* it encoded survives: `11` still says "no load/performance infrastructure before launch." That posture is wrong here, and is the one place where `16` overrules `11`.) | Use Supavisor pooled connections (port 6543) for Edge Functions. Index every RLS-filtered column. **Do run one load test anyway**: a 30-minute k6 run against the feed query costs nothing, and RLS-per-row slowdowns never appear in unit tests — every row is re-checked against every applicable policy. |
| **Abuse is free for attackers too** | PostgREST exposes your tables as REST to any authenticated client — scraping is *easier* than it was on Firestore. `14` §5 prohibits it but nothing enforces it. | Supabase's built-in rate limits; `.limit()` on every query; row-count caps; a Worker in front for per-IP throttling (100k req/day free); audit that the `anon` key can read nothing it shouldn't. A permissive RLS policy plus a client-side key is the classic Supabase breach — make the RLS matrix in the rewritten `08_api_contract.md` a reviewable artifact. |

---

## 9. The upgrade ladder — what to buy first

Design every step so it's a config change. In the order I'd spend money:

| # | Purchase | Cost | Unlocks | Trigger to buy |
|---|---|---|---|---|
| 0 | **Domain name** | ~$10/**year** | Uncapped edge-cached R2 video delivery; SPF/DKIM for email deliverability; a real privacy-policy URL | Immediately if you can afford $10 once. Otherwise the `workers.dev` path holds to ~10k video views/day. |
| 1 | **Google Play** | **$25 once** | Android distribution. No renewal, ever. | As soon as a build is installable — and start the 12-tester/14-day clock the same day (§7.2). |
| 2 | **Supabase Pro** | **$25/mo** | **50x** API egress (5 GB → 250 GB), 8 GB DB, no 7-day pause, **daily backups**, dedicated pooling, 500 Realtime connections, 5M Realtime messages | The moment you have real users. The pause and the absent backups are reasons to buy this *before* you need the egress. |
| 3 | **Apple Developer Program** | **$99/yr** | **Native** iOS: Sign in with Apple, APNs push with no install step, Background Sync, Universal Links, App Store discovery | When web analytics show real iOS traffic *and* low Home-Screen install conversion — i.e. the install step is measurably costing retention. Or first revenue/funding. |
| 4 | **TMDB commercial license** | ~$149/mo **[verify]** | Richer catalog + hosted imagery, cleaner rights than CC BY-SA | Only if TVmaze coverage gaps start hurting, or on monetization. |
| 5 | **Server-side HLS ladder** (Oracle free → Bunny Stream / Cloudflare Stream) | $0 → $50–100/mo | Adaptive bitrate; fixes the one real quality concession in §3 | When buffering shows up in PostHog playback metrics. |
| 6 | **Paid moderation** (Hive / Sightengine) | ~$29/mo+ | Video-native moderation, deeper categories, higher accuracy | When the false-positive rate in your review queue costs more founder time than $29/mo. |

**Note what's absent:** no paid hosting, no paid CDN, no paid CI, no paid
analytics, no paid auth, no paid email, no paid database until step 2. The
first *three* purchases total **$35 one-time + $10/year** and get you a live,
distributed, backed-up product on Android and Web.

---

## 10. What zero budget does not cover

Be honest about these in the roadmap rather than discovering them at launch:

- **Legal review.** `14` and `15` are structural drafts that both say they need
  counsel. That costs money. Free-tier mitigations: your jurisdiction's
  startup/legal-aid clinics, bar-association pro bono programs, and template
  review services. But the **TVmaze CC BY-SA share-alike question**, the
  **drama-poster/cast-photo IP question** (`14` §7's closing note), the **CSAM
  reporting obligations**, and the **age-gating/COPPA-GDPR-K question** are not
  things to guess about. If you can spend money on exactly one thing, spend it
  here.
- **The founder's time as the moderation team.** `12` §3's 24-hour SLA is, at
  $0, a promise about your personal availability. That's a real cost with a
  real ceiling, and it's the thing that breaks first when the app succeeds.
- **A Mac.** Native iOS development needs macOS. Free GitHub Actions macOS
  runners cover CI (§7.1), but interactive iOS debugging needs a physical Mac —
  and you also need **an iPhone to test the web version properly**, because the
  Home-Screen-installed standalone PWA behaves differently from a Safari tab
  (that's where Web Push, storage eviction, and video autoplay all differ). If
  you can borrow or buy one used device, make it an iPhone: it's now your
  highest-value test device precisely *because* you're not shipping a native
  iOS app.
- **Human QA on real devices.** `11` §4 asks for a manual QA pass on real
  devices per phase. At $0 that's your own devices plus your 12 Play testers.
  Recruit them deliberately and treat them as QA, not just as a
  closed-testing-requirement checkbox — they're the same 12 people.

---

## 11. What zero budget forces you to cut from the spec

Not "nice to have later" — these are cuts the free tiers actively require:

| Cut | Forced by | Note |
|---|---|---|
| **Native iOS app** | Apple entitlements (§4) | **Not a reach cut** — iOS users get the web PWA with home-screen install + $0 Web Push. Keep `iosMain` compiling and tested in CI. |
| **Video *posting* on web** | No viable browser-side transcode (§4) | Android-only posting; web **watches** video fine. Android = creator client, Web = reader/discusser client. |
| **TikTok-style swipe feed on web** | Canvas/Skia can't host `<video>` without fragile DOM interop (§4) | Tap-to-play grid on web at MVP; vertical swipe on Android. |
| **Server-side video transcoding / HLS ladder** | No free transcode compute worth operating (§3) | Client-side compress to one 720p rendition; the one real quality concession |
| **Supabase Storage entirely** | 1 GB cap = ~100 videos (§2.1) | All media to R2 |
| **Group DMs → 1:1 only** | 200 concurrent Realtime connections (§2.3) | Fewer simultaneous sockets, simpler membership model, less moderation surface |
| **Channels** | Not free-tier-forced — but `08` already says *"Create: none in MVP (creator program gated)"*, so **no user can create one** | You'd be designing (`05` #31–32), building and testing a dead feature. Cut wholesale. |
| **Analytics in Postgres** | 1M events ≈ 40% of the 500 MB DB (§2.2) | PostHog free instead |
| **Paid moderation team** | $0 (§6) | Founder + community moderation + free automated screening |
| **ML recommendations** | Already out of scope in `02` | Rule-based ranking is one SQL query |
| **Always-on Realtime** | 200 connections (§2.3) | Subscribe only inside an open thread; poll everywhere else |

**What survives intact:** the entire product thesis, the drama-hub information
architecture, per-episode discussion threads, communities, the follow graph,
search (with TVmaze AKAs feeding your `aliases` table), spoiler handling, and
the episode-cadence retention hook. **Nothing in `01_vision_and_case.md` or
`02_product_spec.md` is weakened by the zero-budget constraint** — the cuts are
infrastructure sophistication, not product. And with §4's revision, even the
platform-reach cut mostly evaporates: **you ship to Android *and* iOS users from
day one, for $0.**

That's the real answer to "can this work on no budget and work well": **yes,
because the thing that makes Hallyu interesting is the fandom graph and the
weekly episode rhythm, and neither of those costs money.** What costs money is
*native* iOS polish, adaptive-bitrate video, and other people's moderation
labor — and all three are legitimately deferrable until the data says you need
them.

### One consequence to plan for: video supply will be thin

Restricting video posting to Android narrows the top of the Explore funnel — and
note you **cannot seed it with official clips**, because drama footage is
somebody else's copyright (exactly the IP problem `14` §7 flags). Fan edits and
reactions are user-generated or they don't exist.

So at launch Explore will be sparse, and **an empty TikTok-style feed is the
worst possible first impression** for the exact users `01` calls "The Curious
Newcomer." Three free mitigations:

- **PostHog feature flags are free** (§1). Gate the Explore tab on a
  server-side content threshold so it doesn't appear until there's enough video
  to fill a swipe. This is what the free flag allowance is *for*.
- **Let Explore be image-first initially.** Fan edits, poster galleries, and
  cast stills are a legitimate short-form discovery surface and `02` §4 doesn't
  forbid it — only `05` screen #19 assumes video. Images post fine from web
  *and* Android, so supply is much larger.
- **Design `05` screen #16 (feed empty state) to cover Explore too**, and seed
  the *drama hubs* rather than the video feed — the catalog arrives fully
  populated from TVmaze on day one, so the app never looks empty even when
  user-generated video is scarce.

---

## 12. Status of the pack — what's been fixed and what's still missing

This section was originally a proposed diff ("if you want me to apply these").
Most of it has now been applied. Rewritten as a status table so it doesn't read
as a to-do list for work that's already done.

| # | Document | Status |
|---|---|---|
| 1 | **`03_technical_architecture_expo.md`** | ✅ **Rewritten from scratch** (was `..._kmp.md`). R2 + custom domain, `expo-image-picker` `videoQuality` for client-side re-encode, PostHog, `@supabase/supabase-js` first-party client with the correct RN wiring, two-tier storage strategy replacing Firestore's offline persistence, Android-native + Web-PWA targeting with iOS CI-compiled but undistributed, and the §4 platform-capability table. |
| 2 | **`04_ai_studio_build_prompt.md`** | ✅ **Rewritten** — this was the P0 fix. It previously told the model to scaffold Kotlin/Gradle/Koin/Ktor/SQLDelight/Voyager and would have generated the wrong app entirely. Now Expo SDK 57, expo-router, expo-video, TanStack Query, NativeWind, with explicit "never import `expo-av`" and Supabase client wiring rules baked in. |
| 3 | **`06_prd_and_roadmap.md`** | ✅ Applied — Supabase Auth (Apple sign-in dropped); **60 s → 5-minute cap, enforced both client-side and server-side**; Firestore listeners → Supabase Realtime; Cloud Functions → Postgres triggers; FCM → Expo Push + Web Push; Phase 0 rebuilt around the Expo scaffold, the Supabase client, and **`09` + the RLS test suite**; Phase 1 "Android + iOS parity" → **"Android + web parity"**; web moved up out of Phase 2. ⚠️ *Still open:* Brevo SMTP configuration and the `pg_dump`→R2 backup cron aren't in Phase 0 yet. |
| 4 | **`08_api_contract.md`** | ✅ **Rewritten** as a Supabase contract: RLS-gated table surface, column-level grant rules, the trigger/`pg_cron`/Edge-Function split, Realtime channel topology, the R2-vs-Supabase-Storage split, and a versioning section that accounts for **link-distributed APKs running old code indefinitely**. ⚠️ *Still open:* `?select=` column allowlists and explicit rate limits. |
| 5 | **`11_test_strategy.md`** | ✅ **Rewritten** — was KMP `commonTest`/Turbine/Compose and, worse, tested **Firestore Security Rules**. Now Jest + `jest-expo` + RNTL, and §2 makes **RLS policy testing the non-negotiable layer** against local Supabase, including the deny-cases that actually leak. Realtime tested against a real instance, never mocked. Maestro + Playwright for E2E. |
| 6 | **`12_content_moderation_policy.md`** | ✅ Partially — Firestore admin-tool reference → service-role Postgres; `moderationStatus` → `moderation_status`. ⚠️ *Still open:* Perspective API → OpenAI Moderation, the frame-sampling table in §6, the free CSAM tooling table, and the week-1 vendor applications. |
| 7 | **`15_privacy_policy.md`** | ✅ Partially — processors named (Supabase, Cloudflare, Expo, PostHog, TVmaze); analytics → PostHog; §7 Security now describes RLS; §8 International Transfers made concrete with an **open region decision**. ⚠️ *Still open:* §5's 30-day purge needs the CSAM evidence carve-out. |
| 8 | **`01_vision_and_case.md`** | ✅ §5's "one codebase, three platforms / KMP + Compose" claim → **Expo, two shipping surfaces, web as the iOS vehicle**. |
| 9 | **`05_stitch_uiux_prompt.md`** | ✅ Screen #37's 60 s cap indicator → **5-minute**, plus the web variant notice. ⚠️ *Still open:* TVmaze attribution (#50), DOB/age gate (#4–#10), dropping Channels screens #31–#32, the iOS "Add to Home Screen" explainer on #14, a new "Install Hallyu" screen, an Explore low-supply state, a tap-to-play web variant of #19, and a web variant of #34 with video **visibly disabled** rather than silently hidden. |

### Still missing — and these are the blockers

| Document | Why it blocks |
|---|---|
| **`09_database_schema.md`** | **The critical path.** `04` has an explicit `[paste the schema here]` hole, and `08` describes a surface without DDL. Postgres tables, columns, constraints, **every RLS policy**, trigger bodies, `pg_cron` jobs, and migration ordering. `11` §2's policy tests can't be written without it. **Write this next.** |
| **`07_analytics_tracking_plan.md`** | PostHog event schema covering `02`'s three success signals (episode-day WAU spike, community join rate, feed-engagement-to-video-view ratio) plus **playback buffering metrics** — that's your trigger for the HLS upgrade in §3. |
| **`13_data_retention_and_privacy_handling.md`** | `15` §5 promises a 30-day purge that is nowhere defined, and the CSAM evidence carve-out has to live somewhere. Also where the **video storage retention policy** belongs now that ~220 clips is the R2 free ceiling (§2.1). |
| **`19_pwa_and_web_delivery.md`** *(new, proposed)* | Web is a primary delivery channel — arguably *the* primary one, since it's how every iOS user arrives. Needs: `manifest.json` with `display: standalone`, service worker or Declarative Web Push, VAPID key generation/rotation, `display-mode: standalone` detection, iOS-Safari-only install messaging, **Expo Router static-rendering + `generateMetadata` config for the crawlable Drama Hub / episode routes** (§4a), and the APK download page with the Play Protect walkthrough (§7.2). |
