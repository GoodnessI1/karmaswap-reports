# KarmaSwap — Central Consolidated Report

**Last updated:** June 2026
**Overall progress:** ~78%

| Area | Progress |
|---|---|
| Core trading loop | 90% |
| Karma economy | 70% |
| Social features | 75% |
| Notifications & trades UI | 65% |
| User profiles | 50% |
| Admin dashboard | 60% |
| Karma Bounties | 5% |
| Search | 40% |

---

## What KarmaSwap Is

A Nigerian cashless barter marketplace. Users trade pre-owned goods using a points currency called Karma instead of real money. Built with Spring Boot (backend) and React (frontend).

**Core loop:** User lists item → another user claims it → Karma is locked in a Karma Hold → seller dispatches → buyer confirms receipt → Karma releases to seller (minus 5% platform fee).

---

## Tech Stack

- **Backend:** Spring Boot 3.3.4, PostgreSQL, JdbcTemplate + JPA/Hibernate, WebSocket/STOMP
- **Frontend:** React 18, React Router v6, Axios, lucide-react, plain CSS
- **No Git** — code transferred via flash drive between developers
- **Deployment:** Not yet live, running locally

---

## Team Structure

- **Lead backend engineer** — core features, escrow, auth, social, reviews
- **Subordinate backend engineer** — built the entire admin system
- **Frontend engineer** — all UI including My Trades, TradeDetail, Social, Admin dashboard
- **Product owner** — founder, non-technical, receives plain English documents for review

---

## Karma & Economy

### Karma-to-Naira Relationship
Pricing logic anchors Karma to real-world Naira value internally. Seller inputs what the item costs in the real world; the system converts to a suggested Karma value using a fixed exchange rate. Users never see Naira mentioned anywhere on the platform.

### Karma Pricing Formula
Base Karma value = estimated Naira value ÷ 10. Condition multipliers: NEW = 100%, LIKE_NEW = 85%, GOOD = 70%, FAIR = 50%, POOR = 30%. AI suggests the result; user can accept or override.

### Platform Fee
5% deducted at Karma release. Seller receives 95%. Deducted automatically — no action from either party. Example: 200 Karma locked → seller receives 190, platform keeps 10.

### Trust Score
- Starts at 50 (neutral) for every new user. Capped 0–100.
- `adjustTrustScore()` enforces the cap on every call.
- Badge tier computed on the fly — never stored separately.
- **Proposed movements (pending owner sign-off):** completed trade +2, 5-star review received +3, 4-star review +1, 3-star or below -2, buyer cancellation penalty -5, fraud flag -10, account suspended -20.
- Full calculation metrics not yet confirmed by owner.

### Buyer Cancellation Penalty
Fires on the 3rd consecutive cancellation and every consecutive one after. Penalty: 5% of Karma locked in that trade deducted from buyer's balance + trust score decrease. Any completed trade resets the consecutive count to zero. Buyer notified each time penalty fires — seller gets standard cancellation notification only, no mention of penalty.

> ⚠️ Not yet built — depends on consecutive cancellation tracking which does not exist in the codebase yet.

### Karma Expiry
After 90 days of inactivity (no login, listing, or trading), user's full Karma balance is zeroed. A `KARMA_EXPIRED` transaction is written to their history. User is notified with explanation. Any platform activity resets the clock.

> ⚠️ Not yet built. Depends on confirming `last_seen_at` interceptor is sufficient as the activity signal.

### Karma Rewards (Built)
- Welcome/first listing bonus: 50 Karma when user lists their first item
- First swap bonus: 50 Karma each for buyer and seller on first completed trade
- Rating bonus: 5 Karma for reviewer on submission
- Daily login streaks: built

### Referral System
- Referral codes are **system-generated** on registration — users cannot create their own
- Give 50 / Get 50 on signup — both referrer and new user receive 50 Karma
- Referrer gets +25 when friend lists their first item
- Referrer gets +25 when friend completes their first swap

> ⚠️ Partially built. Signup bonus works. The +25 on friend's first listing (commented out) and +25 on friend's first swap are not yet wired.

### Karma Gifting
Users can transfer Karma to other users. `KARMA_GIFTED` notification sent to receiver. Activity log written on transfer.

### Cash-to-Karma Purchase
Discussed, kept on the table. Not confirmed for MVP.

---

## Escrow & Trade Flow

### Confirmed Flow (Option A)
| Stage | Endpoint | Actor | Result |
|---|---|---|---|
| 1 | `POST /api/escrow/start/{itemId}` | Buyer | Status: `ESCROW`, Karma deducted, item `CLAIMED` |
| 2 | `POST /api/escrow/{id}/dispatch` | Seller | Status: `DISPATCHED`, 48hr release timer starts |
| 3 | Auto-cron (48hr after dispatch) | System | Auto-completes → `COMPLETED`, Karma released |
| Confirm | `POST /api/escrow/{id}/complete` | Buyer | Status: `COMPLETED`, Karma released early |
| Acknowledge | `PATCH /api/escrow/{id}/acknowledge` | Either | Trade moves to History |
| Cancel | `POST /api/escrow/{id}/cancel` | Buyer or Seller | Only during `ESCROW` — Karma refunded, item restored |
| Abandon | Auto-cron (72hr of inactivity) | System | Karma refunded, item restored |

`DELIVERED` exists in the `TransactionStatus` enum but is **not part of the active flow**. Nothing sets it. The auto-release cron queries `DISPATCHED` trades older than 48 hours.

### Key Escrow Decisions
- **Auto-release window:** 48 hours after dispatch
- **Auto-abandonment:** 72 hours of inactivity after Karma Hold creation
- **Cancellation before dispatch:** either party can cancel, full Karma refund
- **Dispute window:** only valid after seller marks dispatched and before buyer confirms. 48hr auto-release clock pauses while dispute is open
- **Suspension triggers escrow cancellation:** when a user is suspended, all their active seller escrows are cancelled, buyers refunded, items restored

---

## My Trades & UI

### Naming
- "Seller Desk" renamed to **"My Trades"** everywhere
- **"Karma Hold"** replaces "Escrow" in all user-facing copy. "Escrow" only appears in code and internal documentation

### Trade Reference Format
Raw UUIDs are never shown to users. Trade identifiers shown as human-readable short references (e.g. `#KS-2405`).

### My Trades Tabs
Four tabs: Active Trades (default), Selling, Buying, History.
- Active Trades shows all in-progress trades combined — buying and selling in one feed
- Selling trades: amber left border. Buying trades: blue left border
- Completed trades stay in Active until user explicitly acknowledges — they do not auto-move to History

### Trade Detail Page
A full dedicated page (not a side panel) containing:
- Item summary
- Visual progress timeline
- In-trade chat (always visible, not hidden behind a button)
- Role-aware action panel

Seller sees dispatch form with tracking and photo upload. Buyer sees confirm received and report a problem.

### Chat
- Chat does not exist outside a trade. No pre-trade messaging.
- Conversation only starts after a Karma Hold is created.
- Chat is tied 1:1 to escrow.

### Trade Update Indicator
Red dot on the My Trades navbar item when there is new activity on any active trade. Clears immediately on navigating to `/trades` or deeper.

---

## Notifications

### Confirmed Types
`TRADE_STARTED`, `TRADE_DISPATCHED`, `TRADE_COMPLETED`, `TRADE_CANCELLED`, `TRADE_ABANDONED`, `NEW_FOLLOWER`, `KARMA_GIFTED`, `NEW_MESSAGE`, `ACCOUNT_SUSPENDED`

### Still Needed
`KARMA_EXPIRED`, `TRADE_CANCELLED_PENALTY`, bounty-related notifications

### Routing on Tap
- All trade notifications → Trade Detail page
- `NEW_MESSAGE` → Trade Detail scrolled to chat
- `NEW_FOLLOWER` → that user's profile
- `KARMA_GIFTED` → Karma history
- `ACCOUNT_SUSPENDED` → suspension notice with reason, date, and contact info (admin username never shown)

### Notification Centre
Bell icon in navbar with unread indicator. Tapping opens full notifications screen. Dropdown shows last 5; full page at `/notifications`.

### Suspension Notice
When a suspended user tries to log in, backend returns 403 with suspension reason and date. Frontend renders a suspension card at the login screen with that data. Generic error message is never shown.

---

## Social

- Social feed and leaderboard cover **all of Nigeria** — not city or neighbourhood scoped
- Feed shows new listings and completed trades from followed users only — login and system events excluded
- **Feed blending:** 0 following = 100% discovery. 1–5 = 30/70. 6–15 = 60/40. 16+ = 80/20. Shortfall backfills with discovery so total always reaches 50 entries
- Feed split ratios are fixed product decisions — do not change without approval
- **Leaderboard formula:** `(tradeCount × 10) + (karmaEarned × 0.5) + (trustScore × 2) + (avgRating × 20)`. Logged-in user's rank always shown even if outside top 10
- Follow button hidden on own leaderboard card
- Suggested traders (4 max in sidebar) and full Discover section should not show the same users. "See More" routes to Discover tab
- Location excluded as a ranking or filtering factor — user location is free text, not reliable for filtering

---

## Location

Three fields: State (dropdown), City (dropdown based on selected state), Popular area or landmark (optional, free-text). Data served from a static JSON file bundled in the backend — no external API dependency.

> ⚠️ Not yet built. Static JSON file not yet sourced. `GET /api/locations` endpoint not yet built.

---

## Karma Bounty

User posts a bounty with title, description, category, and Karma amount. Karma locked upfront at posting. Other users respond by linking a matching listed item. Bounty creator approves or rejects responses. If approved, normal escrow flow begins using the locked Karma. If cancelled or expired unmatched, full Karma refund. Auto-cancels after 30 days.

> ⚠️ Not yet built. Entity, endpoints, cron, activity log all unstarted.

---

## Profile

### Editable Fields
Username, bio, city, profile photo.

### Username Changes
Username changes cascade everywhere — trade history, chat messages, leaderboard, activity feed, all past records update.

> ⚠️ Profile update endpoint not yet built. Avatar upload UI not yet built (field exists on entity).

---

## Admin

### Roles
- **SUPER_ADMIN** (founder only) — full access, creates other admins
- **MODERATOR** — view activity, suspend users, delist items. Cannot see Karma economy internals
- **OBSERVER** — read-only, overview and activity feed only

### Bootstrap
`POST /api/admin/bootstrap` — creates first SUPER_ADMIN. Works only once when `admin_users` table is empty. Validates against a secret in application config. Always hardcodes SUPER_ADMIN.

### Fraud Detection
Four rules run hourly — flags created for human review, no automatic suspensions:
- Karma farming (500 Karma in 24 hours)
- Listing flood (10 listings in 2 hours)
- Instant confirm (receipt within 5 minutes of dispatch)
- Circular swap (reciprocal trade between same two users in 24 hours)

Rule 5 (IP-based multiple accounts) — skipped, no IP capture exists.

24hr deduplication window per user+flagType to avoid flag spam.

### Dispute Queue
When a dispute is raised, all admins are notified and it appears in a disputes queue. Admin reviews chat history, pre-shipment photo, tracking number. Resolution is binary: release Karma to seller or refund to buyer. 48hr auto-release clock pauses for the duration.

---

## Infrastructure Decisions

- **`@Lob` banned on String fields** — use `@Column(columnDefinition = "TEXT")` instead
- **Every Karma balance update must be paired with `profileRepository.save()`** — recurring bug pattern, non-negotiable
- **Activity log writes must be `@Async`** — use `ActivityLogWriter`, never `activityLogRepository.save()` directly
- **JWT secret must move to environment variable** — non-negotiable before any public deployment
- **SQL comments banned inside JdbcTemplate strings** — causes `PreparedStatementCallback` errors in PostgreSQL

---

## Active Bugs

| Bug | Status |
|---|---|
| `GET /admin/overview` — karma in circulation and completed trades not displaying | Undiagnosed |
| Admin dispute endpoints missing `SecurityConfig` role restriction | Not confirmed applied |
| Referral block in `createItem()` still commented out | Easy fix — `referredBy` confirmed on `User` |
| Suggested traders and Discover tab showing same users | Pending backend fix |
| Social feed showing "Trade Ongoing" for cancelled trades | Backend not yet sending `TRADE_CANCELLED` activity type |
| Activity log timestamps showing as dash | Likely frontend, uninvestigated |
| N+1 in `EscrowService.mapToResponse()` for single-record lookups | Outstanding |

---

## Pending Owner Decisions

- Trust score calculation metrics — formula not yet confirmed
- Karma expiry: full balance zeroed or partial? Not yet finalised
- Abandoned escrow threshold — 72 hours assumed and implemented, not yet explicitly confirmed by owner

---

## What Is Not Yet Built

### Features
- Dispute resolution — backend built, frontend contract match not yet confirmed; admin `SecurityConfig` restriction unconfirmed
- Karma Bounty — entirely unstarted
- Karma Expiry cron
- Buyer cancellation penalty tracking
- `GET /api/locations` — static JSON and endpoint
- Search — `GET /api/items/search?q=` (title + description matching)
- Profile update endpoint and edit UI
- Public profile screen (`PublicProfile.jsx`) — backend endpoint ready, frontend not built
- Avatar upload UI
- Marketplace pagination

### Referral
- +25 on friend's first listing — commented out
- +25 on friend's first swap — not wired

### Notifications Not Yet Wired (Backend)
`TRADE_DISPATCHED`, `TRADE_COMPLETED`, `TRADE_ABANDONED`, `NEW_MESSAGE`, all karma reward notifications

### Admin
- `GET /admin/overview` karma/completed trades bug unresolved
- `GET /admin/users/{id}/karma-history` — cumulative running balance not built
- `GET /admin/activity-log` — status unknown, was "in progress" per handover
