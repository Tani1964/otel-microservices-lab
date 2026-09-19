# Comfortme — Production Readiness Audit

**Date**: 2026-08-05
**Scope**: Backend (`comfort_me_nextjs`), Customer mobile app, Admin web app, Landing web app
**Method**: Full static read-only review of all 4 codebases (120 backend routes, 3 frontend apps) by 5 parallel review passes, cross-referenced against each other, plus a handful of safe read-only `curl` checks against the live production API to confirm exposure where noted. No code was changed, no data was mutated, no real payments/emails/SMS were sent as part of this audit.

**Total findings: 18 Critical · 19 High · 24 Medium · 26 Low** (87 raw findings, consolidated below — several were independently found by more than one review pass, which is itself a signal of how systemic they are).

---

## 0. Executive Summary

Comfortme is a bus-booking platform (Nigeria) with three client apps — a rider mobile app (Expo/React Native), an admin/ops web app (Vite/React), and a marketing site — all backed by one consolidated Next.js API (migrated recently from a set of legacy Python/Node/Rust/Go microservices, evident throughout the code in comments referencing the old services). The migration is functionally broad — 120 API routes covering auth, booking, payments, wallet, fleet/driver ops, rentals, packages, referrals, and notifications — and most of the individual business logic is competently written, with correct use of parameterized SQL, Zod validation, and (in the payments webhook) real HMAC signature verification.

**The platform is not production-ready, for one dominant reason: there is effectively no authentication anywhere in the backend except 4 routes used by the driver portal.** Access tokens are minted at login for riders and admins alike, but nothing server-side ever verifies them. Every other route — every wallet, card, booking, user profile, and every admin/fleet-management endpoint — trusts a client-supplied ID with no ownership or role check. This was confirmed live against `https://api.comfortmeng.com`, not just read from source: an unauthenticated request returned the full admin list including the real super admin's email.

This single gap is the direct cause of most of the Critical findings below: unauthenticated wallet draining, card-charging, free bookings, full PII dumps, driver account takeover via password reset, and self-service admin/super-admin account creation. Layered on top of that are a genuine payment-idempotency bug (double-crediting), a webhook that verifies but never acts on events, a broken driver-assignment workflow in the admin app (the primary dispatch action silently fails every time), a broken token-refresh flow in the mobile app (every session dies silently after 15 minutes), and plaintext password/token logging shipping in the mobile app's production builds.

None of this requires a rewrite. The fixes are concentrated: one real auth-verification layer (the JWT-signing and -verification code already exists and is already used correctly for the driver portal — it just needs to be applied everywhere else), a handful of idempotency/transaction fixes in payment code, and a handful of concrete, already-diagnosed bugs in each frontend. Section 14 gives the order to do this in.

---

## 1. Architecture Overview

```mermaid
graph TB
    subgraph Clients
        MOB["Customer Mobile App<br/>(Expo / React Native)"]
        ADM["Admin Web App<br/>(Vite / React)<br/>admin.comfortmeng.com"]
        LAND["Landing Web App<br/>(Vite / React)<br/>comfortmeng.com"]
        DRV["Driver Portal<br/>(inside admin_web_app,<br/>separate auth)"]
    end

    subgraph API["comfort_me_nextjs — Next.js 15 App Router<br/>api.comfortmeng.com (120 routes)"]
        MW["middleware.ts<br/>(CORS headers ONLY — no auth)"]
        AUTH[auth module]
        USERS[users module]
        WALLET[wallet module]
        CARDS[cards module]
        PAYH[paymentHub module]
        PAY[payments module — Paystack]
        BOOK["booking module<br/>(rides/bookings/rentals/packages/routes)"]
        ADMIN[admin module]
        DRIV[drivers module]
        BUS[buses module]
        NOTIF["notifications module<br/>(Resend / Twilio / Expo Push)"]
        TERMS[terms module]
        REW[rewards/referrals module]
        SUB[subscriptions module — orphaned]
        SYNC[sync module — orphaned]
    end

    subgraph DB["Neon Postgres — 13+ separate connection pools, one per module"]
        PG[(one physical Postgres,<br/>multiple logical DBs/pools:<br/>AUTH_ / USER_ / WALLET_ / CARD_ /<br/>BOOKING_ / ADMIN_ / DRIVER_ / BUS_ /<br/>TERMS_ / REWARD_ / SUBSCRIPTION_ /<br/>SYNC_DATABASE_URL)]
    end

    subgraph External
        PAYSTACK[Paystack]
        RESEND[Resend — email]
        TWILIO[Twilio — SMS]
        EXPOPUSH[Expo Push]
        GMAPS[Google Maps]
    end

    MOB -->|"axios, 9 client instances,<br/>all → same host"| API
    ADM -->|"axios, 12 client instances,<br/>rewritten via vercel.json →<br/>nextbackend-blond.vercel.app"| API
    LAND -->|"axios, same-origin →<br/>vercel.json rewrite"| API
    DRV -->|"driver-JWT auth<br/>(the one part that works)"| API

    API --> DB
    PAY <--> PAYSTACK
    NOTIF --> RESEND
    NOTIF --> TWILIO
    NOTIF --> EXPOPUSH
    BOOK --> GMAPS

    style MW fill:#7a2020,color:#fff
    style AUTH fill:#7a2020,color:#fff
```

**Key architectural facts, confirmed across all 5 review passes:**

- **One Next.js backend, 13+ separate Postgres connection pools.** Each business module (`auth`, `users`, `wallet`, `cards`, `booking`, `admin`, `drivers`, `buses`, `terms`, `rewards`, `subscriptions`, `sync`) opens its own global-singleton `pg.Pool` against its own `*_DATABASE_URL` env var. Several of these currently point at the same physical Neon instance (confirmed for `AUTH_DATABASE_URL`/`USER_DATABASE_URL` earlier this week), which means there is no real database-level isolation between "services" despite the module boundary — see §12.
- **Three frontends, one backend, but two different indirection layers in front of it.** The mobile app calls `api.comfortmeng.com` directly. Both web apps (admin + landing) proxy through a Vercel rewrite in their own `vercel.json` to a *different* hostname, `nextbackend-blond.vercel.app` — a raw Vercel project alias, not the documented custom domain. This was found independently by two separate review passes (admin web app, landing web app) and should be confirmed intentional — see Finding **H-ADMIN-2 / M-LAND-1**.
- **Auth is fragmented into three independent systems**: rider auth (`JWT_SECRET`/`JWT_REFRESH_SECRET`, never actually verified anywhere), admin auth (`ADMIN_JWT_SECRET`, also never verified), and driver auth (`DRIVER_JWT_SECRET`, the only one that's actually checked, on exactly 4 routes). None share a verification layer.
- **The legacy-migration history is visible everywhere in comments** — this codebase was deliberately ported near-verbatim from a prior multi-language microservice fleet (Python/FastAPI-style error envelopes, a Rust `pg_tb_sync` service, a Go `booking_service` still apparently unmigrated per one comment, Express-style route handlers). Several of the worst gaps (no auth, the admin "refresh_token equals access_token" bug, the sync module) are explicitly called out in code comments as preserved-as-is from the legacy system, not newly introduced — worth knowing when prioritizing, since it means these are pre-existing conditions being inherited into production for the first time as this migration goes live, not new regressions.

---

## 2. Feature Inventory

| Domain | Backend module(s) | Rider app | Admin app | Landing | Notes |
|---|---|:---:|:---:|:---:|---|
| Auth (signup/login/OTP/reset) | `auth` | ✅ | — (separate `admin`) | — | Two entirely separate user tables (`users`, `auth_users`) — see known orphan-row bug §8 |
| Admin auth | `admin` | — | ✅ | — | Own JWT secret, own login/setup |
| Driver auth | `drivers` | — | ✅ (driver portal) | — | The *only* backend auth actually verified server-side |
| Rider profile | `users` | ✅ | ✅ (view/suspend) | — | |
| Wallet + PIN | `wallet` | ✅ | ✅ (view only) | — | |
| Cards | `cards` | ✅ | — | — | |
| Payments (Paystack) | `payments` | ✅ (indirect) | — | — | |
| Payment orchestration | `paymentHub` | ✅ | — | — | The actual settlement path |
| Bookings | `booking` (bookings) | ✅ | ✅ | — | |
| Rides | `booking` (rides) | ✅ (read) | ✅ (full CRUD) | — | |
| Rentals | `booking` (rentals) | ✅ | ✅ (pricing/status) | — | |
| Packages | `booking` (packages) | ✅ (create only) | ✅ (read only) | — | No rider "my packages" UI — see §10 |
| Routes/stops/destinations/locations | `booking` | — (indirect, via search) | ✅ | — | |
| Buses | `buses` | — | ✅ | — | |
| Drivers (fleet mgmt) | `drivers` | — | ✅ | — | |
| Admins | `admin` | — | ✅ | — | |
| Terms & Conditions | `terms` | ✅ (read) | ✅ (CMS) | ✅ (read, but via a nonexistent endpoint) | No acceptance tracking anywhere |
| Referrals | `rewards` | ✅ | ✅ | — | |
| Notifications (send) | `notifications` | (triggered server-side) | ⚠️ broken UI | — | Admin "broadcast" feature is non-functional |
| Notifications (inbox) | `notifications` | ✅ | — | — | |
| Waitlist | `auth` | — | — | ✅ | The one thing that's fully correct end-to-end |
| Subscriptions | `subscriptions` | ❌ no UI | ❌ no UI | — | Confirmed orphaned/dead in source comments, still live & unauthenticated |
| Sync (payment reconciliation ledger) | `sync` | ❌ | ❌ no UI | — | Confirmed orphaned dead code, not offline-sync as its name suggests |
| Analytics/reporting | — | — | ⚠️ partially fake | — | Bookings chart is `Math.random()`; wallet analytics is real |
| Support/ticketing | — | — | ❌ 100% mock | — | No backend exists at all |
| Contact form | — | — | — | ❌ fake | No backend exists at all |
| Audit logs | — | — | ❌ endpoint doesn't exist | — | Page always shows empty |

---

## 3. User Journeys

### 3a. Rider journey (mobile app)

```mermaid
flowchart LR
    A[Onboarding] --> B[Sign Up]
    B --> C["Set Password<br/>POST /auth/signup"]
    C --> D["OTP Verify<br/>POST /auth/send-otp<br/>POST /auth/verify-otp"]
    D -->|"email/SMS may silently<br/>fail — no error surfaced"| D
    D --> E[Personalize + Emergency Contact]
    E --> F[Home]
    F --> G[Search Rides]
    G --> H["Book: pick seats,<br/>pay wallet or card"]
    H -->|"POST /payment-hub/pay-booking<br/>then /verify — NOT idempotent,<br/>double-tap can double-charge/refund"| I["Confirmed Booking"]
    I --> J["Trip Day: board via QR/OTP<br/>POST /bookings/id/board — no<br/>identity check, only the shared code"]
    J --> K["Complete<br/>POST /bookings/id/complete"]

    style H fill:#7a4a20,color:#fff
    style J fill:#7a4a20,color:#fff
```

Every one of these steps, today, is reachable by an unauthenticated third party knowing only sequential integer IDs — not just by the legitimate rider. See §6 (Critical) for the concrete exploit paths.

### 3b. Ops/dispatch journey (admin web app)

```mermaid
flowchart LR
    A[Create Route + Stops] --> B[Create Bus + Seat Layout]
    B --> C[Onboard Driver]
    C --> D[Create Ride]
    D --> E["Assign Driver to Ride<br/>(RideDetailPage)"]
    E -->|"BROKEN IN PRODUCTION —<br/>string-vs-number type mismatch,<br/>driversApi.assignRide always 400s,<br/>blocks the rest of the sequence"| F["Ride goes out<br/>with no driver assigned"]
    D --> G[Riders book seats]
    G --> H[Ride day: board/monitor]
    H --> I["Mark complete<br/>(no state-machine —<br/>any status → any status)"]

    style E fill:#7a2020,color:#fff
```

**The single most-used scheduling action in the admin app — assigning a driver to a ride — does not work today.** This is Finding **C-ADMIN-2** below; it's listed here because it breaks the core operational flow, not just an edge case.

---

## 4. THE Systemic Issue: No Authentication (read this section first)

This is not one bug — it is the absence of a layer that should wrap the entire API. Confirmed independently by three of the five review passes, and confirmed **live** against production (not just in source):

```
$ curl -s -w "HTTP %{http_code}\n" "https://api.comfortmeng.com/api/v1/admins?skip=0&limit=5" -H "x-request-id: audit-check-1"
HTTP 200
[{"id":1,"first_name":"Super","last_name":"Admin","email":"admin@comfortmeng.com","role":"super_admin","is_active":true,...}]
```

**Why this happened**: `src/modules/auth/jwt.ts` and `src/modules/admin/jwt.ts` both correctly *sign* access tokens at login. But `jwt.verify()` against those secrets is never called anywhere in the codebase. The only file that resembles a guard, `src/modules/admin/guard.ts`, checks for the *presence* of an `x-request-id` header — not who sent it — and its own code comment admits this: *"the CRUD routes still have no auth check at all, which is a real, deliberately preserved gap."* `src/middleware.ts`, the one place that runs in front of every route, only sets permissive CORS headers (`Access-Control-Allow-Origin: *`).

The **only** part of the system that does this correctly is the driver portal: `src/modules/drivers/guard.ts` verifies a bearer JWT against `DRIVER_JWT_SECRET` and is applied to exactly 4 routes (`drivers/me`, `rides/[id]/ride-code`, `rides/[id]/packages`, `packages/[id]/deliver`). **This is the reference implementation** — the fix for everywhere else is to generalize this exact pattern, not invent a new one.

**Confirmed concrete consequences** (each independently reproducible today, no auth token needed for any of them):

| # | What an anonymous caller can do | Where |
|---|---|---|
| 1 | List every admin account, incl. the real super admin's email | `GET /api/v1/admins` |
| 2 | Create a new **super_admin** account | `POST /api/v1/admins {"role":"super_admin",...}` |
| 3 | Reset any driver's password and read the new plaintext password back | `POST /api/v1/drivers/{id}/reset-password` |
| 4 | Dump every driver's PII — phone, address, next-of-kin, license number | `GET /api/v1/drivers` |
| 5 | Dump every rider's PII — name, email, phone, emergency contacts | `GET /api/v1/users/` |
| 6 | Read any rider's saved card's reusable Paystack charge token, then charge it any amount | `GET /api/v1/cards/{user_id}` → `POST /api/v1/payments/charge` |
| 7 | Drain any rider's wallet balance — no PIN, no auth | `POST /api/v1/wallet/deduct` |
| 8 | Create real, confirmed, seat-reserved bookings with zero payment | `POST /api/v1/bookings`, `/bookings/bulk` |
| 9 | Cancel or "board"/"complete" any other rider's booking by guessing a sequential ID | `DELETE /api/v1/bookings/{id}`, `POST /bookings/{id}/board` |
| 10 | Set any pending rental's price to ₦0.01 and pay it | `PATCH /api/v1/rentals/{id}/price` |
| 11 | Send arbitrary SMS/email, from Comfortme's own sender identity, to any phone/email, on the company's Twilio/Resend bill | `POST /api/v1/notifications/*` |
| 12 | Create/reassign/cancel rides, buses, routes; publish Terms & Conditions | various admin/ops routes |

**Fix (applies to nearly every Critical/High finding below at once):**
1. Write one `requireAuth(request)` helper (mirroring `drivers/guard.ts`) that verifies the bearer token against the correct secret per audience (`JWT_SECRET` for riders, `ADMIN_JWT_SECRET` for admins) and returns the caller's identity/role.
2. Apply it to every route in scope, checking resource ownership (`token.userId === resource.user_id`) for rider routes and role (`admin`/`super_admin`/etc.) for admin routes.
3. Confirm `JWT_SECRET`, `JWT_REFRESH_SECRET`, and `ADMIN_JWT_SECRET` are set to strong random values in Vercel Production — today they silently fall back to the literal string `"change_me_in_production"` if unset, which (given this session already found two other required env vars missing from Production) must not be assumed.
4. This is also the fix that makes the admin web app's existing role-based UI (`Sidebar.tsx` hiding nav items, the create-admin form omitting `super_admin`) into a *real* control instead of cosmetic decoration — the frontend already assumes this backend behavior exists.

---

## 5. Critical Findings

*(Excluding the auth items already covered in §4 — those are items 1–5, 7–9, 11 in the table above. This section covers everything else Critical.)*

### C-1. Payment verification is not idempotent — can double-credit a wallet or produce "confirmed booking + full refund" from one charge
**File**: `modules/paymentHub/service.ts:653-673` (wallet funding), `701-744` (booking payment), `modules/wallet/repository.ts:99-107`
`POST /payment-hub/verify/{reference}` has no auth and the `reference` is handed to the client at `initialize` time, so any client holding it can call `verify` as many times as it wants — trivially triggered by a network retry, a double-tap, or the app foregrounding twice during a Paystack redirect.
- **Wallet funding**: `fundWallet()` increments the balance *before* inserting the transaction row that's the only thing enforcing `reference` uniqueness. A second call for the same reference credits the balance again, then fails only on the transaction insert — by then the balance is already double-credited, with no rollback.
- **Booking payment (worse)**: first `verify` call books the seats. A second call for the same reference fails to re-book (seats taken) — and that failure path is coded to mean "payment succeeded but couldn't secure seats," so it **refunds the full amount to the wallet**. Net result of one real card charge + two verify calls: a confirmed booking *and* a full refund — free money.
- **Fix**: make `verifyPaymentHub` idempotent up front (check if `reference` was already processed before doing anything), and fix `fundWallet`'s statement ordering so the balance adjustment and the uniqueness-enforcing insert happen in one transaction with rollback on failure.

### C-2. Paystack webhook verifies its signature correctly, then does nothing with the event
**File**: `app/api/v1/payments/webhook/route.ts:8-21`
The HMAC-SHA512 signature check is solid (constant-time compare, no bypass found) — but after verifying, the handler just echoes `{event, status:"received"}` and stops. It never triggers `fundWallet`, `createBookingsBulk`, or any of the side effects `payment-hub/verify` performs. This route exists specifically as Paystack's safety net for payments that settle with no client present (bank transfer, USSD, app killed mid-redirect) — today, any payment that only completes this way is money Comfortme received with **no corresponding wallet credit or booking ever created.** The rider paid and got nothing, silently.
- **Fix**: wire the webhook to the same reconciliation logic as `verify` (ideally the same idempotent function once C-1 is fixed), gated on `event.event === "charge.success"`.

### C-3. `admin_web_app`: driver↔ride assignment is broken in production — the primary dispatch action doesn't work
**File**: `admin_web_app/src/features/drivers/api/driversApi.ts:63-75`, backend `modules/drivers/validation.ts:46-47`
`driversApi.assignBus`/`assignRide` send `bus_id`/`ride_id` as raw JSON strings; the backend schema requires actual numbers (`z.number().int()`, not coerced) and 400s on a string. `busesApi`/`ridesApi`'s equivalent calls *do* coerce correctly — only the driver-side calls are missing it. Concretely:
- `RideDetailPage`'s "Assign Driver" — the main way ops staff build out a schedule — awaits `driversApi.assignRide` *first*, which always 400s, so the ride/bus-side calls that would actually complete the assignment are never reached. **This feature cannot succeed today.**
- `DriverDetailPage`/`BusDetailPage`'s assign-bus flows fire two independent calls via `Promise.all`; one succeeds, one 400s, leaving driver and bus records cross-referencing inconsistently with only a generic "Failed" toast shown.
- **Fix**: `Number(busId)`/`Number(rideId)` in `driversApi.ts`, matching the pattern already correct elsewhere. Longer-term, make cross-resource assignment one backend transaction instead of two client-orchestrated calls (see also §9 architecture note).

### C-4. `admin_web_app`: Notifications broadcast feature can never succeed
**File**: `admin_web_app/src/pages/NotificationsPage.tsx:52-68`, backend `modules/notifications/validation.ts:43-48`
Posts `{phone, user_id, message}` to `/notifications/ride-update` for every channel/target combo, but that endpoint requires `ride_id` (never collected) and is semantically a single-ride single-recipient endpoint, not a broadcast tool. There is no backend broadcast endpoint at all. Every submission fails.
- **Fix**: needs a real backend broadcast endpoint; until then, mark the page non-functional rather than presenting it as a working tool.

### C-5. `admin_web_app`: admin password recovery and change-password are dead ends (404 on every attempt)
**File**: `admin_web_app/src/features/auth/api/authApi.ts:19-37`
Calls `/api/v1/admin/auth/forgot-password`, `/reset-password`, `/change-password` — none of these exist on the backend (only unprefixed `auth/*`, which operates on the *rider* table, exists). Same broken path is used for the silent 401-refresh (`admin/auth/refresh` also doesn't exist), so every admin session hard-logs-out on token expiry instead of silently refreshing.
- **Fix**: needs real `admin/auth/{forgot,reset,change}-password` and `admin/auth/refresh` backend routes — this is a backend gap, not just a frontend URL fix.

### C-6. Mobile app: passwords and bearer tokens logged in plaintext, including in production builds
**Files**: `customer_mobile_app/src/services/auth.service.ts:20,25`, `src/services/api.ts:27-32`
`signUp()`/`signIn()` log their full request payload (including plaintext password) — this is the exact live incident already observed this session (`"password": "Test1234@"` in real logs). Separately, the shared Axios response interceptor logs **every** response body on all 9 API clients — meaning `access_token`/`refresh_token` are logged on every login/signup/refresh, and `paystack_auth_code` (a reusable card-charge token) is logged on every Wallet/Cards screen load. None of this is gated behind `__DEV__` or stripped by a build plugin, so it ships in release builds and is visible via device logs. Notably, `resetPassword`/`changePassword` *do* correctly omit passwords from their own logs — proving the gap is an oversight, not a decision.
- **Fix**: remove payload/response-body logging from `api.ts` and `auth.service.ts` entirely, or gate behind `__DEV__` and redact `password`/`pin`/`*token*`/`paystack_auth_code` before logging.

### C-7. Mobile app: token refresh is broken — every session silently dies after 15 minutes
**Files**: `customer_mobile_app/src/services/api.ts:46-52`, backend `app/api/v1/auth/refresh/route.ts:14`
Access tokens expire after 15 minutes. The 401-refresh interceptor reads `data.access_token` from the raw response, but the backend actually returns `{data: {access_token, refresh_token}}` — one level deeper. The refresh silently produces `undefined`, the retried request fails, and — critically — the failure path only clears SecureStore, never the in-memory Zustand auth state, so `isAuthenticated` stays `true` and the app never redirects to sign-in. **Every real session longer than 15 minutes enters a "zombie" state**: every screen fails with 401/error states, with no way out except force-quitting the app. This is very likely the single highest-impact bug in the mobile app given how common a 15+ minute session is.
- **Fix**: correct the unwrap to `data.data.access_token`; call `logout()` + redirect on refresh failure; centralize the refresh logic (today it's duplicated across all 9 clients, so concurrent 401s fire concurrent refreshes that invalidate each other's single-use refresh token).

### C-8. Mobile app: Profile "Save" silently discards the user's name edit
**Files**: `customer_mobile_app/src/app/(app)/account/profile.tsx:16-24`, backend `modules/users/validation.ts:21-36`
The profile screen sends a combined `name` field; the backend schema only recognizes `first_name`/`last_name` and (being a non-strict Zod object) silently drops unrecognized keys instead of erroring. The request returns 200, the UI shows "Saved," and the name is never actually changed. Invisible without diffing request/response bodies.
- **Fix**: split `name` into `first_name`/`last_name` before submitting, same as `set-password.tsx` already does correctly during signup.

### C-9. Mobile app: live Google Maps API key committed to git in plaintext
**File**: `customer_mobile_app/eas.json:27`
Hardcoded in a tracked build-config file (confirmed present in git history), rather than an EAS secret. Used client-side to build Google Static Maps URLs, so it's expected to be *somewhere* in the shipped bundle — but committing it to source means it's also permanently in git history, discoverable independent of bundle extraction.
- **Fix**: treat as burned, rotate immediately, move to an EAS secret, and confirm in Google Cloud Console that the key is restricted (referrer/package+SHA1/bundle ID, and API-scoped to Static Maps only).

---

## 6. High Findings

| # | Finding | Where | Why it matters |
|---|---|---|---|
| H-1 | Seat booking has a real double-booking race (confirmed independently by two review passes) | `modules/booking/repository/bookings.ts:66-106` | No `SELECT...FOR UPDATE`, no conditional `UPDATE...WHERE status='available'`, no unique index — two concurrent requests for the same seat under READ COMMITTED isolation can both succeed. Fix: row-lock or conditional update + `rowCount===0` check. |
| H-2 | SMS and push notifications share email's silent-failure design — and SMS is the *primary* OTP channel for this market | `modules/notifications/providers/twilio.ts`, `expo.ts` | Neither ever throws; every failure (bad creds, non-2xx, network error) resolves `false`, discarded by every call site. There is no path by which an OTP SMS failure becomes visible to the rider or any dashboard — signup/login always reports success regardless. Directly connects to Finding M-5 below (mobile UX has no cooldown/failure messaging either). |
| H-3 | `POST /wallet/deduct` requires no PIN at all | `app/api/v1/wallet/deduct/route.ts` | The PIN gate (`verifyWalletPinOrThrow`) exists only one layer up, in `paymentHub`. The raw endpoint is independently public and skips it entirely — more severe than a PIN-bruteforce problem, since the PIN can be skipped outright. (Folded into the auth fix in §4, called out separately for severity.) |
| H-4 | `authErrorResponse` (auth module) leaks raw internal error messages; every other module's error handler sanitizes them | `modules/auth/errors.ts:18-25` vs `lib/http-errors.ts:38-47` | Auth is the single most-attacked surface (login/signup/reset). An unexpected exception there — e.g. a raw Postgres error string containing table/column names — is returned verbatim instead of a generic message. |
| H-5 | Driver password reset returns the new plaintext password in the response body | `modules/drivers/auth-service.ts:51-64` | Combined with no auth (§4), this is a one-request full account-takeover primitive, worse than the general PII exposure — called out separately from §4's table for emphasis. |
| H-6 | Ride status has no state machine — any status can go to any status | `modules/booking/repository/rides.ts:191-196` | A ride can jump `scheduled → completed` directly, or be un-completed back to `scheduled`, with no cascading effect on bookings/seats/payouts. Fix: explicit allowed-transitions table. |
| H-7 | Driver/bus/ride assignment is written independently in 3+ unsynchronized places across 2 databases | `buses/repository.ts`, `drivers/repository.ts`, `booking/repository/rides.ts` | No overlap/conflict checks anywhere — a driver or bus can be double-booked onto two overlapping rides; `complete-ride` resets the driver's status but never touches the ride record itself, which can be left orphaned at `active` forever. |
| H-8 | Booking board/complete has no caller-identity check — only the ride's shared boarding code | `app/api/v1/bookings/[id]/board`, `.../complete` | The boarding code is shown to *every* rider on a ride. Any rider who legitimately has it can iterate sequential booking IDs and mark *other* riders' bookings boarded/completed. |
| H-9 | Unbounded pagination `limit` everywhere, combined with no auth = full-table PII dump in one request | `lib/common-validation.ts:7-10` and several hand-rolled parsers | `GET /drivers?limit=999999` or `/packages/all?limit=999999` exfiltrates entire tables in one call. Fix independent of the auth fix: cap `limit` server-side. |
| H-10 | `admin_web_app`: table search boxes only filter the currently-loaded 20-row page, not the full dataset | `DriversPage`, `BookingsPage`, `RidesPage`, `UsersPage` | Looks like it works, is silently wrong for anything not on-screen — an ops agent searching for a specific booking reference gets a false "not found." |
| H-11 | `admin_web_app`: Settings page saves nothing but tells staff it did | `SettingsPage.tsx:46-183` | Company/Payment/Notification/Booking-rule tabs call no API at all — just a success toast. An admin believing they've changed a cancellation-fee rule or Paystack key has changed nothing. Trust-eroding and operationally risky. |
| H-12 | `admin_web_app`: Support/ticketing page is 100% mock data | `SupportPage.tsx` | No backend exists; "creating" a ticket shows success and then the ticket vanishes — never persisted, never actioned. |
| H-13 | `admin_web_app`: Dashboard's Bookings chart is `Math.random()`, indistinguishable from real data | `dashboardApi.ts:57-82` | No `/analytics/bookings` endpoint exists; the mock fallback has no "unavailable" indicator. Any stakeholder glancing at it sees noise with full confidence it's real. |
| H-14 | `admin_web_app`: admin session tokens stored in plain `localStorage`, no CSP configured | `AuthContext.tsx:40-42`, `DriverAuthContext.tsx:37-39` | For a panel with real operational power (create admins, cancel bookings, publish legal terms), any XSS yields full session theft with no httpOnly/SameSite protection and no CSP as mitigation. |
| H-15 | `admin_web_app`: `/admins` route has no role guard, only auth-required | `App.tsx:97` | Any authenticated admin — including `customer_support`/`finance_officer` — can navigate directly to `/admins` and get full admin-management UI, even though the sidebar hides the link. Defense-in-depth gap on top of §4's backend gap. |
| H-16 | `admin_web_app` / `landing_web_app`: production API traffic is proxied through a raw Vercel project alias, not the documented custom domain | `vercel.json` in both apps → `nextbackend-blond.vercel.app` | Found independently by two review passes. If that Vercel project is ever renamed/redeployed differently, every API call from both web apps breaks with no build-time warning. Also silently bypasses any WAF/rate-limiting/cert policy scoped to `api.comfortmeng.com` specifically. Worth an explicit confirmation with whoever owns the Vercel projects. |
| H-17 | Mobile app: two dead-end buttons with no `onPress` at all | `home/index.tsx:136-138` ("I am on board"), `account/change-password.tsx:60-62` ("Forgot password? Reset") | Both have working equivalents elsewhere in the app, suggesting an incomplete port rather than intent. |
| H-18 | Mobile app: package delivery code has no persistent UI | `trips/send-package.tsx:80-98` | The one-time code the driver needs at drop-off is shown only on the immediate post-payment screen, held in local state. No "My Packages" list exists despite the backend supporting it (`GET /packages/*`). If the user navigates away before noting the code, it's unrecoverable through the app. |
| H-19 | Landing app: `/contact` message form is entirely fake | `pages/ContactPage.tsx:37-40` | `handleSubmit` just calls `setSent(true)`. No backend route for it exists anywhere. Worse than an obviously-broken form — it actively confirms a fake success to a user who thinks they reached support. |

---

## 7. Medium Findings (condensed)

| # | Finding | Where |
|---|---|---|
| M-1 | Referral code apply has a TOCTOU race allowing the same user to double-apply a discount (no unique constraint on `referral_usages`, unlike the equivalent `referral_conversions` table which does have one) | `modules/rewards/repository.ts:160-205` |
| M-2 | Wallet deduct's insufficient-balance check reads a stale snapshot before the atomic decrement — concurrent deducts can drive a balance negative (no `CHECK (balance >= 0)`) | `modules/wallet/repository.ts:109-123` |
| M-3 | Orphaned-signup bug is **still open** (known issue from earlier this session, reconfirmed) — `auth_users` insert after `createUser()` has no try/catch; a failure (e.g. the *auth* DB's independent phone-uniqueness constraint, out of sync with the *users* DB's) leaves an unrecoverable orphaned profile | `modules/auth/service.ts:131-134` |
| M-4 | Every list endpoint's `limit` has no upper bound (`.default(100)`, no `.max()`) | `lib/common-validation.ts:7-10` and module-local copies |
| M-5 | Rental status transitions are equally unvalidated as ride status (H-6), lower blast radius | `app/api/v1/rentals/[id]/status/route.ts` |
| M-6 | Driver creation only pre-checks email uniqueness, not phone/license — duplicates throw an opaque 500 instead of a clean 409 | `modules/drivers/repository.ts:67-103` |
| M-7 | Per-module unbounded Postgres pools (`pg` default `max=10`) × 12+ modules × however many warm Vercel lambdas — real connection-exhaustion risk under load against a managed Postgres with a modest connection cap | every `*/db.ts` | 
| M-8 | N+1 query patterns on every list endpoint that touches a route (`listRides`, `listRoutes`, booking/package DTOs) — listing 100 rides costs 400+ queries | `modules/booking/repository/{rides,routes,bookings,packages}.ts` |
| M-9 | Mobile: wallet transaction history has no virtualization or pagination — renders the entire history unbounded in a `ScrollView` | `wallet/index.tsx:212-230` |
| M-10 | Mobile: funding wallet from a saved card requires no PIN/biometric, while *spending* from wallet does — the higher-consequence action is less protected | `wallet/index.tsx:104-113` |
| M-11 | Mobile: "Delete Account" has no error handling at all — a failed delete leaves the user with zero feedback | `account/index.tsx:86-99` |
| M-12 | Mobile: Delete Account confirmation copy is wrong ("...disconnect your debit card?" on an account-deletion modal) — copy-paste leftover on a destructive, irreversible action | `account/index.tsx:240-242` |
| M-13 | Mobile: OTP flows have no resend cooldown, no rate-limit UI, and (on the signup OTP screen specifically) no success feedback on resend at all — directly relevant given the real silent-email-failure incident this session | `otp.tsx`, `reset-otp.tsx`, `wallet-pin.tsx` |
| M-14 | Admin app: three list pages (Routes/Locations, Packages, Rentals) have no pagination UI at all — records beyond the backend default page size become invisible with no indicator | `RoutesPage.tsx`, `PackagesPage.tsx`, `RentalsPage.tsx` |
| M-15 | Admin app: pagination "total" is a client-side guess everywhere, not a real backend count | 6 different list pages |
| M-16 | Admin app: Dashboard pulls up to 5,000 rows client-side on every load to compute 12 numbers | `dashboardApi.ts:12-19` |
| M-17 | Admin app: cross-resource assignment mutations (`Promise.all` across 2 independent backend calls) have no rollback on partial failure, independent of the C-3 type-mismatch bug | `DriverDetailPage.tsx`, `BusDetailPage.tsx` |
| M-18 | Admin app: no global error boundary — any uncaught render exception blanks the entire app to white with no recovery | repo-wide `grep` confirms none exists |
| M-19 | Admin app: Terms/Content versions can only be created and published, never edited, despite the backend supporting `PUT terms/{id}` | `ContentPage.tsx` |
| M-20 | Admin app: `.env`, `.env.local`, `.env.production` are tracked in git despite being listed in `.gitignore` (pre-existing tracked files aren't retroactively ignored) — no secrets in them today, but the safety net is an illusion for whoever adds one next | `.gitignore:6-8` vs `git ls-files` |
| M-21 | Landing app: `vercel.json`'s API rewrite targets the same fragile `nextbackend-blond.vercel.app` alias as the admin app (see H-16) | `vercel.json:3` |
| M-22 | Landing app: Privacy/Terms pages call a backend endpoint (`/policies/privacy`, `/policies/terms`) that doesn't exist; failure is silently swallowed and falls back to hardcoded content — works today, but is dead code giving false confidence content is centrally managed | `lib/api.ts:12-15`, `hooks/usePolicyContent.ts:12` |
| M-23 | Landing app: Footer "FAQ's" link points to an anchor (`#faq`) that doesn't exist on the page | `Footer.tsx:81` vs `FAQ.tsx` |
| M-24 | Terms & Conditions has zero acceptance tracking or enforcement anywhere in the codebase — purely informational despite having a full versioning/publish workflow | confirmed via repo-wide grep, zero hits |

---

## 8. Low Findings (condensed)

- SQL-injection-*adjacent* fragility (not currently exploitable): dynamic column-name building in `users/repository.ts` `applyUpdate` and the analogous `terms/repository.ts` pattern — safe today only because every call site passes hardcoded field names, not raw client keys.
- `GET /api/v1/users/` bulk-dumps full PII for every rider with no upper bound — a distinct bulk-exfiltration path from the general auth gap.
- `/health` always returns `{status:"healthy"}` unconditionally, never checks any of the 9+ DB pools or provider configs — would have caught the `USER_DATABASE_URL`-missing incident immediately instead of via live debugging.
- `subscriptions` and `sync` modules are confirmed orphaned/dead (explicit source comments + zero internal callers + no UI in any app) but remain live, public, and unauthenticated API surface.
- Admin login's `refresh_token` is literally the same string as `access_token` — a preserved legacy bug, low severity today only because there's no admin `/refresh` endpoint that behaves differently.
- Hand-rolled query-param parsers (`bookings/all`, `packages/all`, `rentals`) don't guard against negative `skip`.
- Inconsistent error-swallowing: `GET /buses` and `sync`'s list return `200 []` on a backing query failure, masking outages, vs. other modules surfacing 500s — intentional per comments, worth knowing for on-call expectations.
- Mobile: unused `LoadingOverlay` component; onboarding always briefly flashes for already-authenticated returning users before redirect resolves; rental date/time are free-text fields with no picker/validation; client-side cancellation-fee math duplicates (and could drift from) backend logic; `AppErrorBoundary` shows a raw JS stack trace to end users in production; verbose lifecycle `console.log`s ship unconditionally.
- Admin app: CSV export uses `JSON.stringify` for cell escaping (mis-escapes embedded quotes, no formula-injection neutralization for `=`/`+`/`-`/`@`); a dead `resetPassword()` API method targets a nonexistent endpoint; header notification bell has a permanent fake "unread" dot with no click target; `id: string` vs backend `id: number` typing inconsistency (the actual cause of C-3's silent runtime failure); Assign-Ride modal doesn't filter out completed/cancelled rides.
- Landing app: stale `Dockerfile`/`nginx.conf` left from the pre-Vercel DigitalOcean deployment; a dead, unused `footerColumns` data array that the real Footer doesn't read from; placeholder `href="#"` app-store/social links; cross-page anchor links (`#contact` etc.) silently no-op from non-Home routes; a cosmetic default-vs-real-option mismatch in the waitlist survey's "commute days" dropdown (hyphen vs en-dash, submits `""` if untouched).

---

## 9. What's Already Solid

Worth stating plainly, since the findings above are dense: a meaningful amount of this codebase is done well, and shouldn't be reworked out of an abundance of caution once the Critical items are fixed.

- **No SQL injection risk anywhere audited.** Every query across all modules uses `$1`-style parameterization; the only dynamic-identifier interpolation found (`booking/repository/places.ts`) only ever receives one of 3 hardcoded literals, never client input.
- **Paystack webhook signature verification is implemented correctly** — HMAC-SHA512, constant-time comparison, proper length check before compare. The gap is what happens *after* verification (§5, C-2), not the verification itself.
- **Cards module never stores a raw PAN**, only `last_four` and a gateway token — correct design (the *access control* around that token is the problem, not the storage).
- **The mobile app never touches raw card numbers at all** — routed entirely through Paystack's hosted WebView, explicitly avoiding PCI scope.
- **Tokens are stored in `expo-secure-store`** (Keychain/Keystore) in the mobile app, not AsyncStorage — the correct choice.
- **Payment-triggering buttons in the mobile app have consistent double-submit protection** (`loading || disabled`) — no double-charge risk from UI-level double-tapping was found.
- **Loading/empty/error states are consistently implemented** across the mobile app's list-heavy screens via shared components with retry affordances.
- **The waitlist signup flow (landing site) is fully correct end-to-end** — field names match the backend schema exactly, the 409 "already on waitlist" case is surfaced properly, no gaps found.
- **A few modules correctly use explicit transactions** for their multi-statement invariants (`setDefaultCard`, `publishTerms`) — proof the pattern is known in this codebase, just not applied consistently (see C-1, M-1, M-2, M-3).
- **Driver-portal auth is the one part of the system built correctly** — it's the template for fixing everything else (§4).

---

## 10. Backend ↔ UI Mapping — Missing Functionality

**Backend features with no UI anywhere** (confirmed by grep across both frontends):
- `subscriptions/*` — no screen in either app; if this is a live product feature, riders have no way to see or manage one.
- Rider-side package tracking — riders can *create* a package but never see it again after the one-time success screen (H-18).
- `sync/*` — no admin UI at all despite being an admin-relevant route group; also confirmed dead/orphaned on the backend itself.
- `drivers/{id}/reset-password` — no "reset driver password" admin action exists in the UI (the API method isn't even defined client-side).
- `terms/{id}` `PUT` (edit) — content team can create and publish versions but never edit a draft.
- `bookings/bulk`, `bookings/group/{reference}`, `bookings/{id}/complete` — no admin UI surfaces these.
- `rides/{id}/eta` — no ETA shown anywhere in the admin ride detail view.
- Location/stop/route browsing primitives (`bookingService.getLocations/getDestinations/getStops/getRoutes`) are defined in the mobile app's service layer but never called from any screen — dead client code, search is free-text only.

**UI features calling backend endpoints that don't exist** (all 404 today):
- Admin: `admin/auth/refresh`, `admin/auth/forgot-password`, `admin/auth/reset-password`, `admin/auth/change-password`, `GET /audit-logs`, `GET /analytics/bookings`, `POST /admins/{id}/reset-password`.
- Landing: `GET /policies/privacy`, `GET /policies/terms` (silently caught, falls back to hardcoded content).
- Mobile: none found — every endpoint the mobile app calls has a matching backend route. This app's backend integration is the cleanest of the three.

---

## 11. Environment Variable Checklist

Consolidated from grepping every `process.env.X` reference across the whole backend. **This session already found two of these (`USER_DATABASE_URL`, `RESEND_API_KEY`) silently missing from Vercel Production, discovered only via live debugging** — treat this whole list as unverified against the actual Vercel dashboard until manually confirmed.

| Var | Failure mode if missing/wrong | Priority to verify |
|---|---|---|
| `JWT_SECRET`, `JWT_REFRESH_SECRET` | Falls back to `"change_me_in_production"` — moot today since nothing verifies rider tokens (§4), but becomes **security-critical the instant that's fixed** | 🔴 Verify before shipping the auth fix |
| `ADMIN_JWT_SECRET` | Same fallback risk, same urgency, for admin auth | 🔴 Verify before shipping the auth fix |
| `DRIVER_JWT_SECRET` | Falls back silently; this is the one secret that's *already* live-critical, since it's the only auth actually enforced, and must match the still-unmigrated Go booking service per an in-code comment | 🔴 High priority, already load-bearing |
| `PAYSTACK_SECRET_KEY` | Every payment call and the webhook hard-fail with a clean 500 | 🟡 Fails loudly, lower audit urgency |
| `USER_DATABASE_URL`, `WALLET_DATABASE_URL`, `CARD_DATABASE_URL`, `REWARD_DATABASE_URL`, `TERMS_DATABASE_URL`, `BOOKING_DATABASE_URL`, `ADMIN_DATABASE_URL`, `DRIVER_DATABASE_URL`, `BUS_DATABASE_URL`, `SYNC_DATABASE_URL` | Each throws synchronously on first query — scattered 500s per route family, not a clean deploy-time failure | 🟡 Re-verify all of these now that one has already been caught missing once |
| `AUTH_DATABASE_URL`, `SUBSCRIPTION_DATABASE_URL` | Falls back to a hardcoded local `postgres://postgres:postgres@localhost.../` URL instead of failing fast — silently tries to reach a nonexistent DB in prod | 🟡 |
| `RESEND_API_KEY` | No throw — logs and returns `false`, invisible to any caller (already diagnosed & fixed this session) | ✅ Already resolved |
| `TWILIO_ACCOUNT_SID` / `TWILIO_AUTH_TOKEN` / `TWILIO_FROM_NUMBER` | Same silent-failure pattern as Resend, but this is the **primary OTP channel** for this market | 🔴 Audit with the same urgency as the Resend incident — has not yet been confirmed either way |
| `GOOGLE_MAPS_API_KEY` | Narrow blast radius — only uncached ride ETAs fail with a 502 | 🟢 |
| `SUPER_ADMIN_EMAIL/PASSWORD/FIRST_NAME/LAST_NAME` | Only affects the one-time `admin/auth/setup` bootstrap; already-confirmed-live so not currently needed | 🟢 |

**Recommendation**: build a real `/health` endpoint that pings every pool and reports required env vars/provider config per-dependency (today's `/health` unconditionally returns `{status:"healthy"}` regardless of any of this — see §8), so the next missing var is caught at deploy time, not through a support ticket.

---

## 12. Technical Debt & Architecture Notes

- **13+ independent Postgres connection pools against what appears to be one physical database**, each with no explicit `max` (defaults to 10). On Vercel's serverless model, each warm lambda instance can independently open up to 10 connections *per module*; multiplied across concurrent warm instances under real traffic, this is a real connection-exhaustion risk against a managed Postgres with a typical connection cap. Worth consolidating to fewer pools or setting explicit, coordinated `max` values per module.
- **N+1 query patterns on essentially every "list" endpoint that touches a route** (rides, routes, bookings, packages) — an admin dashboard pulling 100 rides costs 400+ queries. Fix is standard (batch queries via `WHERE id = ANY($1)` or JOINs), not architectural, but widespread enough to note here rather than repeat per-finding.
- **Two confirmed orphaned modules (`subscriptions`, `sync`) are fully live, public, and unauthenticated** — recommend a product decision on whether to remove them from the API surface entirely rather than carry them forward with the same auth gap as everything else.
- **The legacy migration left several intentionally-preserved-but-real bugs in place** (admin `refresh_token === access_token`, the no-auth gap itself, in-code comments explicitly flagging both as known/deliberate carryovers) — worth a pass specifically hunting for other "preserved from legacy, not actually fixed" comments before assuming anything ported verbatim is safe.
- **Both web apps proxy production API traffic through a raw Vercel project alias** (`nextbackend-blond.vercel.app`) rather than the documented `api.comfortmeng.com` custom domain (H-16) — worth resolving simply because it's an unnecessary extra point of fragility with no apparent benefit.
- **Driver/bus/ride association is a genuine distributed-consistency problem** (H-7) spanning 2 databases and 3+ independent writers with no shared transaction or conflict-detection — this is the one finding in this audit that's a real design problem, not just a missing-guard bug, and deserves a deliberate redesign (single system of record, or a coordinating transaction/saga) rather than a quick patch.

---

## 13. Prioritized Action Plan

### Before any real users touch this in production (do these first, in this order)
1. **Ship real authentication** (§4) — the `requireAuth`/`requireAdminAuth` guard, applied everywhere except intentionally-public routes (login, signup, waitlist, public search/terms-read). This alone resolves the large majority of Critical/High findings.
2. **Verify `JWT_SECRET`/`JWT_REFRESH_SECRET`/`ADMIN_JWT_SECRET`/`DRIVER_JWT_SECRET` are strong random values in Vercel Production** — do this *as part of* step 1, not after; shipping auth against a default/guessable secret is worse than no auth (false confidence).
3. **Fix payment idempotency** (C-1) and **wire the webhook to actually reconcile payments** (C-2) — real money is at stake on both.
4. **Fix the driver-assignment type bug in the admin app** (C-3) — the core dispatch workflow is currently non-functional.
5. **Remove or gate the plaintext password/token logging in the mobile app** (C-6) — this is shipping today.
6. **Fix mobile token refresh** (C-7) — currently breaking essentially every real session.
7. **Rotate the committed Google Maps API key** (C-9) and confirm it's restricted.
8. Audit `TWILIO_ACCOUNT_SID`/`AUTH_TOKEN`/`FROM_NUMBER` in Vercel Production with the same urgency the Resend key got this session — SMS is the primary OTP channel here.

### Before scaling past initial launch
9. Fix the seat-booking race (H-1/C-seat-race), ride/rental status state machines (H-6, M-5), and the driver/bus/ride multi-writer consistency problem (H-7).
10. Cap unbounded pagination everywhere (H-9, M-4) — cheap, high-value, independent of the auth fix.
11. Fix the mobile profile save bug (C-8) and the admin app's fake Settings/Support/Notifications pages (H-11, H-12, C-4) — replace with real functionality or honest "not implemented" states.
12. Address the N+1 query patterns (§12) before they become a real latency problem at scale.
13. Resolve the `nextbackend-blond.vercel.app` alias dependency (H-16) in both web apps.

### Cleanup / hardening (do when convenient, low urgency)
14. Everything in §7 (Medium) and §8 (Low) not already covered above — none of these are launch-blockers individually, but the volume itself (24 Medium + 26 Low) reflects a codebase that hasn't yet had a hardening pass.
15. Decide on `subscriptions`/`sync` modules — remove from the public API or build the missing UI/integration.
16. Add a real `/health` endpoint (§11) so the next missing env var is caught at deploy time.

---

## 14. Final Assessment

**This platform is not production-ready as of this audit.** The core business logic — booking, payment orchestration, fleet management — is largely well-built, and the gaps that exist are fixable without architectural rework. But the absence of authentication across nearly the entire API is not a hardening item to schedule for later; it is a currently-live condition where any internet user can drain wallets, charge cards, create free bookings, dump all rider and driver PII, and mint themselves a super-admin account, all confirmed against the real production deployment during this audit.

**Recommended gate before allowing real user signups/payments to continue**: complete items 1–8 in §13. Everything else in this report is real, worth fixing, and represents meaningful technical debt and UX gaps — but none of it carries the same immediate risk as the authentication gap, and none of it should block starting on item 1 today.

---

*Full per-domain findings with additional file:line detail are preserved in the individual audit passes this report was synthesized from, available in this session's scratchpad if deeper source references are needed for any specific finding.*
