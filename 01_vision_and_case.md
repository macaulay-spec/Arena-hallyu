# Hallyu — Vision & The Case for Building It

## The problem

K-drama fandom is huge and growing, but it has no home. Right now a fan's
experience is scattered across:

- **X/Twitter** — real-time reactions, but not drama-native; a firehose with no structure
- **TikTok/Instagram** — great for short-form clips, but discovery is algorithmic noise, not fandom-organized
- **MyDramaList / DramaBeans / Viki forums** — good databases and recaps, but static, dated, not social-first
- **Reddit** — decent per-drama discussion threads, but clunky UX, no video-native experience, not built for this audience
- **Discord servers** — real community, but fragmented, invite-gated, and undiscoverable
- **Naver / Korean-native apps** — deep but language- and region-gated for global fans

No single product gives a fan: *discover a drama → watch clips → talk about it
with people who are actually watching the same episode this week → follow the
actors and creators → get a feed built around their fandom.* Hallyu's bet is
that the fandom deserves a purpose-built social home instead of five
half-fitting tools stitched together.

## Why now

- K-drama has moved from niche to mainstream global entertainment (Netflix,
  Viki, and others have made it a primary content category, not a side genre).
- The audience skews Gen Z / younger Millennial — a demographic that already
  behaves in short-form-video + community-feed patterns (TikTok, Discord,
  BeReal-style apps), so the UX model is proven, just not applied to this niche.
- Weekly episode-release cadence (most K-dramas air 2 episodes/week) creates a
  **built-in engagement rhythm** — similar to how sports apps ride game-day
  spikes. A drama-hub-centric app can turn "new episode Wednesday" into a
  recurring retention hook no generic platform is built to exploit.

## Core benefits of building Hallyu

1. **A single, fandom-native home** — discovery, discussion, short-form video,
   and messaging in one product instead of five.
2. **Structural retention hook** — episode-release cadence gives a natural
   reason to open the app weekly (arguably twice weekly) per active drama,
   which generic social apps don't replicate.
3. **Network effects around communities** — drama-specific and actor-specific
   communities compound: more fans → richer discussion → more reason for new
   fans to join that community specifically (not just the app generally).
4. **A fandom graph as a long-term asset** — even with no monetization now,
   what's being built is a dataset of who follows which dramas/actors/creators
   and how they engage. That's the foundation for personalization, and later,
   monetization (ads, creator tools, merch/ticketing) without having to
   retrofit it in.
5. **One codebase, two shipping surfaces** — building on **React Native + Expo**
   means the product logic and most of the UI is written once and shipped to
   Android (native) and Web (via `react-native-web`), instead of maintaining
   separate teams/codebases. The web build doubles as the iOS delivery vehicle —
   an installable home-screen app that reaches iPhone users for $0, with no
   Apple developer account. For a project without funding pressure, that's a
   large time-and-cost advantage. See `03_technical_architecture_expo.md`.

## Who this is for

- **The Superfan** — watches everything, wants deep discussion and to follow
  every cast update.
- **The Casual Binger** — wants good recommendations and light-touch community
  without commitment.
- **The Creator/Reactor** — wants an audience and a channel to broadcast to,
  not just a personal feed.
- **The Curious Newcomer** — discovered K-drama recently, wants an easy way in.

## Competitive read

Nobody currently owns "social home for K-drama fans" as a category. The risk
isn't a single dominant competitor — it's fragmentation across generic
platforms. The opportunity is being the first fandom-native, video-and-community
combined product in the space, built cleanly rather than bolted onto an
existing generic app.
