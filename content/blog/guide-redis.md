---
title: "redis"
date: 2026-07-12
categories: ["System Guide"]
---

# Redis Reference

SumiCare uses Redis as an in-memory store for ephemeral, high-churn, or live-state data that does not belong in PostgreSQL. Redis holds short-lived security artifacts (MFA challenges, token revocation, rate-limit counters, cancellation codes), the live therapist decking queue, live room/bed occupancy, and the active WebSocket session registry. PostgreSQL remains the durable system of record; Redis holds derived or transient state that is either rebuilt, expired, or reconciled.

All access is through Spring Data Redis. Two templates are exposed (see [Connection configuration](#connection-configuration)): every key documented below is written and read through `StringRedisTemplate`.

---

## Key reference

### Authentication and security

#### `mfa:challenge:{id}:user`

|Property|Value|
|--------|-----|
|Key template|`mfa:challenge:{challengeId}:user` where `{challengeId}` is a random `UUID`|
|Data type|String|
|Value|The user's `UUID` as a string|
|Purpose|Binds an in-flight MFA challenge to a user. Lives in Redis because the challenge is short-lived, single-use, and must auto-expire without a cleanup job; persisting it to PostgreSQL would leave dangling rows.|
|Written by|`MfaService.create`, `MfaService.resend`|
|Read by|`MfaService.resend`, `MfaService.verify`|
|TTL / expiry trigger|15 minutes (`Duration.ofMinutes(15)`), set on write|
|Eviction behavior|Expires automatically after TTL; also explicitly deleted by `MfaService.clear` on successful verify, on exceeding `MAX_RESENDS` (3), or on exceeding `MAX_ATTEMPTS` (5)|

#### `mfa:challenge:{id}:code`

|Property|Value|
|--------|-----|
|Key template|`mfa:challenge:{challengeId}:code`|
|Data type|String|
|Value|The 6-digit numeric verification code (`%06d`)|
|Purpose|Stores the expected MFA code for comparison at verify time. Redis chosen for the same auto-expiry reason as the `:user` key.|
|Written by|`MfaService.create`, `MfaService.resend`|
|Read by|`MfaService.verify`|
|TTL / expiry trigger|15 minutes, set on write|
|Eviction behavior|Expires after TTL; explicitly deleted by `MfaService.clear` on successful verify or attempt/resend exhaustion|

#### `mfa:challenge:{id}:attempts`

|Property|Value|
|--------|-----|
|Key template|`mfa:challenge:{challengeId}:attempts`|
|Data type|String (integer counter)|
|Value|Incrementing count of verify attempts, initialized to `"0"`|
|Purpose|Enforces the per-challenge verify attempt cap (`MAX_ATTEMPTS = 5`).|
|Written by|`MfaService.create` (init to `0`); incremented by `MfaService.verify` via `increment`|
|Read by|`MfaService.verify` (via the return value of `increment`)|
|TTL / expiry trigger|15 minutes, set on initial write in `create`|
|Eviction behavior|Expires after TTL; explicitly deleted by `MfaService.clear`|

#### `mfa:challenge:{id}:resends`

|Property|Value|
|--------|-----|
|Key template|`mfa:challenge:{challengeId}:resends`|
|Data type|String (integer counter)|
|Value|Incrementing count of resend requests, initialized to `"0"`|
|Purpose|Enforces the per-challenge resend cap (`MAX_RESENDS = 3`).|
|Written by|`MfaService.create` (init to `0`); incremented by `MfaService.resend` via `increment`|
|Read by|`MfaService.resend` (via the return value of `increment`)|
|TTL / expiry trigger|15 minutes, set on initial write in `create`. Note: `resend` re-writes the `:user` and `:code` keys with a fresh 15-minute TTL but does not reset the `:resends` key's TTL.|
|Eviction behavior|Expires after TTL; explicitly deleted by `MfaService.clear`|

#### `revoked:jti:{jti}`

|Property|Value|
|--------|-----|
|Key template|`revoked:jti:{jti}` where `{jti}` is the JWT `id` claim|
|Data type|String|
|Value|`"1"` (presence sentinel)|
|Purpose|JWT revocation deny-list. A token whose `jti` is present is treated as revoked. Lives in Redis so the JWT filter can check revocation on every request without a database lookup, and entries self-expire when the underlying token would have expired anyway.|
|Written by|`JwtService.revoke`|
|Read by|`JwtService.isRevoked` (called from `JwtAuthenticationFilter.doFilterInternal` and `AuthService.refresh`)|
|TTL / expiry trigger|TTL passed by the caller. In `AuthService.refresh` and `AuthService.logout`, the TTL is the remaining lifetime of the token being revoked: `max(claims.getExpiration().getTime() - System.currentTimeMillis(), 0)`. For a refresh token this is bounded by its 7-day expiry.|
|Eviction behavior|Expires automatically once the revoked token's own expiry has passed; never explicitly deleted|

#### `user:{userId}:tokens-since`

|Property|Value|
|--------|-----|
|Key template|`user:{userId}:tokens-since`|
|Data type|String|
|Value|An epoch-milliseconds timestamp; tokens issued before this instant are treated as revoked|
|Purpose|Bulk revocation cutoff for a single user — invalidates every token issued before the recorded instant in one entry, instead of listing individual `jti`s.|
|Written by|`JwtService.revokeAllForUser` (note: this method is defined but has no callers in the current source)|
|Read by|`JwtService.isTokenIssuedBeforeRevocation` (called from `JwtAuthenticationFilter.doFilterInternal` and `AuthService.refresh`)|
|TTL / expiry trigger|TTL equals the refresh-token expiry: `Duration.ofMillis(appProperties.jwt().refreshExpiryMs())` (default 604800000 ms = 7 days)|
|Eviction behavior|Expires automatically after the refresh-token expiry window; never explicitly deleted|

#### `ratelimit:login:{key}`

|Property|Value|
|--------|-----|
|Key template|`ratelimit:login:{key}` where `{key}` is a caller-supplied discriminator. `AuthService.login` calls the limiter three times per attempt with `ip:{ip}`, `user:{username}`, and `ip:{ip}:user:{username}`, producing keys such as `ratelimit:login:ip:{ip}`.|
|Data type|String (integer counter)|
|Value|Request count within the current 1-minute window|
|Purpose|Sliding fixed-window rate limiting on the login endpoint. Redis counters with TTL give atomic increment plus automatic window reset without storing per-request rows.|
|Written by|`LoginRateLimiter.tryConsume` (`increment`, and `expire` on the first hit when the counter equals `1`)|
|Read by|`LoginRateLimiter.tryConsume` (compares the incremented count against `properties.rateLimit().loginPerMinute()`, default 10)|
|TTL / expiry trigger|1 minute (`Duration.ofMinutes(1)`), set only when the counter is first created (`count == 1`)|
|Eviction behavior|Expires after the 1-minute window; never explicitly deleted|

### Booking cancellation

#### `cancel:code:{bookingId}`

|Property|Value|
|--------|-----|
|Key template|`cancel:code:{bookingId}`|
|Data type|String|
|Value|`{6-digit code}:{normalized email}` — the issued code concatenated with the trimmed, lower-cased email|
|Purpose|Holds the emailed self-service cancellation code for a public booking, bound to the requester's email. Short-lived and single-use, so Redis with TTL is preferred over a database column.|
|Written by|`BookingCancellationCodeService.issue`|
|Read by|`BookingCancellationCodeService.verify`|
|TTL / expiry trigger|15 minutes (`TTL = Duration.ofMinutes(15)`), set on write|
|Eviction behavior|Expires after TTL; explicitly deleted by `BookingCancellationCodeService.clear` and on exceeding `MAX_ATTEMPTS` (5) during verify|

#### `cancel:attempts:{bookingId}`

|Property|Value|
|--------|-----|
|Key template|`cancel:attempts:{bookingId}`|
|Data type|String (integer counter)|
|Value|Incrementing count of verify attempts|
|Purpose|Caps verification attempts per booking at `MAX_ATTEMPTS = 5` to throttle brute-force guessing of the cancellation code.|
|Written by|`BookingCancellationCodeService.verify` (`increment`, plus `expire` to 15 minutes on the first hit when count equals `1`); deleted in `issue` to reset the counter when a new code is issued|
|Read by|`BookingCancellationCodeService.verify` (via the `increment` return value)|
|TTL / expiry trigger|15 minutes, set on the first increment within a cycle|
|Eviction behavior|Expires after TTL; explicitly deleted by `clear` and on attempt exhaustion|

#### `cancel:cooldown:{bookingId}`

|Property|Value|
|--------|-----|
|Key template|`cancel:cooldown:{bookingId}`|
|Data type|String|
|Value|`"1"` (presence sentinel)|
|Purpose|Enforces a resend cooldown so a new cancellation code cannot be requested more than once per minute.|
|Written by|`BookingCancellationCodeService.issue`|
|Read by|`BookingCancellationCodeService.onCooldown` (presence check via `hasKey`)|
|TTL / expiry trigger|60 seconds (`RESEND_COOLDOWN = Duration.ofSeconds(60)`), set on write|
|Eviction behavior|Expires after TTL; never explicitly deleted|

### Therapist decking

All decking keys are organization-scoped. Constants: `decking:active:`, `decking:shift:`, `decking:skip:`, `decking:flag:`.

#### `decking:active:{orgId}`

|Property|Value|
|--------|-----|
|Key template|`decking:active:{organizationId}`|
|Data type|Sorted Set|
|Members / scores|Member = therapist `UUID` string; score = ordering value. Append uses `Instant.now().toEpochMilli()`; front-insert uses `lowestScore - 1.0`; backup insert uses the midpoint between neighbors.|
|Purpose|The live therapist lineup (decking queue). A sorted set gives ordered, atomic position operations (front, back, midpoint insert) without locking. Live operational state that is broadcast over WebSocket, not a durable record.|
|Written by|`DeckingService.appendToBack`, `prependToFront`, `rotateToBack`, `remove`, `insertBackup` (all via `opsForZSet().add` / `remove`)|
|Read by|`DeckingService.currentLineup`, `lowestScore`, `insertBackup`, `rotateToBack` (via `rangeWithScores` / `score`)|
|TTL / expiry trigger|None — no expiry set|
|Eviction behavior|Members removed explicitly via `DeckingService.remove` (`ZREM`). The key itself is not expired and persists until empty.|

#### `decking:shift:{orgId}:{shiftId}`

|Property|Value|
|--------|-----|
|Key template|`decking:shift:{organizationId}:{shiftId}`|
|Data type|Set|
|Members|Therapist `UUID` strings belonging to the given shift|
|Purpose|Tracks which therapists were added under a particular shift, supporting the latest-shift-first decking model.|
|Written by|`DeckingService.appendToBack`, `prependToFront` (via `opsForSet().add` when `shiftId` is non-null)|
|Read by|Not read elsewhere in the documented source (written but not consumed by `DeckingService` methods reviewed)|
|TTL / expiry trigger|None — no expiry set|
|Eviction behavior|No explicit removal or expiry in `DeckingService`|

#### `decking:skip:{orgId}:{therapistId}`

|Property|Value|
|--------|-----|
|Key template|`decking:skip:{organizationId}:{therapistId}`|
|Data type|String|
|Value|`"1"` (presence sentinel)|
|Purpose|Marks a therapist as on a timed skip/break. Presence means skipped; auto-expiry ends the skip without a scheduled job.|
|Written by|`DeckingService.skip` (with a caller-supplied `Duration` TTL)|
|Read by|`DeckingService.isSkipped`, `DeckingService.currentLineup` (presence check via `hasKey`)|
|TTL / expiry trigger|TTL set per call to `skip(...)` from the supplied `Duration` (the business rule caps a break at 30 minutes)|
|Eviction behavior|Expires after the supplied TTL; explicitly deleted by `DeckingService.cancelSkip` for early self-cancel|

#### `decking:flag:{orgId}:{therapistId}`

|Property|Value|
|--------|-----|
|Key template|`decking:flag:{organizationId}:{therapistId}`|
|Data type|String|
|Value|A `DeckingFlag` enum name: `NONE`, `REQUESTED`, `SCRUB`, `ORDINARY`, `BACKUP`, or `MANUAL`|
|Purpose|Annotates a therapist's lineup entry with a flag (e.g. `BACKUP` for manually inserted backups, `REQUESTED` for specifically requested therapists).|
|Written by|`DeckingService.setFlag` (also set to `BACKUP` inside `insertBackup`)|
|Read by|`DeckingService.currentLineup`|
|TTL / expiry trigger|None — no expiry set|
|Eviction behavior|Explicitly deleted by `DeckingService.clearFlag`; otherwise persists|

### Room occupancy

#### `room:{roomId}:bed:{bedId}`

|Property|Value|
|--------|-----|
|Key template|`room:{roomId}:bed:{bedId}`|
|Data type|Hash|
|Fields|`status` (`OCCUPIED`), `clientNickname`, `lockerNumber`, `therapistNickname`, `genderLock`, `ownerItemId`, `startedAt` (epoch ms). Empty optional values are stored as `""`.|
|Purpose|Live per-bed occupancy state for the room map. A hash holds the full bed state in one key for fast reads and WebSocket broadcast. Live state, kept out of PostgreSQL so the room map reads from Redis rather than the durable session tables.|
|Written by|`RoomOccupancyService.occupy` (`opsForHash().putAll`)|
|Read by|`RoomOccupancyService.read` and `countOccupiedBeds` (`opsForHash().entries`); scanned and read by `RoomOccupancyReconcilerJob.reconcileForOrg`|
|TTL / expiry trigger|None — no expiry set|
|Eviction behavior|Explicitly deleted by `RoomOccupancyService.release` (`delete`). Orphaned beds (status `OCCUPIED` in Redis with no matching `ACTIVE` session) are released by `RoomOccupancyReconcilerJob`, which runs on a fixed delay of 120000 ms with an initial delay of 30000 ms, scans keys matching `room:*:bed:*` (SCAN, count 200), and calls `release` for any orphan.|

### WebSocket session registry

#### `ws:sessions:decking`

|Property|Value|
|--------|-----|
|Key template|`ws:sessions:decking` (constant)|
|Data type|Set|
|Members|STOMP `simpSessionId` strings|
|Purpose|Tracks active STOMP sessions subscribed to decking updates. Set membership in Redis allows multiple application instances to share session presence.|
|Written by|`WebSocketSessionRegistry.onConnected` (`opsForSet().add` on `SessionConnectedEvent`)|
|Read by|Not read elsewhere in the documented source (membership maintained but not consumed by the reviewed code)|
|TTL / expiry trigger|None — no expiry set|
|Eviction behavior|Members removed by `WebSocketSessionRegistry.onDisconnect` (`opsForSet().remove` on `SessionDisconnectEvent`)|

#### `ws:sessions:roommap`

|Property|Value|
|--------|-----|
|Key template|`ws:sessions:roommap` (constant)|
|Data type|Set|
|Members|STOMP `simpSessionId` strings|
|Purpose|Tracks active STOMP sessions subscribed to room-map updates.|
|Written by|`WebSocketSessionRegistry.onConnected` (`opsForSet().add` on `SessionConnectedEvent`)|
|Read by|Not read elsewhere in the documented source|
|TTL / expiry trigger|None — no expiry set|
|Eviction behavior|Members removed by `WebSocketSessionRegistry.onDisconnect` on `SessionDisconnectEvent`|

 > 
 > Note: `onConnected` adds every connecting session to both `ws:sessions:decking` and `ws:sessions:roommap` unconditionally; the registry does not distinguish per-topic subscriptions.

---

## Connection configuration

Redis connectivity is defined in `common/redis/RedisConfig.java`.

The connection factory is built from a single URL property:

````java
@Bean
public RedisConnectionFactory redisConnectionFactory() {
    RedisURI uri = RedisURI.create(redisProperties.url());
    RedisStandaloneConfiguration config = new RedisStandaloneConfiguration(uri.getHost(), uri.getPort());
    if (uri.getPassword() != null) {
        config.setPassword(String.valueOf(uri.getPassword()));
    }
    if (uri.getDatabase() > 0) {
        config.setDatabase(uri.getDatabase());
    }
    return new LettuceConnectionFactory(config);
}
````

The URL is parsed into a `RedisStandaloneConfiguration`, so the deployment topology is a single standalone Redis node (no cluster or sentinel configuration). Host, port, password, and database index are all derived from the URL via `io.lettuce.core.RedisURI`. The driver is **Lettuce** (`LettuceConnectionFactory`).

The URL comes from the bound property record:

````java
@ConfigurationProperties(prefix = "spring.data.redis")
public record RedisProperties(String url) {}
````

In `application.yml`:

````yaml
spring:
  data:
    redis:
      url: ${REDIS_URL:redis://localhost:6379}
      timeout: 2s
      lettuce:
        pool:
          max-active: 16
          max-idle: 8
          min-idle: 0
````

* `spring.data.redis.url` — defaults to `redis://localhost:6379`, overridable via the `REDIS_URL` environment variable.
* `spring.data.redis.timeout` — `2s` command timeout.
* `spring.data.redis.lettuce.pool.max-active` — `16`
* `spring.data.redis.lettuce.pool.max-idle` — `8`
* `spring.data.redis.lettuce.pool.min-idle` — `0`

Note: `RedisConfig` reads only the `url` property directly into `RedisProperties`. The `timeout` and `lettuce.pool.*` values under `spring.data.redis` are standard Spring Boot Redis properties; they apply to the auto-configured Lettuce client settings rather than being read by the custom `RedisConfig` bean, which constructs `LettuceConnectionFactory` from the standalone configuration only.

### Templates

Two beans are exposed:

|Bean|Type|Serialization|
|----|----|-------------|
|`stringRedisTemplate`|`StringRedisTemplate`|String keys and values|
|`redisTemplate`|`RedisTemplate<String, Object>`|`StringRedisSerializer` for keys/hash-keys; `GenericJackson2JsonRedisSerializer` (using the application `ObjectMapper`) for values/hash-values|

Every key documented above is accessed through `StringRedisTemplate`. The JSON-serializing `RedisTemplate<String, Object>` is defined but is not used by any of the services documented here.

---

## Redis failure and unavailability handling

There is **no explicit Redis failover, retry, or degraded-mode fallback in the application code.** No code path catches `RedisConnectionFailureException`, `QueryTimeoutException`, or any Redis-specific exception to substitute alternate behavior. The implications follow from how each call site handles (or does not handle) a thrown exception:

* **Login MFA, rate limiting, token revocation checks, decking, and room occupancy writes** propagate any Redis exception to the caller. If Redis is unreachable, these operations fail with an error (surfaced to the global exception handler), and the affected flow does not complete. There is no in-memory or database fallback for these paths.
* **`JwtAuthenticationFilter.doFilterInternal`** wraps token parsing in `try { ... } catch (JwtException ignored) {}`. This catches only `JwtException`. A Redis failure during `isRevoked` / `isTokenIssuedBeforeRevocation` is a `RuntimeException`, not a `JwtException`, so it would propagate out of the filter rather than being silently ignored.
* **`AuthService.logout`** wraps each `jwtService.revoke(...)` call in `try { ... } catch (Exception ignored) {}`, so a Redis failure during logout is swallowed and logout still clears the refresh cookie. This is the only place where a Redis error is tolerated by design.
* **`RoomOccupancyReconcilerJob`** is defensive: the top-level `reconcile()` catches `Exception` and logs `"RoomOccupancyReconcilerJob failed"`; a failed Redis `SCAN` is caught and logged (`"Redis SCAN failed in reconciler"`) and that organization's reconciliation is skipped; per-key errors are caught and logged without aborting the run. A Redis outage degrades reconciliation to a no-op rather than crashing the scheduler.

In short, aside from logout (intentionally swallowed) and the reconciler job (defensively logged), Redis unavailability causes the dependent request to fail. The system treats Redis as a required dependency, consistent with the project's stated requirement that Redis is mandatory.
