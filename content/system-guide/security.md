---
title: "security"
date: 2026-07-13
categories: ["System Guide"]
---

# SumiCare Security Reference

This document describes the security implementation of the SumiCare backend (`apps/sumicare-api`) as it exists in source. Every statement below is grounded in code under `apps/sumicare-api/src/main/java/com/sumicare/`. Where behaviour is not determinable from the code, this is stated explicitly.

Key source files referenced:

|Concern|File|
|-------|----|
|Filter chain, CORS, password encoder|`auth/config/SecurityConfig.java`|
|Bearer token authentication|`auth/filter/JwtAuthenticationFilter.java`|
|Token issuance, parsing, revocation|`auth/service/JwtService.java`|
|Login, MFA orchestration, refresh, logout, lockout counting|`auth/service/AuthService.java`|
|Email MFA challenge|`auth/service/MfaService.java`|
|Login rate limiting|`auth/service/LoginRateLimiter.java`|
|Credential loading|`auth/service/UserDetailsServiceImpl.java`|
|Account state fields|`user/domain/User.java`|
|Account unlock|`user/controller/UserController.java`, `user/service/UserService.java`|
|Centralised error rendering|`common/GlobalExceptionHandler.java`|
|Configuration binding|`common/config/AppProperties.java`, `resources/application.yml`|

---

## 1. Authentication Flow

Authentication is stateless (`SessionCreationPolicy.STATELESS`). There are no server-side HTTP sessions; identity is re-established per request from the bearer token.

### 1.1 Credential submission

`POST /api/auth/login` is `permitAll` in `SecurityConfig`. The controller binds and validates the body:

````java
@PostMapping("/login")
public LoginResponse login(@Valid @RequestBody LoginRequest request,
                           HttpServletRequest httpRequest,
                           HttpServletResponse response) {
    return authService.login(request, httpRequest, response);
}
````

`LoginRequest` is `record LoginRequest(@NotBlank String username, @NotBlank String password)`.

### 1.2 `AuthService.login`

The login method executes the following ordered steps:

1. **Rate limit.** Three sliding-window counters are consumed (`ip:{ip}`, `user:{username}`, `ip:{ip}:user:{username}`) using a non-short-circuiting `&` so all three increment on every attempt. If any counter is exceeded, a `LockedException("Too many login attempts")` is thrown (see §4).
1. **Credential verification.** `AuthenticationManager.authenticate(...)` is invoked with a `UsernamePasswordAuthenticationToken`. This delegates to `DaoAuthenticationProvider`, which uses `UserDetailsServiceImpl` to load the user and `BCryptPasswordEncoder` (cost 12) to compare the password.
1. **Authentication exception mapping:**
   * `DisabledException` (inactive account) → `AccessDeniedException("User is deactivated")`.
   * `LockedException` (account already locked) → re-thrown with an administrator-contact message.
   * `BadCredentialsException` → `registerFailedAttempt(username)` is called (§4), then a generic `BadCredentialsException("Invalid credentials")` is thrown. The message is deliberately generic to avoid user enumeration.
   * Any other `AuthenticationException` → `BadCredentialsException("Invalid credentials")`.
1. **Email-verified gate.** After successful authentication the `User` is re-loaded. If `user.isEmailVerified()` is false, an `AccessDeniedException` is thrown instructing the user to verify their email. No tokens are issued.
1. **Reset failure counters.** On success, `resetFailedAttempts(user)` clears `failedLoginAttempts` and `accountLocked` if either is set.
1. **Role branch:**
   * If the user's role code is `SUPERADMIN` (case-insensitive), `completeLogin(...)` runs immediately and `LoginResponse.authenticated(tokenResponse)` is returned — **no MFA challenge**.
   * For all other roles, an email address is required. If none is on file, an `AccessDeniedException` is thrown. Otherwise `MfaService.create(userId)` produces a challenge, the code is emailed via `EmailService.sendMfaCodeEmail(...)`, and `LoginResponse.mfaChallenge(challengeId, maskedEmail)` is returned. The email is masked as `f***@domain` before being returned to the client.

### 1.3 MFA challenge (`MfaService`, Redis)

`MfaService.create` generates a six-digit numeric code using `java.security.SecureRandom` and stores four Redis keys, each with a **15-minute TTL**:

|Key|Purpose|
|---|-------|
|`mfa:challenge:{id}:user`|user id bound to the challenge|
|`mfa:challenge:{id}:code`|the expected code|
|`mfa:challenge:{id}:attempts`|incorrect-attempt counter|
|`mfa:challenge:{id}:resends`|resend counter|

Limits enforced server-side:

* `MAX_ATTEMPTS = 5` — on the sixth verify the challenge is cleared and the user must sign in again.
* `MAX_RESENDS = 3` — on the fourth resend the challenge is cleared.

`verify` increments the attempts counter first, then compares the trimmed submitted code against the stored code. On success the challenge keys are deleted and the bound `userId` is returned. Any failure path raises `BadCredentialsException` with a user-facing message.

### 1.4 Token issuance (`POST /api/auth/mfa/verify`)

````java
@PostMapping("/mfa/verify")
public TokenResponse verifyMfa(@Valid @RequestBody MfaVerifyRequest request,
                               HttpServletRequest httpRequest,
                               HttpServletResponse response) {
    return authService.verifyMfa(request.challengeId(), request.code(), httpRequest, response);
}
````

`verifyMfa` resolves the challenge to a `userId`, loads the user, and calls `completeLogin`, which:

* issues an access token and a refresh token (`JwtService`),
* writes the refresh token as the `sumicare_refresh` httpOnly cookie,
* records a `USER.LOGIN` audit entry with the resolved client IP,
* returns `TokenResponse(accessToken, "Bearer", expiresInSeconds, roleCode)`.

The access token is delivered in the response body (consumed in-memory by the frontend); the refresh token is delivered only as a cookie.

`POST /api/auth/mfa/resend` re-issues the code for an existing challenge, subject to the resend cap.

### 1.5 Refresh flow (`POST /api/auth/refresh`)

`AuthService.refresh` reads the `sumicare_refresh` cookie and:

1. Rejects a missing cookie (`BadCredentialsException("Missing refresh token")`).
1. Parses and verifies the JWT signature; rejects any token whose `type` claim is not `refresh`.
1. Rejects the token if its `jti` is on the deny-list (`isRevoked`) or if it was issued before the user's `tokens-since` watermark (§3.4).
1. Loads the user and rejects refresh if the account is inactive, locked, or email-unverified.
1. **Rotates:** revokes the presented refresh token's `jti` for its remaining TTL, issues a new access token and a new refresh token, and writes the new refresh cookie.

This is full refresh-token rotation: each successful refresh invalidates the token that was used.

### 1.6 Logout (`POST /api/auth/logout`)

`AuthService.logout`:

* If an `Authorization: Bearer` header is present and parseable, the access token's `jti` is revoked for its remaining lifetime.
* If a refresh cookie is present and parseable, the refresh token's `jti` is revoked for its remaining lifetime.
* A `sumicare_refresh` cookie with `maxAge=0` is written to clear it on the client.

Parsing failures during logout are swallowed so logout is best-effort and never errors.

---

## 2. Authorization Model

### 2.1 Role hierarchy

````
SUPERADMIN > ADMIN > MANAGER > RECEPTIONIST
````

The hierarchy is not expressed to Spring Security as a `RoleHierarchy` bean. Instead, each protected operation lists the roles it permits explicitly via `hasAnyRole(...)`. "Higher" roles gain access only because they are included in those lists.

### 2.2 From JWT role claim to authority

`JwtAuthenticationFilter` reads the `role` claim and grants exactly one authority, prefixed with `ROLE_`:

````java
String role = claims.get("role", String.class);
...
UsernamePasswordAuthenticationToken auth = new UsernamePasswordAuthenticationToken(
        principal, token, List.of(new SimpleGrantedAuthority("ROLE_" + role)));
````

The authenticated principal is a record `AuthenticatedPrincipal(String userId, String organizationId, String role)`, populated from the `sub`, `org`, and `role` claims. Controllers receive it via `@AuthenticationPrincipal`. `UserDetailsServiceImpl` likewise grants a single `ROLE_{code}` authority for the username/password authentication path; it rejects accounts with no role assigned (`UsernameNotFoundException`) rather than defaulting to any role.

### 2.3 Method-level enforcement

`@EnableMethodSecurity` is set on `SecurityConfig`. Authorization is enforced with `@PreAuthorize("hasAnyRole(...)")` at both the controller and service layers. `hasAnyRole('ADMIN')` matches the `ROLE_ADMIN` authority (Spring prepends `ROLE_`). Some endpoints use `@PreAuthorize("isAuthenticated()")` (for example the content upload endpoint).

### 2.4 URL-level rules

From `SecurityConfig.filterChain`, the public (`permitAll`) matchers are:

* `OPTIONS /**` (CORS preflight)
* `/api/auth/login`, `/api/auth/mfa/verify`, `/api/auth/mfa/resend`, `/api/auth/refresh`, `/api/auth/logout`, `/api/auth/verify`, `/api/auth/reset-password`, `/api/auth/contact-admin-reset`, `/api/auth/invitations/redeem`
* `/api/public/**`
* `/api/webhooks/**`
* `/uploads/**`
* `/ws/**`
* `/actuator/health`, `/actuator/info`

`/api/content/upload` requires `authenticated()`. Every other request requires authentication (`anyRequest().authenticated()`).

### 2.5 What each role can access (from `@PreAuthorize` usage)

Roles are interpreted inclusively — a list of four roles means all four may call the method.

|Capability area|Roles permitted (`hasAnyRole`)|Representative source|
|---------------|------------------------------|---------------------|
|User create / update / deactivate / reactivate / unlock / send-reset-link|`SUPERADMIN, ADMIN`|`user/controller/UserController.java`|
|List deactivated users|`SUPERADMIN, ADMIN, MANAGER`|`user/controller/UserController.java`|
|Packages, vouchers (configuration)|`SUPERADMIN, ADMIN, MANAGER`|`cashier/service/PackageService.java`, `voucher/service/VoucherService.java`|
|Ledger accounts, ledger listing/balance/revenue/CSV export|`SUPERADMIN, ADMIN, MANAGER`|`cashier/controller/LedgerController.java`|
|Internal website content management|`SUPERADMIN, ADMIN, MANAGER`|`content/controller/ContentController.java`|
|Contact-message CSV export|`SUPERADMIN, ADMIN, MANAGER`|`contact/controller/ContactMessageController.java`|
|Orders (create, payment, lifecycle)|`SUPERADMIN, ADMIN, MANAGER, RECEPTIONIST`|`cashier/service/OrderService.java`, `cashier/controller/OrderController.java`|
|Discount templates|`SUPERADMIN, ADMIN, MANAGER, RECEPTIONIST`|`cashier/controller/DiscountTemplateController.java`|
|Attendance (list, absence, day-off)|`SUPERADMIN, ADMIN, MANAGER, RECEPTIONIST`|`attendance/controller/AttendanceController.java`|
|Dashboard (recent reservations, summary)|`SUPERADMIN, ADMIN, MANAGER, RECEPTIONIST`|`dashboard/service/DashboardService.java`|
|Contact-message read / unread-count / mark-read|`SUPERADMIN, ADMIN, MANAGER, RECEPTIONIST`|`contact/controller/ContactMessageController.java`|
|Decking (therapist queue) controller|`RECEPTIONIST, MANAGER, ADMIN, SUPERADMIN`|`therapist/controller/DeckingController.java`|
|Content upload|any authenticated user|`content/controller/ContentController.java`|

The valid role set is `SUPERADMIN`, `ADMIN`, `MANAGER`, `RECEPTIONIST`; there is no `STAFF` role. `RECEPTIONIST` is the lowest tier and the entry point for day-to-day operations.

Tenant isolation is layered on top of role checks: controllers derive `organizationId` from the authenticated principal and pass it into services (for example `userService.listForOrganization(principal.organizationId())`), and `UserService` calls `requireSameOrganization(...)` before mutating a target user.

---

## 3. JWT Lifecycle

Tokens are signed with HMAC-SHA (`Keys.hmacShaKeyFor` over the UTF-8 bytes of `sumicare.jwt.secret`). The algorithm is HMAC; the exact bit-length is selected by the JJWT library from the key size and is not pinned in code.

### 3.1 Access token

Issued by `JwtService.issueAccessToken`:

````java
return Jwts.builder()
        .id(UUID.randomUUID().toString())
        .subject(userId.toString())
        .claim("org", organizationId.toString())
        .claim("role", role)
        .claim("type", "access")
        .issuedAt(Date.from(now))
        .expiration(Date.from(now.plusMillis(appProperties.jwt().accessExpiryMs())))
        .signWith(signingKey)
        .compact();
````

Claims: `jti` (random UUID), `sub` (user id), `org`, `role`, `type=access`, `iat`, `exp`. Default lifetime is `JWT_EXPIRY_MS = 900000` ms (15 minutes) per `application.yml`.

### 3.2 Refresh token

Issued by `JwtService.issueRefreshToken`: same shape but with `type=refresh` and **no role claim**. Default lifetime is `JWT_REFRESH_EXPIRY_MS = 604800000` ms (7 days).

### 3.3 Validation (`JwtAuthenticationFilter`)

The filter runs before `UsernamePasswordAuthenticationFilter`. For each request:

1. If there is no `Authorization: Bearer` header, the chain continues with no authentication set.
1. The token is parsed and signature-verified via `jwtService.parse`. A `JwtException` (bad signature, expired, malformed) is caught and **ignored** — the request proceeds without authentication.
1. If the `jti` is revoked, the filter continues without authenticating.
1. If the token was issued before the user's `tokens-since` watermark, the filter continues without authenticating.
1. If `type` is not `access`, the filter continues without authenticating (a refresh token cannot be used as a bearer credential).
1. If the `role` claim is missing, the filter continues without authenticating.
1. Only when all checks pass is a `UsernamePasswordAuthenticationToken` placed in the `SecurityContext`.

The filter **never returns an error itself**. An invalid, expired, or revoked token simply leaves the request anonymous. Authorization then fails downstream and the `authenticationEntryPoint` in `SecurityConfig` produces the 401:

````java
.exceptionHandling(ex -> ex.authenticationEntryPoint((request, response, authException) -> {
    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    response.setContentType("application/json");
    response.getWriter().write("{\"message\":\"Authentication is required to access this resource.\"}");
}))
````

### 3.4 Revocation (deny-list)

Two complementary mechanisms, both backed by Redis (`StringRedisTemplate`):

* **Per-token deny-list.** `JwtService.revoke(jti, ttl)` writes `revoked:jti:{jti} = "1"` with a TTL equal to the token's remaining lifetime. `isRevoked(jti)` checks key existence. Used on logout (access and refresh) and on refresh rotation (the consumed refresh token).
* **Per-user watermark.** `JwtService.revokeAllForUser(userId)` writes `user:{userId}:tokens-since = {nowEpochMillis}` with the refresh TTL. `isTokenIssuedBeforeRevocation(userId, iat)` treats any token whose `iat` precedes the watermark as invalid, allowing all of a user's outstanding tokens to be invalidated at once without enumerating their `jti` values. Both the filter and the refresh path consult this watermark.

---

## 4. Account Lockout

### 4.1 State fields (`user/domain/User.java`)

````java
@Column(name = "failed_login_attempts", nullable = false)
private int failedLoginAttempts = 0;

@Column(name = "account_locked", nullable = false)
private boolean accountLocked = false;
````

### 4.2 Counting failures (`AuthService`)

`MAX_FAILED_ATTEMPTS = 2`. On each `BadCredentialsException`, `registerFailedAttempt(username)` runs:

````java
int attempts = user.getFailedLoginAttempts() + 1;
user.setFailedLoginAttempts(attempts);
if (attempts > MAX_FAILED_ATTEMPTS) {
    user.setAccountLocked(true);
}
userRepository.save(user);
````

Because the comparison is `attempts > 2`, the account is locked on the **third** consecutive failed attempt. The counter (and lock) are reset on the next successful authentication via `resetFailedAttempts`.

### 4.3 Locked-account responses

* A locked account that attempts to log in fails authentication with `LockedException` (the lock flag is surfaced through `UserDetailsServiceImpl`, which sets the `accountNonLocked` flag from `!user.isAccountLocked()`). `AuthService` re-throws it with an administrator-contact message.
* Rate-limit exhaustion also throws `LockedException`.
* `GlobalExceptionHandler.handleLocked` maps every `LockedException` to **HTTP 429 (Too Many Requests)** with `{"message": ...}`.

Note that lockout (persistent, per-account) and rate limiting (transient, per minute) are distinct mechanisms that share the same HTTP status code.

### 4.4 Rate limiter (`LoginRateLimiter`)

A per-key counter `ratelimit:login:{key}` is incremented in Redis; on first increment a 1-minute expiry is set. The request is allowed while the count is `<= sumicare.rateLimit.loginPerMinute` (default `10`). This is a fixed-window-per-minute limiter, applied to three keys per login attempt as described in §1.2.

### 4.5 Unlock

`POST /api/users/{userId}/unlock`, restricted to `SUPERADMIN, ADMIN`:

````java
public UserResponse unlockUser(UUID actorId, String actorRole, UUID userId) {
    User target = userRepository.findById(userId).orElseThrow();
    requireSameOrganization(actorId, target);
    enforceTierForTarget(actorRole, target);
    target.setAccountLocked(false);
    target.setFailedLoginAttempts(0);
    target.setUpdatedAt(OffsetDateTime.now());
    userRepository.save(target);
    return toResponse(target);
}
````

Unlock clears both the lock flag and the failure counter, after verifying the actor and target belong to the same organization and that the actor's tier permits acting on the target.

---

## 5. Input Validation

Request DTOs are immutable `record` types annotated with Bean Validation constraints (for example `LoginRequest` uses `@NotBlank`). Controllers opt in with `@Valid @RequestBody`.

When validation fails, `MethodArgumentNotValidException` is rendered by `GlobalExceptionHandler.handleValidationError` as **HTTP 400** with both a single summary `message` and a per-field `fields` map:

````java
Map<String, String> fields = new LinkedHashMap<>();
ex.getBindingResult().getFieldErrors()
        .forEach(e -> fields.putIfAbsent(e.getField(),
                e.getDefaultMessage() != null ? e.getDefaultMessage() : "Invalid value"));
String message = fields.entrySet().stream()
        .map(e -> e.getKey() + ": " + e.getValue())
        .findFirst()
        .orElse("Validation failed");
Map<String, Object> body = new LinkedHashMap<>();
body.put("message", message);
body.put("fields", fields);
return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(body);
````

Related malformed-input cases are also mapped to 400: unreadable/malformed body (`HttpMessageNotReadableException`), missing request parameter (`MissingServletRequestParameterException`), and type mismatch (`MethodArgumentTypeMismatchException`). `IllegalArgumentException` and `IllegalStateException` likewise return 400 with the exception's own message.

---

## 6. CORS

CORS is configured in `SecurityConfig.corsConfigurationSource()`. As written, the configuration uses an **explicit allow-list of origins with credentials enabled**:

````java
CorsConfiguration cors = new CorsConfiguration();
cors.setAllowedOrigins(List.of(
    "http://localhost:4200",
    "https://frontend-production-1b53.up.railway.app",
    "https://newlasemaspa.up.railway.app"
));
cors.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
cors.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With"));
cors.setExposedHeaders(List.of("Authorization"));
cors.setAllowCredentials(true);
````

Because credentials are allowed (required so the browser will send and accept the `sumicare_refresh` cookie cross-site), the origins are an explicit list rather than a wildcard — the CORS specification forbids `Access-Control-Allow-Origin: *` together with credentials. The registered list is hardcoded in the filter chain and is applied to `/**`.

A `sumicare.cors.allowedOrigins` property exists (`AppProperties.Cors`, defaulting to `*` in `application.yml`), but **the active `CorsConfigurationSource` does not read it**. The refresh cookie's `Secure`/`SameSite` attributes are not derived from this property either — `AuthService` derives them from whether the incoming request is over HTTPS (`isSecureRequest(request)`). This divergence between the property default and the hardcoded CORS list is a property of the current code and is recorded here rather than reconciled.

**Angular dev proxy.** In development the frontend runs on `http://localhost:4200`. Although that origin is in the allow-list, teams typically route API calls through the Angular dev-server proxy so that requests are same-origin from the browser's perspective; this avoids preflight on every request and keeps cookie handling straightforward during local development. The exact proxy configuration lives in the frontend project and is not part of this backend reference.

---

## 7. HTTPS / Production Posture

What the **code** sets:

* Stateless sessions; CSRF disabled (appropriate for a token + non-session API). The refresh cookie is `HttpOnly` and `Path=/`, written via `ResponseCookie`.
* The refresh cookie's `Secure` flag and `SameSite` attribute are derived from whether the request is over HTTPS (`isSecureRequest(request)` in `AuthService`): over HTTPS the cookie is `Secure; SameSite=None` (so it flows cross-site to the Railway frontend subdomain and a reload silently refreshes the session); on localhost http it is non-secure with `SameSite=Lax`.
* `Asia/Manila` is the configured Jackson and JDBC time zone.

What the code does **not** set:

* There is **no security-header configuration in code** — no HSTS, `Content-Security-Policy`, `X-Content-Type-Options`, `X-Frame-Options`, or `Referrer-Policy` are added by the filter chain.
* There is no `requiresChannel()` / forced HTTPS redirect in the security configuration.

These are expected to be provided at the platform/proxy layer. The application is intended to run behind a TLS-terminating reverse proxy (the deployment target is a Railway-hosted container); HTTPS enforcement, HSTS, and any additional response headers are handled there, not in application code. The presence of `X-Forwarded-For` handling in `AuthService.clientIp(...)` confirms the app expects to sit behind a proxy. Whether `server.forward-headers-strategy` is configured for that proxy is **not determinable from the files reviewed**.

---

## 8. Sensitive Data Handling

* **Client identity.** Operational records identify clients by nickname only; real names are not stored in operational records. This is a product rule reflected in the domain model rather than something enforced by the security layer.

* **Password storage.** Passwords are hashed with BCrypt at **cost factor 12** (`new BCryptPasswordEncoder(appProperties.bcrypt().cost())`, with `sumicare.bcrypt.cost: 12`). Plaintext passwords exist only transiently on the `LoginRequest` record during authentication and are never persisted.

* **Email masking.** The MFA challenge response returns a masked email (`f***@domain`) rather than the full address.

* **What must never be logged.** Secrets and credentials must never be written to logs: `JWT_SECRET`, `DB_PASSWORD`, SMTP credentials, the PayMongo secret/webhook keys, the email-provider API key, and any access/refresh token or MFA code. These are sourced from environment variables (`application.yml`) and should remain there. The default log level is `INFO` for both `root` and `com.sumicare`; no logger in the reviewed code emits token or password values.

* **No stack-trace or class-name leakage.** `GlobalExceptionHandler` is the single `@RestControllerAdvice`. Every handler returns a small JSON object (`{"message": ...}`, plus a `fields` map for validation). The catch-all handler logs the exception server-side but returns a fixed generic message:
  
  ````java
  @ExceptionHandler(Exception.class)
  public ResponseEntity<Map<String, String>> handleGeneric(Exception ex) {
      log.error("Unhandled exception", ex);
      return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
              .body(Map.of("message", "An unexpected error occurred."));
  }
  ````
  
  No handler serialises the exception object, its class name, or a stack trace into the response body. `DataIntegrityViolationException` is logged at `warn` and surfaced only as a generic conflict message. Stack traces therefore stay in server logs and are not exposed to clients.
