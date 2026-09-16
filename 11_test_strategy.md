# Hallyu — Test Strategy

## Test pyramid

### 1. Unit tests (largest layer, fully shared via KMP `commonTest`)
- **Domain layer**: every use case (`GetPersonalizedFeedUseCase`,
  `JoinCommunityUseCase`, etc.) — pure Kotlin, no platform mocking needed,
  runs identically on all targets
- **Repositories**: tested against fake data sources (no real Firebase calls)
- **View models**: `StateFlow` emissions tested via `Turbine` given fake
  use case results

### 2. Integration tests
- Firestore Security Rules tested against the **Firebase Emulator Suite**
  using `@firebase/rules-unit-testing` — this is non-negotiable given how
  much access control lives in Rules rather than server code (see
  `08_api_contract.md`)
- Cloud Functions tested against the emulator for trigger behavior (e.g.
  `onPostCreate` correctly fans out to community feed index)

### 3. UI / instrumented tests
- Compose Multiplatform UI tests for critical flows only (not exhaustive
  screen coverage): onboarding completion, post creation, follow/unfollow,
  channel subscribe
- Run on Android emulator + iOS simulator in CI nightly (not every PR — too
  slow for the PR feedback loop)

### 4. Manual/exploratory QA
- Each Phase (per roadmap in `06_prd_and_roadmap.md`) gets a manual QA pass
  on real devices before internal dogfood, focused on: video playback across
  platforms (highest-risk area per `03_technical_architecture_kmp.md`), and
  real-time listener behavior under flaky network conditions

## Coverage targets (guideline, not gate)

- Domain/use case layer: aim for high coverage — this is cheap given it's
  pure Kotlin with no platform dependencies
- Data layer: cover repository logic, not the Firebase SDK itself
- UI layer: cover state-to-render logic in view models; don't chase coverage
  numbers on Composables themselves — visual correctness is better caught by
  the manual QA pass against Stitch mockups

## What NOT to test heavily pre-launch

- Don't build extensive load/performance testing infrastructure before
  there's real traffic — Firebase autoscaling handles MVP-stage load;
  revisit if/when usage data from `07_analytics_tracking_plan.md` shows
  approaching limits
- Don't write exhaustive UI snapshot tests across all three platforms before
  the design system stabilizes — snapshot churn will outpace design
  iteration speed early on
