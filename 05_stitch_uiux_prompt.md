# UI/UX Prompt Pack — Paste into Stitch (Complete)

This replaces the earlier draft, which only covered 10 sample screens and a
thin design system. This version covers brand identity, every auth screen,
every core screen, every edge/empty/error state, and every settings/modal
screen — the full product, not a sample of it.

Run order: **Part 0 → Part 1 → Part 2**, in that sequence. Each screen in
Part 2 should be generated *after* the design system in Part 1 exists, so
Stitch has real tokens to stay consistent against instead of improvising per
screen.

---

## Part 0 — Brand Identity (logo & app icon)

```
Design a brand identity for "Hallyu," a premium social app for K-drama fans.
The brand should feel sophisticated and editorial — closer to a high-end
streaming service or a cultural media brand than a typical fan-app or K-pop
aesthetic. Avoid: generic heart/star icons, bright bubblegum pink, glitter/
sparkle motifs, anime-style mascots, or anything that reads as a fan-made
app rather than a professional product.

Deliverables:
1. A wordmark treatment of "Hallyu" — clean, modern typography, confident
   but understated. Consider a custom letterform detail on one character
   (e.g. the "y" or the double "l") as a subtle signature mark, without
   tipping into decorative/playful territory.
2. An app icon / logomark that works at 3 sizes (iOS rounded-square icon,
   Android adaptive icon with safe-zone padding, browser favicon at 16px)
   — must remain legible and distinct at favicon size.
3. A monochrome (single-color) version of the logomark for use on dark
   surfaces, splash screens, and watermarks.
4. Confirm how the logomark and wordmark combine (lockup) for use in the
   splash screen vs. how the icon alone is used in the app header/nav.
```

## Part 1 — Full Design System

```
Using the Hallyu brand identity above, design a complete design system.
Sophisticated, editorial, premium — think high-end streaming/media app, not
a fan-app. No neon, no excessive rounding, no playful iconography.

1. COLOR
   - Primary: one deep, sophisticated base color (deep plum, ink navy, or
     warm charcoal — not bright red/pink K-pop cliché)
   - Accent: one color used sparingly, only for primary CTAs, likes, and
     the "live/currently airing" indicator
   - Full neutral/grayscale scale for backgrounds, surfaces, borders, and
     text at multiple emphasis levels (primary text, secondary text,
     disabled text)
   - Semantic colors: success, error/destructive, warning, info
   - Full light AND dark theme variants for every color above — dark theme
     is not just "invert," it needs its own contrast-checked palette
   - A distinct treatment for "spoiler-hidden" content (blur/overlay color)
   - A distinct "verified/official account" badge color

2. TYPOGRAPHY
   - One primary sans-serif for UI text with a full scale: display,
     headline, title, body (regular + emphasis), caption, label/overline
   - Confirm line-height and letter-spacing per size step, not just size

3. SPACING & SHAPE
   - 4/8pt-based spacing scale
   - Corner-radius scale for cards, buttons, inputs, sheets/modals — soft
     but restrained, not bubbly

4. ELEVATION & SURFACES
   - Shadow/elevation scale for cards, floating action buttons, and
     modals/sheets — define whether the app uses shadow-based elevation or
     flat surfaces with border/color separation, and apply it consistently

5. ICONOGRAPHY
   - Define icon style: outline vs. filled, stroke weight, corner style —
     pick one system and apply it everywhere (tab bar, actions, inputs)

6. MOTION PRINCIPLES (described, even if Stitch output is static)
   - Define feel for transitions (e.g. subtle fade+slide, no bouncy/
     playful easing), used later as a spec for the engineering handoff

7. IMAGERY TREATMENT
   - A consistent treatment for drama posters and cast photos, which will
     arrive as inconsistent source images — define a duotone/overlay
     treatment or consistent aspect-ratio crop so the visual language
     stays unified regardless of source image quality
   - Video thumbnail treatment (play-button style, duration badge style)

8. EMPTY / LOADING STATE STYLE
   - Define a consistent illustration style (or none — could be
     typography-led instead) for empty states
   - Define the loading-skeleton visual style (shimmer, pulse, static
     gray blocks) used across all list/card loading states

Output every token in a structured, reusable format: exact hex values, type
scale in sp/pt with line-height, spacing values, radius values, shadow
specs.
```

## Part 2 — Complete Screen Inventory

Run this per screen, substituting `[SCREEN NAME]` and using the notes given:

```
Using the Hallyu brand identity and design system above, design this
screen: [SCREEN NAME]. Maintain strict visual consistency with the
established colors, typography, spacing, iconography, and imagery
treatment. [Additional notes for this screen, if any.]
```

### A. Brand & Launch
1. App icon — final export at all required platform sizes
2. Splash / launch screen (logo lockup, brief load state)

### B. Auth flow
3. Welcome / pre-auth landing screen (logo, tagline, Sign Up / Log In CTAs)
4. Sign up (email + password fields)
5. Log in (email + password fields)
6. Social sign-in screen (Google / Apple buttons, "or continue with email")
7. Forgot password (email entry)
8. Check-your-email confirmation screen (post forgot-password / post signup)
9. Reset password (new password entry)
10. Email verification prompt (with resend action)

### C. Onboarding flow
11. Genre selection (multi-select grid, min-3 requirement visible)
12. Favorite drama selection (searchable, selectable grid with posters)
13. Favorite actor selection (optional, skippable — skip affordance must be visually secondary, not hidden)
14. Notification permission prompt (native-style pre-permission explainer screen)

### D. Home / Feed
15. Home Feed — populated, mixed content types (text/image/video post cards)
16. Home Feed — empty state (shouldn't normally occur post-onboarding, but design for edge case)
17. Home Feed — loading skeleton
18. Post detail — expanded view with full comment thread

### E. Explore / Watch
19. Explore — vertical short-form video feed (TikTok-style, overlay UI for like/comment/share)
20. Video detail / comments overlay (expanded comment sheet over paused video)

### F. Drama Hub
21. Drama Hub — Overview tab (synopsis, poster, next-episode countdown)
22. Drama Hub — Cast tab (grid of actor cards)
23. Drama Hub — Episodes tab (episode list with air dates)
24. Episode discussion thread (spoiler-permissive context banner)

### G. Actor
25. Actor profile page (photo, bio, linked dramas)

### H. Communities
26. Communities — discover/list screen (drama-linked + topic communities)
27. Community detail (feed + member count + join/leave state)
28. Create community (topic community creation form)

### I. Messages
29. DM inbox (conversation list)
30. DM thread (message bubbles, input bar)
31. Channels list (subscribed broadcast channels)
32. Channel detail (one-way broadcast feed, reaction-only affordance — visually distinct from a DM thread so users don't expect to reply inline)
33. New message / new group composer (contact picker)

### J. Post composer
34. Composer — entry/type picker (text / image / video)
35. Composer — text post screen
36. Composer — image picker + crop screen
37. Composer — video picker + trim screen (60s cap indicator)
38. Composer — tagging screen (tag drama/actor/community, spoiler toggle)

### K. Search
39. Search — empty state with suggestions/trending
40. Search — results screen (tabbed: Dramas / Actors / Users / Communities / Posts)

### L. Profile
41. Own profile (posts grid/list, followed dramas, edit-profile entry)
42. Other user's profile (follow/message actions, no edit access)
43. Edit profile (avatar, display name, bio, handle)
44. Followers / following list

### M. Notifications
45. Notifications feed (grouped by type: episodes, replies, mentions, channel broadcasts)

### N. Settings
46. Settings — main menu
47. Account settings (email, password change, delete account entry)
48. Notification preferences (per-category toggles, matching `06_prd_and_roadmap.md` §10)
49. Privacy settings (blocked users list entry, data/export entry)
50. About / legal (links to Terms of Service and Privacy Policy)

### O. System states & modals
51. Report content modal (fixed-reason list, per `12_content_moderation_policy.md`)
52. Block user / delete-confirmation modal (generic destructive-action pattern)
53. Generic error state (used app-wide: failed load, no connection to a specific resource)
54. Offline state (banner or full-screen, for lost connectivity mid-session)
55. Generic loading skeleton pattern (reusable list/card skeleton, referenced by #17 but defined once as a system component)

---

## Practical notes for running this in Stitch

- If Stitch supports generating a flow/batch at once, group by section
  letter (A through O above) rather than running all 55 one at a time —
  it'll also help consistency within a flow.
- Screens 51–55 (system states) are easy to skip because they feel like an
  afterthought, but they're what makes the app feel finished rather than a
  prototype — don't leave them for "later."
- Export every screen at 2x/3x resolution plus the raw Figma/code export if
  Stitch offers one — that's what feeds the Figma connector and, downstream,
  the AI Studio build prompt.
- As you generate each section, sanity-check it against the design system
  in Part 1 before moving to the next section — catching drift every 5–8
  screens is much cheaper than catching it at screen 50.
