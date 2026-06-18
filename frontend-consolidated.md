# KarmaSwap — Frontend Consolidated Report

**Last updated:** June 5, 2026
**Stack:** React 18 + Vite + Axios + lucide-react + Plain CSS + React Router DOM v6

---

## Project Structure

```
src/
  components/       ← all React components
  api/
    axios.js        ← configured axios instance — ALWAYS import from here
    escrow.js       ← all escrow API functions
    items.js        ← marketplace item API functions
  contexts/
    AuthContext.jsx ← auth state, user, profile, authReady flag
  styles/
    App.css         ← single monolithic stylesheet (admin.css is separate)
    admin.css       ← admin-only styles, imported only by admin components
  App.jsx           ← main app with React Router
  main.jsx          ← entry point with BrowserRouter + AuthProvider
```

---

## Critical Rules — Read Before Touching Anything

**1. Never use raw axios**
Always import the configured instance:
```js
import api from "../api/axios";
```
This instance auto-attaches the JWT, handles 401 logout, and reads baseURL from `VITE_API_URL`.

**2. Never hardcode `localhost:8080`**
```js
const API_ORIGIN = import.meta.env.VITE_API_URL?.replace("/api", "") || "http://localhost:8080";
```

**3. All styles go in `App.css`** — except admin styles which go in `admin.css`.
No component-level CSS files exist yet. CSS Modules split is on hold.

**4. Always use CSS variables — never hardcode colors:**
```css
--primary: #2d7a3e
--primary-hover: #246330
--primary-light: #e8f5e9
--surface: #ffffff
--background: #fafafa
--text: #2c3e50
--text-light: #6c757d
--border: #e0e0e0
--error: #e74c3c
--success: #27ae60
--shadow: rgba(0,0,0,0.08)
--radius: 12px / --radius-sm: 8px / --radius-lg: 16px
--transition: 0.2s ease
```

**5. Use `profile` over `user` for display data**
`user` only has `email` after login. `profile` has `username`, `id`, `karmaBalance`, `location` etc. Guard all data fetching behind the `authReady` flag.

**6. Always normalize image URLs**
```js
const normalizeUrl = (url) => {
  if (!url) return "";
  if (url.startsWith("http://") || url.startsWith("https://")) return url;
  return `${API_ORIGIN}/uploads/${url}`;
};
```

**7. Always normalize status strings before comparing**
```js
const status = String(trade.status || "").toUpperCase();
```
Backend sends mixed casing (`AWAITING_DISPATCH`, `ESCROW`, `DISPATCHED` etc.). `mapEscrowToOrder()` in `escrow.js` handles this — always run escrow data through it.

**8. Profile ID vs User ID**
`user.id` can be null. Always use `profile.id || profile.accountId` for ownership checks.

**9. Role detection in trades — check multiple fields**
```js
const isSeller =
  trade.sellerId === profile?.id ||
  trade.sellerId === profile?.accountId ||
  trade.sellerName === profile?.username;
```

**10. API error shape is inconsistent — always check all three:**
```js
error.response?.data?.message
error.response?.data         // plain string
error.response?.data?.error
```

**11. Use `Promise.allSettled` over `Promise.all`**
All multi-fetch calls use `Promise.allSettled` so one failing endpoint doesn't crash the page.

**12. Keep mock data fallbacks**
Social page and leaderboard have mock data that shows when API calls fail. Never let an API failure show a blank or crashed page.

---

## Routing

All routes live in `App.jsx` inside `AppContent`. Admin routes are outside `AppContent` at the top-level `App` export — they render with no main navbar.

| Path | Component | Notes |
|---|---|---|
| `/marketplace` | `Marketplace` | |
| `/item-detail` | `ItemDetail` | needs `selectedItem` state |
| `/checkout` | `CheckoutEscrowConfirmation` | |
| `/trades` | `MyTrades` | |
| `/trades/:escrowId` | `TradeDetail` | |
| `/social` | `Social` | |
| `/profile` | `Profile` | |
| `/list-item` | `ListItem` | |
| `/order-tracking` | redirect → `/trades/:escrowId` | legacy — kept alive for stale links |
| `/dispute/:escrowId` | `DisputeResolution` | reads escrowId from route params |
| `/transaction-complete` | `TransactionComplete` | legacy |
| `/admin/login` | `AdminLogin` | isolated |
| `/admin/bootstrap` | `AdminBootstrap` | one-time first SUPER_ADMIN setup |
| `/admin/*` | `AdminApp` | isolated — no main navbar |

---

## Infrastructure & Setup

- **`axios.js`** — `VITE_API_URL` env var, request interceptor normalizes and attaches JWT, 401 interceptor clears session and redirects, token null/undefined guards.
- **`main.jsx`** — wrapped with `BrowserRouter`, `AuthProvider`, `GoogleOAuthProvider`.
- **`vite.config.js`** — COOP headers added for Google OAuth popup.
- **`.env`** — `VITE_API_URL=http://localhost:8080/api` — no hardcoded URLs anywhere.
- **`GoogleLoginButton.jsx`** — deleted. Was a legacy duplicate causing GSI to initialize twice. `Auth.jsx` handles Google login correctly.
- **`supabase.js`** — deleted. Unused, was throwing hard error on missing env vars.

---

## Authentication

**`AuthContext.jsx`**
- `persistSession` helper extracted — shared by `signIn` and `signInWithGoogle`
- `authReady` flag — prevents polling and data fetches from firing before auth is initialized
- `isTokenExpired` — decodes JWT and checks `exp` claim on session load
- `setAuthReady(false)` on logout
- `normalizeToken` strips Bearer prefix
- Reads `id` and `username` from `response.data` directly, not from JWT decode
- HTTP status prefix stripped from error messages via regex before displaying
- Referral code — reads `?ref=` query param via `URLSearchParams` directly inside `signUp` (Option B) — zero changes to `Auth.jsx`'s `handleSubmit`

**`Auth.jsx`**
- Specific error messages from backend — "No account found" vs "Incorrect password"
- Sign-up prompt shown when email not found
- Suspension notice card shown on 403 response with reason from backend
- `suspended` and `suspensionReason` state variables
- Google sign-in rendered only on Sign Up tab, not Login tab

---

## Navbar & Layout

**Desktop Navbar**
- lucide-react icons left of each nav label
- Karma balance display with sparkle icon
- Profile replaced with circular avatar showing first letter of username
- Account ID removed
- Fluid responsive using `clamp()` — no extra media queries needed
- `List Item` button stays in navbar (matches Etsy/Vinted pattern)
- Notification bell added — polls `/notifications/unread-count` every 30s

**Mobile Layout**
- Bottom tab bar fixed at bottom — Marketplace, Social, My Trades, Seller Desk, List Item (floating green circle), Profile avatar
- Top navbar hidden below 768px
- Mobile header shows logo and brand name sticky at top — wrapped in `.mobile-header-brand`; `.mobile-header-actions` for right-aligned bell
- Category filters scroll horizontally on mobile, no wrap
- `padding-bottom: 72px` on main content so tab bar doesn't overlap
- Red dot indicator on "My Trades" / "Trades" tab — polls `GET /api/escrow/unread-summary` every 30s, clears immediately on navigating to `/trades` or deeper

---

## Marketplace

**`ItemCard.jsx`**
- `formatCondition` — maps `LIKE_NEW` → "Like New" etc.
- Email removed from owner fallback chain — falls back to "Unknown seller"
- Unverified badge hidden — only Safe, Trusted, Elite show
- Lock overlay on hover for items user can't afford — shows "Need X more karma"
- Red "Not enough karma" text removed — replaced by overlay and faded karma value
- Follow/Unfollow button per card, right of owner name, hidden on own listings
- "Your Listing" badge on own items — purchase interaction disabled
- Owner location shown below owner name via `ownerLocation` flat field
- `isOwnItem` uses `profile.id` and `profile.accountId` for comparison

**`Marketplace.jsx`**
- `isOwnItem` — compares profile ID against `item.ownerId` and `item.owner?.id`
- `profile` in `useEffect` dependency array — re-evaluates when profile loads
- Follow state lifted to `Marketplace.jsx` — `followingIds` Set, `handleFollow/Unfollow`, fetches `/social/following` on mount
- Optimistic follow updates — reverts on failure
- All cards from same seller update simultaneously
- `API_ORIGIN` from `VITE_API_URL` env var

---

## Social

**`Social.jsx`** — complete rebuild from scratch (original used Tailwind which isn't installed)

**Feed**
- Real backend data from `GET /social/feed`
- Flat backend fields mapped to nested structure — `itemTitle → item.title` etc.
- `locationMap` built from following list before feed normalization — enriches location on entries missing it
- Legacy entries (null item fields) show parsed item name from description string via regex
- Item image shown from `activity.item.imageUrl` when available, falls back to 📦 emoji
- Activity badges — New Listing (green), Trade Completed (dark green), Trade Ongoing (blue), Trade Pending (amber), Trade Cancelled (mapped in `FeedCard.jsx` — backend flagged to send this type on cancellation, still open)
- Feed cards clickable — checks `item.status === "AVAILABLE"` before navigating; shows auto-dismiss toast otherwise
- Toast component — state + auto-dismiss after 3s + CSS in `Social.jsx`

**Leaderboard**
- Live from `GET /social/traders/top?limit=5`
- Medal emojis for top 3
- `isCurrentUser` row highlighted with green border and "You" badge — trusting backend `isCurrentUser` flag (backend fixed, client-side fallback removed)
- Follow button hidden on own row
- Deduplication — prevents same user appearing twice
- Trust badges removed (too cramped in sidebar — revisit later)
- Follower count, trade count, location shown per entry

**Suggested Traders**
- Sidebar shows `suggestions.slice(0, 4)` — capped at 4
- Discover tab shows full unsliced `suggestions` state
- Follower count shown per suggestion
- "More" button routes to Discover tab via `setActiveTab("discover")`

**Follow/Unfollow**
- `followingIds` Set tracks state across leaderboard, suggestions, feed, and user list tabs
- Optimistic updates with revert on failure
- `locationMap` workaround for missing location in feed — pending proper backend fix

---

## Notifications

**`NotificationBell.jsx`**
- Dropdown component in navbar and mobile header
- Polls `/notifications/unread-count` every 30s
- Shows last 5 notifications on open
- Mark-as-read on click, "Mark all as read", "View all" → `/notifications`
- lucide-react icons per notification type (Package, Truck, CheckCircle, XCircle, Clock, UserPlus, Gift, MessageCircle, ShieldAlert)
- Mobile: full-screen takeover via `@media (max-width: 480px)`

**`Notifications.jsx`**
- Full page at `/notifications`
- All/Unread filter, delete per notification, mark all as read
- Navigation: `ESCROW`/`CHAT` referenceType → `/trades/{referenceId}`; `USER`/`KARMA_TRANSACTION` → non-clickable, mark as read only
- Polling at 30s interval (WebSocket deferred for v1)

---

## My Trades

**`MyTrades.jsx`**
- 4 tabs — Active Trades, Selling, Buying, History
- Active → `GET /escrow/activities` (direct call — backend fixed the broken JPA query, `Promise.allSettled` workaround reverted)
- Selling → `GET /escrow/seller/active`
- Buying → `GET /escrow/buyer/active`
- History → `GET /escrow/history`
- Navigates to `/trades/:escrowId` on card click

**`TradeCard.jsx`**
- Amber left border = Selling, Blue left border = Buying
- Role detected via `sellerId === profile.id` with username fallback
- CTA label changes per role and stage:
  - Seller on `ESCROW`/`AWAITING_DISPATCH` → "Mark as Dispatched" (enabled)
  - Buyer on `ESCROW`/`AWAITING_DISPATCH` → "Waiting for dispatch" (disabled)
  - Buyer on `DISPATCHED`/`DELIVERED` → "Confirm Received" (enabled)
- Terminal trades show "Acknowledge" button inline on card
- Acknowledge calls `PATCH /escrow/{id}/acknowledge` and refreshes tab
- Abandoned trades — distinct purple badge, muted card border, "Trade expired due to inactivity" notice
- Seller 72hr abandonment countdown — `useAbandonCountdown` hook, live countdown from `trade.createdAt + 72hrs`, updates every 60s, urgent styling under 12hrs remaining
- Buyer countdown chip on card — rejected (existing 48hr countdown in TradeDetail is sufficient)
- Responsive CSS — `@media (max-width: 480px)` for flex-wrap, chip, and notice font sizing

**`TradeDetail.jsx`**
- Loads via `useParams` — self-contained, no state from `App.jsx` needed
- Item summary — image, title, karma, other party name, status badge
- Visual timeline — hidden entirely for `COMPLETED`/`CANCELLED`/`ABANDONED` trades (was previously frozen on stale stage)
- `TradeChat` sub-component — loads history from `GET /chat/{escrowId}/history`, connects to `ws-chat` WebSocket
- `ActionPanel` sub-component — role and stage aware, `useNavigate()` inside sub-component (not parent)
  - Seller on `ESCROW`: tracking input, file upload (base64, prefix stripped), dispatch button, cancel button
  - Seller on `DISPATCHED`/`DELIVERED`: waiting message, tracking number display
  - Buyer on `ESCROW`: waiting message + "Cancel Trade" button (backend relaxed cancel permission to allow buyer cancellation during `ESCROW`/`AWAITING_DISPATCH`)
  - Buyer on `DISPATCHED`/`DELIVERED`: Confirm Received + Report Problem
  - Buyer on `COMPLETED`: star rating (calls `POST /reviews/{escrowId}`)
- Acknowledge button for terminal trades → navigates back to `/trades`
- Countdown timer showing auto-release time
- Confirm dialog wording conditional on `isSeller` — "refunded to the buyer" vs "refunded to you"

**`escrow.js`**
- `startEscrow` — body removed
- `getEscrowActivities` — escrowId param removed
- `markDelivered` — deleted (endpoint doesn't exist)
- Added: `getBuyerActiveTrades`, `getAllActiveTrades`, `getTradeHistory`, `acknowledgeEscrow`, `rateSeller`
- `mapEscrowToOrder` — normalizes backend status aliases to consistent uppercase (`ESCROW`, `DISPATCHED`, `DELIVERED`, `COMPLETED`, `CANCELLED`, `ABANDONED`); handles both flat and nested EscrowResponse fields; `normalizeUrl` for relative image paths
- `rateSeller` verified against backend contract — hits `POST /api/reviews/{escrowId}` with `{ rating }` body

---

## Dispute Resolution

**`DisputeResolution.jsx`** — rebuilt
- Reads `escrowId` from route params (`/dispute/:escrowId`) instead of prop state
- Real API call to `POST /api/disputes/{escrowId}` with `reason`, `description`, `evidenceImages` (base64, up to 3 images)
- Video evidence removed — backend only accepts images
- Human-readable issue types mapped to backend enum values (`ITEM_NOT_RECEIVED` etc.)
- Success state UI after submission
- Both buyer and seller get "Report a Problem" access during `DISPATCHED`/`DELIVERED`
- Backend dispute endpoint built — contract match not yet confirmed

---

## Admin Dashboard

Isolated at `/admin/*` — completely separate from main app, no shared navbar. Admin styles live in `admin.css`, not `App.css`.

**Components**
- **`AdminApp.jsx`** — full self-contained dashboard, sidebar nav, seven sections: Overview, Activity Log, Users, Karma Economy, Escrow Monitor, Fraud Flags, Admin Management
- **`AdminLogin.jsx`** — dedicated sign-in at `/admin/login`, verifies admin status via `/admin/me` before routing in. Auth error codes (400/401/403) mapped to generic "Invalid email or password" — no backend internals exposed in UI
- **`AdminBootstrap.jsx`** — one-time setup at `/admin/bootstrap` for creating first SUPER_ADMIN via `userId` + `secret` flow, redirects to login on success

**Panels (all rebuilt against real API spec)**
- `KarmaEconomyPanel` — `GET /admin/karma-economy`, field names corrected (`totalKarmaInCirculation`, `usersActiveLastThirtyMins` etc.), `karma-concentration` added
- `EscrowPanel` — live endpoint calls, GREEN/AMBER/RED flag thresholds at 48hr/72hr driving badge and text color
- `FlagsPanel` — live, `flagCount` and `suspended` fields added
- `UsersPanel` — live, includes Registrations tab (tab not separate nav item — keeps sidebar at seven items)

**Role-based access**
- SUPER_ADMIN sees everything
- MODERATOR sees most
- OBSERVER is read-only
- UI hides sections/buttons by role; doc explicitly notes backend is the actual source of truth — verify against direct API calls
- `GET /admin/me` → role check on mount, 403 redirects to marketplace

**Destructive actions**
- Suspend (`POST /admin/users/{id}/suspend`) and Delist (`POST /admin/listings/{id}/delist`) both require a typed reason and a confirm step before firing

**Documentation**
- `KarmaSwap_Admin_Dashboard_Guide.docx` — covers routes, role/access matrix, section walkthrough, bootstrap flow, known limitations, test checklist

---

## WebSocket Details

Two WebSocket endpoints:

| Endpoint | Purpose |
|---|---|
| `ws://localhost:8080/ws-chat` | Buyer-seller chat per escrow |
| `ws://localhost:8080/ws-escrow-updates` | Escrow status updates |

> **Note:** Backend handover specifies JWT goes in STOMP CONNECT headers, not URL query param, and to use `http://` not `ws://` for SockJS. Frontend currently passes JWT as URL query param — verify which is correct with backend before changing.

Send format:
```js
{ destination: `/app/chat/${escrowId}`, content: text, imageUrl: "" }
```

Subscribe format:
```js
{ destination: `/topic/escrow/${escrowId}` }
```

Notifications: `/user/queue/notifications`
Escrow updates: `/user/queue/updates`

---

## Backend Endpoints — Quick Reference

### Auth
- `POST /api/auth/login` → `{ token, id, username }`
- `POST /api/auth/google` → same
- `GET /api/profile/me` → full profile
- `GET /api/profile/{username}` → public profile

### Items
- `GET /api/items` → marketplace listing with ranking
- `GET /api/items/{id}` → single item with owner enrichment
- `POST /api/items` → create listing (FormData)

### Escrow / Trades
- `POST /api/escrow/start/{itemId}` → no body
- `GET /api/escrow/{id}` → single escrow
- `POST /api/escrow/{id}/dispatch` → `{ trackingNumber, preShipmentPhoto: base64 }`
- `POST /api/escrow/{id}/complete` → no body (buyer only)
- `POST /api/escrow/{id}/cancel` → optional reason string (buyer or seller during ESCROW/AWAITING_DISPATCH)
- `PATCH /api/escrow/{id}/acknowledge` → 204
- `GET /api/escrow/activities` → all active trades
- `GET /api/escrow/seller/active` → seller active trades
- `GET /api/escrow/buyer/active` → buyer active trades
- `GET /api/escrow/history` → completed/cancelled acknowledged
- `GET /api/escrow/unread-summary` → unread trade update count
- `POST /api/reviews/{escrowId}` → `{ rating: 1-5 }` (buyer only)

### Social
- `GET /api/social/feed` → blended activity feed
- `GET /api/social/following` → users you follow
- `GET /api/social/followers` → users following you
- `GET /api/social/suggestions` → suggested traders
- `GET /api/social/traders/top` → leaderboard
- `POST /api/social/follow/{userId}` → follow
- `DELETE /api/social/unfollow/{userId}` → unfollow

### Notifications
- `GET /api/notifications` → all notifications
- `GET /api/notifications/unread-count` → unread count

### Chat
- `GET /api/chat/{escrowId}/history` → message history

### Disputes
- `POST /api/disputes/{escrowId}` → `{ reason, description, evidenceImages: base64[] }`

### Admin
- `GET /api/admin/me` → role check
- `GET /api/admin/overview` → platform metrics
- `GET /api/admin/today` → daily stats
- `GET /api/admin/activity-log` → filterable log
- `GET /api/admin/users/{id}` → user detail
- `GET /api/admin/karma-economy` → karma metrics (SUPER_ADMIN only)
- `POST /api/admin/users/{id}/suspend` → suspend user
- `POST /api/admin/listings/{id}/delist` → delist item
- `POST /api/admin/admins` → create admin
- `PATCH /api/admin/admins/{id}/role` → update admin role

---

## What Is Not Yet Done

### Needs Testing
- TradeDetail end-to-end with two real users — seller dispatch → buyer confirm → acknowledge
- WebSocket chat live testing
- Pre-shipment photo — confirm backend accepts pure base64 (prefix stripped on frontend)
- Dispute flow — backend contract match not yet confirmed

### Pending Backend
- `TRADE_CANCELLED` activity type not yet sent by backend on cancellation (frontend `FeedCard.jsx` is ready)
- `TRADE_CANCELLED_PENALTY` and `KARMA_EXPIRED` notification types — backend's next push
- Onboarding location survey — backend not built
- Admin bootstrap endpoint — backend to build

### Legacy Components — Not Yet Retired
`BuyerEscrowView.jsx`, `TransactionComplete.jsx`, `SellerDashboard.jsx` still exist. `/order-tracking` redirects rather than being deleted. Do not remove until `TradeDetail.jsx` is fully tested and stable.

### On Hold — Needs Team Decision
- CSS Modules split — monolithic `App.css`, deferred
- Full mobile layout review beyond current responsive CSS
- `PublicProfile.jsx` — clickable usernames, `GET /api/profile/{username}` exists but component not built
- Fix 7 — use `owner.isFollowing` from items response to init follow state without extra API call (reverted, needs clean impl)
- Sensitive localStorage data — security review pending
- Trust badges on leaderboard — removed, revisit when more space available
- Suggested traders dedup in Discover tab — backend question unanswered
- Search matching descriptions — backend query change, unanswered

### Tier 4 — Big Builds, Unscoped
- Bounties feature (full MVP push item)
- Profile UI revamp + edit profile UI
- Location → 2 dropdowns + 1 text field
- `/checkout` adjustments — awaiting detail on what's needed
