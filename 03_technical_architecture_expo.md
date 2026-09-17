# Hallyu — Technical Architecture (React Native + Expo)

> **This file was `03_technical_architecture_kmp.md`.** The Kotlin Multiplatform /
> Compose Multiplatform plan it described is not the stack this product is being
> built on. Rewritten 2026-09-17 for **React Native + Expo**, which is also the
> better fit for a zero-budget build — see `16_zero_budget_stack.md`.
>
> If any other document in this pack still says KMP, Compose, Gradle, Koin, Ktor,
> SQLDelight or Voyager, **that document is stale.** Known offenders at the time
> of writing: none remaining (`04`, `11`, `01` §5 and `06` Phase 0 were updated
> in the same pass).

## Targets

Two shipping targets from one codebase:

- **Android** — native app, distributed by **direct APK link** at MVP (no Play
  Store; see `16_zero_budget_stack.md` §4)
- **Web** — the same React Native code via `react-native-web`, deployed as a
  static/SSR bundle. **This is also how iOS users get Hallyu**, as an
  installable home-screen web app.

**iOS native is deliberately not an MVP target.** Not because it's hard — because
it cannot be done at $0. The free Apple ID tier withholds the push and Sign in
with Apple entitlements, and distribution costs $99/year. Web Push reaches iOS
users for $0 with no Apple account at all, so the product still ships to them.
Keep the iOS build path working in CI anyway (Expo builds iOS on the free tier —
15/month) so native iOS stays a config switch later rather than a port.

One codebase, three *platforms* (`android`, `ios`, `web`) via `Platform.OS`, but
only two shipping surfaces at MVP.

## Architecture style

Feature-first, with a thin data layer. React Native rewards a flatter structure
than the clean-architecture layering the KMP draft assumed — don't port that
ceremony across.

```
app/                     # Expo Router file-based routes (universal: native + web)
  (auth)/                #   sign-in, sign-up, onboarding group
  (tabs)/                #   Home, Explore, Communities, Messages, Profile
  drama/[id]/            #   Drama Hub — index, cast, episodes
  episode/[id]/          #   per-episode discussion thread
  post/[id]/
  +not-found.tsx
src/
  features/              # one folder per feature: feed/, explore/, dramahub/,
                         #   communities/, messages/, composer/, search/,
                         #   notifications/, profile/, onboarding/
    ├── components/      #   presentational, no data fetching
    ├── hooks/           #   useFeed(), useDrama(), useFollow() — TanStack Query
    └── api.ts           #   the ONLY file that touches supabase for this feature
  components/            # shared UI kit (Button, Card, Avatar, Skeleton…)
  lib/                   # supabase client, query client, storage, analytics
  theme/                 # design tokens from Stitch (05_stitch_uiux_prompt.md)
  types/                 # generated DB types + shared domain types
supabase/
  migrations/            # SQL, version-controlled — the real schema source
  functions/             # Edge Functions (Deno)
  seed.sql               # catalog seed
```

**Rules that keep this from rotting:**

- Screens compose; they don't fetch. Data fetching lives in `features/*/api.ts`,
  wrapped by hooks. A component that calls `supabase` directly is a bug.
- **Generated types, not hand-written ones.** `supabase gen types typescript`
  produces `types/database.ts` from the live schema. Run it in CI and fail the
  build if it drifts. This replaces SQLDelight's compile-time safety with
  something equivalent and free.
- Platform differences go in `*.native.tsx` / `*.web.tsx` file pairs or
  `Platform.OS` branches — not in separate app trees.

## Tech stack

| Concern | Choice | Why |
|---|---|---|
| Framework | **Expo SDK 57** (React Native, New Architecture) | Officially recommended over bare RN CLI. Free forever; you only pay for EAS cloud services. New Architecture is mandatory since SDK 55, so start there. |
| Routing | **Expo Router** | File-based, universal — the same `app/` dir becomes native stacks and web routes. Supports **static rendering (build-time HTML)** and `generateMetadata` for per-route SEO, which matters more than usual here (see *Web delivery* below). |
| Backend | **Supabase** (Postgres + Auth + Realtime + Edge Functions) | Relational fit for dramas↔cast↔episodes↔threads↔follows. `@supabase/supabase-js` is the reference client — no third-party-maintainer risk. |
| Server state | **TanStack Query** | Cache, retry, pagination, optimistic updates — all of which the feed and follow buttons need. Don't hand-roll this. |
| Client state | **Zustand** | Minimal, no provider boilerplate. Session/UI state only; server data stays in TanStack Query. |
| Media (video) | **`expo-video`** | `expo-av` was **removed in SDK 55** — do not use it. `expo-video` splits `VideoPlayer` (logic) from `VideoView` (UI), has **first-class web support**, native thumbnail generation, PiP, and exposes `averageBitrate`/`peakBitrate`. |
| Media (audio) | `expo-audio` | Same split from the old `expo-av`. |
| Images | `expo-image` | Built-in caching + progressive loading; better than third-party alternatives on New Architecture. |
| Video capture/pick | `expo-image-picker` | `videoQuality` option re-encodes on pick — the free fix for the HEVC problem (see *Video* below). |
| Lists | `@shopify/flash-list` v4 | Required for feed/Explore scroll performance. v4 needs New Architecture. |
| Animation | `react-native-reanimated` v4 | Gesture-driven vertical swipe feed. v4 needs New Architecture. |
| Local cache | `expo-sqlite` | Offline read cache. Web falls back to IndexedDB/OPFS via a storage adapter. |
| Secure storage | `expo-secure-store` | Auth tokens. Encrypted, unlike AsyncStorage. |
| Push | `expo-notifications` + Expo Push Service | Free, and it wraps FCM for Android. Web uses Web Push (VAPID) — see §4 of `16_...`. |
| OTA updates | `expo-updates` / EAS Update | **The reason link-distribution is viable.** Ship JS fixes with no store and no re-download. Free to 1,000 MAU. |
| Styling | **NativeWind v4** (or Unistyles 3) | Tailwind on native + web; both work on web. Plain `StyleSheet` is statically extracted to CSS automatically if you prefer zero deps. |
| Analytics | **PostHog** (`posthog-react-native`) | Free to 1M events/month, plus free feature flags. Do **not** put events in Postgres — see `16_...` §2.2. |
| CI | **GitHub Actions, public repo** | Unlimited free minutes including macOS runners. Local Expo builds are free; EAS free tier adds 15 Android + 15 iOS builds/month. |

## Supabase client setup (get this right once)

The official React Native pattern — deviations here cause session-loss bugs that
look like auth being broken:

```ts
// lib/supabase.ts
import 'react-native-url-polyfill/auto'          // required: RN lacks full URL
import { AppState, Platform } from 'react-native'
import * as SecureStore from 'expo-secure-store'
import { createClient, processLock } from '@supabase/supabase-js'

const url = process.env.EXPO_PUBLIC_SUPABASE_URL!
const key = process.env.EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY!

// SecureStore adapter — encrypted at rest, unlike AsyncStorage
const storage = {
  getItem: (k: string) => SecureStore.getItemAsync(k),
  setItem: (k: string, v: string) => SecureStore.setItemAsync(k, v),
  removeItem: (k: string) => SecureStore.deleteItemAsync(k),
}

export const supabase = createClient(url, key, {
  auth: {
    ...(Platform.OS !== 'web' ? { storage } : {}),  // web uses localStorage
    autoRefreshToken: true,
    persistSession: true,
    detectSessionInUrl: false,                      // NOT a web-URL session flow
    lock: processLock,
  },
})

// Register ONCE, at app root. Without this, tokens go stale after backgrounding.
if (Platform.OS !== 'web') {
  AppState.addEventListener('change', (s) => {
    s === 'active' ? supabase.auth.startAutoRefresh() : supabase.auth.stopAutoRefresh()
  })
}
```

Non-negotiables:

- **`react-native-url-polyfill/auto`** must be imported before the client is
  created, or Realtime and some auth paths fail with opaque errors.
- **Create the client once.** Never inside a screen or hook.
- **`onAuthStateChange` is wired once at the root**, and startup is treated as
  async — a valid session and a loaded profile are two different states.
- **Never ship `service_role` in the bundle.** Only the publishable/anon key.
  Everything privileged goes in an Edge Function.
- Env vars must be `EXPO_PUBLIC_*` to reach the client bundle.
- **Known gotcha:** Realtime pulls WebSocket dependencies that reference Node's
  `stream` module. If Metro fails on it, either add a stream polyfill or import
  `@supabase/gotrue-js` and `@supabase/postgrest-js` separately and only add
  `@supabase/realtime-js` where you actually subscribe (DM threads).

## Video

This is much simpler than the KMP draft assumed. The whole MVP pipeline is:

```
pick (expo-image-picker, videoQuality set → H.264 re-encode)
  → read duration, reject if over cap
  → upload original to Cloudflare R2
  → insert row: r2_key, duration_s, poster_key, moderation_status
  → play with expo-video (progressive MP4, byte-range, edge-cached)
```

**No transcoding, no ffmpeg, no HLS ladder.** Phones record H.264 MP4 that plays
everywhere; R2 egress is $0 uncapped. `expo-video` has web support built in, so
the same component works on Android and in the browser.

The one real trap: **iPhones record HEVC/H.265 by default.** Safari plays it, but
Chrome and Edge on Windows need a hardware decoder plus a paid Microsoft
extension, and Firefox desktop had it disabled through v136 — so raw iPhone clips
silently fail to play for a chunk of your web audience. Setting `videoQuality` on
`expo-image-picker` re-encodes to H.264 at pick time and solves it for free.

Poster frames come from `expo-video`'s native thumbnail generation — no server
work.

See `16_zero_budget_stack.md` §3 for the storage and egress math.

## Web delivery

Two things worth stating plainly, because they change the plan:

**SEO is solved by the framework.** Compose/Wasm rendered to a canvas and was
uncrawlable. **Expo Router does static rendering (build-time HTML) on web today,
and `generateMetadata` for per-route SEO since SDK 56.** So Drama Hub and episode
threads can be real, indexable HTML pages — which matters a lot for a product
whose acquisition channel is people Googling *"crash landing on you episode 5
ending explained."* Use `generateMetadata` per drama/episode route and serve
`og:image` from the TVmaze poster. This is a free growth channel, not an
afterthought.

**Video on web is not a problem.** `react-native-web` renders real DOM, so
`expo-video` maps to a `<video>` element. There is no canvas-overlay interop
issue — that entire risk belonged to the KMP/Wasm plan and is gone.

What *is* different on web:

- **Storage is disposable.** Safari evicts script-writable storage after ~7 days
  of non-use. The `expo-sqlite`/IndexedDB cache is a performance optimization
  that can vanish — never a source of truth, and **no local-only write queue on
  web.** Optimistic UI only, with server confirmation before a row counts as
  durable. Android can afford a real outbox; web cannot.
- **Push requires home-screen install.** `display: standalone` in the manifest is
  mandatory or `pushManager` is `undefined`. The install step must be Safari —
  Chrome on iOS is WebKit underneath and can't do it.
- **Camera/upload parity is thinner.** Video posting is Android-native-only at
  MVP; web watches but doesn't post video. Text and image posting work
  everywhere.

## Platform capability matrix

| Capability | Android (native) | Web / PWA (incl. iOS) |
|---|---|---|
| Feed, Drama Hub, episode threads, Communities, Search, Profile | ✅ | ✅ |
| Post text / image | ✅ | ✅ |
| **Post video** | ✅ | ❌ restricted — no viable browser-side re-encode |
| **Watch video** | ✅ vertical swipe (Reanimated) | ✅ tap-to-play at MVP |
| DMs (1:1), real-time | ✅ | ✅ WebSocket |
| Push | ✅ FCM via Expo Push | ✅ Web Push (VAPID), $0, no Apple account |
| Sign-in | Google + email | Google + email |
| OTA updates | ✅ EAS Update | n/a — deploy is instant |
| Offline read cache | ✅ durable | ⚠️ disposable |

**The framing that makes this a product decision rather than a list of cuts:
Android is the creator client, Web is the reader-and-discusser client.** That
matches real behavior — short-form video *creation* is a mobile-native act, while
reading episode threads and arguing about a finale happens happily in a browser.

## Data model

Full DDL, RLS policies and migrations belong in **`09_database_schema.md`**,
which does not exist yet and should be written before any code is generated. The
sketch below is the corrected entity list — the KMP draft's version was
Firestore-shaped (ID arrays on rows, no `Episode` entity) and would not have
survived contact with Postgres.

**Core:** `users`, `profiles`, `follows` (join tables, *not* ID arrays),
`dramas`, `actors`, `drama_cast`, `episodes`, `genres`, `drama_genres`,
`aliases`, `external_ids`
**Content:** `posts`, `media`, `comments`, `likes`, `communities`,
`community_members`, `reports`
**Messaging:** `conversations`, `conversation_members`, `messages`,
`channel_subscriptions`
**System:** `notifications`, `push_tokens`, `moderation_actions`

Every content row carries `moderation_status`, `is_spoiler` where relevant, and
`deleted_at` for soft delete. `episodes.air_date` is `timestamptz` — K-dramas air
on KST and a global audience sees countdowns in its own zone; a naive timestamp
produces wrong "next episode" pushes for most users.

## Build & distribution

- **Android:** local `eas build -p android --profile preview` (free) or EAS cloud
  (15/month free). Output a **universal APK**, not an AAB, for direct
  distribution. **Back up the signing keystore — lose it and you can never ship
  an update.**
- **Web:** `npx expo export --platform web` → Cloudflare Pages. $0, unlimited
  static bandwidth.
- **Updates:** `eas update` for JS-only fixes. Free to 1,000 MAU, then it stops
  sending (apps keep working). Hermes bytecode diffing keeps update payloads
  small. The Updates protocol is an open spec — self-hosting is supported if you
  outgrow the free tier.
- **Min OS:** Android 7+, iOS 15.1+. Android builds target API 36, which is what
  Google Play requires if you add it later.

## Handoff note

This document, `02_product_spec.md`, and the design tokens from Stitch
(`05_stitch_uiux_prompt.md`) are the three inputs the build prompt in
`04_ai_studio_build_prompt.md` consumes. That prompt was rewritten for Expo in
the same pass — **do not use any earlier version of it**, it instructed an AI to
scaffold a Kotlin/Gradle project.
