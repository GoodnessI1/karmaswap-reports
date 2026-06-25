# KarmaSwap — Backend Consolidated Report

**Last updated:** June 25, 2026
**Stack:** Spring Boot 3.3.4 + JPA/Hibernate + JdbcTemplate + Spring Security + JWT + WebSocket/STOMP + PostgreSQL
**ddl-auto:** `update` — Hibernate auto-creates/alters columns on startup. No manual migrations needed for entity changes.

> **Two project copies exist:** `spring-karma-one` (dev) and `spring-karma-host` (production). Always keep them in sync. Editing the wrong one is a common mistake.

---

## Critical Rules — Read Before Touching Code

**Rule 1 — PostgreSQL UUID Binding**
Every UUID passed to JdbcTemplate must be `.toString()` and SQL must use `CAST(? AS uuid)`. No exceptions.
```java
// WRONG
jdbcTemplate.query(sql, mapper, someUUID);

// CORRECT
jdbcTemplate.query(sql, mapper, someUUID.toString());
// SQL: WHERE id = CAST(? AS uuid)
```
Never use `WHEN ? IS NOT NULL` in SQL. Handle null checks in Java and build different SQL strings instead.

**Rule 2 — Escrow Has Two Status Fields**
`Escrow` entity has both `status` (`EscrowStatus`) and `statuss` (`TransactionStatus`). `EscrowStatus` is decommissioned. Always use `getStatuss()` and `setStatuss()` with `TransactionStatus`. Never call `getStatus()` or `setStatus()` on `Escrow`. The double-s is known technical debt — do not rename it, too many callsites depend on it.

`TransactionStatus` values: `ESCROW`, `DISPATCHED`, `DELIVERED`, `COMPLETED`, `CANCELLED`, `ABANDONED`

> `DELIVERED` exists in the enum but is **not part of any active flow** under the confirmed Option A escrow model. `releaseExpiredEscrows()` queries `DISPATCHED`, not `DELIVERED`.

**Rule 3 — Two Item Mapping Methods**
- `mapItemRow(ResultSet rs)` — JdbcTemplate, returns full enriched response including `ownerLocation`, `isFollowing`, full image URL. Use for all marketplace and item detail queries.
- `mapToResponse(Item item)` — JPA, simpler, no `isFollowing` context. Use only for the user's own item list.
- `itemMapper.toItemResponse()` — **removed from the codebase.** The last call site (`updateItem()`) now routes through `ItemService.updateItem()`.

**Rule 4 — Two `ProfileService.mapToDTO()` Overloads**
- `mapToDTO(Profile profile)` — `isFollowing` defaults to false. Use for logged-in user's own profile.
- `mapToDTO(Profile profile, UUID requestingUserId)` — runs COUNT query, sets `isFollowing` correctly. Use for public profile views.
Using the wrong one silently returns `isFollowing: false` everywhere.

**Rule 5 — Activity Log Writes**
Always use `activityLogWriter.write(log)`, never `activityLogRepository.save()` directly. The writer is `@Async`.
```java
ActivityLog log = ActivityLog.builder()
    .userId(user.getId())
    .username(user.getUsername())
    .itemId(item.getId())           // nullable
    .activityType("TRADE_COMPLETED")
    .category("TRANSACTION")        // USER, LISTING, TRANSACTION, KARMA, ADMIN
    .actorId(actor.getId())
    .targetId(target.getId())       // nullable
    .description("human readable")
    .build();
activityLogWriter.write(log);
```

**Rule 6 — ItemCondition**
Always use `ItemCondition.fromString()`, never `ItemCondition.valueOf()` directly.

**Rule 7 — Lombok on Entities**
Never use `@Data` on JPA entities with lazy-loaded relations. Use `@Getter @Setter @NoArgsConstructor` instead. `@Data` on `Escrow` was the root cause of many `cannot find symbol getBuyer()` errors. If Maven fails with `cannot find symbol` for getters/setters but IntelliJ shows them fine — `annotationProcessorPaths` in `pom.xml` handles this, already configured, do not remove it.

**Rule 8 — SQL Comments in JdbcTemplate**
Never put SQL comments (`--`) inside JdbcTemplate query strings. They cause `PreparedStatementCallback` errors in PostgreSQL. Put them as Java comments above the string.

**Rule 9 — Transaction Rollback**
Escrow creation is `@Transactional`. If any step after `escrowRepository.save()` throws — including a notification insert — the entire transaction rolls back. Always ensure dependent tables exist before wiring new writes inside `@Transactional` methods.

**Rule 10 — Every karma balance update must be paired with `profileRepository.save()`**
This has been a recurring bug pattern across multiple methods. Never update a balance without immediately saving.

**Rule 11 — `@Lob` banned on String fields**
Use `@Column(columnDefinition = "TEXT")` instead. `@Lob` on a `String` field causes PostgreSQL/Hibernate to treat it as an `oid` LOB requiring an open transaction to stream.

**Rule 12 — Standard Escrow Query Methods**
- `findByParticipantWithDetails` — standard for any escrow list endpoint needing buyer/seller/item (FETCH JOIN, avoids N+1 and lazy loading)
- `findByIdWithParticipants` — standard for single escrow lookups needing participant data

**Rule 13 — JWT Subject is Email, Not UUID**
JWT subject is the user's **email address**. To get UUID from Principal:
```java
userRepository.findByEmail(principal.getName()).getId()
```
`Profile.id` always equals `User.id` via `@MapsId`.

---

## Trade Flow Reference

| Stage | Endpoint | Actor | Result |
|---|---|---|---|
| 1 | `POST /api/escrow/start/{itemId}` | Buyer | status: `ESCROW`, karma deducted, item `CLAIMED` |
| 2 | `POST /api/escrow/{id}/dispatch` | Seller | status: `DISPATCHED`, `releaseTime` set |
| 3 | Auto-cron (48hr after dispatch) | System | Auto-completes → `COMPLETED`, karma released |
| Confirm | `POST /api/escrow/{id}/complete` | Buyer | status: `COMPLETED`, karma released early |
| Acknowledge | `PATCH /api/escrow/{id}/acknowledge` | Either | Trade moves to History |
| Cancel | `POST /api/escrow/{id}/cancel` | Buyer or Seller | Only during `ESCROW` status — karma refunded, item restored |
| Abandon | Auto-cron (72hr of inactivity) | System | Karma refunded, item restored |

> **Option A confirmed:** Buyer goes `DISPATCHED → COMPLETED` directly via one "Confirm Received" button. `DELIVERED` is backend/cron-only and not part of the active flow.

---

## WebSocket Reference

| Endpoint | Protocol | Purpose |
|---|---|---|
| `http://localhost:8080/ws-chat` | SockJS — use `http://` not `ws://` | Buyer-seller chat per escrow |
| `http://localhost:8080/ws-escrow-updates` | SockJS | Escrow status updates |

- JWT in STOMP CONNECT headers as `Authorization: Bearer token` — **never as URL query param**
- Chat messages: `/topic/escrow/{escrowId}`
- Notifications: `/user/queue/notifications`
- Escrow updates: `/user/queue/updates`
- App prefix: `/app` (e.g. `/app/chat/{escrowId}`)

---

## Authentication

**What's built:**
- `POST /auth/login` and `POST /auth/google` return `{ token, id, username }`
- 401 with readable messages for bad email vs wrong password
- Suspended users blocked at login with 403 and suspension reason
- Google OAuth dual username bug fixed — `user.username` and `profile.username` now generated once and shared
- `GoogleIdTokenVerifier` converted to `@PostConstruct` singleton
- `USER_REGISTERED` / `USER_LOGIN` activity logs wired into both regular and Google auth flows (boolean flag used in Google auth to distinguish new vs returning users — lambdas can't reassign outer variables)

**Design decisions:**
- `login()` returns `User` not `String` — controller owns token generation, single DB call
- Auth response shape `{ token, id, username }` locked as the standard across both login methods

---

## Items

**What's built:**
- `ItemCondition.GOOD` hardcoding fixed in 4 places — all use `ItemCondition.fromString()`
- `ownerUsername`, `ownerLocation`, nested `owner` object with `isFollowing` added to `ItemResponse`
- `GET /items/{id}` rebuilt to use same enriched mapper as `GET /items`
- Weighted marketplace ranking — recency decay + reputation boost (capped 0.3) + condition boost + karma value nudge + randomness (0.25)
- Seller diversity pass — no seller appears more than twice consecutively
- PostgreSQL UUID binding fixed throughout
- `createItem()` built from scratch in `ItemService` — handles image storage, item save, first listing karma bonus, activity log. Controller delegates to this method.
- `claimItem()` dead code removed — overlapped with `createEscrow()`
- `ITEM_CLAIMED` activity log wired into `claimItem()`
- `DELISTED` added to `ItemStatus`
- `delistItem()` added to `ItemService`
- **Hardcoded `localhost` image URLs removed** — `ItemServiceImpl` had `BASE_IMAGE_URL = "http://localhost:8080/uploads/"` as a static constant; moved to `application.yml` as `app.base-url`, injected via `@Value("${app.base-url}")`. Deploying now only requires changing one config line, and every image URL across the app updates automatically.
- Fixed a malformed-URL bug introduced during the base URL refactor — some occurrences were written as `baseUrl + imageUrl`, missing the `/uploads/` segment (produced URLs like `http://localhost:8080avatar-123.jpg`). Corrected across `mapItemRow()`, `mapToResponse()`, and `mapToDTO()`.
- **`itemMapper.toItemResponse()` — fully removed from the codebase.** The one remaining call site (`ItemController.updateItem()`) was replaced by routing through the newly-built `ItemService.updateItem()`.
- **`ItemService.updateItem()` — built.** Handles ownership check (403 if not owner), partial field updates (all fields optional), image replacement with old file deletion, activity log write. Controller delegates cleanly, no business logic left in the controller.

**Design decisions:**
- `mapItemRow()` is the canonical mapper for marketplace and item detail
- `mapToResponse()` only for user's own item list
- Marketplace order intentionally varies slightly per load (randomness factor) — not a bug
- Base URL extracted to config, not hardcoded — `app.base-url` in `application.yml` is the single source of truth for the server URL used in image paths. No `localhost` strings remain hardcoded in service logic.

---

## Profile

**What's built:**
- `location` and `avatarUrl` fields added to `Profile` entity
- `RegisterRequest.setBio()` fixed — was always saving null
- `mapToDTO()` fully populated — `followerCount`, `followingCount`, `itemsListedCount`, `itemsClaimedCount`, `avatarUrl`, `location`, `trustBadge`
- `mapToDTO(profile, requestingUserId)` overload added for `isFollowing` on public profile views
- `GET /profile/{username}` passes requesting user context when authenticated
- `avatarUrl` now returns a full absolute URL, not just a filename — `mapToDTO()` was setting it directly from the DB value with no base URL prefix; fixed to prepend `baseUrl + "/uploads/"` (matches how `ItemServiceImpl` already handled image URLs)
- **`PUT /api/profile/me`** — confirmed fully built. Multipart endpoint, accepts `data` (JSON part) + optional `avatar` file
  - Username uniqueness check, syncs to both `Profile` and `User` entities
  - `fullName`, `bio`, `location`, `avatarUrl` all updatable
  - Returns full `ProfileDTO` immediately in the same response — no separate GET needed
  - Avatar stored via `fileStorageService.storeFile()`, old avatar deleted before new one is saved
- Duplicate `updateProfile(UUID userId, Profile updatedProfile)` method removed — old dead code that only updated username/bio and returned a raw `Profile` instead of `ProfileDTO`

**Design decisions:**
- `followerCount`/`followingCount` always live-queried, never stored/cached on `Profile`
- Trust badge tier computed on the fly via `getTrustBadge()` from numeric `trustScore` — never stored as a separate field (avoids drift risk)
- Two `mapToDTO()` overloads are intentional and must both stay (Rule 4) — using the wrong one silently returns `isFollowing: false` everywhere
- Profile update returns the full DTO immediately, including the updated `avatarUrl` as a full absolute URL — no second GET needed after `PUT /api/profile/me`
- Username change notification to followers deferred — fan-out to potentially hundreds of followers on a single change is disproportionate at this stage; revisit post-launch if users report confusion

---

## Social

**What's built:**
- Fixed `getUsersFollowedBy()`, `getUsersFollowing()`, `getFollowersFeed()` — all had wrong JOIN direction
- Fixed `mapRowToProfileDTO()` — was reading nonexistent `id` and `trust_badge` columns
- Dynamic mixed feed: 0 following=100% discovery, 1–5=30/70, 6–15=60/40, 16+=80/20, shortfall backfills to discovery
- Feed enriched with `location` and nested item fields via LEFT JOIN
- Feed filtered — login/system events excluded; `?type=` filter available
- All UUID params fixed with `CAST(? AS uuid)` across `SocialService`
- SQL comment inside JdbcTemplate string removed
- Suggestions `ORDER BY` fixed, limited to 3 with randomness `(follower_count * 0.7 + RANDOM() * 0.3)`
- `followerCount` subquery added to all three profile list queries
- Follow/unfollow response: `{ success, following, followerCount }`
- Self-follow returns 400 instead of silently no-op
- `followUser()` — added `userRepository.findById(followerId)` lookup; corrected all variable references (previously referenced non-existent variables)
- `NEW_FOLLOWER` notification wired into `followUser()`

**Design decisions:**
- Feed split ratios are fixed product decisions — do not change without approval
- Discovery uses higher randomness factor (0.4) than followed content (0.2)
- Suggestions capped at 3, with randomness for rotation
- Location-based suggestion filtering deferred — user location is free text, not reliable for filtering
- Unfollowed users move to discovery bucket next load rather than disappearing

---

## Leaderboard

**What's built:**
- `GET /social/traders/top` built from scratch
- Score formula: `(tradeCount * 10) + (karmaEarned * 0.5) + (trustScore * 2) + (avgRating * 20)`
- Logged-in user's rank always appended with `isCurrentUser: true` if outside top N
- SQL comment inside JdbcTemplate removed; UUID comparison in `buildLeaderboardEntry()` normalized
- `limit` query param, defaults to 10

**Design decisions:**
- Location excluded as a ranking factor — usable as optional filter param only
- Trade count primary, karma secondary, trust and reviews as boosts — perfect 5-star seller gets max +100, can't override trade count dominance

---

## Reviews

**What's built:**
- Full system: `Review.java`, `ReviewRepository.java`, `ReviewService.java`, `ReviewController.java`, `ReviewResponse.java`
- `POST /api/reviews/{escrowId}` — 5 validations: escrow exists, is buyer, is `COMPLETED`, no duplicate, rating 1–5
- `GET /api/reviews/seller/{userId}` — average rating + count
- `findByIdWithParticipants` added to `EscrowRepository` — avoids `LazyInitializationException`
- 5 karma bonus wired to reviewer on submission
- Activity log written on review submission
- Review score feeds into leaderboard formula and `EscrowResponse.sellerRating`

**Design decisions:**
- Only buyer reviews seller, never reverse
- Rating only, no text comment, 1–5 stars
- One review per escrow — enforced via DB-level unique constraint on `escrow_id`
- Restricted to `COMPLETED` escrows only

---

## Escrow

**What's built:**
- `EscrowStatus` decommissioned — `TransactionStatus`/`statuss` is sole source of truth
- `@Data` replaced with `@Getter @Setter @NoArgsConstructor` on `Escrow`
- `DELIVERED` and `ABANDONED` added to `TransactionStatus` enum
- `createEscrow()` marks item `CLAIMED` — race condition closed; buyer karma now persisted (`profileRepository.save()` added)
- `cancelTransaction()` — refunds karma to buyer, restores item; allows both buyer and seller to cancel but only during `ESCROW` status; notifies other party; `TRADE_CANCELLED` activity log added
- `processSellerDispatch()` — `releaseTime` and `disputeDeadline` now set **before** `escrowRepository.save()`; status guard added rejecting dispatch if not in `ESCROW`; seller karma persisted
- `releaseKarma()` — status check fixed to accept `DISPATCHED` and `DELIVERED`; seller karma persisted (`profileRepository.save()` added); `escrowCompleted` and `successfulTransactions` incremented for both parties; `totalKarmaEarned` updated
- `GET /escrow/activities` — root cause fixed (wrong JPA property resolution on `@ManyToOne` relations), now delegates to `escrowService.getUserActivities()` via `findByParticipantWithDetails`
- `GET /escrow/seller/active` — expanded to include `DISPATCHED` and `DELIVERED`, not just `ESCROW`
- `PATCH /escrow/{id}/acknowledge` — built; CORS fixed (`PATCH` added to `allowedMethods`)
- `GET /escrow/buyer/active` — built
- `GET /escrow/history` (combined buyer+seller) — built
- Active trades filter excludes acknowledged trades
- `EscrowResponse` enriched: `buyerId`, `sellerId`, `itemImageUrl`, owner info, `deliveryMethod`, `lockedKarma`, `escrowReleaseAt`, `sellerRating`, `disputeStatus`
- N+1 fixed via `findByParticipantWithDetails` and FETCH JOIN repository methods
- `EscrowAutoReleaseJob` — migrated to `TransactionStatus`; dispute check moved to top of loop before any state change; query corrected from `DELIVERED` to `DISPATCHED` with 48hr `releaseTime` condition; karma/item restore correct
- `markAbandonedEscrows()` cron — 72hr threshold, uses `TransactionStatus.ESCROW`
- `setStatuss()` self-assignment bug fixed
- `AdminService.suspendUser()` — cancels seller's active escrows, refunds buyers, restores items, notifies buyers, logs each cancellation; trust score penalty (-20) and `ACCOUNT_SUSPENDED` notification now fire **once per suspension** outside the escrow loop
- `@Lob` removed from `preShipmentPhoto` String field — replaced with `@Column(columnDefinition = "TEXT")`
- `disputeStatus` denormalized field added to `Escrow` entity and `EscrowResponse`

**Unread trade indicator:**
- `ChatReadStatus` entity built (`userId`, `escrowId`, `lastReadAt`, unique constraint)
- `GET /api/escrow/unread-summary` — returns `hasAnyUnread` + `unreadEscrowIds`
- `PATCH /api/chat/{escrowId}/mark-read` — updates `lastReadAt`

**Design decisions:**
- Option A confirmed — buyer goes `DISPATCHED → COMPLETED` directly, one "Confirm Received" button
- Auto-release window confirmed at 48 hours
- Abandoned threshold confirmed at 72 hours
- User suspension triggers automatic escrow cancellation for buyer protection
- `DELIVERED` status dormant under Option A — exists in enum only

---

## Chat

**What's built:**
- Sender verified against JWT — impersonation prevented
- SYSTEM check moved to top of `sendMessage()`
- Participant validation on send and history
- Broadcast sends persisted message (correct `id`/`timestamp`), not raw input
- `mapToDTO()` made public
- `GET /api/chat/{escrowId}/history` — built with participant-only access
- `/api/chat/upload` deferred — removed from `SecurityConfig` `permitAll`

**Design decisions:**
- Chat tied 1:1 to escrow, no standalone messaging
- JWT in STOMP CONNECT headers, never URL query param
- Use `http://` not `ws://` for SockJS

---

## Notifications

**What's built:**
- Generic `Notification.java` entity (replaces old escrow-specific one) — fields: `recipientId`, `type`, `title`, `message`, `referenceId`, `referenceType`, `read`, `createdAt`
- `NotificationRepository`, `NotificationDTO`, `NotificationService`, `NotificationController` — fully rewritten and confirmed working
- `notify()` implemented — pushes via `messagingTemplate.convertAndSendToUser` to `/user/queue/notifications`
- Ownership checks on acknowledge/delete added

**Wired call-sites:**
- `TRADE_STARTED` → `createEscrow()` (to seller)
- `TRADE_CANCELLED` → `cancelTransaction()` (to other party)
- `ACCOUNT_SUSPENDED` → `suspendUser()`
- `NEW_FOLLOWER` → `followUser()`
- `KARMA_GIFTED` → `KarmaService.transfer()` — variable references corrected (`receiver`, `giverUser.getUsername()`)

**Not yet wired:**
- `TRADE_DISPATCHED` → buyer (in `processSellerDispatch()`)
- `TRADE_COMPLETED` → seller (in `releaseKarma()`)
- `TRADE_ABANDONED` → buyer (in `markAbandonedEscrows()`)
- `NEW_MESSAGE` → other party (in `ChatController.sendMessage()`)
- All karma reward notifications — first listing bonus, first swap bonus, referral milestones (bonuses applied, no notifications sent yet — flagged twice)

**Notification click routing (planned, frontend to implement):**
- Trade events → Trade Detail page
- `NEW_MESSAGE` → Trade Detail scrolled to chat
- `NEW_FOLLOWER` → profile
- `KARMA_GIFTED` → karma history
- `ACCOUNT_SUSPENDED` → dedicated suspension page

---

## Karma Rewards System

**What's built:**
- First listing bonus — 50 karma, triggers in `createItem()` when item count == 1
- First swap bonus — 50 karma each for buyer and seller, triggers in `releaseKarma()` when `successfulTransactions == 1`
- Rating bonus — 5 karma for reviewer, triggers in `ReviewService.submitReview()`
- Trade count increment — `escrowCompleted` and `successfulTransactions` incremented for both parties in `releaseKarma()`
- `totalKarmaEarned` updated alongside `karmaBalance` on seller karma release
- `KARMA_GIFTED` activity log wired into `KarmaServiceImpl.transfer()`; missing `note` field fixed so gifted-karma sum queries work

**Referral system:**
- `referralCode` and `referredBy` fields added to `User` entity
- Referral code auto-generated on registration — 8-character uppercase UUID substring
- `referralCode` added to `RegisterRequest` with manual getter/setter
- `findByReferralCode()` added to `UserRepository`
- Referral handling wired into `AuthServiceImpl.register()` — looks up referrer, sets `referredBy`, grants 50 karma to referrer, sends notification

---

## Trust Score

**What's built:**
- `adjustTrustScore()` utility in `ProfileService` — enforces 0–100 cap on every call
- Badge tier computed on the fly via `getTrustBadge()` — not stored
- `-10` trust penalty wired into `FraudDetectionJob.flagIfNew()` — applies once per new flag, guarded by existing 24hr duplicate-flag check
- `-20` trust penalty wired into `AdminService.suspendUser()` — fires exactly once per suspension

**Not yet built:**
- `-5` trust penalty for consecutive buyer cancellations — deferred, depends on the cancellation-tracking feature which doesn't exist yet
- Full trust score calculation metrics — on hold, pending owner decision

**Design decisions:**
- Cancellation penalty (-5 trust) and karma deduction (5%) are additive, not exclusive — both will apply in the same transaction when the feature is built

---

## Dispute Resolution

**What's built:**
- `Dispute` entity — `escrowId`, `raisedBy`, `reason`, `description`, `evidenceImageUrls`, `status`, `resolution`, `createdAt`
- `DisputeRepository` — `existsByEscrowIdAndStatus`, `findByEscrowId`, `findByStatus()`
- `DisputeService.raiseDispute()` — validates participant, validates status is `DISPATCHED`/`DELIVERED`, prevents duplicate open disputes, stores evidence images, notifies all admins
- `storeBase64Image()` in `FileStorageService` — decodes base64, strips data URI prefix, writes to disk
- `POST /api/disputes/{escrowId}` — accepts `{ reason, description, evidenceImages: base64[] }`
- `DisputeService.resolveDispute()` — binary outcome only (`REFUND_BUYER` or `RELEASE_SELLER`), validates dispute is `OPEN`, moves karma accordingly, updates `Escrow.statuss` and `Escrow.disputeStatus`, notifies both parties, writes activity log
- `GET /api/disputes/open` — admin queue
- `POST /api/disputes/{disputeId}/resolve` — admin resolution endpoint
- `disputeStatus` denormalized on `Escrow` — all writes restricted to `DisputeService`

> ⚠️ Both admin dispute endpoints need `hasAnyRole("SUPER_ADMIN", "MODERATOR")` restriction in `SecurityConfig` — **not yet confirmed applied.**

**Design decisions:**
- `disputeStatus` denormalized on `Escrow` (Option B) — avoids N+1 on bulk escrow list endpoints
- Resolution is binary only — no split option (no business rule exists for split percentage)
- Dispute window is `DISPATCHED`/`DELIVERED` only — before dispatch, cancellation handles it; after `COMPLETED`, karma is irreversible
- Evidence images stored via base64 decode → file storage, not raw base64 in DB

---

## Admin

All admin routes isolated under `/admin/*`. Role-gated throughout.

**Entities and roles:**
- `AdminUser` entity, `AdminRole` enum: `SUPER_ADMIN`, `MODERATOR`, `OBSERVER`
- `SUPER_ADMIN` sees everything; `MODERATOR` sees most; `OBSERVER` read-only
- `CustomUserDetailsService` checks `admin_users` table and merges admin authority with base user authority
- `UserPrincipal` refactored to accept injected authorities
- `SecurityConfig` — `permitAll()` rules ordered before wildcard rules (was shadowing `/api/admin/bootstrap`)
- `AdminUser` field injection bug fixed — `User` entity was wrongly injected as a Spring bean via `@RequiredArgsConstructor`

**Endpoints built:**

| Endpoint | Access |
|---|---|
| `POST /api/admin/bootstrap` | One-time, always creates `SUPER_ADMIN` |
| `GET /api/admin/me` | Any admin |
| `GET /api/admin/overview` | Any admin |
| `GET /api/admin/today` | Any admin |
| `GET /api/admin/activity-log` | Any admin |
| `GET /api/admin/users` | Any admin |
| `GET /api/admin/users/registrations` | Any admin |
| `GET /api/admin/users/{id}` | Any admin |
| `GET /api/admin/users/{id}/karma-history` | Any admin |
| `POST /api/admin/users/{id}/suspend` | SUPER_ADMIN, MODERATOR |
| `GET /api/admin/escrows` | Any admin |
| `GET /api/admin/flags` | Any admin |
| `PATCH /api/admin/flags/{id}/review` | SUPER_ADMIN, MODERATOR |
| `GET /api/admin/karma-economy` | SUPER_ADMIN only |
| `GET /api/admin/karma-concentration` | SUPER_ADMIN only |
| `POST /api/admin/admins` | SUPER_ADMIN only |
| `PATCH /api/admin/admins/{id}/role` | SUPER_ADMIN only |
| `POST /api/admin/listings/{id}/delist` | SUPER_ADMIN, MODERATOR |

**Implementation notes:**
- Activity log filtering uses dynamic JdbcTemplate SQL — JPQL couldn't infer types for null filter parameters
- Karma history running balance computed via SQL window function `SUM() OVER` — never stored as a column
- Escrow monitor flag thresholds: GREEN <48hrs, AMBER 48–72hrs, RED >72hrs
- Bootstrap always hardcodes `SUPER_ADMIN` — no role choice option

**ActivityLog:**
- `category`, `actorId`, `targetId`, `metadata` fields added
- `escrowId` field added — used to deduplicate feed entries per escrow (one entry per escrow, latest wins, via `DISTINCT ON` subquery in both feed SQL queries)
- Logs wired across: login, register, delist, claim, karma gift, escrow events, review submission, `TRADE_CANCELLED`, admin activity

**Fraud detection:**
- `AdminFlag` entity + `AdminFlagRepository` + `FraudDetectionJob` (hourly)
- 4 rules built: `KARMA_FARMING`, `LISTING_FLOOD`, `INSTANT_CONFIRM`, `CIRCULAR_SWAP`
- Rule 5 (IP-based) — skipped, no IP capture exists
- 24hr deduplication window per user+flagType to avoid flag spam
- `-10` trust penalty on new flag via `flagIfNew()`

> ⚠️ `GET /admin/overview` — karma in circulation and completed trades not showing correctly. Not yet diagnosed — endpoint code not yet reviewed.

---

## Infrastructure

**What's built:**
- `last_seen_at` + `LastSeenInterceptor` + `WebMvcConfig` registration
- `ActivityLogWriter` — `@Async` wrapper, all log writes use it
- `GlobalExceptionHandler` — app-wide 400 on `IllegalArgumentException`
- Maven Lombok annotation processing fixed (`annotationProcessorPaths` in `pom.xml`)

---

## What Is Not Yet Done

### Immediate / Blocking

- `GET /admin/overview` — karma in circulation and completed trades not displaying correctly, undiagnosed
- Admin dispute endpoints — `SecurityConfig` role restriction not confirmed applied
- Referral block inside `createItem()` — still commented out, never reactivated after `referredBy` confirmed on `User`
- Notification wiring — `TRADE_DISPATCHED`, `TRADE_COMPLETED`, `TRADE_ABANDONED`, `NEW_MESSAGE`, all karma reward notifications still unwired

### Features Not Started

- `GET /api/locations` — static JSON file with Nigerian states/cities/popular places not sourced or built
- Search — `GET /api/items/search?q=` matching title and description not built
- Karma Bounty — entity, 6 endpoints, cron, activity log, all unstarted
- Karma Expiry — daily cron for 90-day inactivity not built; depends on confirming `last_seen_at` interceptor is sufficient
- `GET /admin/users/{id}/karma-history` — cumulative running balance query not built

### Trust Score

- Full calculation metrics — on hold pending owner decision
- Cancellation penalty (-5) — deferred until cancellation-tracking feature is built
- Caps (min 0, max 100) — `adjustTrustScore()` enforces this, but default value of 50 on entity not confirmed updated

### On Hold — Do Not Touch Without Lead Engineer

- `setStatuss()` naming on `Escrow` — do not rename
- Pre-shipment photo — still base64, needs retrofitting to proper file upload (same approach now used for disputes)
- Fraud Rule 5 (IP-based) — no IP capture
- Karma refund on admin-triggered cancellation — cron handles it, admin endpoint unconfirmed
- `GET /admin/activity-log` status — was "in progress by subordinate" per handover; current status unknown
- JWT secret hardcoded in `JwtUtil` — needs moving to `application.properties`
- No pagination on marketplace — hard capped at 100
- Reviews → `updateTrustScore()` integration — average rating not yet feeding into trust score
