# Master Prompt — Paste into Google AI Studio (Gemini)

Fill in the `[[DESIGN TOKENS FROM STITCH]]` section below with the actual
colors/type/spacing Stitch produces, then paste the whole thing in.

---

```
You are acting as a senior Kotlin Multiplatform engineer. Build the initial
codebase scaffold for a social app called "Hallyu" — a mobile-first social
platform for K-drama fans, combining a personalized feed, short-form video
discovery, drama-specific communities, one-way broadcast channels, and DMs.

TARGETS: Android, iOS, and Web (Wasm), all sharing one Kotlin Multiplatform
codebase using Compose Multiplatform for UI.

ARCHITECTURE (follow strictly):
- Clean layered architecture: presentation (Compose UI + StateFlow-based
  view models) → domain (use cases, pure Kotlin) → data (repositories with
  remote + local data sources).
- Dependency injection via Koin.
- Networking via Ktor Client.
- Serialization via kotlinx.serialization.
- Local caching via SQLDelight.
- Async via Kotlin Coroutines/Flow throughout.
- Image loading via Coil 3.
- Backend: Supabase (Postgres, Auth, Realtime, Storage) accessed via Ktor.

MODULE STRUCTURE (create this Gradle module layout):
Hallyu/
├── shared/ (domain/, data/, di/)
├── composeApp/ (ui/feed, ui/explore, ui/dramahub, ui/communities,
  ui/messages, ui/profile, ui/onboarding, navigation/, theme/,
  androidMain/, iosMain/, wasmJsMain/)
├── androidApp/
├── iosApp/
├── webApp/
└── build-logic/

CORE DATA MODELS (define as Kotlin data classes in shared/domain/model):
User, Drama, Actor, Post, Community, Channel, Message/ChannelPost, Notification
— fields as follows: [paste the "Data model sketch" section from
03_technical_architecture_kmp.md here]

MVP FEATURES TO SCAFFOLD (screens + view models + use cases, UI can be
placeholder-populated with mock data for now, wired to real repository
interfaces so backend can be plugged in later):
1. Auth & profile
2. Onboarding (pick genres/dramas/actors)
3. Home feed (mixed algorithmic + follow-based)
4. Explore (vertical short-form video feed)
5. Drama Hub (synopsis, cast, episode discussion threads)
6. Communities (list, join, post)
7. Messages (DMs) + Channels (one-way broadcast, distinct data model from DMs)
8. Post composer (text/image/video, taggable to drama/actor/community)
9. Search (dramas, actors, users, communities, posts)
10. Notifications
11. Follow system (users, dramas, actors, creators, communities)

NAVIGATION: Bottom navigation with Home, Explore, Communities, Messages,
Profile. Use Voyager (or Compose Multiplatform Navigation) for screen
navigation, shared across all three platforms.

DESIGN SYSTEM — apply these tokens consistently across every screen via a
shared `theme/` package (Color.kt, Type.kt, Shape.kt):
[[DESIGN TOKENS FROM STITCH — paste colors, typography scale, corner radii,
spacing scale, and component styling notes here once Stitch has generated
them]]

DESIGN DIRECTION: sleek, modern, minimal, confident — inspired by premium
media/streaming apps. Avoid anything flashy, neon-heavy, or childish. Favor
generous whitespace, restrained color use, and clean typography over
decorative UI.

DELIVERABLE: Generate the full Gradle project structure, build files
(build.gradle.kts at root and per-module, settings.gradle.kts, version
catalog libs.versions.toml), the shared domain/data/di scaffolding with
interfaces and mock implementations, and Compose Multiplatform screen
composables for each MVP feature wired to view models exposing StateFlow.
Leave clear TODOs where the real Supabase backend integration should be
plugged in. Prioritize a codebase that compiles and runs on all three
targets with mock data over completeness of every feature.
```
