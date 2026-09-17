# Hallyu — Test Strategy

> **Rewritten 2026-09-17 for Expo / React Native.** The previous version was
> written for Kotlin Multiplatform (`commonTest`, Turbine, `StateFlow`, Compose
> UI tests) and for **Firebase Firestore Security Rules** — both wrong now. The
> backend is Supabase, so access control lives in **RLS policies**, and that
> shifts the centre of gravity of the whole test plan.

## The one thing that must not be under-tested

Hallyu's authorization model is almost entirely **Postgres Row Level Security**.
There is little server-side code between a user and a row — RLS *is* the
security boundary. A bug in a policy is not a failing test, it's a data leak or
a silently broken feed, and neither will show up in a Jest run.

**Treat RLS policy tests as production code.** Everything else in this document
is negotiable; this is not.

## Test pyramid

### 1. Unit tests — Jest + `jest-expo`

- **Preset:** `jest-expo` (multi-platform: run the same suite against
  `jest-expo/ios`, `jest-expo/android`, `jest-expo/web`).
- **Components & hooks:** `@testing-library/react-native`. Query by role and
  accessible text, never by testID-first — the web target needs real semantics
  anyway.
- **Feature `api.ts` modules:** mock `@supabase/supabase-js` with a small
  hand-written fake that records calls and returns fixtures. Test the query
  *shape* (filters, ordering, pagination cursor math) because that's where the
  feed logic actually lives.
- **Hooks:** `@tanstack/react-query` has a test utils package — use
  `QueryClientProvider` with a throwaway client. Test loading / error /
  optimistic-rollback paths explicitly; these are the states users see.
- **Pure logic** gets the most coverage because it's cheapest: duration-cap
  validation, spoiler collapsing, follow-graph merging, countdown/KST timezone
  math, feed de-duplication.

> `episode.air_date` is `timestamptz` and the audience is global. **Timezone
> math needs explicit tests with a fixed clock** — "next episode in 3 hours"
> being wrong by 9 hours is a churn event, not a nit.

### 2. RLS / database tests — local Supabase, non-negotiable

Run a real Postgres, not a mock. `supabase start` boots the full local stack
(Auth, PostgREST, Realtime, Storage) in Docker on any machine and in GitHub
Actions for free.

Two layers:

- **pgTAP / SQL tests** (`supabase test db`) for policy logic that is naturally
  expressed in SQL — constraints, triggers, the `deleted_at` soft-delete filter,
  episode air-date ordering.
- **Policy tests through the API** as *different identities*. This is the layer
  that catches real leaks. Mint JWTs for distinct test users against the local
  `JWT_SECRET`, then assert:

  | Assertion | Must be |
  |---|---|
  | Anonymous read of a public post | ✅ allowed |
  | Anonymous read of a DM message | ❌ denied |
  | Read a DM in a conversation you're not a member of | ❌ denied |
  | Read a community post from a private community you haven't joined | ❌ denied |
  | Update another user's `profiles` row | ❌ denied |
  | Insert a `follows` row with someone else's `follower_id` | ❌ denied |
  | Insert a `likes` row twice for the same post | ❌ blocked by unique constraint |
  | Read a `posts` row where `deleted_at` is set | ❌ filtered |
  | Write `moderation_status` directly as a user | ❌ denied — service role only |

  Write this suite as data-driven: a table of `{role, method, path, expect}`.
  Every new policy in `09_database_schema.md` gets rows added in the same PR.

**Storage bucket policies are RLS too.** Test that a user can only write under
their own prefix and cannot overwrite another user's object.

### 3. Edge Functions — Deno

- Unit-test pure logic (frame-sampling scheduler, moderation fan-out,
  notification payload shaping) with `Deno.test`.
- Supabase ships a test helper for mocking the Functions runtime; use it so
  function bodies stay testable without a live invocation.
- One **integration smoke test per function** against local Supabase, asserting
  the auth header path — an Edge Function that trusts an unverified JWT is a
  hole, and that class of bug only appears when you call it for real.

### 4. Realtime — test against a real instance

Do not mock Realtime. Spin up local Supabase, open two clients, and assert:

- a DM inserted by A appears for B within a bounded interval
- **B stops receiving after leaving the conversation** (channel cleanup leaks are
  the classic bug)
- the AppState listener path: background → foreground reconnects and resumes
  auto-refresh
- reconnection after a dropped socket (kill the container network briefly)

These are flaky by nature, so **quarantine them from the PR-blocking suite** and
run nightly.

### 5. E2E — Maestro (mobile) + Playwright (web)

- **Maestro** for Android flows: open-source CLI, YAML flows, no test-code
  language to maintain. Best effort-to-value ratio at $0.
- **Playwright** for web: free and open source, and it can run against the same
  exported static bundle you ship to Cloudflare Pages.

Critical flows only — about six, not sixty:

1. Sign up → onboarding (pick genres/dramas) → land on a populated feed
2. Watch a video in the Explore vertical feed, including the tap-to-play path on web
3. Create a text post tagged to a drama → it appears in the Drama Hub thread
4. Follow a user → their posts appear in Home
5. Send a DM → receive it on the second account
6. Join a community → post → see it in the community feed

Run in CI **nightly**, not per-PR. Per-PR E2E on a free GitHub Actions runner
will make the feedback loop slower than the code is worth.

### 6. Manual / exploratory QA

Each Phase (per `06_prd_and_roadmap.md`) gets a real-device pass before
dogfooding, focused on the two things automation can't see:

- **Video playback across surfaces** — the same clip on Android native, Chrome
  desktop, Safari desktop, Safari iOS home-screen. This is where the HEVC trap
  shows up: an iPhone-recorded clip that plays fine on your Mac and silently
  fails on a Windows Chrome browser.
- **Install and update path** — sideload the APK from a link, then ship an
  `eas update` and confirm it lands. On web, confirm Add to Home Screen →
  `display: standalone` → Web Push permission actually granted (if the manifest
  is wrong, `pushManager` is simply `undefined` and nothing tells you).

## Coverage targets (guideline, not gate)

| Layer | Target | Note |
|---|---|---|
| Pure logic (feed math, dates, caps) | high | cheapest coverage you'll get |
| Feature `api.ts` query builders | medium-high | catches wrong filters silently returning wrong data |
| Hooks: loading/error/rollback | medium | these are user-visible states |
| **RLS policies** | **exhaustive** | **every policy, both allow and deny** |
| Components | low | don't chase numbers; verify against Stitch mockups by eye |
| Screens (full render) | very low | E2E covers the flows that matter |

Gate the build on **RLS coverage** and on `supabase gen types typescript`
matching the checked-in `src/types/database.ts`. Don't gate on component
snapshot coverage.

## CI shape (GitHub Actions, public repo — free, macOS runners included)

Per PR: typecheck → lint → Jest → `supabase start` + RLS/pgTAP suite → Edge
Function unit tests. ~8–12 minutes; that's the budget, don't exceed it.

Nightly: Realtime suite, Maestro E2E on an Android emulator, Playwright on web,
one `eas build` to prove the release profile still compiles.

Per release: `eas update --branch preview` to a test channel and run the manual
QA checklist against the installed build.

## What NOT to build pre-launch

- **No load/performance infrastructure.** Supabase and Cloudflare handle MVP
  load; revisit only when real usage data says otherwise.
- **No exhaustive snapshot tests across three platforms.** The design system
  will churn faster than the snapshots. Snapshot the shared UI kit (buttons,
  cards, avatars) and nothing else.
- **No visual-regression pipeline.** Manual comparison against Stitch mockups is
  faster and cheaper at this team size.
- **No iOS simulator testing.** There is no iOS build at MVP. Keep `eas build
  -p ios` compiling in the nightly job so the path doesn't rot, but don't test
  against it.
- **Don't test the Supabase JS client itself.** Test what you *ask* it for, not
  that it asked correctly.
