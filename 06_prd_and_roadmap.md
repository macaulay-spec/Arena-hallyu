# Hallyu — PRD & Roadmap

## 1. Auth & Profile

**User story:** As a user, I can create an account and build a profile that
reflects my fandom.

**Acceptance criteria:**
- Sign up via email or Google sign-in (Supabase Auth). *No Apple sign-in at MVP — there is no native iOS build and no Apple developer account.*
- Profile has: handle (unique), display name, avatar, bio (160 char max)
- Profile shows followed dramas/actors as visible chips/badges
- Editing profile reflects instantly across feed/comments (denormalized display name/avatar updated by a Postgres trigger on profile change)

## 2. Onboarding

**User story:** As a new user, I want the app to understand my taste
immediately so my first feed isn't empty or irrelevant.

**Acceptance criteria:**
- User must select ≥3 genres and ≥1 drama before reaching Home
- Actor selection is optional and skippable
- Selections write to `users/{uid}.followedDramaIds` / `followedGenres` before first feed load
- Home feed on first load contains only content matching onboarding selections (no cold-start generic content)

## 3. Home Feed

**User story:** As a user, I want a feed that mixes what I follow with
relevant discovery.

**Acceptance criteria:**
- Feed ranks: (a) posts from followed users/dramas/communities, (b) posts tagged to onboarding-selected dramas/genres, (c) trending posts as filler
- Infinite scroll, paginated (20 items/page)
- Pull-to-refresh re-ranks rather than just re-fetching page 1
- Empty state (no follows yet) never shown post-onboarding (see #2)

## 4. Explore (short-form video)

**Acceptance criteria:**
- Vertical swipe feed, autoplay on focus, mute by default with persistent unmute toggle
- Like/comment/share visible as overlay, non-blocking to video
- Video upload capped at **5 minutes** for MVP. Cap is enforced client-side at pick time *and* server-side before the row becomes visible — a client-only cap is bypassable.
- Video **posting is Android-only**; web can watch but not post video (see `16_zero_budget_stack.md` §4)

## 5. Drama Hub

**Acceptance criteria:**
- Each drama has: synopsis, poster, cast list (linked to actor profiles), episode list
- Each episode has its own discussion thread (not one thread per drama)
- "Currently airing" dramas show next-episode countdown
- Joining a drama's community is one tap from the Hub page

## 6. Communities

**Acceptance criteria:**
- Two types: drama-linked (auto-created per drama) and topic (user-creatable)
- Join/leave is instant, no approval step in MVP
- Community feed shows only posts tagged to that community

## 7. Messages & Channels

**Acceptance criteria:**
- DMs: 1:1 and group, real-time via Supabase Realtime (`postgres_changes` subscriptions on the messages table, scoped by conversation)
- Channels: one-way broadcast — subscribers cannot reply into the channel thread, only react; replies route to creator's DMs
- Channel data model is distinct from group-chat data model (no shared collection)

## 8. Post composer

**Acceptance criteria:**
- Supports text (500 char max), single image, or single video (≤5 minutes)
- Optional tagging to one drama, one actor, and/or one community
- Tagged content appears in the relevant Hub/Community feed within 5 seconds (real-time listener, not polling)

## 9. Search

**Acceptance criteria:**
- Tabbed results: Dramas / Actors / Users / Communities / Posts
- Debounced query (300ms), minimum 2 characters before firing

## 10. Notifications

**Acceptance criteria:**
- Triggered for: new episode of a followed drama, reply/mention, new channel broadcast from a followed channel
- Delivered via push when app backgrounded (Expo Push Service → FCM on Android; Web Push/VAPID on web), in-app banner when foregrounded
- Notification settings allow disabling each category independently

## 11. Follow system

**Acceptance criteria:**
- Follow/unfollow is a single tap, optimistic UI update (don't wait on network round-trip to reflect state)
- Follow counts denormalized on the profile row, updated by a Postgres trigger (not client-side increment, to avoid race conditions)

---

## Roadmap

### Phase 0 — Foundation (weeks 1–3)
- Project scaffold (**Expo SDK 57 app, expo-router route tree**, Supabase project + first migration, GitHub Actions pipeline)
- Supabase client wired correctly once (`react-native-url-polyfill`, SecureStore session, AppState auto-refresh) — see `03_technical_architecture_expo.md`
- **`09_database_schema.md` written and the RLS policy test suite standing up against local Supabase** (`11_test_strategy.md` §2)
- Design system finalized in Stitch, tokens implemented in `src/theme` via NativeWind
- Auth + profile working end-to-end on Android **and web**
- Web build exporting to Cloudflare Pages, PWA manifest with `display: standalone`

### Phase 1 — MVP core (weeks 4–9)
- Onboarding, Home Feed, Drama Hub, Communities, Post composer
- **Android + web parity** — web is the iOS delivery vehicle, so it ships alongside Android rather than trailing it (`16_zero_budget_stack.md` §4)
- Internal dogfood build, distributed by **direct APK link** + the Cloudflare Pages web URL (no store submission)
- EAS Update wired so dogfood fixes ship without a re-download

### Phase 2 — MVP complete (weeks 10–13)
- Explore (short-form video), Messages + Channels, Search, Notifications
- Web Push live (install-to-home-screen flow + VAPID) so iOS users get notifications
- Closed beta — recruited via shared link, install prompt on web, sideload on Android

### Phase 3 — Post-MVP (weeks 14+)
- Recommendation ranking beyond rule-based
- Live watch-parties, polls
- Creator tools, monetization exploration
- Public launch readiness (moderation tooling hardened, ToS/Privacy finalized — see other docs in this pack)

## Analytics note

See `07_analytics_tracking_plan.md` for the event schema this roadmap assumes
is instrumented from Phase 1 onward — instrumenting late is expensive to
retrofit. **That document does not exist yet.** `16_zero_budget_stack.md` §2.2
settles the tool (PostHog, free to 1M events/month — events must *not* go into
Postgres); the event schema itself is still unwritten.
