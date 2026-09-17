# Hallyu — Content Moderation Policy

This is a working policy draft to build the moderation *system* around, not a
finished legal document — have it reviewed by counsel before publishing
externally (see disclaimer at the bottom).

## Why this matters before launch, not after

Hallyu has posts, comments, short-form video, and DMs/channels — every
surface where users generate content needs a moderation plan, because
retrofitting moderation onto a live UGC product with an existing user base is
far harder than building it in from Phase 1.

## Moderation layers

### 1. Automated pre-screen (on create, before content is publicly visible)
- Cloud Function (`moderateContentOnCreate`, per `08_api_contract.md`) runs
  on every `post`/`comment`/`broadcast` create
- Text: run through a moderation API (Google Cloud's Perspective API or
  similar) checking for harassment, hate speech, spam, and explicit content
- Images/video: run through an image-moderation API (Google Cloud Vision
  SafeSearch, or similar) checking for explicit/violent content
- Content flagged by automated screening is set to `moderationStatus:
  "pending"` (not immediately visible) and queued for human review, not
  auto-published and not auto-rejected — false positives shouldn't silently
  remove legitimate fan content

### 2. User reporting
- Report action available on every post, comment, video, profile, and
  channel broadcast
- Report reasons (fixed list, not free text, to keep triage fast):
  harassment/bullying, spam, hate speech, sexual content, spoilers posted
  without warning (drama-fandom-specific category), impersonation, other
- Reports write to a `reports/{reportId}` collection with content reference,
  reporter ID, reason, status — separate from the content's own document

### 3. Human review queue
- MVP-stage: a small moderation team (even 1–2 people initially) reviews the
  `pending` queue and escalated reports through an internal admin
  view/dashboard — doesn't need to be user-facing, can be a simple internal
  tool reading directly from Postgres with the **service-role key** (server-side
  only; that key must never reach a client bundle). A Supabase Studio SQL view
  over `posts`/`comments`/`reports` filtered to `moderation_status = 'pending'`
  is enough for the first months
- SLA target for MVP: review within 24 hours; tighten as volume grows

### 4. Enforcement actions (graduated)
1. Content removal (sets `moderation_status = 'removed'`, not hard-deleted —
   retained internally for appeals/audit per
   `13_data_retention_and_privacy_handling.md`)
2. Warning to user
3. Temporary posting restriction (read-only mode)
4. Account suspension
5. Account ban (severe/repeated violations)

### 5. Appeals
- Any enforcement action must have a stated reason visible to the affected
  user, plus a way to appeal (even a simple support-email flow at MVP stage)

## Fandom-specific consideration: spoilers

Given the drama-hub/episode-thread structure, spoiler-tagging is a real
community-health issue, not just a generic content-safety one. Recommend:
- Post composer includes an optional "contains spoilers" toggle when tagging
  a drama, which blurs media/truncates text until tapped
- Episode discussion threads are implicitly spoiler-permissive for that
  episode's content (users entering a thread for Episode 5 are opting into
  Episode 5 spoilers), but the general Home Feed is not

## Non-negotiable baseline (regardless of resourcing)

- No tolerance for CSAM, credible threats of violence, or doxxing — these
  get immediate removal and reporting to relevant authorities where legally
  required, with no "pending" queue step
- Harassment and hate speech policies apply equally to DMs/channels, not just
  public posts, even though DMs are private — private doesn't mean unmoderated
  when reported

---

**Disclaimer:** This is an operational draft to guide product/engineering
work, not a legal document. Have a lawyer review the final moderation policy,
especially around legal reporting obligations (e.g., CSAM reporting
requirements vary by jurisdiction) before public launch.
