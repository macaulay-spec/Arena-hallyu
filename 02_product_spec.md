# Hallyu — Product Specification

## Personas

| Persona | Motivation | Key needs |
|---|---|---|
| **The Superfan** | Deep engagement, community status | Drama hubs, episode-thread discussion, follow actors/creators |
| **The Casual Binger** | Good recs, light engagement | Personalized feed, easy discovery, low-friction browsing |
| **The Creator/Reactor** | Build an audience | Post short-form video, broadcast channel, follower analytics (later) |
| **The Newcomer** | Find a way into the fandom | Onboarding by genre/drama, curated starter feed |

## MVP feature set

1. **Auth & Profile**
   - Sign up / log in, profile with avatar, bio, favorite dramas/actors shown on profile
2. **Onboarding**
   - Pick favorite genres, dramas, and actors → seeds the personalized feed
3. **Home Feed**
   - Algorithmic + follow-based mixed feed: posts (text/image/video), drama news, community activity
4. **Explore / Watch**
   - TikTok-style vertical short-form video feed of clips, fan edits, reactions
5. **Drama Hub pages**
   - Per-drama profile: synopsis, cast, episode list, per-episode discussion threads, ratings
6. **Communities**
   - Drama-specific and topic-specific communities (join, post, discuss) — Reddit-style structure
7. **Channels / Messaging**
   - 1:1 and group DMs, plus one-way broadcast **Channels** for creators/fan pages (WhatsApp Channels model)
8. **Post composer**
   - Text, image, and short video posting, taggable to a drama/actor/community
9. **Search & Discovery**
   - Search dramas, actors, users, communities, posts
10. **Notifications**
    - New episode alerts for followed dramas, replies, mentions, channel broadcasts
11. **Follow system**
    - Follow users, dramas, actors, creators, communities

### Explicitly out of MVP scope (Phase 2+)

- Live watch-parties / synced viewing
- Polls, badges, leaderboards, gamification
- Creator monetization tools, tipping, subscriptions
- ML-based recommendation engine (MVP uses simple rule-based personalization: followed tags/dramas weighted higher in feed ranking)
- Multi-language localization beyond English

## Information architecture (bottom navigation)

1. **Home** — personalized feed
2. **Explore** — short-form video discovery
3. **Communities** — browse/search drama & topic communities
4. **Messages** — DMs + Channels
5. **Profile** — own profile, settings, followed dramas/actors

Search is globally accessible from a top bar on Home/Explore/Communities.

## Core user flows

### Onboarding flow
Sign up → pick 3+ favorite genres → pick favorite dramas (searchable list) →
pick favorite actors (optional) → land on Home with a pre-seeded feed.

### Posting flow
Tap compose → choose type (text/image/video) → optionally tag a drama/actor/
community → write caption → post → appears in home feeds of followers and the
tagged drama hub/community.

### Drama engagement flow
Search or discover a drama → Drama Hub page → browse cast/synopsis → open an
episode discussion thread → post/reply → optionally join the drama's community
for persistent access.

### Channel broadcast flow
Creator opens their Channel → posts an update (text/image/video) → all
followers/subscribers of that channel receive it as a one-way broadcast (not a
group chat) → followers can react but replies go to the creator only, not the
whole channel.

## Success signals for MVP (qualitative, since no monetization yet)

- Weekly-active usage that spikes on known episode-release days (validates the
  drama-hub retention hypothesis)
- Community creation/joining behavior around individual dramas (validates the
  fandom-graph hypothesis)
- Ratio of feed engagement (likes/comments/shares) to short-form video views
