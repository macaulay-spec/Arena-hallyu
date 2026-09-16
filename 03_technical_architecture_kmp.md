# Hallyu — Technical Architecture (Kotlin Multiplatform)

## Targets

- **Android** (Compose Multiplatform on Android)
- **iOS** (Compose Multiplatform rendering via Skia, thin SwiftUI shell for entry point)
- **Web** (Compose Multiplatform for Web, Wasm target)

One shared codebase for business logic and (almost) all UI, native shells only
where platform APIs demand it.

## Architecture style

Clean, layered architecture, shared across all platforms:

```
Presentation (Compose UI + ViewModel/StateFlow)
        ↓
Domain (Use Cases — plain Kotlin, no platform deps)
        ↓
Data (Repositories → Remote data source / Local data source)
```

- **Presentation**: Compose Multiplatform screens + `ViewModel`-equivalents
  exposing `StateFlow<UiState>`. No business logic in composables.
- **Domain**: Use case classes (e.g. `GetPersonalizedFeedUseCase`,
  `JoinCommunityUseCase`) — pure Kotlin, fully testable, fully shared.
- **Data**: Repository interfaces in `commonMain`, implementations combining a
  remote data source (API) and local data source (cache/offline).

## Tech stack

| Concern | Choice | Why |
|---|---|---|
| UI | Compose Multiplatform | Single UI codebase across Android/iOS/Web |
| Navigation | Voyager (or Compose Multiplatform Navigation) | Multiplatform-native, screen-based |
| DI | Koin | Lightweight, multiplatform-first, no codegen required |
| Networking | Ktor Client | Multiplatform HTTP client, same API surface everywhere |
| Serialization | kotlinx.serialization | Standard, multiplatform |
| Local DB / cache | SQLDelight | Type-safe SQL, multiplatform, offline-first support |
| Async | Kotlin Coroutines + Flow | Standard |
| Image loading | Coil 3 | Multiplatform support (Android/iOS/Web) |
| Backend | Supabase (Postgres + Auth + Realtime + Storage) | Fastest path to a working backend without a dedicated backend team; Realtime covers chat/channels; swappable later for a custom Ktor server if needed |
| Video storage/CDN | Supabase Storage → migrate to Cloudflare Stream/Mux if video volume grows | Keep MVP simple, avoid premature infra investment |

> Note on video on Web: short-form video playback on the Wasm/Web target is
> the one area needing early prototyping — Compose Multiplatform video
> support is younger there than on Android/iOS. Plan a fallback of an
> HTML5 `<video>` interop layer for the Web target if native playback support
> is insufficient at build time.

## Module structure

```
Hallyu/
├── shared/                      # commonMain business logic, shared everywhere
│   ├── domain/                  # use cases, models
│   ├── data/                    # repositories, remote/local data sources
│   └── di/                      # Koin modules
├── composeApp/                  # commonMain UI, shared across all platforms
│   ├── ui/
│   │   ├── feed/
│   │   ├── explore/
│   │   ├── dramahub/
│   │   ├── communities/
│   │   ├── messages/
│   │   ├── profile/
│   │   └── onboarding/
│   ├── navigation/
│   ├── theme/                   # design tokens (colors, type, shapes) — from Stitch output
│   └── androidMain / iosMain / wasmJsMain   # platform-specific actuals (camera, notifications, etc.)
├── androidApp/                  # thin Android entry point
├── iosApp/                      # thin iOS/SwiftUI entry point wrapping Compose
├── webApp/                      # thin Web entry point (Wasm)
└── build-logic/                 # shared Gradle convention plugins
```

## Data model sketch (core entities)

- `User` — id, handle, displayName, avatarUrl, bio, followedDramaIds, followedActorIds
- `Drama` — id, title, synopsis, posterUrl, castIds, episodeCount, airSchedule
- `Actor` — id, name, photoUrl, dramaIds
- `Post` — id, authorId, type (text/image/video), mediaUrl, caption, taggedDramaId, taggedCommunityId, createdAt
- `Community` — id, name, type (drama/topic), memberCount, description
- `Channel` — id, ownerId, name, subscriberCount (one-way broadcast, distinct from group chat)
- `Message` / `ChannelPost` — for DMs vs. channel broadcasts respectively
- `Notification` — id, userId, type, payload, read

## Build/handoff note

This document plus `02_product_spec.md` and the design tokens produced by
Stitch (see `05_stitch_uiux_prompt.md`) are the three inputs the AI Studio
prompt in `04_ai_studio_build_prompt.md` is built to consume.
