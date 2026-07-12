---
title: "frontend-modules"
date: 2026-07-12
categories: ["System Guide"]
---

# SumiCare Frontend Module Reference

This document is a technical reference for the SumiCare Angular frontend  
(`apps/sumicare-web/src/app`). It is derived directly from component and service  
source. Every selector, input/output, signal, observable, and HTTP endpoint  
listed here was read from the actual code.

## Conventions and global facts

* **Components** are Angular standalone components using  
  `ChangeDetectionStrategy.OnPush` and signals for local state.
* **API base URL** is `environment.apiBaseUrl`, a build-time placeholder  
  (`'${API_BASE_URL}'` substituted by `scripts/set-env.js`). In local development it  
  resolves empty, so every HTTP path is a relative `/api/**` request served through the  
  dev proxy.
* **Organization scoping.** Public endpoints are slugged  
  (`environment.defaultOrganizationSlug`). Internal realtime topics are suffixed  
  with the organization id resolved from the JWT (`AuthService.organizationId()`,  
  which base64-decodes the `org` claim of the access token).
* **Realtime** uses STOMP over WebSocket via `core/realtime/stomp.service.ts`  
  (`@stomp/rx-stomp`). Topics seen in code:  
  `/topic/bookings/{orgId}`, `/topic/orders/{orgId}`,  
  `/topic/messages/{orgId}`, `/topic/feedback/{orgId}`,  
  `/topic/decking-updates/{orgId}`, `/topic/room-updates/{orgId}`.
* **Auth tokens** are held in memory only (`AuthService.session`, a signal). The  
  refresh token lives in an httpOnly cookie; requests that need it use  
  `withCredentials: true`.
* **No emojis / no code comments** are used in the source, per project rules.

---

## Core services (`core/*`)

### `AuthService` — `core/auth/auth.service.ts`

In-memory session holder and auth API client.

* **State:** `session = signal<AuthSession | null>` where  
  `AuthSession = { accessToken, role, expiresAt }`.

* **Methods and endpoints:**
  
  |Method|Endpoint|Returns|
  |------|--------|-------|
  |`bootstrapSession()`|`POST /api/auth/refresh` (withCredentials)|applies token, swallows error|
  |`login(username, password)`|`POST /api/auth/login`|`LoginResponse { mfaRequired, challengeId, email, token }`|
  |`verifyMfa(challengeId, code)`|`POST /api/auth/mfa/verify`|`TokenResponse`|
  |`resendMfa(challengeId)`|`POST /api/auth/mfa/resend`|`void`|
  |`refresh()`|`POST /api/auth/refresh`|`TokenResponse`, deduped via `shareReplay`|
  |`logout()`|`POST /api/auth/logout`|clears `session`|

* **Helpers:** `isAuthenticated()` (checks `expiresAt > Date.now()`),  
  `hasRole(roles[])`, `organizationId()` (decodes `org` claim).

* `bootstrapSession()` is wired as an app initializer in `app.config.ts`, so a  
  refresh attempt runs on app load to restore the session from the cookie.

### `authGuard` / `roleGuard(roles)` — `core/auth/auth.guard.ts`

`CanActivateFn`s. `authGuard` requires `isAuthenticated()`; `roleGuard` also  
requires `hasRole(allowedRoles)`. Both redirect to `/sumicare/login` on failure.

### `authInterceptor` — `core/auth/auth.interceptor.ts`

Attaches `Authorization: Bearer <accessToken>` to requests whose URL contains  
`/api/`. On `401` (except the refresh endpoint itself) it calls  
`AuthService.refresh()`, retries the original request with the new token, and on  
failure navigates to `/sumicare/login`.

### `IdleTimeoutService` — `core/auth/idle-timeout.service.ts`

* Logs out after 60 minutes idle (`IDLE_MS`). Activity events  
  (`mousemove`, `keydown`, `click`, `scroll`, `touchstart`) reset the timer  
  (throttled to 30s).
* Runs a refresh loop every 60s: if the session is within 3 minutes of expiry and  
  the user was active within the last 2 minutes, it calls `AuthService.refresh()`.
* On expiry it calls `logout()` and routes to `/sumicare/login?reason=idle`.
* Started/stopped by `InternalShellComponent`.

### `StompService` — `core/realtime/stomp.service.ts`

* `connect(token)` configures and activates an `RxStomp` client against  
  `environment.wsUrl` (http→ws rewrite), passing `Authorization` connect header,  
  `reconnectDelay: 5000`.
* `watch<T>(destination)` returns an `Observable<T>` that JSON-parses message  
  bodies. `disconnect()` deactivates.

### `NotificationFeedService` — `core/notifications/notification-feed.service.ts`

* Unread-count signals: `unreadBookings`, `unreadOrders`, `unreadMessages`,  
  `unreadFeedback`; plus an `events$` Subject of `NotificationEvent`.
* `start(orgId)` seeds counts from  
  `GET /api/contact-messages/unread-count` and `GET /api/feedback/unread-count`,  
  then subscribes to `/topic/{bookings|orders|messages|feedback}/{orgId}`,  
  incrementing the matching counter and emitting on `events$`.
* `markRead(key)` zeroes a counter; `unreadFor(key)` reads it. Used by  
  `InternalShellComponent` for sidebar badges and toast notifications.

### `BrandingService` — `core/branding/branding.service.ts`

* `branding = signal<OrganizationBranding | null>`.
* `loadPublicBranding(slug)` → `GET /api/public/branding/{slug}`.
* `loadCurrentBranding()` → `GET /api/organization/branding`.
* `applyTheme(branding)` sets CSS custom properties on `document.documentElement`  
  (`--sumi-primary/-secondary/-accent`, app fonts, `--sumi-login-bg`) and updates  
  the favicon. `resetToDefault()` removes them.

### `PaymentDetailsService` — `shared/components/payment-details/payment-details.service.ts`

Promise-based modal bridge. `open(method, amount)` shows the  
`PaymentDetailsModalComponent` and resolves with captured `PaymentDetails`  
(card/GCash fields) or `null` on cancel. `respond(result)` is called by the modal.

---

## Public site features

All public routes are children of `PublicShellComponent` at path `''`, except  
`pay/authorize` which is a top-level route.

### Public shell — `LandingComponent` host

* **Route:** `''` (layout wrapper).
* **Component:** `PublicShellComponent`, selector `sumi-public-shell`,  
  template `public-shell.component.html`.
* **Injects:** `BrandingService` (exposed as `branding`), `Router`.
* **Imports:** `RouterOutlet`, `RouterLink`, `RouterLinkActive`, `ToastHostComponent`.
* **Signals:** `menuOpen`, `routeToken` (bumped per navigation to retrigger the  
  `routeFade` animation). On each `NavigationEnd` it closes the mobile menu and  
  scrolls to top.
* **Interactions:** `toggleMenu()` / `closeMenu()` for the responsive nav.

### Home — `LandingComponent`

* **Route:** `''` (index child).
* **Selector:** `sumi-landing`, template `landing.component.html`.
* **Imports:** `RouterLink`, plus reveal/parallax/counter directives  
  (`SumiRevealDirective`, `SumiParallaxDirective`, `SumiCounterDirective`).
* **Purpose:** Marketing landing page. Pure presentational — exposes static  
  `heroImage`, `mosaicLarge`, `mosaicSmall[]`, `stripImages[]` data. No HTTP, no  
  signals, no services.

### About — `AboutComponent`

* **Route:** `about`. **Selector:** `sumi-public-about`.
* **Imports:** `RouterLink`, `SumiRevealDirective`. Static content only.

### Services — `ServicesComponent`

* **Route:** `services`. **Selector:** `sumi-services`.
* **Injects:** `HttpClient`. **Imports:** `RouterLink`, `SumiRevealDirective`.
* **Signals:** `services = signal<ServiceItem[]>`, `loadingInitial`.
* **Endpoint:** `GET /api/public/services/{slug}` → `ServiceItem[]`  
  (id, code, name, durationMinutes, price, category, vip, fixedRate, description,  
  imageUrl).
* **Behavior:** `descriptionFor(s)` falls back to a built-in keyword→copy map when  
  a service has no description; `photoFor(index)` rotates three stock images.

### Packages — `PackagesComponent`

* **Route:** `packages`. **Selector:** `sumi-public-packages`.
* **Imports:** `RouterLink`, `DecimalPipe`, `SumiRevealDirective`.
* **Purpose:** Static price book. Exposes `highlights[]` (package descriptions),  
  `fullPackageRates[]` / `massagePackageRates[]` (weekday/weekend regular & promo),  
  and `extras[]`. No HTTP.

### Recommendation — `RecommendationComponent`

* **Route:** `recommendation`. **Selector:** `sumi-recommendation`.
* **Purpose:** A **weighted-scoring quiz UI**. The user answers five questions;  
  the answers are posted to the backend scoring engine, which returns a primary  
  recommendation plus alternatives and a disclaimer.
* **Injects:** `HttpClient`, `Router`. **Imports:** `FormsModule`, `RouterLink`,  
  `SumiRevealDirective`.
* **Static quiz model:** `questions[]` with codes  
  `PRESSURE`, `FOCUS_AREA`, `TEXTURE`, `DURATION`, `GOAL`, each with option codes  
  (e.g. `LIGHT/MEDIUM/FIRM/VERY_FIRM`, `FULL_BODY/BACK/FEET/HEAD`, etc.).
* **Signals:** `answers = signal<Record<string,string>>`, `result`,  
  `submittedOnce`, `submitting`; computed `allAnswered`.
* **Endpoint:** `POST /api/public/recommendation/{slug}` with body  
  `{ answers: [{ questionCode, optionCode }] }` → `RecommendationResponse`  
  (`primary`, `alternatives[]`, `rationale`, `aiUsed`, `disclaimer`).
* **State flow:** `setAnswer()` updates the `answers` signal → `allAnswered`  
  recomputes to enable submit → `submit()` posts and sets `result` → template  
  renders the recommended service(s) and disclaimer.
* **Interactions:** `goToBook(serviceId)` navigates to `/book` with  
  `queryParams: { serviceId }`.

### Book — `BookComponent`

* **Route:** `book`. **Selector:** `sumi-book`.

* **Purpose:** Public multi-attendee booking form for soft (`SOFT`) and hard  
  (`HARD`, prepaid) reservations, with voucher application and a PayMongo-style  
  payment flow.

* **Injects:** `HttpClient`, `ActivatedRoute`, `Router`, `PaymentDetailsService`.

* **Imports:** `FormsModule`, `DecimalPipe`, `QRCodeComponent` (angularx-qrcode),  
  `PaymentDetailsModalComponent`, `RouterLink`.

* **Key signals:** `services`, `packages`, `bookingItems` (array of  
  `BookingItemForm` with per-attendee tier/locker/gender), `appliedVoucher`,  
  `confirmation`, `bookingRef`, `orNumber`, plus confirmation field signals.

* **Computed:** `grossTotal`, `voucherDiscount`, `estimatedTotal`.

* **Endpoints:**
  
  |Action|Endpoint|
  |------|--------|
  |Load services|`GET /api/public/services/{slug}`|
  |Load packages|`GET /api/public/packages/{slug}` (filters `active`)|
  |Check voucher|`GET /api/public/vouchers/{slug}/check?code=&email=`|
  |Create booking|`POST /api/public/bookings/{slug}`|
  |Initiate payment|`POST /api/public/bookings/{slug}/payment/initiate`|
  |Confirm payment|`POST /api/public/bookings/{slug}/payment/confirm`|

* **State flow:** package/tier/room selection mutate `bookingItems` (cloned to  
  preserve immutability) → totals recompute → `submit()` validates, builds the  
  payload, and posts. For `HARD` reservations with an `orderId`, it opens the  
  GCash details modal when needed and starts payment; `mock_` intents redirect to  
  `/pay/authorize`, real intents redirect to `res.redirectUrl`. On return,  
  `ngOnInit` detects `paymongoReturn` and calls the confirm endpoint.

* **Interactions:** `addItem()`/`removeItem()`, `onItemPackageChange()`,  
  `setItemRoom()`, `setItemAttendeeTier()`, `setItemAttendeeGender()`,  
  `applyVoucher()`, `bookAnother()` (resets the form).

### Visit — `VisitComponent`

* **Route:** `visit`. **Selector:** `sumi-public-visit`.
* **Injects:** `DomSanitizer`. Renders a sanitized Google Maps embed (`mapUrl`)  
  plus static directions content. No HTTP.

### Feedback (public) — `FeedbackComponent`

* **Route:** `feedback`. **Selector:** `sumi-public-feedback`.
* **Injects:** `HttpClient`, `ActivatedRoute`. **Imports:** `FormsModule`.
* **Signals:** `rating`, `sessionRef`, `orNumber`, `submitted`, `submitting`,  
  `loadingInitial`, `recent = signal<Feedback[]>`.
* **Endpoints:**
  * `GET /api/public/feedback/{slug}` → recent feedback list.
  * `POST /api/public/feedback/{slug}` with  
    `{ ratingStars, comment, nickname, orNumber }`.
* **Behavior:** reads `session` / `slip` / `or` query params to pre-fill the  
  reference (the `or` value is sent as `orNumber`). Star buttons set `rating`;  
  submit posts then reloads the recent list. `stars(n)` renders filled/empty stars.

### Contact — `ContactComponent`

* **Route:** `contact`. **Selector:** `sumi-contact`.
* **Injects:** `DomSanitizer`, `HttpClient`. **Imports:** `FormsModule`.
* **Endpoint:** `POST /api/public/contact/{slug}` with `{ name, email, message }`.
* **Behavior:** client-side required-field and email-regex validation; handles  
  `429` (rate limit) and `400` distinctly. Renders a sanitized map embed.  
  `sendAnother()` resets the success state.

### Cancel — `CancelComponent`

* **Route:** `cancel`. **Selector:** `sumi-cancel`.
* **Purpose:** Self-service booking cancellation via an emailed verification code.
* **Imports:** `FormsModule`, `DecimalPipe`, `DatePipe`, `RouterLink`.
* **Signals:** `step = signal<'request'|'code'|'review'|'done'>`, `busy`, `error`,  
  `notice`, `details = signal<CancellationDetails | null>`.
* **Endpoints (base `…/api/public/bookings/{slug}/cancel`):**
  * `POST …/request` `{ reference, email }` → sends code, advances to `code`.
  * `POST …/verify` `{ reference, email, code }` → `CancellationDetails`,  
    advances to `review`.
  * `POST …/confirm` `{ reference, email, code }` → advances to `done`.
* **Interactions:** `requestCode()`, `resendCode()`, `verifyCode()`,  
  `confirmCancel()`, `restart()`. The `step` signal drives a wizard template.

### Terms — `TermsComponent`

* **Route:** `terms`. **Selector:** `sumi-public-terms`. Static content,  
  imports `RouterLink` and `SumiRevealDirective`.

### PayMongo authorize — `PaymongoAuthorizeComponent`

* **Route:** `pay/authorize` (top-level, outside the public shell).
* **Selector:** `sumi-paymongo-authorize`.
* **Purpose:** Mock payment-gateway authorization page used for `mock_` intents.
* **Injects:** `ActivatedRoute`. Reads `intent`, `amount`, `method`, `return`  
  query params into signals.
* **Interactions:** `authorize()` → completes with `succeeded`,  
  `cancel()` → `cancelled`; both append `status=...` to the return URL and set  
  `window.location.href` (falling back to `/sumicare/app/cashier`). No HTTP.

---

## Auth features (`/sumicare/...`)

### Login — `LoginComponent`

* **Route:** `sumicare/login`. **Selector:** `sumi-login`.
* **Injects:** `AuthService`, `Router`, `ActivatedRoute`.
* **Imports:** `FormsModule`, `PasswordInputComponent`, `ContactAdminModalComponent`.
* **Signals:** `busy`, `error`, `verifiedBanner` (`success`/`expired`),  
  `idleBanner`, `showContactModal`, `mfaRequired`, `maskedEmail`, `resendNotice`.
* **Flow:** `submit()` calls `auth.login()`. If the response requires MFA it stores  
  `challengeId`, sets `maskedEmail`, and flips `mfaRequired` to show the code form;  
  otherwise it navigates to `/sumicare/app/dashboard`. `verify()` calls  
  `auth.verifyMfa()`, `resend()` calls `auth.resendMfa()`,  
  `backToSignIn()` resets the MFA view.
* Reads `verified` and `reason=idle` query params for banners. Opens the  
  contact-admin modal for credential help.

### Verify (email/MFA link) — `VerifyComponent`

* **Route:** `sumicare/verify`. **Selector:** `app-verify`.
* **Injects:** `ActivatedRoute`, `HttpClient`. **Imports:** `RouterLink`.
* **Signal:** `state = signal<'loading'|'success'|'expired'|'invalid'>`.
* **Behavior:** If `verified=1|already` → success, `verified=expired` → expired;  
  otherwise it calls `GET /api/auth/verify?token=...` and sets success/invalid by  
  response.

### Reset password — `ResetPasswordComponent`

* **Route:** `sumicare/reset-password`. **Selector:** `sumi-reset-password`.
* **Imports:** `FormsModule`, `PasswordInputComponent`, `PasswordStrengthComponent`,  
  `RouterLink`.
* **Signals:** `submitting`, `error`, `success`.
* **Endpoint:** `POST /api/auth/reset-password` `{ token, newPassword }`.  
  Reads `token` from query params; validates password match client-side.

### Invite (activate account) — `InviteComponent`

* **Route:** `sumicare/invite`. **Selector:** `sumi-invite`.
* **Imports:** `FormsModule`, `RouterLink`, `PasswordInputComponent`,  
  `PasswordStrengthComponent`.
* **Signals:** `submitting`, `error`, `success`, `invalidToken`.
* **Endpoint:** `POST /api/auth/invitations/redeem` `{ token, password }`.  
  Distinguishes used/expired/invalid tokens from other errors.

### Contact-admin modal — `ContactAdminModalComponent`

* **Selector:** `sumi-contact-admin-modal` (used inside Login).
* **Input:** `show` (setter that toggles internal `open` and clears state).  
  **Output:** `closed: EventEmitter<void>`.
* **Endpoint:** `POST /api/auth/contact-admin-reset` `{ name, email, message }`.
* `canSubmit()` requires name and email; `close()` emits `closed`.

---

## Internal shell and dashboard (`/sumicare/app`)

The `app` route is guarded by `authGuard` and renders `InternalShellComponent`.  
Children are individually role-guarded (see routing table at the end).

### Internal shell — `InternalShellComponent`

* **Route:** `sumicare/app` (layout). **Selector:** `sumi-internal-shell`.
* **Injects:** `AuthService`, `Router`, `BrandingService`, `ConfirmService`,  
  `IdleTimeoutService`, `StompService`, `NotificationFeedService`, `DestroyRef`.
* **Imports:** `RouterOutlet`, `RouterLink`, `RouterLinkActive`,  
  `NotificationToastComponent`, `ToastHostComponent`.
* **Signals:** `sidebarOpen` (persisted to `localStorage` key  
  `sumi_sidebar_open`), `routeToken`, plus `session` (from `AuthService`).
* **`visibleGroups` (computed):** filters the static `groups` nav definition by  
  the current role. Role tiers: `STAFF_ROLES` = RECEPTIONIST+,  
  `MANAGER_PLUS`, `ADMIN_PLUS`.
* **Lifecycle (`ngOnInit`):** loads current branding, restores sidebar state,  
  starts idle timeout, **connects STOMP** with the access token, and starts the  
  notification feed for the org id. On each `NavigationEnd` it bumps `routeToken`  
  and clears the unread badge for the visited section.
* **`ngOnDestroy`:** stops idle timeout, stops the feed, disconnects STOMP.
* **Badges:** `unreadFor(route)` maps `bookings/orders/messages/admin-feedback`  
  routes to the corresponding feed counter.
* **Interactions:** `toggleSidebar()`, `signOut()` (confirm dialog → `logout()` →  
  navigate to login).

### Dashboard — `DashboardComponent`

* **Route:** `dashboard`. **Selector:** `sumi-dashboard`.
* **Injects:** `HttpClient`, `AuthService`, `BrandingService`, `StompService`.
* **Imports:** `RouterLink`, `DecimalPipe`.
* **Signals:** `recentReservations`, `todaysBookings`, `activeSessions`,  
  `completedSessions`, `lineupCount`, `occupiedBeds`, `dailyRevenue`,  
  `revenueSeries`, `loading`. Computed role flags `isReceptionist/isManager/isAdmin`  
  and `sparklinePoints` (SVG polyline string from `revenueSeries`).
* **Endpoints:**
  * `GET /api/dashboard/summary` → `DashboardSummary` (today bookings, active /  
    completed sessions, therapists in lineup, beds occupied).
  * `GET /api/dashboard/recent-reservations` → `RecentReservation[]`.
  * `GET /api/cashier/ledger/daily-revenue?from=&to=` (manager+ only) → 14-day net  
    revenue series for the sparkline.
* **Realtime:** subscribes to `/topic/bookings/{orgId}` and `/topic/orders/{orgId}`;  
  any message debounces (300 ms) a reload of the summary and recent reservations.

---

## Operations features

### Bookings — `BookingsComponent`

* **Route:** `bookings` (STAFF+). **Selector:** `sumi-bookings`.

* **Purpose:** Day list of bookings with start-session, end/extend, edit, and  
  expandable per-order/per-attendee detail. Heavy realtime consumer.

* **Injects:** `HttpClient`, `Router`, `ConfirmService`, `StompService`,  
  `AuthService`, `ToastService`.

* **Imports:** `FormsModule`, `RouterLink`, `SortableColumnDirective`,  
  `SortIconComponent`, `LockerLabelPipe`, `PaginatorComponent`.

* **Key signals:** `selectedDate`, `bookings`, `services`, `lineup`, `rooms`,  
  `ordersByBooking` (Map), `expandedBookingId`, the entire start-session form  
  (`startBooking`, `startPrimaryTherapistId`, `startSecondaryTherapistId`,  
  `startRoomId`, `startBedId`, `startSpecificallyRequested`, etc.), plus edit-form  
  signals and `sortState`/`searchTerm`/`currentPage`.

* **Computed:** `filteredBookings`, `sortedBookings`, `pagedBookings`,  
  `availableTherapists`, `onBreakTherapists`, `selectedStartService`,  
  `needsSecondary`, `canStart`, `startItemId`, `filteredStartRooms`,  
  `pickedRoomNumber`, `pickedBedLabel`.

* **Endpoints:**
  
  |Action|Endpoint|
  |------|--------|
  |Load bookings for date|`GET /api/bookings?from=&to=`|
  |Services reference|`GET /api/services`|
  |Lineup reference|`GET /api/decking/lineup`|
  |Rooms reference / refresh|`GET /api/rooms`|
  |Order for a booking|`GET /api/cashier/orders/{orderId}`|
  |Order by booking|`GET /api/cashier/orders/by-booking/{bookingId}`|
  |Start booking session|`POST /api/bookings/{bookingId}/sessions`|
  |Start attendee session|`POST /api/bookings/attendees/{attendeeId}/sessions`|
  |Session by booking|`GET /api/sessions/by-booking/{bookingId}`|
  |End session|`POST /api/sessions/{id}/end`|
  |Extend session|`POST /api/sessions/{id}/extend?minutes=60`|
  |Edit booking|`PATCH /api/bookings/{id}`|
  |Export|`GET /api/bookings/export.csv?from=&to=` (blob)|

* **Realtime:** four subscriptions —  
  `/topic/bookings/{orgId}` and `/topic/orders/{orgId}` (debounced 300 ms reload),  
  `/topic/room-updates/{orgId}` (debounced 200 ms rooms reload),  
  `/topic/decking-updates/{orgId}` (refresh lineup).

* **Start-session UX:** `openStart()`/`openStartForAttendee()` populate the form;  
  bed selection enforces gender/occupancy rules  
  (`isRoomSelectableForGender`, `isBedSelectableForGender`, `bedClass`),  
  `pickBed()` sets room/bed. `submitStart()` confirms then posts to the booking or  
  attendee session endpoint and reloads. Errors surface via `ToastService`.

### Reception (room map) — `ReceptionComponent`

* **Route:** `reception` (STAFF+). **Selector:** `sumi-reception`.
* **Purpose:** Live bed/room occupancy map.
* **Injects:** `HttpClient`, `StompService`, `AuthService`. **Imports:** `LockerLabelPipe`.
* **Signal:** `rooms = signal<RoomView[]>` (each bed carries an  
  `occupancy` record with status, genderLock, clientNickname, lockerNumber, etc.).
* **Endpoint:** `GET /api/rooms`.
* **Realtime:** subscribes to `/topic/room-updates/{orgId}` and reloads on any  
  message.
* **Rendering:** `bedClass(bed)` colors by `genderLock` (M → slate, F → pink,  
  otherwise white); `elapsed(startedAtMillis)` formats time since session start.

### Lineup (decking) — `LineupComponent`

* **Route:** `lineup` (STAFF+). **Selector:** `sumi-lineup`.

* **Purpose:** Therapist queue management — flags, skip/break, rotation, manual  
  insertion of backups.

* **Injects:** `HttpClient`. **Imports:** `FormsModule`.

* **Signals:** `lineup`, `allTherapists`, `allShifts`, `showAddModal`.  
  Computed: `activeTherapists`, `activeLineup`, `onBreakLineup`, `groupedLineup`  
  (groups active queue by shift label).

* **Endpoints:**
  
  |Action|Endpoint|
  |------|--------|
  |Load lineup|`GET /api/decking/lineup`|
  |Therapists / shifts|`GET /api/therapists`, `GET /api/shifts`|
  |Set flag|`POST /api/decking/{id}/flag?flag=`|
  |Skip (break)|`POST /api/decking/{id}/skip?minutes=30`|
  |Cancel skip|`POST /api/decking/{id}/skip/cancel`|
  |Rotate to end|`POST /api/decking/{id}/rotate`|
  |Remove from lineup|`DELETE /api/decking/{id}`|
  |Add to lineup|`POST /api/decking/{id}?shiftId=`|

* **Rendering:** `statusLabel`/`statusClass` reflect skipped, on-call, and flags  
  (`REQUESTED`, `BACKUP`, `SCRUB`); `avatarClass(gender)` colors the avatar.  
  Each mutation reloads the lineup. (This component reloads on action; it does not  
  itself subscribe to STOMP — realtime lineup refresh is handled by other views.)

### Cashier (POS) — `CashierComponent`

* **Route:** `cashier` (STAFF+); `pos` redirects here. **Selector:** `sumi-cashier`.

* **Purpose:** Build an order (packages + attendees + room type + discounts +  
  voucher), capture payments, and create or edit an order. Also handles PayMongo  
  redirect returns.

* **Injects:** `HttpClient`, `Router`, `ActivatedRoute`, `ConfirmService`,  
  `PaymentDetailsService`.

* **Imports:** `FormsModule`, `DecimalPipe`, `RouterLink`,  
  `PaymentDetailsModalComponent`.

* **Key signals:** `searchResults`, `selectedClient`, `clientLocked`, `packages`,  
  `cart = signal<CartItem[]>`, `weekend`, `roomAvailability`, `payments`,  
  `chargeLedgers`, `discountConfig`, `savedTemplates`, `discountSummary`,  
  `voucherId`, `editingOrderId`, `submitting`, `error`, `tax`.

* **Computed totals:** `groupBooking`, `roomSurcharge`, `totalAttendees`,  
  `itemsSubtotal`, `subtotal`, `attendeeDiscountTotal`, `totalDiscount`, `total`,  
  `paid`, `due`; plus `chargeOptions`, `hasVoucherApplied`, `hasManualDiscount`.

* **Endpoints:**
  
  |Action|Endpoint|
  |------|--------|
  |Charge ledgers|`GET /api/cashier/ledger/accounts`|
  |Packages|`GET /api/cashier/packages`|
  |Discount templates|`GET/POST/DELETE /api/cashier/discount-templates[/{id}]`|
  |Room availability|`GET /api/cashier/room-availability`|
  |Clients search / register|`GET /api/clients?q=`, `POST /api/clients`|
  |Voucher check|`GET /api/vouchers/check?code=&subtotal=&clientId=`|
  |Load order for edit|`GET /api/cashier/orders/{id}`|
  |Create order|`POST /api/cashier/orders`|
  |Update order|`PUT /api/cashier/orders/{id}`|
  |Extra payment|`POST /api/cashier/orders/{id}/payments`|
  |PayMongo initiate|`POST /api/cashier/orders/{id}/paymongo/initiate`|
  |PayMongo confirm|`POST /api/cashier/orders/{id}/paymongo/confirm`|

* **Flow:** `addPackage()` builds a `CartItem` with attendees seeded from the  
  default tier; `onAttendeeMassageChange()`/`toggleWeekend()` reprice line items;  
  discounts and vouchers (mutually exclusive) feed `discountSummary`.  
  `addPayment()` validates against `total`/`paid`, opening the GCash details modal  
  when needed, and queues an `AddedPayment`. `checkout()` confirms, then:
  
  * if a single gateway payment (`GCASH/CREDIT/DEBIT`) is queued, creates/updates  
    the order without an initial payment and starts PayMongo (`startPayMongo`,  
    which redirects to `/pay/authorize` for `mock_` intents or `redirectUrl`);
  * otherwise creates/updates the order (first payment inlined as `initialPayment`  
    on create) and records any extra payments sequentially, then routes to the  
    order detail page.
* **Returns:** `ngOnInit` detects `paymongoReturn` and confirms via  
  `paymongo/confirm`, routing to the order detail (with `paymentError` on failure).

### Orders list — `OrdersListComponent`

* **Route:** `orders` (STAFF+). **Selector:** `sumi-orders-list`.
* **Injects:** `HttpClient`. **Imports:** `DecimalPipe`, `FormsModule`,  
  `RouterLink`, `PaginatorComponent`.
* **Signals:** `orders`, `loading`, `filter` (default `PAID`),  
  `sortColumn`/`sortDirection`, `searchTerm`, `currentPage`/`pageSize`. Computed  
  `filteredOrders`, `sortedOrders`, `paginatedOrders`.
* **Endpoint:** `GET /api/cashier/orders[?status=OPEN|PAID|CANCELLED]` (no param  
  for `ALL`).
* **Behavior:** `setFilter()` reloads; client-side search/sort/paginate.  
  `slipId(o)` resolves the treatment-slip id from order or attendees;  
  `itemSummary(o)` builds a short label. `statusClass` colors order status.

### Order detail — `OrderDetailComponent`

* **Route:** `orders/:id` (STAFF+). **Selector:** `sumi-order-detail`.

* **Injects:** `HttpClient`, `ActivatedRoute`, `Router`, `ConfirmService`,  
  `PaymentDetailsService`.

* **Imports:** `DecimalPipe`, `FormsModule`, `RouterLink`,  
  `PaymentDetailsModalComponent`, `LockerLabelPipe`.

* **Signals:** `order`, `audits`, `error`, `paymentNotice`, `busy`, `loading`,  
  `showCancel`, `showRefund`, plus inline payment/cancel/refund form fields.

* **Endpoints:**
  
  |Action|Endpoint|
  |------|--------|
  |Load order|`GET /api/cashier/orders/{id}`|
  |Order audit trail|`GET /api/audit-logs/by-target?entity=ORDER&id={id}`|
  |Record payment|`POST /api/cashier/orders/{id}/payments`|
  |Mark paid|`POST /api/cashier/orders/{id}/mark-paid`|
  |Reopen|`POST /api/cashier/orders/{id}/open`|
  |Cancel payment|`POST /api/cashier/orders/{id}/cancel-payment`|
  |Cancel order|`POST /api/cashier/orders/{id}/cancel`|
  |Refund|`POST /api/cashier/orders/{id}/refund`|
  |PayMongo initiate/confirm|`POST /api/cashier/orders/{id}/paymongo/{initiate,confirm}`|
  |Receipt PDF|`GET /api/cashier/orders/{id}/receipt.pdf` (blob download)|

* **Behavior:** CASH payments post directly; GCash/card payments go through the  
  PayMongo flow (with the same `mock_`→`/pay/authorize` redirect logic as the  
  cashier). Cancel/refund use confirm dialogs and small modals. `editOrder()`  
  routes to the cashier with `?orderId=`.

### Messages — `MessagesComponent`

* **Route:** `messages` (STAFF+). **Selector:** `sumi-messages`.
* **Injects:** `HttpClient`. **Imports:** `FormsModule`.
* **Signals:** `messages`, `filter` (`all`/`unread`, default `unread`),  
  `searchTerm`, `selected`, `loading`, `exporting`. Computed `filteredMessages`.
* **Endpoints:**
  * `GET /api/contact-messages?unread=` → list.
  * `POST /api/contact-messages/{id}/mark-read` → updated message.
  * `GET /api/contact-messages/export.csv?from=&to=` (blob).
* **Behavior:** `open(m)` selects a message; `markRead(m)` updates it in place.

### Registered clients — `RegisteredClientsComponent`

* **Route:** `registered-clients` (STAFF+). **Selector:** `sumi-registered-clients`.
* **Injects:** `HttpClient`, `ConfirmService`. **Imports:** `FormsModule`,  
  `PaginatorComponent`.
* **Signals:** `clients`, `loading`, `searchTerm`, `currentPage`/`pageSize`,  
  `showAdd`, `saving`, `addError`. Computed `filteredClients`, `paginatedClients`.
* **Endpoints:**
  * `GET /api/clients` → list.
  * `POST /api/clients` → create (nickname + email required).
  * `DELETE /api/clients/{id}` → remove from active list (history preserved).
* **Behavior:** `openAdd()`/`submitAdd()` for registration; `remove()` confirms  
  then deletes. Clients are identified by nickname (no real names).

### Treatment slips list — `TreatmentSlipsComponent`

* **Route:** `treatment-slips` (STAFF+). **Selector:** `sumi-treatment-slips`.
* **Injects:** `HttpClient`. **Imports:** `FormsModule`, `RouterLink`,  
  `SortableColumnDirective`, `SortIconComponent`.
* **Signals:** `selectedDate`, `slips`, `sortState`, `searchTerm`; computed  
  `filteredSlips`, `sortedSlips`.
* **Endpoints:**
  * `GET /api/treatment-slips?from=&to=` (day bounds in `+08:00`).
  * `GET /api/treatment-slips/export.csv?from=&to=` (blob).
* **Behavior:** date picker reloads; client-side filter/sort. Shows TSN, nickname,  
  service, therapists, room, time range, VIP flag.

### Treatment slip detail — `TreatmentSlipDetailComponent`

* **Route:** `treatment-slips/:id` (STAFF+). **Selector:** `sumi-treatment-slip-detail`.
* **Injects:** `HttpClient`, `ActivatedRoute`, `ToastService`.
* **Imports:** `RouterLink`, `QRCodeComponent`, `DecimalPipe`, `FormsModule`,  
  `LockerLabelPipe`. Has a component CSS file (`treatment-slip-detail.component.css`).
* **Signals:** `slip`, `editing`, `saveError`; plus an `edit` form object.
* **Endpoints:**
  * `GET /api/treatment-slips/{id}` → detail.
  * `PATCH /api/treatment-slips/{id}` → save edits (VIP-only fields nulled for  
    non-VIP slips).
  * `GET /api/treatment-slips/{id}/slip.pdf` (blob download).
* **Behavior:** `startEdit()`/`saveEdit()`/`cancelEdit()`; `print()` triggers  
  `downloadPdf()`. `feedbackUrl(tsn)` builds a public feedback link  
  (`/feedback?slip={tsn}`) used by the QR code.

### Attendance — `AttendanceComponent`

* **Route:** `attendance` (STAFF+). **Selector:** `sumi-attendance`.
* **Injects:** `HttpClient`. **Imports:** `FormsModule`.
* **Signals:** `records`, `loading`, `fromDate`, `toDate`.
* **Endpoint:** `GET /api/attendance?from=&to=`.
* **Behavior:** date-range filter; `eventBadgeClass` colors  
  `CLOCK_IN/CLOCK_OUT/ABSENT/DAY_OFF`. Read-only viewer.

---

## Finance features (MANAGER+)

### Reports — `ReportsComponent`

* **Route:** `reports`. **Selector:** `sumi-reports`.

* **Purpose:** Tabbed reporting with CSV export across services, daily, monthly,  
  commissions (shift/daily/cutoff/monthly), and decking.

* **Injects:** `HttpClient`, `ToastService`. **Imports:** `FormsModule`,  
  `DecimalPipe`, `PaginatorComponent`.

* **Signals:** `tab`, `commissionTab`, `loading`, the various report signals  
  (`servicesReport`, `dailyReport`, `monthlyReport`, `commissionShiftReport`,  
  `commissionDailyReport`, `commissionCutoffReport`, `commissionMonthlyReport`,  
  `deckingReport`), `shifts`, and `dailyPage`. Computed `pagedDailyRows`.

* **Endpoints (each has a paired `/export.csv`):**
  
  |Report|Endpoint|
  |------|--------|
  |Shifts reference|`GET /api/shifts`|
  |Cutoff services|`GET /api/reports/cutoff/services?from=&to=&shiftId=`|
  |Daily|`GET /api/reports/daily?date=`|
  |Monthly|`GET /api/reports/monthly-detailed?year=&month=`|
  |Commissions by shift|`GET /api/reports/commissions/shift?shiftId=&date=`|
  |Commissions daily|`GET /api/reports/commissions/daily?date=`|
  |Commissions cutoff|`GET /api/reports/commissions/cutoff?year=&month=&half=`|
  |Commissions monthly|`GET /api/reports/commissions/monthly?year=&month=`|
  |Decking daily|`GET /api/reports/decking/daily?date=`|

* **Behavior:** `setTab`/`setCommissionTab` switch views; each `load*` populates a  
  signal; `export*Csv` downloads a blob via the shared `downloadBlob` helper  
  (toast on failure). Date ranges use Manila `+08:00` bounds.

### Ledger — `LedgerComponent`

* **Route:** `ledger`. **Selector:** `sumi-ledger`.
* **Injects:** `HttpClient`, `ToastService`. **Imports:** `FormsModule`, `RouterLink`.
* **Signals:** `entries`, `balance`, `loading`, `fromDate`, `toDate`,  
  `selectedMethod`, `searchQuery`, `accountBalances` (Map), `customLedgers`,  
  `customLedgerBalances` (Map), `showCreate`, `createError`, plus new-ledger form  
  fields. Computed `accounts`, `filteredAccounts`, `filteredCustomLedgers`,  
  `selectedAccount`.
* **Endpoints:**
  * `GET /api/cashier/ledger/accounts` → custom ledgers; `POST` to create one.
  * `GET /api/cashier/ledger/balance?from=&to=&method=` → per-method balance.
  * `GET /api/cashier/ledger?from=&to=&method=` → entries for a selected account.
  * `GET /api/cashier/ledger/export.csv?from=&to=&method=` (blob).
* **Behavior:** built-in methods `CASH/GCASH/CREDIT/DEBIT` plus user-created  
  ledgers. Selecting an account loads its entries and balance; `statusClass`/  
  `statusLabel` render entry status (Completed/Refund/Reversed). The append-only  
  ledger is view/export only here.

### Analytics — `AnalyticsComponent`

* **Route:** `analytics`. **Selector:** `sumi-analytics`.
* **Purpose:** Revenue trend (line) and net-by-method (bar) charts via Chart.js.
* **Injects:** `HttpClient`. **Imports:** `FormsModule`, `DecimalPipe`. Uses  
  `viewChild` refs `trendCanvas`/`methodCanvas`.
* **Signals:** `from`, `to`, `method`, `loading`, `points`, `methodSlices`.  
  Computed `totalNet`, `totalInflow`, `totalOutflow`, `totalCount`, `bestDay`.
* **Endpoints (combined with `forkJoin`):**
  * `GET /api/cashier/ledger/daily-revenue?from=&to=&method=` → trend points.
  * `GET /api/cashier/ledger/balance?from=&to=&method=` for each of  
    CASH/GCASH/CREDIT/DEBIT → method breakdown.
* **Behavior:** `load()` fetches both, then `renderTrend()`/`renderMethods()`  
  create or update Chart.js instances. Chart colors are read from the branding  
  CSS variables. Charts are destroyed on `ngOnDestroy`.

---

## Administration features

### Settings (profile) — `SettingsComponent`

* **Route:** `settings` (STAFF+; not role-restricted beyond the shell guard).
* **Selector:** `sumi-settings`. **Injects:** `HttpClient`, `ConfirmService`.  
  **Imports:** `FormsModule`.
* **Endpoints:**
  * `GET /api/users/me` → current display name and email.
  * `PATCH /api/users/me/profile` `{ displayName }`.
  * `POST /api/users/me/request-password-reset`.
* **Signals:** `savingProfile`/`profileSuccess`/`profileError`,  
  `requestingReset`/`resetSuccess`/`resetError`. Self-service profile name update  
  and password-reset email request (confirm dialog).

### Users (admin) — `UsersComponent`

* **Route:** `users` (MANAGER+). **Selector:** `sumi-users`.

* **Injects:** `HttpClient`, `AuthService`, `ConfirmService`.

* **Imports:** `FormsModule`, `UserAuditDrawerComponent`,  
  `SortableColumnDirective`, `SortIconComponent`.

* **Signals:** `users`, `deactivated`, `resetSent`, `showForm`, `error`,  
  `auditUserId`, `auditUsername`, `sortState`; create-form fields. Computed  
  `sortedUsers`, `myRole`, `canManage`, `allowedRoles`; method `canManageUser`.

* **Endpoints:**
  
  |Action|Endpoint|
  |------|--------|
  |List users|`GET /api/users`|
  |Deactivated users|`GET /api/users/deactivated` (manage roles only)|
  |Create user|`POST /api/users`|
  |Deactivate|`DELETE /api/users/{id}`|
  |Reactivate|`POST /api/users/{id}/reactivate`|
  |Unlock|`POST /api/users/{id}/unlock`|
  |Send reset link|`POST /api/users/{id}/send-reset-link`|

* **Behavior:** role-based action gating (SUPERADMIN cannot manage SUPERADMINs;  
  ADMIN limited to MANAGER/RECEPTIONIST). `openAudit(user)` opens the audit  
  drawer. Destructive actions use confirm dialogs.

### User audit drawer — `UserAuditDrawerComponent`

* **Selector:** `sumi-user-audit-drawer` (used inside Users).
* **Inputs:** `userId`, `username`. **Output:** `close: EventEmitter<void>`.
* **Endpoint:** `GET /api/audit-logs?actorUserId={userId}&page=&size=25`.
* **Behavior:** `ngOnChanges` loads page 0 when `userId` is set; `prevPage()`/  
  `nextPage()` paginate.

### Audit log — `AuditComponent`

* **Route:** `audit` (ADMIN+). **Selector:** `sumi-audit`.
* **Injects:** `HttpClient`. **Imports:** `SlicePipe`, `FormsModule`,  
  `SortableColumnDirective`, `SortIconComponent`.
* **Signals:** `entries`, `page`, `totalPages`, `totalElements`, `pageSize`,  
  `sortState`; `fromDate`/`toDate` filters. Computed `sortedEntries`.
* **Endpoint:** `GET /api/audit-logs?page=&size=&from=&to=` → Spring `Page`.
* **Behavior:** server-side pagination plus client-side sort; date filter via  
  `applyFilter()`/`clearFilter()`.

### Branding (admin) — `BrandingComponent`

* **Route:** `branding` (MANAGER+). **Selector:** `sumi-branding`.
* **Injects:** `HttpClient`, `BrandingService`. **Imports:** `FormsModule`, `DatePipe`.
* **Signals:** `form = signal<OrganizationBranding | null>`, `saving`, `savedAt`,  
  `customFont`.
* **Endpoints:**
  * `GET /api/organization/branding` → load.
  * `PUT /api/organization/branding` → save.
* **Behavior:** color/font/logo/background editing with **live local preview**  
  (`previewLocally()` calls `BrandingService.applyTheme`). Font dropdown supports a  
  `__custom__` option backed by `customFont`. On save it re-applies the theme.

### Content (CMS) — `ContentComponent`

* **Route:** `content` (MANAGER+). **Selector:** `sumi-content`.
* **Injects:** `HttpClient`. **Imports:** `FormsModule`.
* **Signals:** `blocks`, `loading`, `savingId`, `addingNew`; `newBlock` draft.
* **Endpoints:**
  * `GET /api/content` → blocks.
  * `POST /api/content` → create.
  * `PUT /api/content/blocks/{id}` → update.
  * `POST /api/content/upload` (multipart) → `{ url }` for block images.
* **Behavior:** edit and publish website content blocks; `uploadImage()` posts a  
  file then saves the block with the returned URL.

### Therapists (admin) — `TherapistsAdminComponent`

* **Route:** `admin/therapists` (MANAGER+). **Selector:** `sumi-admin-therapists`.
* **Injects:** `HttpClient`, `ConfirmService`. **Imports:** `FormsModule`,  
  `SortableColumnDirective`, `SortIconComponent`.
* **Signals:** `therapists`, `deactivated`, `shifts`, `showForm`,  
  `editingTherapist`, `formError`, `sortState`; create/edit form fields. Computed  
  `sortedTherapists`.
* **Endpoints:**
  * `GET /api/therapists`, `GET /api/therapists/deactivated`, `GET /api/shifts`.
  * `POST /api/therapists`, `PATCH /api/therapists/{id}`.
  * `DELETE /api/therapists/{id}` (deactivate),  
    `POST /api/therapists/{id}/reactivate`.
* **Behavior:** create/edit with staff number, nickname, gender, backup flag, and  
  current shift; deactivate/reactivate with confirm dialog.

### Shifts (admin) — `ShiftsAdminComponent`

* **Route:** `admin/shifts` (MANAGER+). **Selector:** `sumi-admin-shifts`.
* **Injects:** `HttpClient`, `ConfirmService`. **Imports:** `FormsModule`.
* **Signals:** `shifts`, `showForm`; form fields (`formLabel`, `formStart`,  
  `formEnd`, `formExpected`).
* **Endpoints:**
  * `GET /api/shifts`.
  * `POST /api/shifts` (`startTime`/`endTime` suffixed with `:00`).
  * `DELETE /api/shifts/{id}` (deactivate, confirm dialog).

### Rooms (admin) — `RoomsAdminComponent`

* **Route:** `admin/rooms` (MANAGER+). **Selector:** `sumi-admin-rooms`.

* **Injects:** `HttpClient`, `ConfirmService`, `StompService`, `AuthService`.

* **Imports:** `FormsModule`, `PaginatorComponent`, `LockerLabelPipe`.

* **Signals:** `rooms`, `beds` (Map roomId→Bed\[\]), `occupancy`  
  (Map bedId→record), `showRoomForm`, `bedFormForRoomId`,  
  `currentPage`/`pageSize`; room/bed form fields. Computed `pagedRooms`.

* **Endpoints:**
  
  |Action|Endpoint|
  |------|--------|
  |Rooms + beds|`GET /api/admin/rooms/with-beds?includeInactive=true`|
  |Beds for room|`GET /api/admin/rooms/{roomId}/beds?includeInactive=true`|
  |Live occupancy|`GET /api/rooms`|
  |Create room|`POST /api/admin/rooms`|
  |Deactivate / reactivate room|`DELETE /api/admin/rooms/{id}` / `PATCH …/reactivate`|
  |Create bed|`POST /api/admin/rooms/{roomId}/beds`|
  |Deactivate / reactivate bed|`DELETE /api/admin/beds/{id}` / `PATCH …/reactivate`|

* **Realtime:** subscribes to `/topic/room-updates/{orgId}` and debounces (200 ms)  
  an occupancy reload, so the admin grid reflects live bed status.

* **Rendering:** `bedClass`/`bedStateLabel`/`occupant*` derive color and labels  
  from occupancy records; `roomGenderLock(roomId)` reports the room's active gender  
  lock.

### Services (admin) — `ServicesAdminComponent`

* **Route:** `admin/services` (MANAGER+). **Selector:** `sumi-admin-services`.
* **Injects:** `HttpClient`, `ConfirmService`. **Imports:** `FormsModule`,  
  `DecimalPipe`, `SortableColumnDirective`, `SortIconComponent`.
* **Signals:** `services`, `showForm`, `editingId`, `sortState`; create form and  
  edit-media fields. Computed `sortedServices`.
* **Endpoints:**
  * `GET /api/services`.
  * `POST /api/admin/services`, `PATCH /api/admin/services/{id}` (description/image).
  * `DELETE /api/admin/services/{id}` (deactivate, confirm).
  * `GET /api/services/export` (blob CSV).
* **Behavior:** create services (code, name, duration, category, price,  
  commission, fixed-rate, two-therapist/tandem flags); inline edit of  
  description/image; CSV export.

### Packages (admin) — `PackagesAdminComponent`

* **Route:** `admin/packages` (MANAGER+). **Selector:** `sumi-packages-admin`.
* **Injects:** `HttpClient`, `ConfirmService`. **Imports:** `CommonModule`,  
  `FormsModule`, `SortableColumnDirective`, `SortIconComponent`.
* **Signals:** `packages`, `services`, `error`, `loading`, `editing`, `showForm`,  
  `sortState`. Computed `sortedPackages`.
* **Endpoints:**
  * `GET /api/cashier/packages/all` (admin list incl. inactive),  
    `GET /api/services` (tier service options).
  * `POST /api/cashier/packages`, `PATCH /api/cashier/packages/{id}`.
  * `DELETE /api/cashier/packages/{id}` (deactivate),  
    `POST /api/cashier/packages/{id}/reactivate`.
* **Behavior:** create/edit packages with attendee count, couple/VIP/private-room  
  flags and per-service price **tiers** (weekday/weekend). `onTierService()`  
  auto-fills code/name and seeds prices from the chosen service.

### Vouchers (admin) — `VouchersAdminComponent`

* **Route:** `admin/vouchers` (MANAGER+). **Selector:** `sumi-admin-vouchers`.
* **Injects:** `HttpClient`. **Imports:** `FormsModule`, `DecimalPipe`,  
  `SortableColumnDirective`, `SortIconComponent`.
* **Signals:** `vouchers`, `packages`, `showForm`, `editingVoucher`, `sortState`;  
  form fields. Computed `sortedVouchers`.
* **Endpoints:**
  * `GET /api/vouchers`, `GET /api/cashier/packages/all` (target options).
  * `POST /api/vouchers`, `PUT /api/vouchers/{id}`.
* **Behavior:** create/edit vouchers with fixed-amount or percent discount,  
  validity window, usage limit, active flag, and optional target package  
  (`packageName(id)` shows "Whole order" when unset).

### Feedback (admin) — `FeedbackAdminComponent`

* **Route:** `admin/feedback` (MANAGER+). **Selector:** `sumi-admin-feedback`.
* **Injects:** `HttpClient`, `NotificationFeedService`. **Imports:** `FormsModule`.
* **Signals:** `feedback`, `searchTerm`, `expandedId`, `exporting`;  
  `exportFrom`/`exportTo`. Computed `filteredFeedback`.
* **Endpoints:**
  * `GET /api/feedback` → `{ content: Feedback[] }`.
  * `POST /api/feedback/mark-all-read` (on init; also clears the feed badge).
  * `GET /api/feedback/export.csv?from=&to=` (blob).
* **Behavior:** on load it marks all feedback read and clears the  
  `feedback` notification counter. `toggle(id)` expands a row; `stars(n)` renders  
  the rating.

---

## Route map (summary)

Public (`PublicShellComponent`, no guard): `''` (home/`LandingComponent`),  
`about`, `packages`, `services`, `recommendation`, `book`, `visit`, `feedback`,  
`contact`, `cancel`, `terms`. Plus top-level `pay/authorize`.

Auth (`/sumicare`, no guard): `login`, `verify`, `reset-password`, `invite`.

Internal (`/sumicare/app`, `authGuard`; children role-guarded):

|Path|Guard|Component|
|----|-----|---------|
|`dashboard`|(shell auth)|`DashboardComponent`|
|`bookings`|STAFF+|`BookingsComponent`|
|`reception`|STAFF+|`ReceptionComponent`|
|`lineup`|STAFF+|`LineupComponent`|
|`cashier` (`pos`→`cashier`)|STAFF+|`CashierComponent`|
|`orders`, `orders/:id`|STAFF+|`OrdersListComponent`, `OrderDetailComponent`|
|`messages`|STAFF+|`MessagesComponent`|
|`registered-clients`|STAFF+|`RegisteredClientsComponent`|
|`treatment-slips`, `treatment-slips/:id`|STAFF+|`TreatmentSlipsComponent`, `TreatmentSlipDetailComponent`|
|`attendance`|STAFF+|`AttendanceComponent`|
|`reports`|MANAGER+|`ReportsComponent`|
|`ledger`|MANAGER+|`LedgerComponent`|
|`analytics`|MANAGER+|`AnalyticsComponent`|
|`settings`|(shell auth)|`SettingsComponent`|
|`users`|MANAGER+|`UsersComponent`|
|`audit`|ADMIN+|`AuditComponent`|
|`branding`|MANAGER+|`BrandingComponent`|
|`content`|MANAGER+|`ContentComponent`|
|`admin/therapists`|MANAGER+|`TherapistsAdminComponent`|
|`admin/shifts`|MANAGER+|`ShiftsAdminComponent`|
|`admin/rooms`|MANAGER+|`RoomsAdminComponent`|
|`admin/services`|MANAGER+|`ServicesAdminComponent`|
|`admin/packages`|MANAGER+|`PackagesAdminComponent`|
|`admin/vouchers`|MANAGER+|`VouchersAdminComponent`|
|`admin/feedback`|MANAGER+|`FeedbackAdminComponent`|

Role tiers: **STAFF+** = `RECEPTIONIST, MANAGER, ADMIN, SUPERADMIN`;  
**MANAGER+** = `MANAGER, ADMIN, SUPERADMIN`; **ADMIN+** = `ADMIN, SUPERADMIN`.  
"STAFF+" is shorthand for the `STAFF_ROLES` guard constant — the four internal roles above; it does **not** include a `STAFF` role (there is no `STAFF` role in the system).  
Unknown paths (`**`) redirect to `''`.
