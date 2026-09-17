# Hallyu — Privacy Policy (Draft Template)

*Draft placeholder — not legal advice. Structural starting point for legal
review, not a publishable document as-is. Fill in [bracketed] fields.*

## 1. What We Collect

| Category | Examples | Source |
|---|---|---|
| Account data | Email, auth provider ID | You, at signup |
| Profile data | Handle, display name, avatar, bio | You |
| Content | Posts, comments, videos, messages | You |
| Usage data | App interactions, feature usage | Automatically, via PostHog (pseudonymous, no advertising ID required) |
| Device data | Device type, OS version, push token | Automatically |

See `13_data_retention_and_privacy_handling.md` for the full internal data
inventory this is based on.

## 2. How We Use It

- To provide the Service (feed, communities, messaging, notifications)
- To personalize your experience (feed ranking based on your follows/genres)
- To maintain safety (content moderation, per
  `12_content_moderation_policy.md`)
- To understand product usage in aggregate (analytics)

We do not sell your personal data. [Update this section if/when
monetization involving advertising or data partnerships is introduced — this
would require updated disclosures, not a quiet amendment.]

## 3. Who We Share It With

- **Service providers**: Supabase (database, authentication, image storage);
  Cloudflare (web hosting and video storage/delivery); Expo (app updates and
  push delivery); PostHog (product analytics); content-moderation API providers
  for automated screening (see `12_content_moderation_policy.md`). Catalog
  metadata is sourced from TVmaze.
- **Legal requirements**: if required by law, subpoena, or to protect safety
- We do not share your content or data with advertisers or data brokers.

## 4. Your Rights

Depending on your location, you may have the right to:
- Access the personal data we hold about you
- Request correction or deletion of your data
- Export your data
- Object to certain processing

[This section needs jurisdiction-specific detail — GDPR (EU/UK), CCPA/CPRA
(California), and other regional frameworks each impose specific
requirements and required disclosures; confirm applicable ones with counsel.]

To exercise these rights: [support email/process]

## 5. Data Retention

See `13_data_retention_and_privacy_handling.md` for the full retention
schedule. In summary: account and content data is retained while your
account is active; deleted content and accounts are purged within a defined
window (30 days) except where retained for safety/legal purposes (e.g.,
moderation reports).

## 6. Children's Privacy

Hallyu is not directed at children under [13/16 — confirm with counsel based
on target markets and COPPA/GDPR-K requirements]. We do not knowingly
collect data from children under this age.

## 7. Security

We use industry-standard security practices: Postgres Row Level Security
policies that restrict every read and write to the rows you're permitted to see,
encrypted transport (TLS) and encrypted storage at rest, encrypted device
keychain storage for session tokens, and restricted internal access to sensitive
data (see `13_data_retention_and_privacy_handling.md`) — but no system is 100%
secure.

## 8. International Data Transfers

Your data is stored in the Supabase project region we select at setup, and
video files are distributed through Cloudflare's global network (anycast), which
means a video you upload may be cached and served from data centers in other
countries in order to play it quickly. Push notifications are relayed through
Expo's push service to your device's platform provider (Google's FCM on Android;
Apple's APNs if you use the iOS web app's push through a browser).

[**Decision still open:** pick the Supabase region before launch and state it
here explicitly. If you expect meaningful EU/UK traffic — and a K-drama fandom
app will have some — `eu-west-1`/`eu-west-2` gives you a straightforward GDPR
story. If your launch audience is West Africa and Southeast Asia, a closer
region (`eu-west-1` for Africa, `ap-southeast-1` for SEA) reduces latency
materially. This is a one-line setting at project creation and expensive to
change later.]

## 9. Changes to This Policy

We'll notify you in-app of material changes to this policy.

## 10. Contact

Questions about this policy: [support email/address]

---

**Disclaimer:** This is a structural draft, not legal advice. Privacy law
varies significantly by jurisdiction (GDPR, CCPA/CPRA, and others) and this
document must be reviewed and finalized by a licensed attorney before
publication.
