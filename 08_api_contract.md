# Hallyu — API Contract

Firebase/Firestore doesn't use a REST/OpenAPI contract the way a custom
backend would — clients talk to Firestore directly via SDK, secured by
Security Rules, with Cloud Functions covering anything requiring server-side
logic (denormalization, fan-out, validation beyond what Rules can express).
This doc defines that contract so it's explicit rather than implicit in
client code.

## Firestore collections (client read/write surface)

| Collection | Client can read | Client can write | Notes |
|---|---|---|---|
| `users/{uid}` | Own doc: full. Others: public fields only | Own doc only, subset of fields | `followerCount`/`followingCount` are Cloud-Function-only writes |
| `dramas/{dramaId}` | All, public | None (admin/CMS only) | Seeded/managed by content team, not user-writable |
| `actors/{actorId}` | All, public | None (admin/CMS only) | Same as dramas |
| `posts/{postId}` | All, public | Create: own posts only. Update/delete: own posts only | `likeCount`/`commentCount` are Cloud-Function-only |
| `posts/{postId}/comments/{commentId}` | All, public | Create: authenticated users. Delete: own comment or post author | — |
| `communities/{communityId}` | All, public | Create: authenticated users (topic communities only — drama communities are auto-created) | — |
| `communities/{communityId}/members/{uid}` | Own membership doc | Own membership doc (join/leave) | Drives `memberCount` via Cloud Function trigger |
| `channels/{channelId}` | All, public | Create: none in MVP (creator program gated) | Broadcast content is a subcollection |
| `channels/{channelId}/broadcasts/{broadcastId}` | Subscribers only | Owner only | Enforced via Security Rules checking subscription doc |
| `conversations/{conversationId}` | Participants only | Participants only | 1:1 and group DMs |
| `conversations/{conversationId}/messages/{messageId}` | Participants only | Participants only | Real-time listener-driven |
| `notifications/{uid}/items/{notificationId}` | Own only | None (Cloud-Function-written only) | Client can only mark as read (update `read` field) |

## Cloud Functions (server-side operations)

| Function | Trigger | Purpose |
|---|---|---|
| `onFollowCreate` / `onFollowDelete` | Firestore trigger on `follows/{id}` | Increment/decrement denormalized follower/following counts on `users` docs |
| `onPostCreate` | Firestore trigger on `posts/{id}` | Fan out to tagged drama/community feed indexes; send notifications to followers |
| `onCommunityMemberChange` | Firestore trigger | Update `memberCount` on `communities` doc |
| `onEpisodeAirDate` | Scheduled function | Push `notification` docs to users following a drama when a new episode airs |
| `onChannelBroadcast` | Firestore trigger on `channels/{id}/broadcasts/{id}` | Fan out FCM push to subscribers |
| `moderateContentOnCreate` | Firestore trigger on `posts`/`comments` create | Run automated moderation check (see `10_content_moderation_policy.md`) before content is marked visible |

## Security Rules principles

- No client ever writes a denormalized count field directly — always via
  Cloud Function using the Admin SDK (bypasses Rules by design, but only
  reachable through the trigger, not client-callable)
- Channel broadcast content is gated on an active subscription doc existing
  — checked in Rules, not just in UI
- All writes require `request.auth.uid` to match the resource's owner field
  except the explicitly server-only fields listed above

## Versioning approach

Since there's no traditional REST versioning here, schema changes are
managed via: (1) additive-only fields where possible, (2) a `schemaVersion`
field on documents where breaking changes are unavoidable, read by clients to
branch parsing logic during migration windows.
