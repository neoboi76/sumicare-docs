---
title: "backend-modules"
date: 2026-07-12
categories: ["System Guide"]
---

# SumiCare Backend — Module Reference

This document describes the Spring Boot backend (`apps/sumicare-api`) module by module. Source root: `apps/sumicare-api/src/main/java/com/sumicare/`. Every entity field, repository query method, service signature, controller endpoint, DTO, and `@PreAuthorize` rule below was transcribed directly from source.

## Conventions observed across the codebase

* **Architecture:** `Controller → Service → Repository → Domain`. Controllers are thin; business logic is in services.
* **Tenancy:** Almost every entity carries `organization_id` (`uuid`). The authenticated principal `AuthenticatedPrincipal(String userId, String organizationId, String role)` is extracted from the JWT by `JwtAuthenticationFilter` and injected via `@AuthenticationPrincipal`.
* **Roles (RBAC):** `SUPERADMIN`, `ADMIN`, `MANAGER`, `RECEPTIONIST` (there is no `STAFF` role). Method security is enabled (`@EnableMethodSecurity`); rules are expressed as `hasAnyRole(...)` on services and/or controllers.
* **IDs:** Most domain aggregates use `@GeneratedValue` `uuid` PKs; reference/lookup tables (`Role`, `Shift`, `ShiftAssignment`, `Service`, `Package`, `PackageTier`, `Commission`, `TransactionLedgerEntry`, `RecommendationWeight`, `TherapistAttendance`, `AuditLog`) use `IDENTITY` `Long` PKs.
* **Timezone:** `Asia/Manila` (`ZoneId.of("Asia/Manila")`) is used throughout for day/shift boundaries.
* **No code comments** anywhere (project convention); code is self-documenting.

---

## auth

### Purpose

JWT issuance/parsing, Spring Security configuration, login with email-based MFA, refresh-token rotation, logout/revocation, login rate limiting, email verification, password reset, and invitation redemption.

### Domain entities

#### `EmailVerificationToken` — `@Table("email_verification_tokens")`

|Field|Column|Type / notes|
|-----|------|------------|
|`id`|`id`|`uuid`, `@GeneratedValue`|
|`userId`|`user_id`|`uuid`, not null|
|`token`|`token`|`varchar(128)`, unique, not null|
|`expiresAt`|`expires_at`|`OffsetDateTime`, not null|
|`tokenType`|`token_type`|`varchar(20)`, not null, default `"VERIFY_EMAIL"` (also `"INVITATION"`)|
|`consumedAt`|`consumed_at`|`OffsetDateTime`|
|`createdAt`|`created_at`|`OffsetDateTime`, not null, not updatable|

Helpers: `isExpired()`, `isConsumed()`.

#### `PasswordResetToken` — `@Table("password_reset_tokens")`

Same shape as above minus `tokenType`: `id`, `userId`, `token` (unique, 128), `expiresAt`, `consumedAt`, `createdAt`. Helpers `isExpired()`, `isConsumed()`.

### Repository layer

#### `EmailVerificationTokenRepository extends JpaRepository<EmailVerificationToken, UUID>`

* `Optional<EmailVerificationToken> findByToken(String token)`
* `Optional<EmailVerificationToken> findByTokenAndTokenType(String token, String tokenType)`
* `@Modifying @Query("DELETE FROM EmailVerificationToken t WHERE t.userId = :userId") void deleteAllByUserId(UUID userId)`

#### `PasswordResetTokenRepository extends JpaRepository<PasswordResetToken, UUID>`

* `Optional<PasswordResetToken> findByToken(String token)`

### Service layer

#### `AuthService`

Constants: `REFRESH_COOKIE = "sumicare_refresh"`, `MAX_FAILED_ATTEMPTS = 2`.

* `LoginResponse login(LoginRequest, HttpServletRequest, HttpServletResponse)` — rate-limits via three keys (`ip:`, `user:`, `ip:…:user:`); authenticates via `AuthenticationManager`. Maps `DisabledException`→`AccessDeniedException("User is deactivated")`, `LockedException`→locked message, `BadCredentialsException`→increments failed attempts (locks after >2). Requires `emailVerified`. `SUPERADMIN` bypasses MFA and logs in directly; everyone else gets an MFA email challenge (`MfaService.create` + `EmailService.sendMfaCodeEmail`). Records `USER.LOGIN` audit on completion.
* `TokenResponse verifyMfa(String challengeId, String code, …)` — `MfaService.verify` → `completeLogin`.
* `void resendMfa(String challengeId)` — `MfaService.resend` then re-sends code email.
* `TokenResponse refresh(HttpServletRequest, HttpServletResponse)` — reads refresh cookie, validates `type=refresh`, checks jti revocation and per-user revocation timestamp, ensures user active/unlocked/verified, revokes old jti, issues new pair, rotates cookie.
* `void logout(…)` — revokes access and refresh jtis (best-effort), clears cookie.
* Throws: `LockedException`, `AccessDeniedException`, `BadCredentialsException`.

#### `JwtService`

HS256 signing key from `appProperties.jwt().secret()`. Backed by `StringRedisTemplate`.

* `String issueAccessToken(UUID userId, UUID organizationId, String role)` — claims `org`, `role`, `type=access`, random jti.
* `String issueRefreshToken(UUID userId, UUID organizationId)` — `type=refresh`.
* `Claims parse(String token)`; `boolean isRevoked(String jti)`; `void revoke(String jti, Duration ttl)`.
* `void revokeAllForUser(UUID userId)` — sets `user:{id}:tokens-since` cutoff.
* `boolean isTokenIssuedBeforeRevocation(String userId, long issuedAtEpochSecond)`.
* `Map<String,String> issuePair(...)`.

#### `LoginRateLimiter`

* `boolean tryConsume(String key)` — `INCR` on `ratelimit:login:{key}`, sets 1-minute expiry on first hit; allowed while count ≤ `properties.rateLimit().loginPerMinute()`.

#### `MfaService`

TTL 15 min, `MAX_ATTEMPTS=5`, `MAX_RESENDS=3`. 6-digit codes via `SecureRandom`.

* `Challenge create(UUID userId)`, `Challenge resend(String challengeId)`, `UUID verify(String challengeId, String code)`.
* Inner record `Challenge(String challengeId, UUID userId, String code)`.

#### `EmailService`

`@Async` HTML email senders (via `EmailSender` abstraction): `sendVerificationEmail`, `sendInvitationEmail`, `sendMfaCodeEmail`, `sendPasswordResetEmail`, `sendAdminPasswordResetNotice`, `sendCancellationCodeEmail` (returns `boolean`), plus booking/order/payment emails used by other modules. Links resolved via `BaseUrlResolver`.

#### `UserDetailsServiceImpl implements UserDetailsService`

* `loadUserByUsername(String)` — loads `User`; rejects null password hash ("Account setup is not complete"); rejects accounts with no role assigned ("Account has no role assigned"); grants `ROLE_{roleCode}`; maps active/locked flags into `UserDetails`.

### Controller layer

#### `AuthController` — `@RequestMapping("/api/auth")` (all endpoints public per `SecurityConfig`)

|Method|Path|Body|Returns|
|------|----|----|-------|
|POST|`/login`|`LoginRequest`|`LoginResponse`|
|POST|`/mfa/verify`|`MfaVerifyRequest`|`TokenResponse`|
|POST|`/mfa/resend`|`MfaResendRequest`|void|
|POST|`/refresh`|— (refresh cookie)|`TokenResponse`|
|POST|`/logout`|—|void|
|POST|`/reset-password`|`ResetPasswordRequest`|void (→ `UserService.consumePasswordReset`)|
|POST|`/contact-admin-reset`|`ContactAdminResetRequest`|`Map<String,String>` — stores a `[PASSWORD_RESET_REQUEST]` `ContactMessage`, emails org admins|
|POST|`/invitations/redeem`|`RedeemInvitationRequest`|void (→ `UserService.redeemInvitation`)|

#### `EmailVerificationController` — `@RequestMapping("/api/auth")`

* `GET /verify?token=...` — `@Transactional`; consumes token, sets `User.emailVerified=true`, HTTP 302 redirect to `/sumicare/login?verified=...` (`1`/`already`/`expired`/`invalid`).

### Security configuration — `SecurityConfig` (`@EnableMethodSecurity`)

* Stateless sessions, CSRF disabled, custom CORS (`*` → `allowedOriginPatterns("*")` with credentials off; otherwise explicit origins with credentials on).
* `permitAll`: all `/api/auth/*` listed endpoints, `/api/public/**`, `/api/webhooks/**`, `/uploads/**`, `/ws/**`, `/actuator/health`, `/actuator/info`, all `OPTIONS`. `/api/content/upload` requires authentication; everything else `authenticated()`.
* `JwtAuthenticationFilter` registered before `UsernamePasswordAuthenticationFilter`.
* `PasswordEncoder` = `BCryptPasswordEncoder(appProperties.bcrypt().cost())`.

### DTOs

|DTO|Fields|Purpose|
|---|------|-------|
|`LoginRequest`|`@NotBlank username`, `@NotBlank password`|credentials|
|`LoginResponse`|`boolean mfaRequired`, `String challengeId`, `String email`, `TokenResponse token`|factory methods `authenticated(token)` / `mfaChallenge(challengeId, maskedEmail)`|
|`TokenResponse`|`accessToken`, `tokenType`, `long expiresIn`, `role`|issued access token|
|`MfaVerifyRequest`|`@NotBlank challengeId`, 6-char `code`|MFA step|
|`MfaResendRequest`|`@NotBlank challengeId`|resend code|
|`ResetPasswordRequest`|`token`, `newPassword`|reset|
|`RedeemInvitationRequest`|`token`, 8–128 `password`|invite acceptance|
|`ContactAdminResetRequest`|`name`, `@Email email`, `message`|admin-assisted reset|

### Inter-module dependencies

Calls `user` (`UserService`, `UserRepository`), `audit` (`AuditService`), `organization` (`OrganizationRepository`), `contact` (`ContactMessageRepository`). Consumed by every protected controller (filter populates the principal).

### Redis interactions

* `revoked:jti:{jti}` (String, TTL) — access/refresh token revocation deny-list.
* `user:{userId}:tokens-since` (String, TTL) — bulk per-user revocation cutoff.
* `ratelimit:login:{key}` (counter, 1-min expiry) — login throttling.
* `mfa:challenge:{challengeId}:{user|code|attempts|resends}` (Strings, 15-min TTL) — MFA challenge state.

---

## user

### Purpose

User CRUD, role assignment, profile self-service, password-reset issuance, invitation flow, account lock/unlock, deactivate/reactivate, with role-tier enforcement.

### Domain entities

#### `Role` — `@Table("roles")`

`id` (`Long`, IDENTITY), `code` (unique, not null), `displayName` (`display_name`, not null), `hierarchyLevel` (`hierarchy_level`, int, not null).

#### `User` — `@Table("users")`

|Field|Column|Notes|
|-----|------|-----|
|`id`|`id`|`uuid`|
|`organizationId`|`organization_id`|`uuid`, not null|
|`username`|`username`|unique, not null|
|`email`|`email`|nullable|
|`passwordHash`|`password_hash`|nullable (invite flow)|
|`role`|`role_id`|`@ManyToOne(EAGER, optional=false)` → `Role`|
|`displayName`|`display_name`||
|`active`|`is_active`|not null, default true|
|`emailVerified`|`email_verified`|not null, default false|
|`failedLoginAttempts`|`failed_login_attempts`|not null, default 0|
|`accountLocked`|`account_locked`|not null, default false|
|`createdAt` / `updatedAt`|`created_at` / `updated_at`|`OffsetDateTime`|

### Repository layer

#### `RoleRepository extends JpaRepository<Role, Long>`

* `Optional<Role> findByCode(String code)`

#### `UserRepository extends JpaRepository<User, UUID>`

* `findByUsername`, `findByEmail`, `findAllByOrganizationId`, `findAllByOrganizationIdAndActiveTrue`, `findAllByOrganizationIdAndActiveFalse`, `existsByUsername`, `existsByEmailIgnoreCase`.

### Service layer — `UserService`

`ADMIN_EDITABLE_ROLES = {MANAGER, RECEPTIONIST}`. Tier helpers: `enforceTierForTarget` (ADMIN may only manage editable roles; SUPERADMIN may not modify another SUPERADMIN) and `enforceTierForNewRole`.

|Method|`@PreAuthorize`|Summary|
|------|---------------|-------|
|`listForOrganization(orgId)`|`SUPERADMIN,ADMIN,MANAGER`|active users → `UserResponse`|
|`listDeactivatedForOrganization(orgId)`|`SUPERADMIN,ADMIN,MANAGER`|inactive users|
|`createUser(orgId, actorRole, CreateUserRequest)`|`SUPERADMIN,ADMIN`|`@Transactional`; rejects duplicate email; invite flow when password blank (null hash, sends INVITATION token, 30 min) else direct create (verify-email token, 24 h)|
|`updateUser(actorId, actorRole, userId, UpdateUserRequest)`|`SUPERADMIN,ADMIN`|same-org + tier checks; updates fields, validates password strength|
|`deactivateUser(actorId, actorRole, userId)`|`SUPERADMIN,ADMIN`|cannot self-deactivate; revokes all tokens via `JwtService.revokeAllForUser`|
|`reactivateUser` / `unlockUser`|`SUPERADMIN,ADMIN`|reactivate / clear lock + attempts|
|`updateProfile(userId, UpdateProfileRequest)`|(self)|updates `displayName`|
|`requestPasswordReset(userId)`|(self)|issues `PasswordResetToken` (1 h), emails reset link|
|`sendResetLink(actorId, actorRole, targetUserId)`|`SUPERADMIN,ADMIN`|admin-initiated reset|
|`consumePasswordReset(token, newPassword)`|—|validates strength, consumes token|
|`redeemInvitation(token, newPassword)`|—|sets hash, `emailVerified=true`, consumes INVITATION token|
|`getById(userId)`|—|self/me lookup|
|`listOrgAdmins(orgId)`|—|ADMIN+SUPERADMIN users|

Password strength: ≥8 chars, upper, lower, digit, special. Throws `AccessDeniedException`, `IllegalArgumentException`, `IllegalStateException`.

### Controller layer — `UserController` `@RequestMapping("/api/users")`

|Method|Path|`@PreAuthorize`|Body|Returns|
|------|----|---------------|----|-------|
|GET|`/`|(service-level)|—|`List<UserResponse>` (org-scoped)|
|GET|`/deactivated`|`SUPERADMIN,ADMIN,MANAGER`|—|`List<UserResponse>`|
|POST|`/`|`SUPERADMIN,ADMIN`|`CreateUserRequest`|`UserResponse`|
|PATCH|`/{userId}`|`SUPERADMIN,ADMIN`|`UpdateUserRequest`|`UserResponse`|
|DELETE|`/{userId}`|`SUPERADMIN,ADMIN`|—|204|
|POST|`/{userId}/reactivate`|`SUPERADMIN,ADMIN`|—|`UserResponse`|
|POST|`/{userId}/unlock`|`SUPERADMIN,ADMIN`|—|`UserResponse`|
|GET|`/me`|—|—|`UserResponse`|
|PATCH|`/me/profile`|—|`UpdateProfileRequest`|`UserResponse`|
|POST|`/me/request-password-reset`|—|—|void|
|POST|`/{userId}/send-reset-link`|`SUPERADMIN,ADMIN`|—|204|

### DTOs

* `CreateUserRequest(username, email, password?, role, displayName)` — blank password ⇒ invite flow.
* `UpdateUserRequest(email, password, role, displayName, Boolean active)` — all optional/patch.
* `UpdateProfileRequest(displayName)`.
* `UserResponse(id, organizationId, username, email, role, displayName, active, accountLocked, createdAt)`.

### Inter-module dependencies

Depends on `auth` (`EmailService`, `JwtService`, token repos). Used by `auth` (`AuthController`, `AuthService`), `audit` (interceptor labels), and indirectly by all role checks.

### Redis interactions

Indirect — `deactivateUser` triggers `JwtService.revokeAllForUser` (writes `user:{id}:tokens-since`).

 > 
 > Bootstrap note: `user/bootstrap/SuperadminBootstrap` seeds an initial SUPERADMIN on startup (out of the documented API surface).

---

## organization

### Purpose

Per-tenant branding/CMS metadata (logo, colors, theme, contact info) exposed publicly by slug and editable internally.

### Domain entity — `Organization` `@Table("organizations")`

`id` (`uuid`), `slug` (`slug`, unique, not null), `displayName` (`display_name`, not null), and nullable branding fields: `logoUrl`, `primaryColor`, `secondaryColor`, `accentColor`, `theme`, `fontFamily`, `loginBackgroundUrl`, `faviconUrl`, `instagramUrl`, `contactPhone`, `contactEmail`, `footerNote`, plus `active` (`is_active`, not null, default true), `createdAt`, `updatedAt`.

### Repository — `OrganizationRepository extends JpaRepository<Organization, UUID>`

* `Optional<Organization> findBySlug(String slug)`

### Service — `OrganizationService`

* `OrganizationBrandingResponse getBranding(UUID organizationId)`
* `OrganizationBrandingResponse getBrandingBySlug(String slug)`
* `OrganizationBrandingResponse updateBranding(UUID organizationId, UpdateBrandingRequest)` — `@Transactional`, `@PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER')")`; null-safe patch (blank → null for URL/contact fields).

### Controller — `OrganizationController`

|Method|Path|Auth|Returns|
|------|----|----|-------|
|GET|`/api/public/branding/{slug}`|public|`OrganizationBrandingResponse`|
|GET|`/api/organization/branding`|authenticated (principal org)|`OrganizationBrandingResponse`|
|PUT|`/api/organization/branding`|service `@PreAuthorize`|`OrganizationBrandingResponse`|

### DTOs

* `OrganizationBrandingResponse` — full read model (id, slug, displayName + all branding fields).
* `UpdateBrandingRequest` — same branding fields, all optional.

### Inter-module dependencies

`OrganizationRepository` is the most widely reused repository: every public controller resolves the org by slug through it. No outbound calls.

### Redis interactions

None.

---

## client

### Purpose

Optional, non-critical client accounts identified by nickname/email for usage tracking, voucher eligibility, and recommendations. Soft-deleted via `deletedAt`.

### Domain entity — `Client` `@Table("clients")`

`id` (`uuid`), `organizationId` (not null), `nickname` (not null), `email`, `facebookHandle` (`facebook_handle`), `gender`, `nationality`, `consentGiven` (`consent_given`, default false), `dataTrackingConsent` (`data_tracking_consent`, default false), `createdAt`, `deletedAt`.

### Repository — `ClientRepository extends JpaRepository<Client, UUID>`

* `Optional<Client> findByOrganizationIdAndEmailAndDeletedAtIsNull(orgId, email)`
* `boolean existsByOrganizationIdAndEmailIgnoreCaseAndDeletedAtIsNull(orgId, email)`
* `List<Client> findAllByOrganizationIdAndDeletedAtIsNullOrderByNicknameAsc(orgId)`
* `List<Client> findTop20ByOrganizationIdAndNicknameContainingIgnoreCaseAndDeletedAtIsNullOrderByNicknameAsc(orgId, fragment)`

### Service layer

None — logic lives directly in the controller.

### Controller — `ClientController`

|Method|Path|`@PreAuthorize`|Body/Params|Notes|
|------|----|---------------|-----------|-----|
|POST|`/api/public/clients/{slug}`|public|`Client`|self-register; requires nickname+email; 409 on duplicate email|
|GET|`/api/public/clients/{slug}/check-email?email=`|public|—|returns availability `boolean`|
|GET|`/api/clients?q=`|`SUPERADMIN,ADMIN,MANAGER,RECEPTIONIST`|—|list or top-20 nickname search|
|POST|`/api/clients`|`SUPERADMIN,ADMIN,MANAGER,RECEPTIONIST`|`Client`|internal create|
|DELETE|`/api/clients/{id}`|`SUPERADMIN,ADMIN,MANAGER`|—|soft delete (sets `deletedAt`); 403 cross-org|

### Inter-module dependencies

Uses `OrganizationRepository`. `ClientRepository` consumed by `booking`, `voucher`.

### Redis interactions

None.

---

## therapist

### Purpose

Therapist profiles plus the live **decking (lineup) queue** maintained in Redis, including skip/break handling, flags, backup insertion, and rotation.

### Domain entity — `Therapist` `@Table("therapists")`

Unique constraints on `(organization_id, staff_number)` and `(organization_id, nickname)`.  
`id` (`uuid`), `organizationId`, `staffNumber` (`staff_number`), `nickname` (not null), `gender` (not null), `backup` (`is_backup`, default false), `active` (`is_active`, default true), `createdAt`.

### Repository — `TherapistRepository extends JpaRepository<Therapist, UUID>`

* `findAllByOrganizationId`, `findAllByOrganizationIdAndActiveTrue`, `findAllByOrganizationIdAndActiveFalse`
* `Optional<Therapist> findByOrganizationIdAndStaffNumber(orgId, staffNumber)`
* `existsByOrganizationIdAndNickname`, `existsByOrganizationIdAndStaffNumber`

### Service layer

#### `DeckingService` (Redis-backed sorted-set queue)

Key helpers: `queueKey = decking:active:{orgId}`, `shiftMembersKey = decking:shift:{orgId}:{shiftId}`, `skipKey = decking:skip:{orgId}:{therapistId}`, `flagKey = decking:flag:{orgId}:{therapistId}`.

* `appendToBack(orgId, therapistId, shiftId)` — `ZADD` score = now; adds to shift set; broadcasts.
* `prependToFront(orgId, therapistId, shiftId)` — score = lowest − 1 (newest shift preempts).
* `rotateToBack(orgId, therapistId)` — re-score to now (post-service rotation).
* `servedRequested(orgId, therapistId)` — requested therapist keeps position; just re-broadcasts.
* `remove(orgId, therapistId)` — `ZREM` + broadcast.
* `skip(orgId, therapistId, Duration)` / `cancelSkip(...)` — set/delete skip key (max 30 min enforced by caller).
* `setFlag` / `clearFlag` — store a `DeckingFlag`.
* `insertBackup(orgId, therapistId, positionFromTop)` — interpolates a score between neighbours, flags `BACKUP`.
* `List<DeckingEntry> currentLineup(orgId)` — `ZRANGE` with scores, resolving flag + skip state.
* `boolean isSkipped(orgId, therapistId)`.
* enum `DeckingFlag { NONE, REQUESTED, SCRUB, ORDINARY, BACKUP, MANUAL }`.

Each mutating op calls `NotificationService.broadcastDecking`.

#### `TherapistService`

* `listForOrganization(orgId)` / `listDeactivatedForOrganization(orgId)` → `TherapistResponse` (with resolved current shift label).
* `create(orgId, CreateTherapistRequest)` — `@PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER')")`, `@Transactional`; uniqueness checks; optional shift assignment; calls `LineupShiftSyncJob.syncNow`.
* `update(orgId, therapistId, CreateTherapistRequest)` — same auth; reassigns shift; re-syncs lineup.
* `requireById(id)`, `toResponse(Therapist)`.

### Scheduler — `LineupShiftSyncJob`

`@Scheduled(fixedDelay=60s, initialDelay=5s)` over all orgs; also `syncNow(orgId)`. Computes who *should* be in the lineup (active, non-backup therapists assigned to shifts covering "now" in Manila), filters to clocked-in therapists when any clock-ins exist that day (`AttendanceService.clockedInTherapistIds`), prepends new entrants to front (latest-shift-first), and removes stale non-`BACKUP`/non-`MANUAL` entries.

### Controller layer

#### `DeckingController` `@RequestMapping("/api/decking")` — class-level `@PreAuthorize("hasAnyRole('RECEPTIONIST','MANAGER','ADMIN','SUPERADMIN')")`

|Method|Path|Params|Returns / behaviour|
|------|----|------|-------------------|
|GET|`/`|—|`List<DeckingEntry>`|
|GET|`/lineup`|—|`List<LineupTherapistResponse>` (excludes BACKUP; marks `onCall` via active sessions)|
|POST|`/{therapistId}/skip`|`minutes` (default 30, capped 30)|break|
|POST|`/{therapistId}/skip/cancel`|—|cancel break|
|POST|`/{therapistId}/flag`|`flag` (`DeckingFlag`)|set flag|
|POST|`/{therapistId}/rotate`|—|rotate to back|
|POST|`/backup/{therapistId}`|`position` (default 0)|manual backup insert|
|DELETE|`/{therapistId}`|—|remove (rejects if active session)|
|POST|`/{therapistId}`|`shiftId?`|add to lineup; without shift, flags `MANUAL`|

#### `TherapistController` `@RequestMapping("/api/therapists")`

|Method|Path|`@PreAuthorize`|
|------|----|---------------|
|GET|`/`|(authenticated)|
|GET|`/deactivated`|`SUPERADMIN,ADMIN,MANAGER`|
|POST|`/`|`SUPERADMIN,ADMIN,MANAGER`|
|PATCH|`/{therapistId}`|`SUPERADMIN,ADMIN,MANAGER`|
|DELETE|`/{therapistId}`|`SUPERADMIN,ADMIN,MANAGER` (deactivate)|
|POST|`/{therapistId}/reactivate`|`SUPERADMIN,ADMIN,MANAGER`|

### DTOs

* `CreateTherapistRequest(staffNumber, nickname, gender, boolean backup, Long shiftId)`.
* `DeckingEntry(UUID therapistId, double position, String flag, boolean skipped)`.
* `LineupTherapistResponse(therapistId, nickname, gender, shiftLabel, flag, boolean skipped, int position, boolean onCall)`.
* `TherapistResponse(id, staffNumber, nickname, gender, boolean backup, boolean active, Long currentShiftId, String currentShiftLabel)`.

### Inter-module dependencies

`DeckingService` → `notification`. `TherapistService`/`LineupShiftSyncJob` → `shift`, `attendance`, `organization`. Consumed by `booking` (session start rotates/serves therapists), `report`, `dashboard`, `cashier` (commissions).

### Redis interactions

`decking:active:{orgId}` (ZSET), `decking:shift:{orgId}:{shiftId}` (Set), `decking:skip:{orgId}:{therapistId}` (String, TTL), `decking:flag:{orgId}:{therapistId}` (String).

---

## shift

### Purpose

Shift definitions and therapist↔shift assignments; resolves the active shift for "latest shift first" lineup logic.

### Domain entities

#### `Shift` — `@Table("shifts")`

`id` (`Long`, IDENTITY), `organizationId`, `label` (not null), `startTime`/`endTime` (`LocalTime`, not null), `expectedCount` (`expected_count`, nullable), `active` (`is_active`, default true).

#### `ShiftAssignment` — `@Table("shift_assignments")`

`id` (`Long`), `shiftId` (`shift_id`, not null), `therapistId` (`therapist_id`, `uuid`, not null), `effectiveDate` (`effective_date`, `LocalDate`, default today).

### Repository layer

#### `ShiftRepository extends JpaRepository<Shift, Long>`

* `List<Shift> findAllByOrganizationIdAndActiveTrue(orgId)`

#### `ShiftAssignmentRepository extends JpaRepository<ShiftAssignment, Long>`

* `findAllByShiftId`, `findAllByTherapistId`, `existsByShiftIdAndTherapistId`
* `@Modifying @Query` `deleteByShiftIdAndTherapistId(shiftId, therapistId)`
* `@Modifying @Query` `deleteAllByTherapistId(therapistId)`

### Service — `ShiftService`

* `List<Shift> listActive(orgId)`
* `Optional<Shift> resolveActiveShiftFor(orgId, LocalTime now)` — covering shift with the latest start time.
* `boolean coversTime(start, end, now)` — handles overnight shifts (end ≤ start).

### Controller — `ShiftController` `@RequestMapping("/api/shifts")`

|Method|Path|`@PreAuthorize`|
|------|----|---------------|
|GET|`/`|(authenticated)|
|POST|`/`|`SUPERADMIN,ADMIN,MANAGER`|
|PATCH|`/{shiftId}`|`SUPERADMIN,ADMIN,MANAGER`|
|DELETE|`/{shiftId}`|`SUPERADMIN,ADMIN,MANAGER` (deactivate)|

Request/response body is the `Shift` entity directly. Cross-org access throws `AccessDeniedException`.

### DTOs

None — entity used directly.

### Inter-module dependencies

Consumed by `therapist` (sync job, decking labels), `report` (commission/decking reports).

### Redis interactions

None directly (lineup membership written by `therapist`).

---

## attendance

### Purpose

Records therapist attendance events (clock-in/out, absence, day-off) and answers "who is clocked in today" for lineup filtering.

### Domain entity — `TherapistAttendance` `@Table("therapist_attendance")`

`id` (`Long`), `therapistId` (`uuid`, not null), `eventType` (`event_type`, not null — `CLOCK_IN`/`CLOCK_OUT`/`ABSENT`/`DAY_OFF`), `eventAt` (`event_at`, not null), `deviceId` (`device_id`), `remarks`.

### Repository — `TherapistAttendanceRepository extends JpaRepository<TherapistAttendance, Long>`

* `findAllByTherapistIdAndEventAtBetween(therapistId, from, to)`
* `findAllByEventAtBetween(from, to)`

### Service — `AttendanceService`

* `Set<UUID> clockedInTherapistIds(from, to)` — keeps the latest event per therapist; includes those whose latest event is `CLOCK_IN`.
* `recordClockIn(therapistId, at, deviceId)`, `recordClockOut(...)`, `recordAbsence(therapistId, at, remarks)`, `recordDayOff(therapistId, at)` — all `@Transactional`.

### Controller — `AttendanceController` `@RequestMapping("/api/attendance")`

|Method|Path|`@PreAuthorize`|Params/Body|Returns|
|------|----|---------------|-----------|-------|
|GET|`/`|`SUPERADMIN,ADMIN,MANAGER,RECEPTIONIST`|`therapistId?`, `from?`, `to?` (defaults today)|`List<AttendanceRecord>`|
|POST|`/absence`|`SUPERADMIN,ADMIN,MANAGER,RECEPTIONIST`|`AbsenceRequest(therapistId, remarks)`|`TherapistAttendance`|
|POST|`/day-off`|`SUPERADMIN,ADMIN,MANAGER,RECEPTIONIST`|`DayOffRequest(therapistId)`|`TherapistAttendance`|

Inner records: `AttendanceRecord(id, therapistId, eventType, eventAt, deviceId, remarks)`, `AbsenceRequest`, `DayOffRequest`.

### Inter-module dependencies

`AttendanceService.clockedInTherapistIds` consumed by `therapist` (`LineupShiftSyncJob`).

### Redis interactions

None.

---

## room

### Purpose

Rooms and beds, plus live bed-occupancy state in Redis (white/available, gender-locked occupied) with gender-segregation rules per room/row.

### Domain entities

#### `Room` — `@Table("rooms")`

`id` (`uuid`), `organizationId`, `roomNumber` (`room_number`, not null), `floor` (nullable), `roomType` (`room_type`, not null — `COMMON`/`PRIVATE`/`VIP`), `rowSegmented` (`row_segmented`, default false), `active` (`is_active`, default true), `vipTier` (`vip_tier`, length 20).

#### `Bed` — `@Table("beds")`

`id` (`uuid`), `roomId` (`room_id`, not null), `bedLabel` (`bed_label`, not null), `rowIndex` (`row_index`, nullable), `active` (`is_active`, default true).

### Repository layer

#### `RoomRepository extends JpaRepository<Room, UUID>`

* `findAllByOrganizationIdAndActiveTrue`, `findAllByOrganizationId`

#### `BedRepository extends JpaRepository<Bed, UUID>`

* `findAllByRoomId`, `findAllByRoomIdIn(List)`, `findAllByRoomIdAndActiveTrue`, `findAllByRoomIdAndActiveTrueOrderByRowIndex`

### Service — `RoomOccupancyService` (Redis-backed)

Key: `room:{roomId}:bed:{bedId}` (Hash).

* `long countOccupiedBeds(orgId)` — scans rooms/beds, counts `status=OCCUPIED`.
* `occupy(orgId, roomId, bedId, clientNickname, lockerNumber, therapistNickname, genderLock, ownerItemId)` — writes hash (`status`, `clientNickname`, `lockerNumber`, `therapistNickname`, `genderLock`, `ownerItemId`, `startedAt`) and broadcasts.
* `release(orgId, roomId, bedId)` — deletes hash, broadcasts `AVAILABLE`.
* `Map<Object,Object> read(roomId, bedId)`.

`RoomGenderConflictException` — thrown when mixing genders in a room/row.

### Controller layer

#### `RoomController` `@RequestMapping("/api/rooms")`

* `GET /` (authenticated) — `List<RoomView>` with live occupancy. Inner records `RoomView(id, roomNumber, floor, roomType, rowSegmented, beds)`, `BedView(id, label, rowIndex, occupancy)`.

#### `RoomAdminController` `@RequestMapping("/api/admin")`

|Method|Path|`@PreAuthorize`|
|------|----|---------------|
|GET|`/rooms?includeInactive=`|(authenticated)|
|GET|`/rooms/with-beds?includeInactive=`|(authenticated) — `List<RoomWithBeds>`|
|POST|`/rooms`|`SUPERADMIN,ADMIN,MANAGER`|
|PATCH|`/rooms/{roomId}`|`SUPERADMIN,ADMIN,MANAGER`|
|PATCH|`/rooms/{roomId}/reactivate`|`SUPERADMIN,ADMIN,MANAGER`|
|DELETE|`/rooms/{roomId}`|`SUPERADMIN,ADMIN,MANAGER` (deactivate)|
|GET|`/rooms/{roomId}/beds?includeInactive=`|(authenticated)|
|POST|`/rooms/{roomId}/beds`|`SUPERADMIN,ADMIN,MANAGER`|
|PATCH|`/beds/{bedId}/reactivate`|`SUPERADMIN,ADMIN,MANAGER`|
|DELETE|`/beds/{bedId}`|`SUPERADMIN,ADMIN,MANAGER` (deactivate; re-indexes remaining rows)|

### Inter-module dependencies

`RoomOccupancyService` → `notification`. Consumed by `booking`/`cashier` (session start/end occupy/release), `dashboard` (occupied bed count).

### Redis interactions

`room:{roomId}:bed:{bedId}` (Hash) — live bed occupancy.

 > 
 > Scheduler `room/scheduler/RoomOccupancyReconcilerJob` heals Redis bed state against DB sessions (outside core API surface).

---

## booking

### Purpose

Online and walk-in bookings, session lifecycle (start/end/cancel/extend/adjust), the 15-minute prep buffer, locker assignment, public payment initiation/confirmation, and public self-service cancellation.

### Domain entities

#### `Booking` — `@Table("bookings")`

|Field|Column|Notes|
|-----|------|-----|
|`id`|`id`|`uuid`|
|`reference`|`reference`|`varchar(8)`|
|`organizationId`|`organization_id`|not null|
|`clientId`|`client_id`|nullable|
|`clientNickname`|`client_nickname`|not null|
|`clientEmail`|`client_email`||
|`lockerNumber`|`locker_number`||
|`serviceId`|`service_id`|`Long`, not null|
|`reservationType`|`reservation_type`|not null (`HARD`/`SOFT`/`WALK_IN`)|
|`scheduledAt`|`scheduled_at`|not null|
|`actualStartAt` / `actualEndAt`|`actual_start_at` / `actual_end_at`||
|`pax`|`pax`||
|`clientGender`|`client_gender`|`varchar(1)`|
|`nationality`|`nationality`||
|`remarks`|`remarks`|`text`|
|`preferredTherapist`|`preferred_therapist`|`varchar(120)`, optional free-text remark only — never allocates a therapist|
|`status`|`status`|not null, default `PENDING` (also `ACTIVE`/`COMPLETED`)|
|`paymentStatus`|`payment_status`|not null, default `UNPAID`|
|`gatewayPaymentId`|`gateway_payment_id`||
|`createdAt`|`created_at`||

#### `Session` — `@Table("sessions")`

`id` (`uuid`), `bookingId` (not null), `organizationId` (not null), `primaryTherapistId`, `secondaryTherapistId`, `roomId`, `bedId` (all `uuid`), `specificallyRequested` (`is_specifically_requested`, default false), `extension` (`is_extension`, default false), `extensionMinutes` (default 0), `status` (not null, default `ACTIVE` → `COMPLETED`/`CANCELLED`), `startedAt`, `expectedEndAt`, `endedAt`, `attendeeId` (`attendee_id`), `createdAt`.

### Repository layer

#### `BookingRepository extends JpaRepository<Booking, UUID>`

* `findAllByOrganizationIdAndScheduledAtBetween`
* `@Query` `findAllByOrganizationIdAndEffectiveDateBetween(orgId, from, to)` — `coalesce(actualStartAt, scheduledAt)` window.
* `existsByOrganizationIdAndClientNicknameIgnoreCaseAndStatusIn(orgId, nickname, statuses)`
* `findAllByOrganizationIdAndClientEmailIgnoreCase`
* `findAllByOrganizationIdAndStatusAndScheduledAtBefore`
* `findAllByOrganizationIdOrderByCreatedAtDesc(orgId, Pageable)`

#### `SessionRepository extends JpaRepository<Session, UUID>`

* `findFirstByBookingId`
* `@Lock(PESSIMISTIC_WRITE) @Query` `findByIdForUpdate(id)`
* `findAllByOrganizationIdAndStartedAtBetween`, `findAllByStatusAndExpectedEndAtBefore`
* `@Query(...UNION...)` `Set<UUID> findActiveTherapistIds(orgId, ids)` — active primary ∪ secondary therapists.
* `existsByOrganizationIdAndPrimaryTherapistIdAndStatus`, `existsByOrganizationIdAndSecondaryTherapistIdAndStatus`
* `findAllByOrganizationIdAndStatus`, `findAllByOrganizationIdAndStatusAndExpectedEndAtBefore`
* `countByOrganizationIdAndStatus`, `countByOrganizationIdAndStatusAndEndedAtBetween`

### Service layer

#### `BookingService` (constant `PREP_BUFFER_MINUTES = 15`)

|Method|`@PreAuthorize`|Summary|
|------|---------------|-------|
|`createBooking(orgId, CreateBookingRequest)`|`permitAll`|`@Transactional`. Validates nickname (required) and email (required unless WALK_IN), no past scheduling, single ongoing booking per nickname; resolves/creates `Client` by email; supports multi-package items; computes room type/charge (PRIVATE ₱500, VIP/PRIVATE bundled), tier pricing, and order total; creates `Order` + items + attendees.|
|`initiatePublicPayment(orgId, orderId, method, PaymentDetailsRequest)`|`permitAll`|starts a gateway intent for an online order|
|`confirmPublicPayment(orgId, orderId, intentId, method)`|`permitAll`|confirms gateway payment, marks order paid|
|`startSession(orgId, bookingId, StartSessionRequest)`|`SUPERADMIN,ADMIN,MANAGER,RECEPTIONIST`|delegates to first attendee's `startAttendeeSession`|
|`startAttendeeSession(orgId, attendeeId, StartSessionRequest)`|same|core start logic: requires PAID order + assigned locker; locker-clash check; gender lock check on common rooms; VIP room only for VIP packages; rejects skipped/on-call therapists; tandem requires secondary; sets `startedAt`, `expectedEndAt = now + duration`; occupies bed; rotates therapist (or `servedRequested`); generates treatment slip|
|`endSession(orgId, sessionId)`|same|`endSessionAt(now)`|
|`endSessionAt(orgId, sessionId, endTime)`|—|marks COMPLETED, completes booking when all attendees done, stamps slip end time, `OrderService.recordCommissionsForSession`, releases bed, sends completion email|
|`cancelSession(orgId, sessionId)`|`SUPERADMIN,ADMIN,MANAGER,RECEPTIONIST`|CANCELLED; releases bed; voids slip; booking→PENDING; order→OPEN|
|`extendSession(sessionId, additionalMinutes)`|same|`findByIdForUpdate`; only `additionalMinutes=60`, single extension, not for fixed-rate; bills ₱800/₱900 (weekday/weekend) into order; ₱60/30 min therapist commission|
|`adjustTimes(sessionId, startAt, endAt)`|same|manual time correction|
|`listBookingsForDay(orgId, dayStart, dayEnd)`|—|effective-date window|
|`recentReservations(orgId, limit)`|—|newest-first|
|`autoEndExpiredSessions(orgId)`|—|sweeper entry point|

Throws `IllegalArgumentException`, `IllegalStateException`, `AccessDeniedException`, `ResponseStatusException(CONFLICT)`.

#### `WalkInService`

* `createWalkIn(orgId, CreateWalkInRequest)` — `@PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER','RECEPTIONIST')")`, `@Transactional`. Enforces gender lock on common rooms (per row when `rowSegmented`); creates ACTIVE `Booking` + `Session` immediately; occupies all selected beds; rotates/serves therapist; builds a `TreatmentSlip` (TSN via `IdSequenceService.nextTsn`) with VIP/jacuzzi/wine fields; creates an ACTIVE `Order`. Returns `WalkInResponse`.

#### `BookingCancellationService`

* `requestCancellation(orgId, reference, email, ip)` — finds cancellable PENDING booking by reference+email, issues code (cooldown-guarded), emails it, audits `BOOKING_CANCEL_REQUESTED`.
* `verify(orgId, reference, email, code)` → `CancellationDetailsResponse`.
* `confirm(orgId, reference, email, code, ip)` — `@Transactional`; verifies, `OrderService.cancelInternal`, clears code, audits `BOOKING_CANCELLED`.

#### `BookingCancellationCodeService` (Redis)

6-digit code, TTL 15 min, resend cooldown 60 s, `MAX_ATTEMPTS=5`.

* `boolean onCooldown(bookingId)`, `String issue(bookingId, email)`, `boolean verify(bookingId, email, code)`, `void clear(bookingId)`.

#### `LockerAssignmentService`

Configurable range `sumicare.locker.min`/`max` (default 1–200). `Set<String> takenLockersForDay(orgId, scheduledAt)` (active PENDING/ACTIVE bookings + attendees), `String assign(gender, taken)` — random unused number prefixed `M`/`F`.

### Scheduler — `SessionAutoEndJob`

`@Scheduled(fixedDelay=60s)`; runs as a synthetic SUPERADMIN, calls `BookingService.autoEndExpiredSessions` for every org.

### Controller layer

#### `BookingController`

|Method|Path|Auth|Body|Returns|
|------|----|----|----|-------|
|POST|`/api/walk-in`|authenticated (service `@PreAuthorize`)|`CreateWalkInRequest`|`WalkInResponse`|
|POST|`/api/public/bookings/{slug}`|public|`CreateBookingRequest`|`BookingResponse`|
|POST|`/api/bookings`|authenticated|`CreateBookingRequest`|`BookingResponse`|
|POST|`/api/public/bookings/{slug}/payment/initiate`|public|`PublicPaymentInitiateRequest`|`PublicPaymentResponse`|
|POST|`/api/public/bookings/{slug}/payment/confirm`|public|`PublicPaymentConfirmRequest`|`PublicPaymentResponse`|
|GET|`/api/bookings?from&to` (LocalDate)|authenticated|—|`List<BookingResponse>`|
|POST|`/api/bookings/{bookingId}/sessions`|authenticated|`StartSessionRequest`|`SessionResponse`|
|POST|`/api/bookings/attendees/{attendeeId}/sessions`|authenticated|`StartSessionRequest`|`SessionResponse`|
|POST|`/api/sessions/{sessionId}/cancel`|authenticated|—|`SessionResponse`|
|POST|`/api/sessions/{sessionId}/end`|authenticated|—|`SessionResponse`|
|POST|`/api/sessions/{sessionId}/extend?minutes=`|authenticated|—|`SessionResponse`|
|POST|`/api/sessions/{sessionId}/adjust-times?startAt&endAt`|authenticated|—|`SessionResponse`|
|GET|`/api/sessions/by-id/{sessionId}`|authenticated|—|`SessionResponse`|
|GET|`/api/sessions/by-booking/{bookingId}`|authenticated|—|`SessionResponse`|
|PATCH|`/api/bookings/{bookingId}`|authenticated|`Map<String,Object>` (serviceId/lockerNumber/clientNickname)|`Map<String,String>`|

#### `BookingExportController` `@RequestMapping("/api/bookings")`

* `GET /export.csv?from&to` (OffsetDateTime) — authenticated; CSV of bookings joined to sessions.

#### `PublicCancellationController` (all public)

* `POST /api/public/bookings/{slug}/cancel/request` — `CancellationRequestRequest` → message map.
* `POST /api/public/bookings/{slug}/cancel/verify` — `CancellationVerifyRequest` → `CancellationDetailsResponse`.
* `POST /api/public/bookings/{slug}/cancel/confirm` — `CancellationVerifyRequest` → message map.

### DTOs

|DTO|Notable fields|
|---|--------------|
|`CreateBookingRequest`|clientId?, clientNickname, clientEmail?, lockerNumber?, serviceId, reservationType, scheduledAt, pax?, clientGender?, packageId?, packageTierId?, nationality?, roomType?, paymentMethod?, `PaymentDetailsRequest`, attendees, items, voucherCode?, remarks, preferredTherapist? (free-text remark)|
|`CreateBookingItemRequest`|packageId, packageTierId?, roomType?, attendees|
|`PublicAttendeeRequest`|packageTierId?, lockerNumber?, clientGender?|
|`CreateWalkInRequest`|clientNickname, serviceId, reservationType?, pax?, clientGender?, lockerNumber?, startTime, endTime?, primary/secondaryTherapistId?, roomId?, bedIds, specificallyRequested, jacuzzi/massageMinutes?, wineIncluded?, or/addOn OR numbers, othersAddOn?, remarks?, totalAmount?, waiverAccepted|
|`StartSessionRequest`|primaryTherapistId?, secondaryTherapistId?, roomId?, bedId?, specificallyRequested|
|`SessionResponse`|id, bookingId, therapist/room/bed ids, specificallyRequested, extension, extensionMinutes, startedAt, expectedEndAt, endedAt, status|
|`BookingResponse`|id, reference, client info, serviceId, reservationType, scheduledAt, projectedEndAt, status, orderId, orderStatus, treatmentSlipId, pax, sessionExtended, nationality, remarks, preferredTherapist|
|`WalkInResponse`|slipId, bookingId, sessionId, tsn|
|`PublicPaymentInitiateRequest`|orderId, paymentMethod, `PaymentDetailsRequest`|
|`PublicPaymentConfirmRequest`|orderId, intentId, paymentMethod?|
|`PublicPaymentResponse`|status, intentId, redirectUrl, orNumber, reference, client/package/service info, scheduledAt, reservationType|
|`Cancellation*`|`CancellationRequestRequest(reference, email)`, `CancellationVerifyRequest(reference, email, code)`, `CancellationDetailsResponse(reference, clientNickname, scheduledAt, reservationType, summary, roomType, total, paid, remarks)`|

### Inter-module dependencies

Heavy hub: depends on `cashier` (`OrderService`/repos), `transaction` (`TreatmentSlipService`, slips, commissions), `room` (`RoomOccupancyService`), `therapist` (`DeckingService`), `service_catalogue`, `client`, `notification`, `audit`, `auth` (`EmailService`), `common` (`IdSequenceService`, `BookingReference`). Consumed by `dashboard`, `cashier` (booking linkage).

### Redis interactions

* `cancel:code:{bookingId}` / `cancel:attempts:{bookingId}` / `cancel:cooldown:{bookingId}` — public cancellation codes.
* Indirectly via `DeckingService` and `RoomOccupancyService`.

---

## cashier

### Purpose

Point-of-sale orders, line items and per-attendee sessions, packages/tiers, discount templates, ledger accounts/reporting, room availability, payments (cash + gateway), refunds, commissions, and order auto-cancellation.

### Domain entities

#### `Order` — `@Table("orders")`

PK `id` (`uuid`). Key fields: `organizationId`, `bookingId?`, `transactorName`, `groupBooking` (`is_group_booking`), `roomType` (default `COMMON`), `roomTypeCharge`, `weekend` (`is_weekend`), `treatmentSlipId?`, `cashierUserId?`, `lastEditedByUserId?`, `orNumber`, `referenceNumber`, `notes`, monetary fields `subtotal`/`discount`/`tax`/`total`/`extensionAmount`/`amountPaid` (all not null, default 0), `extensionMinutes` (default 0), `status` (default `PENDING`; lifecycle `OPEN`/`PAID`/`ACTIVE`/`CANCELLED`/`REFUNDED`/`COMPLETED`), timestamps `createdAt`/`completedAt`/`finishedAt`/`cancelledAt`, `cancelledReason`, `voucherId?`, `completionEmailSentAt`, `paymentEmailSentAt`, `preferredTherapist?` (`preferred_therapist varchar(120)`, optional free-text remark only — never allocates a therapist).

#### `OrderItem` — `@Table("order_items")`

`id` (`uuid`), `orderId`, `organizationId`, `packageId` (`Long`, not null), `quantity` (default 1), `unitPrice`, `lineTotal`, `roomType` (default `COMMON`), `roomTypeCharge`, `position`, `createdAt`.

#### `OrderItemAttendee` — `@Table("order_item_attendees")`

`id` (`uuid`), `orderItemId`, `orderId`, `organizationId`, `serviceId?`, `packageTierId?`, `lockerNumber` (16), `clientGender` (1), `sessionId?`, `treatmentSlipId?`, `position`, `discount` (precision 12 scale 2, default 0), `providedTsn` (64), `createdAt`.

#### `Package` — `@Table("packages")`

`id` (`Long`), `organizationId`, `code` (40), `name` (120), `description`/`benefits` (text), `maxStayHours?`, `defaultPax` (default 1), booleans `couple` (`is_couple`), `includesMassage`, `bundlesPrivateRoom`, `requiresVipRoom`, `active` (`is_active`), `createdAt`.

#### `PackageTier` — `@Table("package_tiers")`

`id` (`Long`), `packageId`, `serviceId?`, `weekdayPrice`, `weekendPrice` (both not null).

#### `DiscountTemplate` — `@Table("discount_templates")`

`id` (`uuid`), `organizationId`, `name` (120), `amountType` (`PERCENT`/fixed, default `PERCENT`), `percent?`, `fixedAmount?`, `createdAt`.

#### `LedgerAccount` — `@Table("ledger_accounts")`

`id` (`uuid`), `organizationId`, `name` (120), `shortName` (40), `type` (20), `createdAt`.

### Repository layer

* `OrderRepository extends JpaRepository<Order, UUID>` — `findByBookingId`, `findAllByBookingIdIn`, `@Query findByOrganizationIdAndOrNumber`, `findAllByOrganizationIdOrderByCreatedAtDesc`, `findAllByOrganizationIdAndStatusInOrderByCreatedAtDesc`, `findAllByOrganizationIdAndCreatedAtBetweenOrderByCreatedAtDesc`.
* `OrderItemRepository` — `findAllByOrderIdOrderByPosition`, `@Modifying deleteAllByOrderId`.
* `OrderItemAttendeeRepository` — `findAllByOrderItemIdOrderByPosition`, `findAllByOrderIdOrderByPosition`, `findAllByOrderIdIn`, `findBySessionId`, `findByTreatmentSlipId`, `@Modifying deleteAllByOrderId`.
* `PackageRepository extends JpaRepository<Package, Long>` — `findAllByOrganizationIdAndActiveTrueOrderByName`, `findAllByOrganizationIdOrderByActiveDescNameAsc`, `findByOrganizationIdAndCode`.
* `PackageTierRepository extends JpaRepository<PackageTier, Long>` — `findAllByPackageId`, `@Modifying deleteAllByPackageId`.
* `DiscountTemplateRepository` — `findAllByOrganizationIdOrderByName`.
* `LedgerAccountRepository` — `findAllByOrganizationIdOrderByCreatedAtDesc`.

### Service layer

#### `OrderService` (selected public methods)

|Method|`@PreAuthorize`|Summary|
|------|---------------|-------|
|`recordCommissionsForSession(orgId, Session)`|—|idempotent (skips if already recorded); resolves service via attendee/booking; base = `service.commissionAmount`; tandem splits 50/50 between primary and secondary|
|`create(orgId, cashierUserId, CreateOrderRequest)`|`SUPERADMIN,ADMIN,MANAGER,RECEPTIONIST`|`@Transactional`; builds order, items, attendees, optional initial payment|
|`update(orgId, orderId, actorUserId, CreateOrderRequest)`|same|rebuilds items/attendees|
|`recordPayment(orgId, orderId, actorUserId, RecordPaymentRequest)`|same|cash/manual payment, updates `amountPaid`/status|
|`initiatePayMongo(...)` / `confirmPayMongo(...)`|same|gateway intent + confirm|
|`settleGatewayPayment(order, actorUserId, intentId, method, amount)`|—|webhook settlement|
|`refundOrder(orgId, orderId, actorUserId, RefundRequest)`|`SUPERADMIN,ADMIN,MANAGER`|gateway/manual refund|
|`markPaid` / `openOrder` / `cancelPayment` / `cancel(reason)`|`SUPERADMIN,ADMIN,MANAGER,RECEPTIONIST`|status transitions|
|`cancelInternal(orgId, orderId, reason)`|—|used by public cancellation|
|`autoCancelElapsedOrders(orgId)`|—|nightly sweep|
|`reconcileRefund(order, amount, refundId)`|—|webhook refund|
|`list`, `get`, `getByBookingId`, `requireOrder`, `ensureForBooking`, `createOpenOrderFromBooking`, `materialiseAttendeeSessions`, `syncBookingPaymentStatus`, `ensureSlipForAttendee`, `nextOrNumber`|—|helpers shared with `booking`|

#### `PackageService`

* `listAll(orgId)`, `create(orgId, PackageRequest)`, `update(orgId, packageId, PackageRequest)`, `deactivate(orgId, packageId)`, `reactivate(orgId, packageId)` — all `@PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER')")`. `deriveInclusions(Package)` builds the inclusion list.

### Scheduler — `OrderAutoCancelJob`

`@Scheduled(cron="0 5 0 * * *", zone="Asia/Manila")`; runs as synthetic SUPERADMIN, `OrderService.autoCancelElapsedOrders` per org.

### Controller layer

#### `OrderController` `@RequestMapping("/api/cashier/orders")` (all `RECEPTIONIST`+ unless noted)

|Method|Path|`@PreAuthorize`|Body|
|------|----|---------------|----|
|POST|`/`|RECEPTIONIST+|`CreateOrderRequest`|
|PUT|`/{id}`|RECEPTIONIST+|`CreateOrderRequest`|
|GET|`/?status=`|RECEPTIONIST+|—|
|GET|`/{id}`|RECEPTIONIST+|—|
|POST|`/{id}/payments`|RECEPTIONIST+|`RecordPaymentRequest`|
|POST|`/{id}/paymongo/initiate`|RECEPTIONIST+|`RecordPaymentRequest` → `PayMongoInitiateResponse`|
|POST|`/{id}/paymongo/confirm`|RECEPTIONIST+|`PayMongoConfirmRequest`|
|POST|`/{id}/refund`|`SUPERADMIN,ADMIN,MANAGER`|`RefundRequest`|
|POST|`/{id}/mark-paid`|RECEPTIONIST+|—|
|POST|`/{id}/cancel`|RECEPTIONIST+|`{reason}`|
|POST|`/{id}/open`|RECEPTIONIST+|—|
|POST|`/{id}/cancel-payment`|RECEPTIONIST+|—|
|GET|`/by-booking/{bookingId}`|RECEPTIONIST+|—|
|GET|`/{id}/receipt.pdf`|RECEPTIONIST+|— (PDF via `ReceiptPdfService`)|

("RECEPTIONIST+" = `hasAnyRole('SUPERADMIN','ADMIN','MANAGER','RECEPTIONIST')`.)

#### `PackageController` `@RequestMapping("/api/cashier/packages")`

* `GET /` (RECEPTIONIST+) — active packages with tiers; `GET /all` (`SUPERADMIN,ADMIN,MANAGER`); `POST /`, `PATCH /{packageId}`, `DELETE /{packageId}`, `POST /{packageId}/reactivate` (`SUPERADMIN,ADMIN,MANAGER`).

#### `PublicPackageController`

* `GET /api/public/packages/{slug}` — public list of active packages.

#### `DiscountTemplateController` `@RequestMapping("/api/cashier/discount-templates")` (RECEPTIONIST+)

* `GET /`, `POST /` (`DiscountTemplateRequest`), `DELETE /{id}`.

#### `LedgerController` `@RequestMapping("/api/cashier/ledger")`

* `GET /accounts` (authenticated), `POST /accounts` (`SUPERADMIN,ADMIN,MANAGER`).
* `GET /`, `GET /balance`, `GET /daily-revenue`, `GET /export.csv` — all `SUPERADMIN,ADMIN,MANAGER`. Inner records: `LedgerAccountResponse`, `CreateLedgerAccountRequest`, `LedgerEntryResponse`, `BalanceResponse(inflow, outflow, balance, count)`, `DailyRevenuePoint(date, net, inflow, outflow, count)`.

#### `RoomAvailabilityController` `@RequestMapping("/api/cashier/room-availability")`

* `GET /` (RECEPTIONIST+) — `Map<String,Integer>` availability by room type.

### DTOs

|DTO|Fields|
|---|------|
|`CreateOrderRequest`|clientId?, clientNickname?, lockerNumber?, clientGender?, scheduledAt?, pax?, serviceIds?, primary/secondaryTherapistId?, roomId?, bedId?, referenceNumber?, notes?, orNumber?, tsNumber?, subtotal/discount/tax/total, `InitialPayment(paymentMethod, amount, referenceNumber?, paymentDetails?)`, transactorName?, groupBooking?, weekend?, roomType?, roomTypeCharge?, voucherId?, items, preferredTherapist? (free-text remark)|
|`CreateOrderItemRequest`|packageId, quantity, unitPrice, lineTotal, roomType, position, attendees|
|`CreateOrderItemAttendeeRequest`|serviceId?, packageTierId?, lockerNumber?, clientGender?, position?, discount?, providedTsn?|
|`OrderResponse`|full read model incl. balance, cashier/lastEditedBy display names, `couplePackage`, `sessionCompleted`, items, `preferredTherapist` (shown in order detail)|
|`OrderItemResponse` / `OrderItemAttendeeResponse`|nested line/attendee read models|
|`RecordPaymentRequest`|paymentMethod, amount, referenceNumber?, `PaymentDetailsRequest`, returnPath?|
|`PaymentDetailsRequest`|card (number/exp/cvc/holder/email) + GCash (name/phone/email) fields|
|`PayMongoConfirmRequest`|intentId, amount?, paymentMethod?|
|`PayMongoInitiateResponse`|status, intentId, redirectUrl|
|`RefundRequest`|amount?, reason?, notes?|
|`PackageRequest` / `PackageResponse`|package config incl. tiers + derived `inclusions`|
|`PackageTierRequest` / `PackageTierResponse`|serviceId, weekday/weekend price (+ resolved service metadata on response)|
|`DiscountTemplateRequest` / `DiscountTemplateResponse`|name, amountType, percent, fixedAmount|

### Inter-module dependencies

Depends on `booking`, `transaction` (commissions, slips), `service_catalogue`, `pos` (`PosTransaction`, ledger, `PayMongoService`/gateway), `voucher`, `room`, `notification`, `print` (`ReceiptPdfService`), `user` (cashier names), `common`. Used by `booking` (order creation/linkage) and `report`.

### Redis interactions

None directly.

---

## service_catalogue

### Purpose

The massage services catalogue (durations, prices, commissions, flags) — public listing, internal listing/CSV export, and admin CRUD.

### Domain entity — `Service` `@Table("services_catalogue")`

`id` (`Long`), `organizationId`, `code` (not null), `name` (not null), `durationMinutes` (`duration_minutes`, not null), `commissionAmount` (not null), `price` (not null), `category`, `requiresTwoTherapists` (`requires_two_therapists`, not null — tandem), `fixedRate` (`is_fixed_rate`, not null), `description` (text), `imageUrl` (512), `active` (`is_active`, default true).

### Repository — `ServiceRepository extends JpaRepository<Service, Long>`

* `List<Service> findAllByOrganizationIdAndActiveTrue(orgId)`

### Controller layer

#### `ServiceCatalogueController`

* `GET /api/services` (authenticated) — active services.
* `GET /api/services/export` (text/csv, authenticated) — CSV export.
* `GET /api/public/services/{slug}` (public) — active services by org slug.

#### `ServiceAdminController` `@RequestMapping("/api/admin/services")`

* `POST /` (`SUPERADMIN,ADMIN,MANAGER`) — create.
* `PATCH /{serviceId}` (`SUPERADMIN,ADMIN,MANAGER`) — patch fields.
* `DELETE /{serviceId}` (`SUPERADMIN,ADMIN,MANAGER`) — deactivate.

Request/response body is the `Service` entity directly.

### Inter-module dependencies

`ServiceRepository` consumed by `booking`, `cashier`, `transaction`, `recommendation`.

### Redis interactions

None.

---

## transaction

### Purpose

Treatment-slip creation/digitization and therapist commission records.

### Domain entities

#### `TreatmentSlip` — `@Table("treatment_slips")`

PK `id` (`uuid`). Fields: `organizationId`, `tsn` (not null), `bookingId` (not null), `sessionId?`, `clientNickname` (not null), `lockerNumber`, `requestedTherapistNickname`, `primaryTherapistNickname`, `secondaryTherapistNickname`, `serviceName` (not null), `roomNumber`, `startTime`/`endTime`, `vip` (`is_vip`), `status` (default `DRAFT`; also `VOIDED`), `pax`, `treatmentMinutes`, `jacuzziMinutes`, `massageMinutes`, `wineIncluded`, `orNumber` (64), `addOnOrNumber` (64), `othersAddOn` (text), `extensionMinutes`, `clientGender` (1), `remarks` (text), `nationality` (64), `packageName` (120), `totalAmount` (12,2), `waiverAccepted` (not null), `waiverAcceptedAt`, `signedAt`, `createdAt`, `attendeeId?`.

#### `Commission` — `@Table("commissions")`

`id` (`Long`), `organizationId`, `sessionId` (not null), `therapistId` (not null), `amount` (not null), `extension` (`is_extension`), `backup` (`is_backup`), `serviceId?`, `serviceType` (50), `specificallyRequested`, `createdAt`.

### Repository layer

#### `TreatmentSlipRepository extends JpaRepository<TreatmentSlip, UUID>`

* `findAllByOrganizationIdAndCreatedAtBetween`
* `@Query` `findAllByOrganizationIdAndScheduleBetween(orgId, from, to)` — `coalesce(startTime, createdAt)` ordered desc.
* `findBySessionId`, `findAllByBookingId`.

#### `CommissionRepository extends JpaRepository<Commission, Long>`

* `findAllByOrganizationIdAndCreatedAtBetween`, `existsBySessionIdAndTherapistId`, `existsBySessionIdAndTherapistIdAndExtensionFalse`.

### Service — `TreatmentSlipService`

* `TreatmentSlip generateForSession(orgId, sessionId)` — `@PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER','RECEPTIONIST')")`; builds/updates the slip from session + booking + service + therapist data.
* `byte[] exportToCsv(orgId, from, to)` — same auth.
* `TreatmentSlip update(orgId, slipId, UpdateTreatmentSlipRequest)` — same auth.

### Controller — `TreatmentSlipController` `@RequestMapping("/api/treatment-slips")`

|Method|Path|Params/Body|Returns|
|------|----|-----------|-------|
|GET|`/{slipId}/slip.pdf`|—|PDF (`TreatmentSlipPdfService`)|
|POST|`/from-session/{sessionId}`|—|`TreatmentSlip`|
|GET|`/export.csv?from&to`|OffsetDateTime|CSV|
|GET|`/{slipId}`|—|`TreatmentSlip`|
|GET|`/?from&to`|OffsetDateTime|`List<TreatmentSlip>`|
|PATCH|`/{slipId}`|`UpdateTreatmentSlipRequest`|`TreatmentSlip`|

### DTOs

* `UpdateTreatmentSlipRequest(tsn?, lockerNumber?, roomNumber?, othersAddOn?, remarks?, orNumber?, addOnOrNumber?, totalAmount?, jacuzziMinutes 0–120?, massageMinutes 0–120?, wineIncluded?, waiverAccepted?, startTime?, endTime?)`.

### Inter-module dependencies

Depends on `booking`, `service_catalogue`, `therapist`, `room`, `print`. `TreatmentSlipService`/commissions consumed by `booking` (start/end), `cashier` (commissions, slip linkage), `report`.

### Redis interactions

None.

---

## pos

### Purpose

Immutable POS transaction records, the append-only transaction ledger, and the PayMongo payment-gateway integration plus its webhook reconciliation.

### Domain entities

#### `PosTransaction` — `@Table("transactions")`

`id` (`uuid`), `organizationId`, `orderId?`, `sessionId?`, `receiptNumber` (not null), `referenceNumber` (100), `subtotal` (not null), `discount` (default 0), `tax` (default 0), `total` (not null), `paymentMethod` (not null), `processedBy?` (`uuid`), `processedAt` (not null), `status` (default `COMPLETED`).

#### `TransactionLedgerEntry` — `@Table("transaction_ledger")` (append-only)

`id` (`Long`), `organizationId`, `transactionId` (not null), `entryType` (not null — e.g. `PAYMENT_RECEIVED`), `amount` (not null), `recordedAt` (not null, not updatable), `metadata`, `paymentMethod`, `status` (default `COMPLETED`), `gatewayReference` (128).

### Repository layer

* `PosTransactionRepository extends JpaRepository<PosTransaction, UUID>` — `findAllByOrganizationIdAndProcessedAtBetween`, `findAllByOrderId`.
* `TransactionLedgerRepository extends JpaRepository<TransactionLedgerEntry, Long>` — `findAllByOrganizationIdAndRecordedAtBetweenOrderByRecordedAtDesc`, `findAllByOrganizationIdAndEntryTypeAndRecordedAtBetween...`, `findAllByOrganizationIdAndPaymentMethodAndRecordedAtBetween...`, `existsByGatewayReference`.

### Gateway / service

#### `PaymentGateway` (interface) / `PayMongoGateway`

Methods: `createIntent`, `createPaymentMethod`, `attachIntent`, `retrieveIntent`, `createCheckoutSession`, `retrieveCheckoutSession`, `retrievePaymentId`, `refund`, `boolean verifyWebhook(payload, signature)`, `capture`. Records: `IntentResult`, `CheckoutResult`, `CardDetails`, `Billing`, `RefundResult`.

#### `PayMongoService`

* `static boolean supports(String paymentMethod)`.
* `ChargeResult charge(...)` (two overloads), `ChargeResult initiate(...)`, `String confirm(String intentId)`, `RefundResult refund(intentId, amount, reason, notes, orderId)`. Records `ChargeResult(intentId, status, nextActionUrl)`, `RefundResult(refundId, status, amount)`.

### Controller — `WebhookController` `@RequestMapping("/api/webhooks")` (public per `SecurityConfig`)

* `POST /paymongo` — verifies `Paymongo-Signature` (401 on failure); handles `payment.paid`/`source.chargeable` → `reconcile` (marks booking/order paid, `OrderService.settleGatewayPayment`, or writes a standalone ledger entry idempotently by `gatewayReference`) and `refund.updated`/`payment.refunded` → `reconcileRefund` (`OrderService.reconcileRefund`). Returns `200 ok` or error.

### Inter-module dependencies

Depends on `booking`, `cashier` (`OrderService`, repos), `common` (`AppProperties`). `PayMongoService`/gateway and ledger consumed by `cashier`.

### Redis interactions

None.

---

## report

### Purpose

Cutoff/day/monthly operational and commission reporting, decking activity reports, and persisted day/monthly report snapshots, all with CSV export.

### Domain entities

#### `DayReport` — `@Table("day_reports")`

`id` (`uuid`), `organizationId`, `reportDate` (`report_date`, `LocalDate`, not null), `payload` (text, not null — serialized snapshot), `generatedAt` (not null).

#### `MonthlyReport` — `@Table("monthly_reports")`

`id` (`uuid`), `organizationId`, `reportYear` (int), `reportMonth` (int), `payload` (text, not null), `generatedAt`.

### Repository layer

* `DayReportRepository` — `findByOrganizationIdAndReportDate`, `findAllByOrganizationIdAndReportDateBetweenOrderByReportDateDesc`.
* `MonthlyReportRepository` — `findByOrganizationIdAndReportYearAndReportMonth`, `findAllByOrganizationIdOrderByReportYearDescReportMonthDesc`.

### Service layer

#### `ReportService`

* `ReportSummary buildCutoffReport(orgId, from, to)` and `byte[] exportCutoffToCsv(orgId, from, to)` — both `@PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER','RECEPTIONIST')")`. Record `ReportSummary`.

#### `CommissionReportService`

* `ShiftReport shift(orgId, shiftId, date)`, `DailyReport daily(orgId, date)`, `MatrixReport cutoff(orgId, year, month, half)`, `MatrixReport monthly(orgId, year, month)` — all `@PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER')")`, each with a `*Csv` companion. Records: `TherapistRow`, `ShiftReport`, `DailyReport`, `MatrixRow`, `MatrixReport`.

#### `DeckingReportService`

* `DeckingDailyReport daily(orgId, date)` (`SUPERADMIN,ADMIN,MANAGER`) + `dailyCsv`. Records: `DeckingGlyph`, `TherapistDeckingRow`, `ShiftGroup`, `DeckingDailyReport`.

#### `OperationsReportService`

* `CutoffServicesReport cutoffServices(orgId, from, to, shiftId?)`, `DailyReportResponse daily(orgId, date)`, `MonthlyReportResponse monthly(orgId, year, month)` — `@PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER','RECEPTIONIST')")`, each with CSV. Records: `ServiceLine`, `CutoffServicesReport`, `DailyRow`, `DailyReportResponse`, `MonthlyReportResponse`.

#### `ReportAggregationService`

* `generateDailyReports()` / `generateMonthlyReports()` (scheduled aggregation), `DayReport generateDayReport(orgId, date)`, `MonthlyReport generateMonthlyReport(orgId, YearMonth)`.

### Controller layer

#### `ReportController` `@RequestMapping("/api/reports")`

|Method|Path|`@PreAuthorize`|
|------|----|---------------|
|GET|`/cutoff?from&to`|(service-level)|
|GET|`/cutoff/export.csv?from&to`|(service-level)|
|GET|`/day?from&to` (LocalDate)|(service-level)|
|POST|`/day/regenerate?date`|`SUPERADMIN,ADMIN,MANAGER`|
|GET|`/monthly`|(service-level)|
|POST|`/monthly/regenerate?year&month`|`SUPERADMIN,ADMIN,MANAGER`|
|GET|`/cutoff/services?from&to&shiftId?` + `/export.csv`|(service-level)|
|GET|`/daily?date` + `/daily/export.csv`|(service-level)|
|GET|`/monthly-detailed?year&month` + `/export.csv`|(service-level)|

#### `CommissionReportController` `@RequestMapping("/api/reports/commissions")` — all `SUPERADMIN,ADMIN,MANAGER`

`GET /shift?shiftId&date` (+csv), `GET /daily?date` (+csv), `GET /cutoff?year&month&half` (+csv), `GET /monthly?year&month` (+csv).

#### `DeckingReportController` `@RequestMapping("/api/reports/decking")` — all `SUPERADMIN,ADMIN,MANAGER`

`GET /daily?date` and `GET /daily/export.csv?date`.

### Inter-module dependencies

Depends on `booking` (`SessionRepository`), `transaction` (`CommissionRepository`), `cashier`, `service_catalogue`, `shift`, `therapist`. Self-contained outputs.

### Redis interactions

None.

---

## notification

### Purpose

STOMP/WebSocket broker configuration, org-scoped topic broadcasting, and a Redis-backed WebSocket session registry.

### Configuration — `WebSocketConfig` (`@EnableWebSocketMessageBroker`)

* Simple broker on `/topic` and `/user`; application prefix `/app`; user prefix `/user`.
* STOMP endpoint `/ws` registered twice — with SockJS fallback and plain — both `setAllowedOriginPatterns("*")`.

### Service — `NotificationService` (wraps `SimpMessagingTemplate`)

|Method|Topic|
|------|-----|
|`broadcastDecking(orgId, payload)`|`/topic/decking-updates/{orgId}`|
|`broadcastRoomUpdate(orgId, roomId, bedId, payload)`|`/topic/room-updates/{orgId}` (body `{roomId, bedId, state}`)|
|`broadcastBookingEvent(orgId, event, bookingId, summary)`|`/topic/bookings/{orgId}`|
|`broadcastOrderEvent(orgId, event, orderId, summary)`|`/topic/orders/{orgId}`|
|`broadcastMessageEvent(orgId, event, messageId, summary)`|`/topic/messages/{orgId}`|
|`broadcastFeedbackEvent(orgId, event, feedbackId, summary)`|`/topic/feedback/{orgId}`|

Event payloads include `event`, an id key, `summary`, `at` (ISO timestamp), and `organizationId`.

### Registry — `WebSocketSessionRegistry`

* `@EventListener onConnected(SessionConnectedEvent)` — adds `simpSessionId` to `ws:sessions:decking` and `ws:sessions:roommap`.
* `@EventListener onDisconnect(SessionDisconnectEvent)` — removes it.

### Inter-module dependencies

Consumed by `therapist` (decking), `room`, `booking`, `cashier`, `feedback`, `contact`.

### Redis interactions

`ws:sessions:decking`, `ws:sessions:roommap` (Sets).

---

## recommendation

### Purpose

A configurable weighted-scoring quiz engine that ranks services for a client based on quiz answers.

### Domain entity — `RecommendationWeight` `@Table("recommendation_weights")`

`id` (`Long`), `organizationId`, `questionCode` (`question_code`, not null), `optionCode` (`option_code`, not null), `serviceId` (`service_id`, not null), `weight` (int, not null).

### Repository — `RecommendationWeightRepository extends JpaRepository<RecommendationWeight, Long>`

* `List<RecommendationWeight> findAllByOrganizationId(orgId)`

### Service — `RecommendationEngine`

* `List<Service> score(UUID organizationId, List<QuizAnswer> answers)` — indexes weights by `(questionCode, optionCode)`, sums per-service weight deltas across all answers, and returns active services that scored, sorted by total descending. Top result = primary recommendation; next two = alternatives.

### Controller — `RecommendationController`

* `POST /api/public/recommendation/{slug}` (public) — body `QuizSubmissionRequest`; resolves org by slug, calls `engine.score`, derives `primary` and up to two `alternatives`, and returns a `RecommendationResponse`. The response always carries a disclaimer constant: *"SumiCare's recommendations are for relaxation purposes only and do not constitute medical advice."*

### DTOs

* `QuizAnswer(String questionCode, String optionCode)`.
* `QuizSubmissionRequest(UUID clientId, List<QuizAnswer> answers)`.
* `RecommendationResponse(Service primary, List<Service> alternatives, String rationale, boolean aiUsed, String disclaimer)`.

### Inter-module dependencies

Depends on `service_catalogue` (`ServiceRepository`, `Service`) and `organization`.

### Redis interactions

None.

---

## voucher

### Purpose

Discount vouchers (amount or percent), per-client redemption tracking, and public/internal validity checks.

### Domain entities

#### `Voucher` — `@Table("vouchers")`

`id` (`uuid`), `organizationId`, `code` (not null), `name`, `discountAmount?`, `discountPercent?` (Integer), `validFrom?`/`validUntil?` (`LocalDate`), `usageLimit?`, `active` (`is_active`, default true), `targetPackageId?` (`Long`).

#### `VoucherRedemption` — `@Table("voucher_redemptions")`

`id` (`uuid`), `organizationId`, `voucherId` (not null), `clientId` (not null), `orderId?`, `redeemedAt` (not null).

### Repository layer

* `VoucherRepository extends JpaRepository<Voucher, UUID>` — `findByOrganizationIdAndCode`, `findAllByOrganizationId`.
* `VoucherRedemptionRepository extends JpaRepository<VoucherRedemption, UUID>` — `existsByVoucherIdAndClientId`, `countByVoucherId`.

### Service — `VoucherService`

* `list(orgId)`, `create(orgId, Voucher)`, `update(orgId, id, Voucher)` — `@PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER')")`.
* `Optional<Voucher> findValid(orgId, code)` — active + within date window.
* `boolean isRedeemableBy(Voucher, clientId)`, `void assertRedeemableBy(orgId, voucherId, clientId)` — enforces usage limit / per-client uniqueness.
* `BigDecimal computeDiscount(Voucher, subtotal)`, `BigDecimal discountForVoucher(orgId, voucherId, subtotal)`.
* `void markRedeemed(voucherId, clientId, orderId)`.

### Controller layer

#### `VoucherController` `@RequestMapping("/api/vouchers")` (auth via service)

* `GET /`, `POST /` (`Voucher`), `PUT /{id}` (`Voucher`).
* `GET /check?code&subtotal&clientId?` → `VoucherCheckResponse(id, code, name, discount)` (404 if invalid).

#### `PublicVoucherController` `@RequestMapping("/api/public/vouchers")` (public)

* `GET /{slug}/check?code` → `PublicVoucherResponse(id, code, name, discountAmount, discountPercent, targetPackageId)`.

### Inter-module dependencies

`VoucherService` consumed by `cashier`/`booking` (order discounting). Uses `client` indirectly (client-scoped redemption).

### Redis interactions

None.

---

## feedback

### Purpose

Public star-rating + comment feedback, internal read/unread management, and CSV export.

### Domain entity — `Feedback` `@Table("feedback")`

`id` (`uuid`), `organizationId`, `sessionId?`, `clientId?`, `ratingStars` (`rating_stars`, int, not null), `comment` (text), `nickname` (120), `orNumber` (50), `submittedAt` (not null, not updatable), `readAt?`, `readByUserId?`.

### Repository — `FeedbackRepository extends JpaRepository<Feedback, UUID>`

* `Page<Feedback> findAllByOrganizationIdOrderBySubmittedAtDesc(orgId, Pageable)`
* `findTop20ByOrganizationIdOrderBySubmittedAtDesc`
* `findAllByOrganizationIdAndSubmittedAtBetweenOrderBySubmittedAtAsc`
* `long countByOrganizationIdAndReadAtIsNull`
* `@Modifying @Query` `int markAllRead(orgId, readAt, userId)`

### Service layer

None — logic in controller.

### Controller — `FeedbackController`

|Method|Path|`@PreAuthorize`|Body/Params|
|------|----|---------------|-----------|
|POST|`/api/public/feedback/{slug}`|public|`PublicFeedbackRequest(ratingStars 1–5, comment, nickname, orNumber)`; broadcasts `FEEDBACK_RECEIVED`|
|GET|`/api/public/feedback/{slug}`|public|top-20 recent|
|GET|`/api/feedback?page&size`|(authenticated)|`Page<Feedback>`|
|GET|`/api/feedback/unread-count`|RECEPTIONIST+|`{count}`|
|POST|`/api/feedback/mark-all-read`|RECEPTIONIST+|`{marked}`|
|GET|`/api/feedback/export.csv?from&to`|`SUPERADMIN,ADMIN,MANAGER`|CSV|

### DTOs

* `PublicFeedbackRequest(int ratingStars [1–5], String comment, String nickname, String orNumber)`.

### Inter-module dependencies

Uses `organization`, `notification`.

### Redis interactions

None.

---

## audit

### Purpose

Immutable per-action audit logging for non-repudiation, auto-captured on mutating requests and explicitly recorded by select flows; queryable by admins.

### Domain entity — `AuditLog` `@Table("audit_logs")`

`id` (`Long`), `organizationId?`, `actorUserId?`, `actorRole`, `actionType` (`action_type`, not null), `targetEntity`, `targetId`, `metadata`, `ipAddress`, `occurredAt` (not null, not updatable).

### Repository — `AuditLogRepository extends JpaRepository<AuditLog, Long>`

* `Page<AuditLog> findAllByOrganizationIdOrderByOccurredAtDesc(orgId, Pageable)`
* `...AndActorUserId...`, `...AndOccurredAtBetween...`, `...AndActorUserIdAndOccurredAtBetween...`
* `List<AuditLog> findAllByOrganizationIdAndTargetEntityAndTargetIdOrderByOccurredAtDesc(orgId, entity, id)`

### Service — `AuditService`

* `@Async void record(orgId, actorUserId, actorRole, actionType, targetEntity, targetId, metadata, ipAddress)` — persists an `AuditLog`.

### Interceptor — `AuditInterceptor implements HandlerInterceptor`

`afterCompletion` records mutating requests (`POST/PUT/PATCH/DELETE`) with status `<400` and an authenticated principal. `resolveTarget` maps the URI to a structured `(entity, targetId, action)` for cashier/orders, bookings, walk-in, sessions, treatment-slips, decking, therapists, shifts, vouchers, clients, users, attendance, organization, content, feedback, contact-messages, reports, and admin (rooms/beds/services). Produces action codes such as `ORDER.REFUNDED`, `SESSION.END`, `DECKING.SKIP`, `USER.DEACTIVATE`, `ROOM.CREATE`.

### Controller — `AuditLogController` `@RequestMapping("/api/audit-logs")`

|Method|Path|`@PreAuthorize`|Params|
|------|----|---------------|------|
|GET|`/`|`SUPERADMIN,ADMIN`|`page`, `size`, `actorUserId?`, `from?`, `to?` (LocalDate)|
|GET|`/by-target`|`SUPERADMIN,ADMIN,MANAGER,RECEPTIONIST`|`entity`, `id`|

### Inter-module dependencies

`AuditService` invoked by `auth` (login), `booking` (cancellation), and the global `AuditInterceptor`. Registered via `audit/config/AuditWebConfig`.

### Redis interactions

None.

---

## content

### Purpose

Editable public-website content blocks (CMS) and image upload for branding/content.

### Domain entity — `WebsiteContentBlock` `@Table("website_content_blocks")`

`id` (`uuid`), `organizationId`, `sectionKey` (`section_key`, not null), `title`, `body` (text), `imageUrl`, `displayOrder` (`display_order`, default 0), `published` (`is_published`, default true), `updatedAt`.

### Repository — `WebsiteContentBlockRepository extends JpaRepository<WebsiteContentBlock, UUID>`

* `findAllByOrganizationIdAndPublishedTrueOrderByDisplayOrderAsc`, `findAllByOrganizationIdOrderByDisplayOrderAsc`

### Controller — `ContentController`

|Method|Path|`@PreAuthorize`|Body|
|------|----|---------------|----|
|GET|`/api/public/content/{slug}`|public|published blocks|
|GET|`/api/content`|`SUPERADMIN,ADMIN,MANAGER`|all blocks|
|POST|`/api/content`|`SUPERADMIN,ADMIN,MANAGER`|`WebsiteContentBlock`|
|PUT|`/api/content/blocks/{id}`|`SUPERADMIN,ADMIN,MANAGER`|`WebsiteContentBlock` (patch)|
|POST|`/api/content/upload`|`isAuthenticated()` (multipart)|`MultipartFile file` → `{url}`|

Upload accepts JPEG/PNG/GIF/WebP only; saves under `uploads/` with a UUID filename and resolves a public URL via `BaseUrlResolver`.

### Inter-module dependencies

Uses `organization`, `common` (`BaseUrlResolver`).

### Redis interactions

None.

---

## contact

### Purpose

Public "contact us" message intake (with per-IP rate limiting) and internal read/unread management plus CSV export.

### Domain entity — `ContactMessage` `@Table("contact_messages")`

`id` (`uuid`), `organizationId`, `name` (255, not null), `email` (255, not null), `message` (text, not null), `ipAddress` (45), `createdAt` (not null, not updatable), `readAt?`, `readByUserId?`.

### Repository — `ContactMessageRepository extends JpaRepository<ContactMessage, UUID>`

* `findAllByOrganizationIdOrderByCreatedAtDesc`, `findAllByOrganizationIdAndCreatedAtBetweenOrderByCreatedAtAsc`, `findAllByOrganizationIdAndReadAtIsNullOrderByCreatedAtDesc`, `countByOrganizationIdAndReadAtIsNull`, `countByIpAddressAndCreatedAtAfter`.

### Controller — `ContactMessageController`

|Method|Path|`@PreAuthorize`|Body/Params|
|------|----|---------------|-----------|
|POST|`/api/public/contact/{slug}`|public|`ContactMessageRequest`; 429 if ≥5 from IP in last hour; broadcasts `MESSAGE_RECEIVED`|
|GET|`/api/contact-messages?unread=`|RECEPTIONIST+|list|
|GET|`/api/contact-messages/unread-count`|RECEPTIONIST+|`{count}`|
|POST|`/api/contact-messages/{id}/mark-read`|RECEPTIONIST+|mark read (403 cross-org)|
|GET|`/api/contact-messages/export.csv?from&to&unread=`|`SUPERADMIN,ADMIN,MANAGER`|CSV|

### DTOs

* `ContactMessageRequest(name [≤255], @Email email [≤255], message [5–5000])`.

### Inter-module dependencies

Uses `organization`, `notification`. `ContactMessageRepository` also used by `auth` (`contact-admin-reset`).

### Redis interactions

None (rate limiting is DB-based via `countByIpAddressAndCreatedAtAfter`).

---

## dashboard

### Purpose

Aggregated operational snapshot and recent reservations for the receptionist/manager home view.

### Service — `DashboardService`

* `List<BookingResponse> recentReservations(orgId, limit)` — `@PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER','RECEPTIONIST')")`; limit capped 1–25; delegates to `BookingService.recentReservations`.
* `DashboardSummary summary(orgId)` — same auth; computes today's bookings, active sessions (`countByOrganizationIdAndStatus`), completed-today, therapists in lineup (`DeckingService.currentLineup().size()`), and occupied beds (`RoomOccupancyService.countOccupiedBeds`).

### Controller — `DashboardController` `@RequestMapping("/api/dashboard")`

* `GET /summary` (authenticated) → `DashboardSummary`.
* `GET /recent-reservations?limit=` (default 8) → `List<BookingResponse>`.

### DTO

* `DashboardSummary(int todayBookings, int activeSessions, int completedSessions, int therapistsInLineup, long bedsOccupied)`.

### Inter-module dependencies

Depends on `booking` (`BookingService`, `SessionRepository`), `therapist` (`DeckingService`), `room` (`RoomOccupancyService`).

### Redis interactions

Indirect — reads decking lineup and bed occupancy (both Redis-backed) via their services.
