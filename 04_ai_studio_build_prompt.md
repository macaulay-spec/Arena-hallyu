# Master Prompt — Paste into Google AI Studio (Gemini)

> **Rewritten 2026-09-17 for Expo / React Native.** The previous version of this
> prompt instructed the model to scaffold a Kotlin Multiplatform + Compose
> project (Koin, Ktor, SQLDelight, Voyager, Gradle modules, `wasmJsMain`). **Do
> not use it** — it would generate an app that is not this product's stack.

Fill in the `[[DESIGN TOKENS FROM STITCH]]` block with the colors/type/spacing
Stitch actually produces, then paste everything inside the fence.

**Before you paste:** this prompt references the data model.
`09_database_schema.md` does not exist yet. Either write it first, or accept that
the model will invent field names and you'll be reconciling them later — which is
the more expensive path. The entity list in the prompt below is already
corrected; only the column-level detail is missing.

---

```
You are acting as a senior React Native engineer working in Expo. Scaffold the
initial codebase for a social app called "Hallyu" — a mobile-first social
platform for K-drama fans, combining a personalized feed, short-form video
discovery, drama-specific communities, one-way broadcast channels, and DMs.

STACK (non-negotiable — follow exactly):
- Expo SDK 57 (React Native, New Architecture, Hermes). Do NOT use the old
  architecture.
- Expo Router for navigation (file-based routing in an `app/` directory).
- react-native-web for the web target. Expo includes it; web must be a
  first-class build, not an afterthought.
- @supabase/supabase-js for the backend (Postgres, Auth, Realtime, Edge
  Functions).
- TanStack Query for all server state; Zustand for client-only UI state.
- expo-video for video playback. **expo-av was removed in SDK 55 — never
  import it.** Use expo-audio for audio.
- expo-image-picker for media selection, expo-image for image display.
- expo-sqlite for the local read cache, expo-secure-store for auth tokens.
- @shopify/flash-list v4 for every long list (feed, Explore, comments, members).
- react-native-reanimated v4 + react-native-gesture-handler for the vertical
  video feed.
- NativeWind v4 for styling, driven by the design tokens below.
- posthog-react-native for analytics.

TARGETS:
- Android (native) — the primary client.
- Web — same codebase via react-native-web. This is ALSO the iOS delivery
  vehicle (installable home-screen PWA); there is no native iOS build at MVP.
- iOS native is out of scope but must not be actively broken.

PROJECT STRUCTURE:
app/                    # Expo Router routes; groups for (auth) and (tabs)
  (tabs)/               # Home, Explore, Communities, Messages, Profile
  drama/[id]/           # index, cast, episodes
  episode/[id]/         # per-episode discussion thread
  post/[id]/
src/
  features/<name>/      # api.ts + hooks/ + components/ — one folder per feature
  components/           # shared UI kit
  lib/                  # supabase.ts, queryClient.ts, storage.ts, analytics.ts
  theme/                # tokens from the design system below
  types/                # generated database.ts + domain types
supabase/
  migrations/           # SQL files, version-controlled
  functions/            # Edge Functions
  seed.sql              # catalog seed

ARCHITECTURE RULES (enforce these in the code you generate):
- Screens compose; they never call supabase directly. All queries live in
  `src/features/<name>/api.ts`, exposed through hooks. A component importing
  supabase is a defect.
- Create the Supabase client exactly once, in `lib/supabase.ts`, with:
    - `import 'react-native-url-polyfill/auto'` at the top of that file
    - `Platform.OS !== 'web' ? { storage: <SecureStore adapter> } : {}`
    - `autoRefreshToken: true, persistSession: true,
       detectSessionInUrl: false, lock: processLock`
  Register the AppState listener (startAutoRefresh / stopAutoRefresh) once at
  the app root, guarded to non-web platforms.
- Wire `onAuthStateChange` once, at the root. Treat a restored session and a
  loaded user profile as two distinct states.
- Never reference the service_role key in client code. Anything privileged
  becomes an Edge Function.
- Read config from `process.env.EXPO_PUBLIC_SUPABASE_URL` and
  `process.env.EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY`.
- Platform differences go in `*.native.tsx` / `*.web.tsx` file pairs or
  Platform.OS branches — never in parallel app trees.

CORE DATA MODEL:
Users, Profiles, Follows (join tables — never ID arrays on a row), Dramas,
Actors, DramaCast, Episodes, Genres, DramaGenres, Aliases, ExternalIds,
Posts, Media, Comments, Likes, Communities, CommunityMembers, Conversations,
ConversationMembers, Messages, ChannelSubscriptions, Notifications,
PushTokens, Reports, ModerationActions.
- Every content row carries moderation_status and deleted_at (soft delete).
- Post and Comment carry is_spoiler.
- Episode.air_date is timestamptz (K-dramas air on KST for a global audience).
Full column-level DDL: [paste the schema from 09_database_schema.md here —
write that document first]

MVP FEATURES TO SCAFFOLD (routes + hooks + api modules + components; UI may be
placeholder-populated with mock data, but wired through the real hook
interfaces so the backend drops in later):
1.  Auth & profile
2.  Onboarding (pick genres / dramas / actors)
3.  Home feed (mixed algorithmic + follow-based, infinite scroll via FlashList)
4.  Explore — vertical short-form video feed (Reanimated paging, expo-video)
5.  Drama Hub (synopsis, cast, episode discussion threads)
6.  Communities (list, join, post)
7.  Messages — DMs (1:1, Supabase Realtime) AND Channels (one-way broadcast).
    These are DIFFERENT data models: DMs use conversations + messages with
    Realtime subscriptions; channels are single-writer broadcasts with a
    subscription table. Do not merge them into one "thread" abstraction.
8.  Post composer (text / image / video, taggable to drama, actor, community)
9.  Search (dramas, actors, users, communities, posts)
10. Notifications
11. Follow system (users, dramas, actors, creators, communities)

VIDEO — keep this simple, do not over-engineer it:
The pipeline is: pick with expo-image-picker (set `videoQuality` so iPhone HEVC
clips get re-encoded to H.264) → check duration against a 5-minute cap → upload
the ORIGINAL file to storage → store key, duration_s, poster_key,
moderation_status → play with expo-video.
There is NO transcoding, NO ffmpeg, NO HLS ladder, NO server-side video
processing. Generate poster frames client-side using expo-video's native
thumbnail generation.
Video POSTING is Android-only. On web, hide the video option in the composer and
show a short explainer; video PLAYBACK works on web.

WEB-SPECIFIC REQUIREMENTS:
- Use Expo Router static rendering so drama and episode routes ship as real,
  crawlable HTML, and implement `generateMetadata` on those routes (title,
  description, og:image from the drama poster). SEO is a core acquisition
  channel for this product.
- Web storage is disposable (Safari evicts after ~7 days idle). The local cache
  is a performance optimization only, never a source of truth. On web, use
  optimistic UI WITHOUT a persistent write queue; only Android gets an offline
  outbox.
- Include a PWA manifest with `display: "standalone"`, plus an install-prompt
  component that instructs Safari users to Add to Home Screen (required for Web
  Push).

NAVIGATION: bottom tabs — Home, Explore, Communities, Messages, Profile. Use
Expo Router groups: `(auth)` for sign-in/sign-up/onboarding, `(tabs)` for the
five main sections. Deep links and web URLs must resolve to the same routes.

DESIGN SYSTEM — apply these tokens consistently via src/theme, consumed by
NativeWind config:
[[DESIGN TOKENS FROM STITCH — paste colors, typography scale, corner radii,
spacing scale, and component styling notes here once Stitch has generated them]]

DESIGN DIRECTION: sleek, modern, minimal, confident — inspired by premium
media/streaming apps. Nothing flashy, neon-heavy, or childish. Favor generous
whitespace, restrained color, clean typography over decorative UI. Respect
system light/dark from day one.

DELIVERABLE: Generate the full project — app.json/app.config.ts, package.json
with pinned Expo SDK 57-compatible versions (use `npx expo install`, never
hand-pick React Native package versions), tsconfig.json, Tailwind/NativeWind
config, the Expo Router route tree, the lib/ layer including the Supabase
client exactly as specified above, the feature api.ts + hooks scaffolding with
mock implementations behind the same interfaces, and the screen components for
each MVP feature. Add a README covering: `npx expo start`, `npx expo start
--web`, `eas build -p android --profile preview`, `eas update`, and how to run
`supabase gen types typescript` to refresh src/types/database.ts.
Prioritize a codebase that starts on Android and web with mock data over
completeness of any single feature. Leave explicit TODOs where real Supabase
integration plugs in.
```

---

## Notes for whoever runs this

- **Run `npx expo install <pkg>`, not `npm install <pkg>`.** It resolves the
  version that matches your SDK. Hand-picked React Native package versions are
  the single most common source of a red screen on startup.
- **The prompt's mock-first framing is deliberate** and worth keeping: it gets
  you something running on a phone the same day, which matters when the Supabase
  schema doesn't exist yet.
- **Expect to iterate.** AI Studio will happily produce 40 files in one go and
  get the Realtime/WebSocket polyfill wrong. Ask for the scaffold first, verify
  it starts on both Android and web, *then* ask for features one at a time.
- **Pin the SDK.** If the model suggests `expo-av`, `expo-router` v2, or New
  Architecture opt-out flags, it is working from pre-SDK-55 knowledge. Correct
  it explicitly.
