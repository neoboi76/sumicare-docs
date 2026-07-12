---
title: "module-interactions"
date: 2026-07-12
categories: ["System Guide"]
---

# SumiCare Inter-Module Interaction Map

This document maps how the backend bounded-context modules under `com.sumicare`  
interact at runtime. Every interaction below was traced from the actual source  
(service constructors, field dependencies, and method calls). Where the code does  
not make a relationship explicit, this is stated as "not determinable from code".

Interaction types used throughout:

* **direct service call** — one module's service holds a constructor-injected  
  dependency on another module's service and calls its methods.
* **shared DB table** — modules read/write the same JPA entity through their own  
  injected repositories (no shared service layer).
* **shared Redis key** — modules coordinate through a common Redis key namespace  
  via `StringRedisTemplate`.
* **WebSocket broadcast** — a mutation publishes a STOMP message through  
  `NotificationService` / `SimpMessagingTemplate`.

---

## Part 1 — Interaction Map

### 1. Cashier (`OrderService`) — the central orchestrator

`OrderService` is the busiest hub in the system. Its constructor injects 21  
collaborators, spanning seven other modules. Key outbound edges:

|From → To|Type|What is exchanged|Why (business requirement)|
|---------|----|-----------------|--------------------------|
|`cashier/OrderService` → `booking/BookingService`|direct service call|`createBooking(...)` from `create(...)`; `resendBookingConfirmation`, `cancelSession`, `materialiseAttendeeSessions` callbacks|Every cashier order materialises a `Booking` (a walk-in order produces a `"WALK_IN"` booking). The cashier never persists bookings directly — it delegates so booking rules stay in one place.|
|`cashier/OrderService` → `booking` (Booking, Session)|shared DB table|`BookingRepository`, `SessionRepository` reads/writes (e.g. `findFirstByBookingId`, set `pax`, sync `paymentStatus`)|The order must keep its booking's `pax` and `paymentStatus` in step (`syncBookingPaymentStatus`) and resolve the session for payment linkage.|
|`cashier/OrderService` → `voucher/VoucherService`|direct service call|`assertRedeemableBy`, `discountForVoucher`, `markRedeemed`|A voucher applied to an order must be validated for the client and usage limit, converted to a peso discount, and recorded as redeemed against the order.|
|`cashier/OrderService` → `pos/PayMongoService`|direct service call|`PayMongoService.supports(...)`, `charge`, `initiate`, `confirm`, `refund` (returns `ChargeResult` / `RefundResult` with `intentId`, `nextActionUrl`)|Card/GCash payments and refunds are processed through the gateway. `PayMongoService.supports` gates which methods route to the gateway versus cash.|
|`cashier/OrderService` → `pos` (PosTransaction, TransactionLedgerEntry)|shared DB table|`PosTransactionRepository`, `TransactionLedgerRepository` inserts|Each payment writes a `PosTransaction` and an append-only `TransactionLedgerEntry` (`PAYMENT_RECEIVED`, `PAYMENT_REFUNDED`, `PAYMENT_REVERSED`, `ORDER_CANCELLED_REVERSAL`, `MANUAL_REFUND_REQUIRED`, `FULLY_DISCOUNTED`). The ledger is the immutable money record.|
|`cashier/OrderService` → `transaction/TreatmentSlipService`|direct service call|`slipService` field (slip generation helpers); also direct `TreatmentSlipRepository` writes in `ensureSlipForAttendee` / void-on-cancel|Paid orders materialise treatment slips per attendee; cancelled orders void them.|
|`cashier/OrderService` → `transaction` (Commission)|shared DB table|`CommissionRepository` insert in `recordCommissionsForSession`|When a session ends, therapist commission is computed (₱ split for tandem) and persisted.|
|`cashier/OrderService` → `room/RoomOccupancyService`|direct service call|`occupancyService.release(orgId, roomId, bedId)` on order cancel|If an order is cancelled while a bed is held without an active session, the bed is freed in Redis.|
|`cashier/OrderService` → `notification/NotificationService`|WebSocket broadcast|`broadcastOrderEvent(orgId, "ORDER_CREATED", orderId, ...)`|The cashier dashboard subscribes to `/topic/orders/{orgId}` to show new orders live.|
|`cashier/OrderService` → `auth/EmailService`|direct service call|`sendCancellationConfirmedEmail`, `sendPaymentConfirmationEmail`|On payment completion or cancellation, the client (if an email is on file) is notified.|
|`cashier/OrderService` → `audit/AuditService`|direct service call|`auditService.record(orgId, null, "SYSTEM", "ORDER_AUTO_CANCELLED", ...)`|Auto-cancellation by the nightly sweep is a system action that must be recorded for non-repudiation.|
|`cashier/OrderService` → `user` (User)|shared DB table|`UserRepository` lookups in `toResponse`|Resolve cashier / last-editor display names for the order response.|
|`cashier/OrderService` → `service_catalogue` (Service)|shared DB table|`ServiceRepository`|Resolve service names, durations, commission amounts, VIP duration validation.|
|`cashier/OrderService` → `common`|direct service call|`IdSequenceService.nextOrNumber / nextReceiptNumber / nextTsn`|Generate OR numbers, receipt numbers, and TSNs from a central sequence.|

### 2. Booking (`BookingService`)

`BookingService` owns the lifecycle of bookings and sessions. It is injected  
`@Lazy OrderService` because the two services are mutually dependent (the lazy  
proxy breaks the circular constructor cycle).

|From → To|Type|What is exchanged|Why|
|---------|----|-----------------|---|
|`booking/BookingService` → `cashier/OrderService`|direct service call (`@Lazy`)|`nextOrNumber`, `recordPaymentInternal`, `materialiseAttendeeSessions`, `notifyOrderPaid`, `settleGatewayPayment`|A public booking creates an `Order` and, for paid/hard reservations, drives payment recording and session materialisation through the cashier.|
|`booking/BookingService` → `cashier` (Order, OrderItem, OrderItemAttendee, Package, PackageTier)|shared DB table|`OrderRepository`, `OrderItemRepository`, `OrderItemAttendeeRepository`, `PackageRepository`, `PackageTierRepository`|A booking builds its own order/items/attendees graph and prices them from package tiers.|
|`booking/BookingService` → `cashier/PackageService`|direct service call|`packageService.deriveInclusions(pkg)`|Email confirmation lists package inclusions.|
|`booking/BookingService` → `room/RoomOccupancyService`|direct service call|`occupy(...)` on session start, `release(...)` on session end/cancel; `read(...)` for gender-lock checks|Starting a session marks a bed OCCUPIED in Redis (locker, therapist nickname, gender lock, start time); ending frees it. The gender-lock read enforces gender-segregated common rooms.|
|`booking/BookingService` → `room` (Room, Bed)|shared DB table|`RoomRepository`, `BedRepository`|Validate room type (VIP/private rules) and enumerate beds for gender-conflict checks.|
|`booking/BookingService` → `therapist/DeckingService`|direct service call|`isSkipped`, `rotateToBack`, `servedRequested`|On assignment: reject therapists on break, rotate the assigned therapist to the back of the decking queue (unless specifically requested, which preserves position).|
|`booking/BookingService` → `therapist` (Therapist)|shared DB table|`TherapistRepository`|Resolve therapist nicknames for bed occupancy and slips.|
|`booking/BookingService` → `transaction/TreatmentSlipService`|direct service call|`generateForSession(orgId, sessionId)` on session start and end|Produce/update the treatment slip for the session.|
|`booking/BookingService` → `transaction` (TreatmentSlip, Commission)|shared DB table|`TreatmentSlipRepository`, `CommissionRepository`|Void slips on cancel; write extension commission (₱60 per 30 min) directly in `extendSession`.|
|`booking/BookingService` → `voucher/VoucherService`|direct service call|`findValid`, `isRedeemableBy`, `computeDiscount`, `markRedeemed` in `applyVoucher`|Apply a voucher code to a public booking's order.|
|`booking/BookingService` → `client` (Client)|shared DB table|`ClientRepository`|Optional client account: created/looked up by email for online bookings; nationality back-filled.|
|`booking/BookingService` → `pos` (PosTransaction, TransactionLedgerEntry)|shared DB table|`PosTransactionRepository`, `TransactionLedgerRepository`|Link transactions to the session on start; describe payment methods for emails.|
|`booking/BookingService` → `pos/PayMongoService`|direct service call|`initiate`, `confirm` in public payment flow|Public-website payment initiation/confirmation.|
|`booking/BookingService` → `notification/NotificationService`|WebSocket broadcast|`broadcastBookingEvent` (`/topic/bookings/{orgId}`), `broadcastRoomUpdate` (`/topic/room-updates/{orgId}`)|Notify dashboards of new bookings and of SESSION_STARTED / SESSION_ENDED / SESSION_CANCELLED bed events.|
|`booking/BookingService` → `auth/EmailService`|direct service call|`sendBookingConfirmationEmail`, `sendCompletionEmail`|Confirm bookings and send completion email with receipt + treatment-slip PDFs.|
|`booking/BookingService` → `print`|direct service call|`ReceiptPdfService.renderReceipt`, `TreatmentSlipPdfService.renderSlip`|Render PDF attachments for the completion email.|
|`booking/BookingService` → `booking/LockerAssignmentService`|direct service call|`takenLockersForDay`, `assign(gender, taken)`|Auto-assign gendered locker numbers for public reservations.|

### 3. Therapist decking (`DeckingService`)

|From → To|Type|What is exchanged|Why|
|---------|----|-----------------|---|
|`therapist/DeckingService` → Redis|shared Redis key|ZSet `decking:active:{orgId}` (queue); Set `decking:shift:{orgId}:{shiftId}` (shift membership); String `decking:skip:{orgId}:{therapistId}` (TTL break); String `decking:flag:{orgId}:{therapistId}` (REQUESTED/BACKUP/etc.)|Ordered lineup with atomic ZADD/ZREM; newest shift prepends to front (`prependToFront`); rotation, skip with auto-expiry, manual backup insertion at a position, requested-therapist flagging.|
|`therapist/DeckingService` → `notification/NotificationService`|WebSocket broadcast|`broadcastDecking(orgId, currentLineup(orgId))` after every mutation|The decking board (`/topic/decking-updates/{orgId}`) reflects queue changes in real time.|

`DeckingService` is consumed by `booking/BookingService` (assignment/rotation) and  
by the therapist controllers (add/remove/skip/flag/backup); it does not call back  
into other domain modules.

### 4. Room occupancy (`RoomOccupancyService`)

|From → To|Type|What is exchanged|Why|
|---------|----|-----------------|---|
|`room/RoomOccupancyService` → Redis|shared Redis key|Hash `room:{roomId}:bed:{bedId}` with `status`, `clientNickname`, `lockerNumber`, `therapistNickname`, `genderLock`, `ownerItemId`, `startedAt`|Live bed occupancy is held in Redis (not Postgres) so the room map can be read cheaply; `genderLock` enforces gender-segregated common rooms.|
|`room/RoomOccupancyService` → `room` (Room, Bed)|shared DB table|`RoomRepository`, `BedRepository`|Enumerate beds to count occupancy.|
|`room/RoomOccupancyService` → `notification/NotificationService`|WebSocket broadcast|`broadcastRoomUpdate(orgId, roomId, bedId, state)` on occupy/release|Room map subscribers get live bed state on `/topic/room-updates/{orgId}`.|

### 5. POS gateway (`PayMongoService`) and webhook

|From → To|Type|What is exchanged|Why|
|---------|----|-----------------|---|
|`pos/PayMongoService` → `pos/gateway/PayMongoGateway`|direct service call|`createIntent`, `createPaymentMethod`, `attachIntent`, `createCheckoutSession`, `retrieveIntent`, `retrieveCheckoutSession`, `retrievePaymentId`, `refund`|All HTTP gateway calls are isolated behind `PaymentGateway`; `PayMongoService` adds mock mode, return-URL building, and status normalisation.|
|`pos/WebhookController` → `cashier/OrderService`|direct service call|`settleGatewayPayment(order, null, gatewayId, method, amount)` on `payment.paid`; `reconcileRefund(order, amount, refundId)` on refund events|The asynchronous PayMongo webhook is the authoritative settlement path for redirect-based payments (GCash/card checkout). It also flips `booking.paymentStatus` to PAID.|
|`pos/WebhookController` → `pos` (TransactionLedgerEntry)|shared DB table|`TransactionLedgerRepository` (idempotency via `existsByGatewayReference`, fallback ledger insert)|If no matching order is found, the payment is still recorded once in the ledger.|
|`pos/WebhookController` → `booking` (Booking)|shared DB table|`BookingRepository` set `paymentStatus`, `gatewayPaymentId`|Reflect the confirmed gateway payment on the booking.|

### 6. Auth, audit, and supporting edges

|From → To|Type|What is exchanged|Why|
|---------|----|-----------------|---|
|`audit/AuditInterceptor` → `audit/AuditService`|direct service call|`record(orgId, actorId, role, action, entity, targetId, null, ip)`|A Spring MVC `HandlerInterceptor` records every successful mutating request (POST/PUT/PATCH/DELETE, status \< 400). Action/entity are derived from the URL path. This is the system-wide all-mutations → audit edge.|
|`audit/AuditService` → `audit` (AuditLog)|shared DB table (`@Async`)|`AuditLogRepository.save`|Append-only audit log, written off the request thread.|
|`auth/AuthService` → `user` (User)|shared DB table|`UserRepository` (load by username/id, lock/unlock, reset failed attempts)|Authentication and account-state management; `UserDetails` is loaded via Spring `AuthenticationManager`.|
|`auth/AuthService` → `auth/JwtService`|direct service call|`issueAccessToken(userId, orgId, role)`, `issueRefreshToken`, `revoke`, `isRevoked`, `isTokenIssuedBeforeRevocation`|Token issuance/rotation; the access token carries the **organization id** used everywhere as `principal.organizationId()` — this is the de facto auth → organization resolution.|
|`auth/JwtService` → Redis|shared Redis key|String `revoked:jti:{jti}` (TTL); String `user:{userId}:tokens-since`|Refresh-token revocation deny-list and per-user mass-revocation timestamp.|
|`auth/AuthService` → `auth/LoginRateLimiter`|direct service call|`tryConsume("ip:...", "user:...")`|Sliding-window rate limiting on login.|
|`auth/AuthService` → `auth/MfaService`|direct service call|`create`, `verify`, `resend`|Email-code MFA for non-SUPERADMIN logins.|
|`auth/AuthService` / `user/UserService` → `auth/EmailService`|direct service call|`sendMfaCodeEmail`, `sendPasswordResetEmail`, `sendVerificationEmail`, `sendInvitationEmail`|Auth-triggered emails.|
|`user/UserService` → `auth` (PasswordResetToken, EmailVerificationToken)|shared DB table|`PasswordResetTokenRepository`, `EmailVerificationTokenRepository`|Issue/consume reset and invitation tokens.|
|`booking/BookingCancellationService` → `cashier/OrderService`|direct service call|`cancelInternal(orgId, orderId, reason)`|Public cancellation cancels the underlying order (which triggers gateway reversal and email).|
|`booking/BookingCancellationService` → `booking/BookingCancellationCodeService`|direct service call|`onCooldown`, `issue`, `verify`, `clear`|Email verification code for public self-service cancellation.|
|`booking/BookingCancellationCodeService` → Redis|shared Redis key|`cancel:code:{bookingId}`, `cancel:attempts:{bookingId}`, `cancel:cooldown:{bookingId}` (all TTL)|6-digit code with 15-min TTL, 5-attempt cap, 60-second resend cooldown.|
|`booking/BookingCancellationService` → `audit/AuditService`|direct service call|`record(... "BOOKING_CANCEL_REQUESTED" / "BOOKING_CANCELLED" ...)`|Public actions are audited with `actorRole = "PUBLIC"`.|
|`report/ReportService` → `booking` + `transaction`|shared DB table|`SessionRepository`, `CommissionRepository` reads|Cutoff report aggregates sessions and commissions per therapist.|
|`report/OperationsReportService` → `booking`, `cashier`, `transaction`, `room`, `shift`, `therapist`, `service_catalogue`|shared DB table|`SessionRepository`, `BookingRepository`, `OrderRepository`, `OrderItem*Repository`, `TreatmentSlipRepository`, `ShiftAssignmentRepository`, etc.|Daily/monthly operations reports are read-only aggregations across the operational tables; CSV bytes are produced in-service.|

### Notification topics (all under `NotificationService`)

|Method|STOMP topic|Triggered by|
|------|-----------|------------|
|`broadcastDecking`|`/topic/decking-updates/{orgId}`|every `DeckingService` mutation|
|`broadcastRoomUpdate`|`/topic/room-updates/{orgId}`|`RoomOccupancyService` occupy/release; `BookingService` session start/end/cancel|
|`broadcastBookingEvent`|`/topic/bookings/{orgId}`|`BookingService.finalizeBooking`|
|`broadcastOrderEvent`|`/topic/orders/{orgId}`|`OrderService.create`|
|`broadcastMessageEvent`|`/topic/messages/{orgId}`|contact module|
|`broadcastFeedbackEvent`|`/topic/feedback/{orgId}`|feedback module|

### Interaction overview (selected edges)

````
                          +------------------+
        REST/STOMP        |  NotificationSvc |<--- DeckingService (Redis ZSet)
        clients  <--------|  (SimpMessaging) |<--- RoomOccupancySvc (Redis Hash)
                          +---------+--------+
                                    ^  ^  ^
        broadcastOrder/Booking/Room |  |  | broadcastRoom
                                    |  |  |
        +-------------+   create    |  |  |   occupy/release   +----------------+
        | OrderService|-------------+  |  +------------------->| RoomOccupancySvc|
        | (cashier)   |---------------------------------------+|  (room)         |
        +------+------+   createBooking / materialise          +----------------+
               | \  \         (BookingService, @Lazy)
               |  \  \------------------> VoucherService (assert/discount/redeem)
               |   \-------------------> PayMongoService --> PayMongoGateway (HTTP)
               |                              ^
               | charge/refund/settle         | settleGatewayPayment / reconcileRefund
               v                              |
        PosTransaction + TransactionLedger    WebhookController <--- PayMongo webhook
               |
               +--> TreatmentSlipService / CommissionRepository (transaction)

        BookingService --> DeckingService (rotate/served/skip checks)
        every mutating request --> AuditInterceptor --> AuditService (AuditLog)
        AuthService --> JwtService (Redis revoked:jti) --> User table
````

---



## Part 2 — End-to-End Data Flows

### Flow 1 — Public website booking paid by GCash

1. **Create booking.** `POST /api/public/bookings/{slug}` →  
   `booking/BookingController.publicBook` resolves the org from the slug  
   (`OrganizationRepository.findBySlug`) → `BookingService.createBooking`.
   * Resolves/creates the optional `Client` (`ClientRepository`, by email).
   * Persists `Booking` (status `PENDING`, with a generated `reference` via  
     `BookingReference.of`) and builds the `Order` + `OrderItem` +  
     `OrderItemAttendee` graph (cashier tables), priced from `PackageTier`.
   * Assigns gendered lockers (`LockerAssignmentService`).
   * `finalizeBooking` → `applyVoucher` (if any) → because GCash is gateway-backed  
     (`PayMongoService.supports("GCASH")` is true), it does **not** mark paid yet;  
     it broadcasts `broadcastBookingEvent("BOOKING_CREATED")` on  
     `/topic/bookings/{orgId}`.
1. **Initiate payment.** `POST /api/public/bookings/{slug}/payment/initiate` →  
   `BookingService.initiatePublicPayment` → `PayMongoService.initiate` →  
   `PayMongoGateway` creates an intent/checkout and returns a `nextActionUrl`  
   (a `PublicPaymentResponse` with `status = awaiting_next_action`).
1. **Redirect & confirm.** The client completes GCash off-site and returns to the  
   `returnPath`. The frontend calls  
   `POST /api/public/bookings/{slug}/payment/confirm` →  
   `BookingService.confirmPublicPayment` → `PayMongoService.confirm(intentId)`.
1. **Authoritative settlement (webhook).** PayMongo also POSTs to  
   `POST /api/webhooks/paymongo` → `pos/WebhookController` verifies the  
   `Paymongo-Signature` (`PayMongoGateway.verifyWebhook`), then on `payment.paid`  
   calls `OrderService.settleGatewayPayment(order, null, gatewayId, method, amount)`  
   and sets `Booking.paymentStatus = "PAID"` + `gatewayPaymentId`.
1. **Order becomes PAID.** `settleGatewayPayment` → `recordPaymentInternal`  
   (writes `PosTransaction` + `TransactionLedgerEntry "PAYMENT_RECEIVED"`,  
   increments `amountPaid`); when fully paid, sets order `PAID`, `finishedAt`, and  
   `materialiseAttendeeSessions` (creates `PENDING` `Session` rows per attendee).
1. **Confirmation email.** `settleGatewayPayment` → `bookingService.resendBookingConfirmation`  
   → `EmailService.sendBookingConfirmationEmail`; `notifyOrderPaid` →  
   `EmailService.sendPaymentConfirmationEmail` (guarded by `paymentEmailSentAt`).
1. **Dashboard STOMP.** `notifyOrderPaid`/order mutations and the booking event  
   keep `/topic/orders/{orgId}` and `/topic/bookings/{orgId}` current.

Tables touched: `clients`, `bookings`, `orders`, `order_items`,  
`order_item_attendees`, `sessions`, `pos_transactions`, `transaction_ledger`.  
Redis: none required on the happy path (Redis is used at session start, not booking).

### Flow 2 — Receptionist walk-in order, then start a session

1. **Create order.** `POST /api/cashier/orders` →  
   `cashier/OrderController` → `OrderService.create(orgId, cashierUserId, request)`.
   * Builds a `CreateBookingRequest` with reservation type `"WALK_IN"` and calls  
     `BookingService.createBooking`, then sets `Booking.pax`.
   * Persists the `Order`, `OrderItem`s and `OrderItemAttendee`s; applies voucher  
     or manual discount; generates the OR number (`IdSequenceService`).
   * If an `initialPayment` covers the total: records payment  
     (`recordPaymentInternal` → `PosTransaction` + ledger), sets order `PAID`, and  
     `materialiseAttendeeSessions` (creates `PENDING` sessions).
   * Broadcasts `broadcastOrderEvent("ORDER_CREATED")` on `/topic/orders/{orgId}`.
1. **Start the session.** `POST /api/bookings/attendees/{attendeeId}/sessions`  
   (or `/api/bookings/{bookingId}/sessions`, which delegates to the first attendee)  
   → `BookingController.startAttendee` → `BookingService.startAttendeeSession`.
   * Guards: order must be `PAID`, locker assigned, no gender/locker clash, chosen  
     therapists not skipped (`DeckingService.isSkipped`) and not already on an  
     `ACTIVE` session.
   * Sets the `Session` `ACTIVE`, `startedAt`, `expectedEndAt = now + duration`.
   * **Occupy bed:** `RoomOccupancyService.occupy(...)` writes Redis hash  
     `room:{roomId}:bed:{bedId}` (client nickname, locker, therapist nickname,  
     `genderLock`, `ownerItemId`, `startedAt`).
   * **Rotate therapist:** `DeckingService.rotateToBack` (or `servedRequested` if  
     specifically requested) updates Redis ZSet `decking:active:{orgId}`.
   * **Treatment slip:** `TreatmentSlipService.generateForSession` creates/updates  
     the `TreatmentSlip`; links its id onto the attendee.
   * **Broadcasts:** `broadcastRoomUpdate(... SESSION_STARTED ...)` on  
     `/topic/room-updates/{orgId}`, and `DeckingService` broadcasts the new lineup  
     on `/topic/decking-updates/{orgId}`.

Tables: `bookings`, `orders`, `order_items`, `order_item_attendees`, `sessions`,  
`treatment_slips`, `pos_transactions`, `transaction_ledger`. Redis:  
`room:{roomId}:bed:{bedId}`, `decking:active:{orgId}`. Topics:  
`/topic/orders`, `/topic/room-updates`, `/topic/decking-updates`.

### Flow 3 — Session auto-ends after one hour with no manual action

1. **Scheduler tick.** `booking/scheduler/SessionAutoEndJob.sweep` runs every  
   60 s (`@Scheduled(fixedDelay = 60_000)`), sets a SYSTEM `SUPERADMIN`  
   `SecurityContext`, and iterates organizations  
   (`OrganizationRepository.findAll`).
1. **Find & end expired sessions.** `BookingService.autoEndExpiredSessions(orgId)`  
   → `SessionRepository.findAllByOrganizationIdAndStatusAndExpectedEndAtBefore("ACTIVE", now)`  
   → for each, `endSessionAt(orgId, sessionId, expectedEndAt)`.
1. **End-session effects** (in `endSessionAt`):
   * `Session` → `COMPLETED`, `endedAt` set; `Booking` → `COMPLETED` (single) or  
     when all group attendees are done.
   * `TreatmentSlipService.generateForSession` finalises the slip (`endTime`);  
     order's `treatmentSlipId` set for single-attendee orders.
   * `OrderService.recordCommissionsForSession` writes therapist `Commission`(s).
   * **Release bed:** `RoomOccupancyService.release` deletes the Redis hash; on  
     Redis failure it logs and relies on the reconciler.
   * **Broadcast:** `broadcastRoomUpdate(... SESSION_ENDED ...)` on  
     `/topic/room-updates/{orgId}`.
   * `maybeSendCompletionEmail` → `EmailService.sendCompletionEmail` with receipt  
     and treatment-slip PDFs (`ReceiptPdfService`, `TreatmentSlipPdfService`),  
     guarded by `completionEmailSentAt`.

Note: the therapist is not auto-rotated here — rotation happens at *assignment*  
time (Flow 2), not at end time. Tables: `sessions`, `bookings`, `treatment_slips`,  
`commissions`, `orders`. Redis: `room:{roomId}:bed:{bedId}` (deleted).

### Flow 4 — Manager generates a monthly report and exports CSV

1. **Detailed monthly (operations).**  
   `GET /api/reports/monthly-detailed?year=&month=` →  
   `report/ReportController.monthlyDetailed` →  
   `OperationsReportService.monthly(orgId, year, month)`, reading across  
   `SessionRepository`, `OrderRepository`, `OrderItem`/`OrderItemAttendee`  
   repositories, `TreatmentSlipRepository`, `ShiftAssignmentRepository`,  
   `TherapistRepository`, `ServiceRepository`, `RoomRepository`.  
   Export: `GET /api/reports/monthly-detailed/export.csv` →  
   `operationsReportService.monthlyCsv(...)` returns CSV bytes with a  
   `Content-Disposition: attachment` header.
1. **Aggregated monthly (persisted rollup).**  
   `POST /api/reports/monthly/regenerate?year=&month=`  
   (`@PreAuthorize SUPERADMIN/ADMIN/MANAGER`) →  
   `ReportAggregationService.generateMonthlyReport` (writes `monthly_reports`);  
   `GET /api/reports/monthly` reads `MonthlyReportRepository`.
1. **Cutoff (commission view).** `GET /api/reports/cutoff` →  
   `ReportService.buildCutoffReport` aggregates `sessions` and `commissions` per  
   therapist; `GET /api/reports/cutoff/export.csv` →  
   `ReportService.exportCutoffToCsv` returns CSV bytes.

The report layer is **read-only** over operational tables (plus the persisted  
`day_reports` / `monthly_reports` rollups). Reports are served by `ReportController`,  
`CommissionReportController`, and `DeckingReportController`. The export format is CSV  
(`text/csv`) — there is no `.xlsx`/Excel generation in the source.

### Flow 5 — Public booking cancellation

1. **Request a code.** `POST /api/public/bookings/{slug}/cancel/request` →  
   `booking/PublicCancellationController.request` →  
   `BookingCancellationService.requestCancellation`.
   * `findCancellable` matches a `PENDING` booking by reference + email whose order  
     is not already CANCELLED/REFUNDED.
   * `BookingCancellationCodeService.onCooldown` / `issue` → writes Redis  
     `cancel:code:{bookingId}` (15-min TTL) and `cancel:cooldown:{bookingId}`  
     (60 s), clears `cancel:attempts:{bookingId}`.
   * `EmailService.sendCancellationCodeEmail` sends the 6-digit code.
   * `AuditService.record("BOOKING_CANCEL_REQUESTED", actorRole "PUBLIC")`.
1. **Verify.** `POST .../cancel/verify` → `BookingCancellationService.verify` →  
   `BookingCancellationCodeService.verify` (increments `cancel:attempts`, 5-attempt  
   cap) → returns `CancellationDetailsResponse` (service name, schedule, paid flag).
1. **Confirm.** `POST .../cancel/confirm` → `BookingCancellationService.confirm`:
   * Re-verifies the code, loads the order, computes `refunded = order PAID`.
   * `OrderService.cancelInternal(orgId, orderId, "Cancelled online by the client")`:  
     order → `CANCELLED`; active session cancelled / bed released  
     (`RoomOccupancyService.release`); booking → `CANCELLED`; slip → `VOIDED`;  
     each completed `PosTransaction` reversed (gateway refund via  
     `PayMongoService.refund` where applicable) with a  
     `TransactionLedgerEntry "ORDER_CANCELLED_REVERSAL"`; `amountPaid` zeroed.
   * **Refund:** gateway (card/GCash) payments are refunded automatically through  
     PayMongo; cash payments are not refunded inline here (the nightly auto-cancel  
     path flags cash via `MANUAL_REFUND_REQUIRED`).
   * `dispatchCancellationEmail` → `EmailService.sendCancellationConfirmedEmail`  
     (includes refund amount/flag).
   * `BookingCancellationCodeService.clear` deletes the Redis code/attempts.
   * `AuditService.record("BOOKING_CANCELLED", actorRole "PUBLIC")`.

Tables: `bookings`, `orders`, `sessions`, `treatment_slips`, `pos_transactions`,  
`transaction_ledger`, `audit_logs`. Redis: `cancel:code/attempts/cooldown:*`.

### Flow 6 — Admin resets a staff member's password

1. **Admin triggers the reset link.**  
   `POST /api/users/{userId}/send-reset-link`  
   (`@PreAuthorize SUPERADMIN/ADMIN`) → `user/UserController.sendResetLink` →  
   `UserService.sendResetLink(actorId, actorRole, targetUserId)`.
   * `requireSameOrganization` + `enforceTierForTarget` (ADMIN may only manage  
     MANAGER/RECEPTIONIST; no one may reset another SUPERADMIN).
   * Delegates to `requestPasswordReset(targetUserId)`.
1. **Issue token.** `UserService.requestPasswordReset` creates a  
   `PasswordResetToken` (random 64-char token, `expiresAt = now + 1h`) via  
   `PasswordResetTokenRepository`, then  
   `EmailService.sendPasswordResetEmail(email, name, token)`. The email links to  
   the **frontend** path `{baseUrl}/sumicare/reset-password?token=...`.
1. **Staff member sets a new password.** The reset page POSTs to the backend  
   `POST /api/auth/reset-password` → `auth/AuthController.resetPassword` →  
   `UserService.consumePasswordReset(token, newPassword)`:  
   validates strength, checks token not consumed/expired, encodes the new password  
   with BCrypt, sets `User.passwordHash`, marks the token consumed.  
   (The self-service variant is `POST /api/users/me/request-password-reset` →  
   `UserService.requestPasswordReset`.)

Tables: `users`, `password_reset_tokens`. Every mutating call in this flow is also  
recorded by `AuditInterceptor` (`USER.SEND_RESET_LINK`, etc.).

---

## Notes and Limitations

* `OrderService` and `BookingService` are mutually dependent; the cycle is broken  
  with `@Lazy OrderService` in `BookingService`'s constructor.
* The authoritative settlement path for redirect-based gateway payments (GCash and  
  card checkout) is the **webhook** (`WebhookController.settleGatewayPayment`); the  
  synchronous `confirm` endpoints are best-effort and idempotent against it.
* The `transaction_ledger` is append-only: cancellations/refunds insert negative  
  reversal entries rather than mutating prior rows.
* No Excel (`.xlsx`) export exists in the current source; reports export as CSV  
  (`text/csv`) through `ReportController`, `CommissionReportController`, and  
  `DeckingReportController`.
* Auth-to-organization resolution is implicit: the org id is embedded in the JWT  
  access token (`JwtService.issueAccessToken`) and read everywhere as  
  `principal.organizationId()`; there is no separate org-resolution service call in  
  the request path.
