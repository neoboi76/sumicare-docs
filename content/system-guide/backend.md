---
title: "backend"
date: 2026-07-13
categories: ["System Guide"]
---

# SumiCare — Backend Reference

 > 
 > Companion document: [`system-overview.md`](./system-overview.md) describes the platform as a whole, the request lifecycle, and the real-time architecture. This document is the implementation-level reference for the Spring Boot backend (`apps/sumicare-api`, Spring Boot 3.2.5, Java 21).

---

## 1. Entry Point & Startup

The application class is `com.sumicare.SumicareApiApplication`:

````java
@SpringBootApplication
@EnableScheduling
@EnableAsync
public class SumicareApiApplication {
    public static void main(String[] args) {
        TimeZone.setDefault(TimeZone.getTimeZone("Asia/Manila"));
        SpringApplication.run(SumicareApiApplication.class, args);
    }
}
````

* `@SpringBootApplication` — component scan rooted at `com.sumicare`.
* `@EnableScheduling` — activates the `@Scheduled` jobs in §6.
* `@EnableAsync` — activates `@Async` methods (audit writes, transactional emails).
* The default JVM time zone is forced to **Asia/Manila** before the context starts, so all `OffsetDateTime.now()` / `LocalTime.now()` calls and cron schedules operate in Manila time.

Startup order of note: Liquibase applies `db/changelog/db.changelog-master.xml`, then Hibernate validates the schema (`spring.jpa.hibernate.ddl-auto: validate`).

---

## 2. Package Convention

Every module lives under `com.sumicare.<module>` and is internally layered as `com.sumicare.<module>.<layer>`:

|Layer package|Contents|
|-------------|--------|
|`domain`|JPA entities (`@Entity`)|
|`repository`|Spring Data JPA repository interfaces|
|`service`|`@Service` business logic|
|`controller`|`@RestController` HTTP adapters|
|`dto`|Immutable `record` request/response types, suffixed `Request` / `Response`|

Cross-cutting infrastructure lives in `com.sumicare.common` (config, redis, util, the global exception handler). Modules present: `auth`, `user`, `booking`, `therapist`, `shift`, `transaction`, `cashier`, `pos`, `report`, `attendance`, `recommendation`, `client`, `notification`, `audit`, `content`, `organization`, `room`, `voucher`, `feedback`, `contact`, `dashboard`.

---

## 3. Spring Security

Configuration class: `com.sumicare.auth.config.SecurityConfig` (`@Configuration`, `@EnableMethodSecurity`).

### Filter chain

````java
http
    .cors(cors -> cors.configurationSource(corsConfigurationSource()))
    .csrf(AbstractHttpConfigurer::disable)
    .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
    .authorizeHttpRequests(...)
    .exceptionHandling(ex -> ex.authenticationEntryPoint(...))   // 401 JSON
    .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
````

* **Stateless** — no HTTP session; every request is authenticated from its bearer token.
* **CSRF disabled** — safe given stateless bearer-token auth and no cookie-based session for API calls.
* **`JwtAuthenticationFilter` runs before** `UsernamePasswordAuthenticationFilter`.

### Public vs protected paths

````java
.requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
.requestMatchers("/api/auth/login", "/api/auth/mfa/verify", "/api/auth/mfa/resend",
        "/api/auth/refresh", "/api/auth/logout", "/api/auth/verify",
        "/api/auth/reset-password", "/api/auth/contact-admin-reset",
        "/api/auth/invitations/redeem").permitAll()
.requestMatchers("/api/public/**").permitAll()
.requestMatchers("/api/webhooks/**").permitAll()
.requestMatchers("/api/content/upload").authenticated()
.requestMatchers("/uploads/**").permitAll()
.requestMatchers("/ws/**").permitAll()
.requestMatchers("/actuator/health", "/actuator/info").permitAll()
.anyRequest().authenticated()
````

`/api/webhooks/**` is `permitAll` at the Spring layer but is independently authenticated by HMAC signature (§9). `/ws/**` is the STOMP/SockJS handshake endpoint.

### Per-request JWT validation

`com.sumicare.auth.filter.JwtAuthenticationFilter` (a `OncePerRequestFilter`):

1. If there is no `Authorization: Bearer …` header, continue unauthenticated.
1. Parse the token (`JwtService.parse`). Any `JwtException` is **caught and ignored** — invalid tokens are silently dropped, never surfaced as a parse error; the request simply proceeds unauthenticated and is later rejected by `anyRequest().authenticated()` if the path is protected.
1. Reject (continue unauthenticated) if: the JTI is revoked (`isRevoked`), the token was issued before a user-wide revocation (`isTokenIssuedBeforeRevocation`), the `type` claim is not `"access"`, or no `role` claim is present.
1. Otherwise build `AuthenticatedPrincipal(userId, organizationId, role)` and set a `UsernamePasswordAuthenticationToken` with authority `ROLE_<role>`.

````java
public record AuthenticatedPrincipal(String userId, String organizationId, String role) {}
````

### Role enforcement

`@EnableMethodSecurity` activates `@PreAuthorize` on service/controller methods. Granted authorities are `ROLE_<role>` (e.g. `ROLE_SUPERADMIN`, `ROLE_RECEPTIONIST`), matching the RBAC hierarchy.

### Authentication entry point

Unauthenticated access to a protected resource returns `401` with a JSON body (no redirect, no HTML):

````json
{"message":"Authentication is required to access this resource."}
````

### CORS — wildcard vs specific behavior

`corsConfigurationSource()` reads `sumicare.cors.allowed-origins`:

````java
boolean isWildcard = "*".equals(origins);
if (isWildcard) {
    cors.setAllowedOriginPatterns(List.of("*"));
    cors.setAllowCredentials(false);
} else {
    cors.setAllowedOrigins(Arrays.stream(origins.split(",")).map(String::trim).toList());
    cors.setAllowCredentials(true);
}
````

* **Wildcard (`*`, dev/evaluation):** origin pattern `*`, **credentials disabled** — the spec forbids `Access-Control-Allow-Origin: *` together with credentials.
* **Specific origins (production):** comma-separated exact origins, **credentials enabled** so the `httpOnly` refresh cookie can flow.
* Allowed methods: `GET, POST, PUT, PATCH, DELETE, OPTIONS`. Exposed header: `Authorization`.

This couples to the refresh cookie's `Secure` flag — see §4.

### Password hashing

`PasswordEncoder` is `BCryptPasswordEncoder` with a configurable cost (`sumicare.bcrypt.cost`).

---

## 4. JWT Implementation

Service: `com.sumicare.auth.service.JwtService`. Tokens are HS256, signed with an HMAC key derived from `sumicare.jwt.secret`.

### Issuance

* **Access token** — `type=access`, claims `sub` (userId), `org` (organizationId), `role`; expiry `sumicare.jwt.access-expiry-ms` (15 minutes). Each token has a random `jti`.
* **Refresh token** — `type=refresh`, claims `sub`, `org`; expiry `sumicare.jwt.refresh-expiry-ms` (7 days); random `jti`.
* `issuePair(...)` returns both.

### Refresh endpoint (rotation)

`POST /api/auth/refresh` (`AuthService.refresh`):

1. Read the refresh token from the `sumicare_refresh` `httpOnly` cookie (constant `AuthService.REFRESH_COOKIE`).
1. Validate it is `type=refresh`, not revoked, and not issued before a user-wide revocation.
1. Load the user; reject if inactive, locked, or email-unverified.
1. **Revoke the presented refresh JTI** (deny-list with TTL = remaining lifetime), issue a **new** access token and a **new** refresh token, and write the new refresh cookie. This is rotation-on-every-use.

The frontend never stores the refresh token; it is only ever the `httpOnly` cookie.

### Logout

`AuthService.logout` revokes both the access JTI (if a bearer header is present) and the refresh JTI (from the cookie), then clears the cookie (`maxAge=0`).

### Deny-list and user-wide revocation (Redis)

* `revoke(jti, ttl)` → `revoked:jti:{jti}` String with TTL equal to the token's remaining lifetime. `isRevoked(jti)` checks key existence.
* `revokeAllForUser(userId)` → `user:{userId}:tokens-since` String (millis), TTL = refresh expiry. `isTokenIssuedBeforeRevocation` rejects any token whose `iat` predates that timestamp — a single write invalidates every outstanding token for a user (e.g. on forced logout or role change) without enumerating JTIs.

### Refresh cookie security

The refresh cookie `sumicare_refresh` is emitted with `ResponseCookie` (written as a `Set-Cookie` header), not the servlet `Cookie` API. `secure` and `sameSite` are derived from whether the request is over HTTPS (`isSecureRequest(request)`):

````java
boolean secure = isSecureRequest(request);
ResponseCookie cookie = ResponseCookie.from(REFRESH_COOKIE, value)
        .httpOnly(true)
        .path("/")
        .maxAge(maxAgeSeconds)
        .secure(secure)
        .sameSite(secure ? "None" : "Lax")
        .build();
response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());
````

Over HTTPS the cookie is `Secure; SameSite=None`, so it flows cross-site to the Railway frontend subdomain and a page reload performs a silent refresh that keeps the session. On localhost http it is non-secure with `SameSite=Lax`. Logout writes the same cookie with `maxAge=0` to clear it.

### MFA

`com.sumicare.auth.service.MfaService` (Redis-backed). On login, when MFA applies, a challenge is created; the user verifies a 6-digit code before tokens are issued. MFA is **bypassed only for `SUPERADMIN`**. Keys, TTLs, and limits are in §7.

---

## 5. Global Exception Handling

`com.sumicare.common.GlobalExceptionHandler` (`@RestControllerAdvice`). Client-facing errors map to specific 4xx codes with a flat `{"message": ...}` body and **no stack traces**; only unexpected server errors are logged.

|Handler (`@ExceptionHandler`)|HTTP status|Body|
|-----------------------------|-----------|----|
|`RoomGenderConflictException`|`409 CONFLICT`|`{message: ex.message}`|
|`IllegalArgumentException`|`400 BAD_REQUEST`|`{message: ex.message}`|
|`IllegalStateException`|`400 BAD_REQUEST`|`{message: ex.message}`|
|`AccessDeniedException`|`403 FORBIDDEN`|`{message}` (default `"Access denied"`)|
|`BadCredentialsException`|`401 UNAUTHORIZED`|`{message}` (default `"Invalid credentials"`)|
|`LockedException`|`429 TOO_MANY_REQUESTS`|`{message}` (default `"Too many attempts"`)|
|`MethodArgumentNotValidException`|`400 BAD_REQUEST`|`{message, fields}` — first field error as `message`, full field map in `fields`|
|`HttpMessageNotReadableException`|`400 BAD_REQUEST`|`{message: "The request body is missing or malformed."}`|
|`MissingServletRequestParameterException`|`400 BAD_REQUEST`|`{message: "<param> is required."}`|
|`MethodArgumentTypeMismatchException`|`400 BAD_REQUEST`|`{message: "<name> has an invalid value."}`|
|`DataIntegrityViolationException`|`409 CONFLICT`|`{message: "The request conflicts with existing data."}` (logged `warn`)|
|`EmptyResultDataAccessException`, `NoSuchElementException`|`404 NOT_FOUND`|`{message: "Resource not found"}`|
|`ResponseStatusException`|`ex.statusCode`|`{message: ex.reason}` (fallback message)|
|`Exception` (catch-all)|`500 INTERNAL_SERVER_ERROR`|`{message: "An unexpected error occurred."}` (logged `error`)|

---

## 6. Scheduled Jobs

Activated by `@EnableScheduling`. All jobs are tenant-aware: they iterate `organizationRepository.findAll()` and process each organization independently, isolating per-org failures. Several set a `ROLE_SUPERADMIN` system authentication into the `SecurityContextHolder` for the duration of the sweep and clear it afterward.

|Job (class)|Schedule|Reads|Writes|
|-----------|--------|-----|------|
|`OrderAutoCancelJob.sweep` (`cashier.scheduler`)|`cron = "0 5 0 * * *"`, zone `Asia/Manila`|Orders per org|Cancels elapsed orders via `OrderService.autoCancelElapsedOrders` (PostgreSQL)|
|`SessionAutoEndJob.sweep` (`booking.scheduler`)|`fixedDelay = 60_000`, `initialDelay = 10_000`|Active sessions per org|Ends expired sessions via `BookingService.autoEndExpiredSessions` (PostgreSQL; cascades to room release / notifications)|
|`LineupShiftSyncJob.syncLineup` (`therapist.scheduler`)|`fixedDelay = 60_000`, `initialDelay = 5_000`|Active shifts, shift assignments, therapists, today's clock-ins (`AttendanceService`), current decking lineup (Redis)|Reconciles the Redis decking queue: prepends therapists who should be in the lineup, removes those who should not (skipping `BACKUP`/`MANUAL` entries)|
|`RoomOccupancyReconcilerJob.reconcile` (`room.scheduler`)|`fixedDelay = 120_000`, `initialDelay = 30_000`|Active sessions (PostgreSQL), `room:*:bed:*` hashes (Redis SCAN, count 200)|Marks ACTIVE-but-ended sessions COMPLETED (PostgreSQL); releases orphaned occupied beds in Redis and broadcasts `BED_RELEASED_BY_RECONCILER`|
|`ReportAggregationService.generateDailyReports` (`report.service`)|`cron = "0 5 6 * * *"`, `@Transactional`|Cutoff summary for yesterday per org|Upserts `day_reports` row (JSON payload)|
|`ReportAggregationService.generateMonthlyReports` (`report.service`)|`cron = "0 30 6 1 * *"`, `@Transactional`|Cutoff summary + day reports for previous month per org|Upserts `monthly_reports` row (JSON payload)|

`LineupShiftSyncJob` and `RoomOccupancyReconcilerJob` use `Asia/Manila` (`ZoneId.of("Asia/Manila")`) explicitly for date/time windows.

---

## 7. Redis Usage

Connection factory: `com.sumicare.common.redis.RedisConfig`. A Lettuce `RedisStandaloneConfiguration` is built from `spring.data.redis.url`. The config exposes a `StringRedisTemplate` (used by every key below) and a JSON `RedisTemplate<String,Object>`. (The Lettuce pool — `max-active 16 / max-idle 8 / min-idle 0`, command timeout `2s` — is configured via Spring Boot properties.)

|Key|Type|Writer|Reader|TTL|
|---|----|------|------|---|
|`mfa:challenge:{id}:user`|String|`MfaService.create`/`resend`|`MfaService.verify`/`resend`|15 min|
|`mfa:challenge:{id}:code`|String|`MfaService`|`MfaService.verify`|15 min|
|`mfa:challenge:{id}:attempts`|String (counter)|`MfaService` (`INCR`)|`MfaService.verify` (max 5)|15 min|
|`mfa:challenge:{id}:resends`|String (counter)|`MfaService.resend` (`INCR`)|`MfaService.resend` (max 3)|15 min|
|`revoked:jti:{jti}`|String|`JwtService.revoke`|`JwtService.isRevoked`|remaining token lifetime|
|`user:{userId}:tokens-since`|String (millis)|`JwtService.revokeAllForUser`|`JwtService.isTokenIssuedBeforeRevocation`|refresh expiry|
|`ratelimit:login:{key}`|String (counter)|`LoginRateLimiter.tryConsume` (`INCR`)|same (`<= loginPerMinute`)|1 min (set on first hit)|
|`cancel:code:{bookingId}`|String (`code:email`)|`BookingCancellationCodeService.issue`|`verify`|15 min|
|`cancel:attempts:{bookingId}`|String (counter)|`verify` (`INCR`)|`verify` (max 5)|15 min|
|`cancel:cooldown:{bookingId}`|String|`issue`|`onCooldown`|60 s|
|`decking:active:{orgId}`|ZSet (score = position)|`DeckingService` (append/prepend/rotate/insert)|`currentLineup`, `LineupShiftSyncJob`|—|
|`decking:shift:{orgId}:{shiftId}`|Set|`DeckingService.appendToBack`/`prependToFront`|(membership)|—|
|`decking:skip:{orgId}:{therapistId}`|String|`DeckingService.skip`|`isSkipped`, `currentLineup`|skip duration (≤ 30 min)|
|`decking:flag:{orgId}:{therapistId}`|String (`DeckingFlag`)|`setFlag`/`insertBackup`|`currentLineup`|—|
|`room:{roomId}:bed:{bedId}`|Hash|`RoomOccupancyService.occupy`|`read`, `countOccupiedBeds`, reconciler|— (deleted on release)|
|`ws:sessions:decking`|Set|`WebSocketSessionRegistry.onConnected`|(registry)|— (removed on disconnect)|
|`ws:sessions:roommap`|Set|`WebSocketSessionRegistry.onConnected`|(registry)|— (removed on disconnect)|

Notes:

* **Decking** is a Redis Sorted Set ordered by score; lower score = nearer the front. `prependToFront` inserts at `lowestScore - 1`, `appendToBack`/`rotateToBack` use `Instant.now().toEpochMilli()`, and `insertBackup` interpolates a midpoint score at the requested index. Every mutating operation broadcasts the recomputed lineup to `/topic/decking-updates/{orgId}`.
* **Room occupancy** hash fields: `status`, `clientNickname`, `lockerNumber`, `therapistNickname`, `genderLock`, `ownerItemId`, `startedAt`. `occupy`/`release` broadcast to `/topic/room-updates/{orgId}`.

---

## 8. Audit

The audit subsystem is an interceptor-plus-async-writer pair.

### Interceptor

`com.sumicare.audit.interceptor.AuditInterceptor` (Spring `HandlerInterceptor`, registered in `audit.config.AuditWebConfig`). It acts in `afterCompletion`:

````java
if (!MUTATING_METHODS.contains(request.getMethod())) return;   // POST/PUT/PATCH/DELETE only
if (response.getStatus() >= 400) return;                       // successful requests only
````

It reads the actor from the `SecurityContextHolder` — only proceeding if the principal is an `AuthenticatedPrincipal` — and resolves a `(entity, targetId, action)` triple from the HTTP method and request URI (a large `switch` over the first path segment maps, e.g., `POST /api/cashier/orders/{id}/refund` → action `ORDER.REFUNDED`, entity `ORDER`). It then calls `AuditService.record(...)` with org id, actor id, role, action, entity, target id, and remote address.

### Writer

`com.sumicare.audit.service.AuditService.record(...)` is **`@Async`** — it persists an `AuditLog` to the `audit_logs` table off the request thread, so audit logging never blocks or fails the originating response. Audit rows are immutable (insert-only), supporting non-repudiation.

---

## 9. PayMongo Integration

Two modules cooperate on payments: **`cashier`** owns orders/ledger domain logic (`OrderService`, `PackageService`, entities `Order`, `OrderItem`, `OrderItemAttendee`, `Package`, `PackageTier`, `DiscountTemplate`, `LedgerAccount`); **`pos`** owns the gateway integration (`PosTransaction`, `TransactionLedgerEntry`, `PayMongoService`, `PayMongoGateway`, `WebhookController`).

### Gateway

`com.sumicare.pos.gateway.PayMongoGateway implements PaymentGateway`, talking to `https://api.paymongo.com/v1` with HTTP Basic auth (secret key, Base64-encoded with a trailing colon).

|Method|PayMongo call|Purpose|
|------|-------------|-------|
|`createIntent`|`POST /payment_intents`|Create an intent; amount converted to centavos; allowed methods derived from the payment method (`card` for CREDIT/DEBIT, `gcash` for GCASH); 3-D Secure requested for cards|
|`createPaymentMethod`|`POST /payment_methods`|Build a `card` or `gcash` payment method with optional billing|
|`attachIntent`|`POST /payment_intents/{id}/attach`|Attach a payment method, supplying `client_key` and `return_url`; returns status + `next_action.redirect.url`|
|`retrieveIntent`|`GET /payment_intents/{id}`|Poll intent status|
|`createCheckoutSession`|`POST /checkout_sessions`|Hosted checkout (used for card flows); returns `checkout_url`|
|`retrieveCheckoutSession`|`GET /checkout_sessions/{id}`|Poll checkout/payment status|
|`retrievePaymentId`|`GET /checkout_sessions/{id}` or `/payment_intents/{id}`|Resolve the concrete `payment` id (needed for refunds)|
|`refund`|`POST /refunds`|Refund a payment by `payment_id`, amount, and normalized reason|
|`verifyWebhook`|(local HMAC)|Verify webhook authenticity (below)|
|`capture`|(no-op)|Returns the intent id unchanged (capture is automatic)|

### Service & redirect flow

`com.sumicare.pos.service.PayMongoService` orchestrates charges:

* `supports(method)` — accepts `CREDIT`, `DEBIT`, `GCASH`.
* **Mock mode** — when `sumicare.payment.paymongo.mock-mode` is true (or an intent id starts with `mock_pi_`), charges and refunds are simulated locally; a set of test card numbers force a decline. No external call is made.
* **Card** charges use a **hosted checkout session** (`createCheckoutSession`) and return its `checkout_url` for redirect.
* **GCash** charges create an intent, create a payment method, and attach it with a `return_url`, returning `next_action` redirect info. The return URL is built from the public base origin (or request origin) and points at the cashier (or supplied return path) with `paymongoReturn=1`, `orderId`, `intent`, `paymentMethod`, and `amount` query parameters.
* `confirm(intentId)` resolves a final status: mock → `succeeded`; `cs_…` → checkout session (`paid` ⇒ `succeeded`); otherwise intent status.
* `refund(...)` resolves the payment id then calls the gateway, normalizing the reason to one of `duplicate`, `fraudulent`, `requested_by_customer`, `others`.

### Webhook verification & reconciliation

`com.sumicare.pos.controller.WebhookController` handles `POST /api/webhooks/paymongo`:

1. **Signature is mandatory.** It reads the `Paymongo-Signature` header and calls `payMongoGateway.verifyWebhook(payload, signature)`. Verification recomputes `Base64(HmacSHA256(webhookSecret + "." + payload))` and compares it to the supplied signature. A missing or invalid signature returns `401 Invalid signature` — the body is never processed.
1. **`payment.paid` / `source.chargeable`** → `reconcile(orderId, amount, method, gatewayId)`:
   * Looks up the `Order` by metadata `orderId`, and its `Booking` by `bookingId`.
   * If a booking exists, sets `paymentStatus = "PAID"` and `gatewayPaymentId`, then saves.
   * If an order exists, calls `OrderService.settleGatewayPayment(order, …)` and returns.
   * Otherwise (no order) writes a `PAYMENT_RECEIVED` `TransactionLedgerEntry` directly, guarded by `existsByGatewayReference(gatewayId)` for idempotency.
1. **`refund.updated` / `payment.refunded`** (status blank or `succeeded`) → `reconcileRefund(orderId, amount, refundId)` → `OrderService.reconcileRefund(...)`.

The `transaction_ledger` is append-only; settlement and refund handling write new entries and propagate `payment_status` back onto `bookings`/`orders` (status writeback).

---

## 10. Liquibase

* **Master changelog:** `apps/sumicare-api/src/main/resources/db/changelog/db.changelog-master.xml`, wired via `spring.liquibase.change-log`.
* **Structure:** the master `<include>`s individual change files under `changes/`, in order, from `0001-init-schema.xml` through `0060-preferred-therapist.xml`.
* **Changeset naming:** ids follow `NNNN-name` (e.g. `0001-organizations`) with `author="sumicare"`. The seeded tenant (`slug = lasema`, `New Lasema Spa Jjimjilbang`) and the 14-service catalogue are inserted by `0002-seed-reference-data.xml`.
* **Startup behavior:** Liquibase runs all pending changesets against PostgreSQL during context startup, then Hibernate validates the schema (`ddl-auto: validate`). Hibernate never alters the schema — migrations are the single source of truth.
* **Failure behavior:** a failing changeset aborts Liquibase, which fails application startup (the context does not come up with a partially migrated or mismatched schema); a schema that does not match the JPA entities likewise fails the subsequent Hibernate `validate` step.

---

## 11. Email Service

### Configuration & strategy

`EmailSender` (`com.sumicare.auth.email.EmailSender`) is the strategy port with two implementations selected by `sumicare.email.provider`:

* `SmtpEmailSender` — `@ConditionalOnProperty(name = "sumicare.email.provider", havingValue = "smtp", matchIfMissing = true)`. The default. Uses Spring's `JavaMailSender` to build a (possibly multipart) MIME message with inline images and attachments; `From` is `sumicare.app.email-from`.
* `BrevoEmailSender` — `@ConditionalOnProperty(... havingValue = "brevo")`. Posts to the Brevo transactional API (`https://api.brevo.com/v3/smtp/email`) with `api-key` header; inline images and attachments are Base64-encoded.

### Orchestrator and triggers

`com.sumicare.auth.service.EmailService` builds the HTML bodies (all branded "New Lasema Spa Jjimjilbang", footer "Powered by SumiCare") and delegates to the active `EmailSender`. Most senders are `@Async`. Triggers:

|Method|Trigger|Notable content|
|------|-------|---------------|
|`sendVerificationEmail`|Account created|Link `/sumicare/verify?token=…`, 24 h|
|`sendPasswordResetEmail`|Password reset requested|Link `/sumicare/reset-password?token=…`, 1 h|
|`sendInvitationEmail`|User invited|Link `/sumicare/invite?token=…`, 30 min|
|`sendMfaCodeEmail`|MFA challenge issued|6-digit code, 15 min|
|`sendCancellationCodeEmail`|Public booking cancellation requested|Code; returns success boolean (recipient masked in logs on failure)|
|`sendCancellationConfirmedEmail`|Cancellation completed|Cancelled services, optional refund line|
|`sendPaymentConfirmationEmail`|Payment received|Reference, OR #, method, amount, balance|
|`sendBookingConfirmationEmail`|Booking confirmed|Packages, reservation type, schedule, room, total|
|`sendCompletionEmail`|Visit completed|Receipt PDF + treatment-slip PDF attachments; inline feedback QR; feedback link|
|`sendAdminPasswordResetNotice`|`contact-admin-reset` submitted|Notifies org admins to send a reset link|

### Link building

`com.sumicare.common.util.BaseUrlResolver.resolve()` returns `sumicare.app.public-base-url` (trailing slashes stripped) or the default `https://newlasemaspa.up.railway.app`. Every email link and the feedback QR are built on this base.
