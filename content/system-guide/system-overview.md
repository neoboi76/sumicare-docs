---
title: "system-overview"
date: 2026-07-12
categories: ["System Guide"]
---

# SumiCare — System Overview

 > 
 > Companion document: [`backend.md`](./backend.md) covers the Spring Boot backend in implementation-level detail. This document describes the system as a whole and the end-to-end flow of a request.

---

## 1. What SumiCare Is

SumiCare is a web-based spa operations management platform built as an academic thesis project, with **New Lasema Spa Jjimjilbang** as the implementation target. The platform computerizes previously paper-based spa workflows and is designed to generalize to other spa enterprises through a multi-tenant data model.

The seeded organization is created by Liquibase changeset `0002-seed-reference-data.xml`:

````xml
<column name="slug" value="lasema"/>
<column name="display_name" value="New Lasema Spa Jjimjilbang"/>
````

So the canonical tenant has `display_name = "New Lasema Spa Jjimjilbang"` and `slug = "lasema"`.

The platform exposes **two surfaces** over the same backend and database:

|Surface|Audience|Auth|Examples|
|-------|--------|----|--------|
|**Public booking website**|Prospective clients|None (`/api/public/**`, `/api/webhooks/**`)|Browse services, book hard/soft reservations, recommendation quiz, leave feedback, self-service booking cancellation|
|**Internal operations system**|Spa staff (RECEPTIONIST → SUPERADMIN)|JWT bearer + `@PreAuthorize`|Decking lineup, room map, treatment slips, cashier/POS, reports, user management, audit logs|

Public endpoints are exposed; internal endpoints are protected (see Section 5).

---

## 2. Modular Monolith — What It Means in This Codebase

SumiCare's backend is a **modular monolith**: a single deployable Spring Boot application (one fat JAR) backed by one PostgreSQL database, internally divided into bounded-context modules.

Concretely:

* **One Spring Boot JAR.** The entry point is `com.sumicare.SumicareApiApplication` (Spring Boot 3.2.5, Java 21). There is no service-to-service network boundary inside the backend.

* **One PostgreSQL database.** All modules persist to the same schema, managed by Liquibase (`ddl-auto: validate`; Hibernate validates the schema but never mutates it).

* **Modules = `com.sumicare.<module>` packages.** Each top-level package under `com.sumicare` is a bounded context: `auth`, `user`, `booking`, `therapist`, `shift`, `transaction`, `cashier`, `pos`, `report`, `attendance`, `recommendation`, `client`, `notification`, `audit`, `content`, `organization`, `room`, `voucher`, `feedback`, `contact`, `dashboard`, plus the cross-cutting `common`.

* **Layered architecture inside each module.** Packages follow `com.sumicare.<module>.<layer>`:
  
  ````
  controller → service → repository → domain
  ````
  
  Controllers are thin HTTP adapters; business logic lives in `service`; persistence lives in Spring Data JPA `repository` interfaces; `domain` holds JPA entities. DTOs (`dto`) carry data across the HTTP boundary as Java `record` types suffixed `Request`/`Response`.

This gives the team module-level separation of concerns without the operational cost of distributed services — appropriate for a single-instance thesis deployment.

---

## 3. NX Monorepo Layout

The repository is an NX workspace containing both backend and frontend plus shared libraries:

````
sumicare/
├── apps/
│   ├── sumicare-api/     Spring Boot 3.2.5 / Java 21 backend (one JAR)
│   └── sumicare-web/     Angular 21 frontend
├── libs/
│   ├── shared-types/     TypeScript DTOs/interfaces shared by the frontend
│   └── ui/               Shared Angular UI component wrappers
├── nx.json
├── docker-compose.yml
└── tsconfig.base.json
````

**Rationale:** a single workspace keeps the API contract, the frontend that consumes it, and shared type definitions versioned together. NX provides task orchestration (build/serve/test) and dependency-graph awareness; the Spring Boot app is wrapped via an NX `run-commands` executor over Maven. Frontend and backend ship independently (Vercel + Railway) but evolve in lockstep within one Git history.

---

## 4. Request Lifecycle (Browser → Response → Re-render)

The following traces an authenticated internal request, for example creating a walk-in booking.

1. **Angular component / service.** A component invokes a feature service that calls `HttpClient`. The base URL comes from `environment.apiBaseUrl`.

1. **`authInterceptor`** (`apps/sumicare-web/src/app/core/auth/auth.interceptor.ts`). For any URL containing `/api/`, if an in-memory session exists, the interceptor clones the request and attaches the bearer header:
   
   ````typescript
   req.clone({ setHeaders: { Authorization: `Bearer ${session.accessToken}` } })
   ````
   
   The access token is held only in an Angular signal (`AuthService.session`) — never in `localStorage`. On a `401`, the interceptor transparently calls `auth.refresh()` and retries the original request once; if refresh fails it routes to `/sumicare/login`.

1. **CORS.** The browser may send a preflight `OPTIONS`. The backend `CorsConfigurationSource` (in `SecurityConfig`) handles it; `OPTIONS /**` is `permitAll`. See [`backend.md`](./backend.md) for the wildcard-vs-credentials behavior.

1. **Spring Security filter chain.** The request enters the stateless filter chain (`SessionCreationPolicy.STATELESS`, CSRF disabled).

1. **`JwtAuthenticationFilter`** (`com.sumicare.auth.filter.JwtAuthenticationFilter`, a `OncePerRequestFilter` registered before `UsernamePasswordAuthenticationFilter`). It reads the `Authorization: Bearer` header, parses and verifies the JWT, and rejects (by simply not authenticating — the token is silently dropped) if the JTI is revoked, the token was issued before a user-wide revocation, the `type` claim is not `access`, or no `role` claim is present. On success it builds an `AuthenticatedPrincipal(userId, organizationId, role)` and sets a `UsernamePasswordAuthenticationToken` with authority `ROLE_<role>` into the `SecurityContextHolder`.

1. **Authorization.** `authorizeHttpRequests` enforces that any non-public path requires authentication; method-level `@PreAuthorize` (enabled via `@EnableMethodSecurity`) enforces role rules on service/controller methods.

1. **Controller.** A `@RestController` method receives the request. The organization is resolved from the principal, e.g.:
   
   ````java
   public WalkInResponse walkIn(@AuthenticationPrincipal AuthenticatedPrincipal principal,
                                @Valid @RequestBody CreateWalkInRequest request) {
       return walkInService.createWalkIn(UUID.fromString(principal.organizationId()), request);
   }
   ````
   
   `@Valid` triggers bean validation; failures are mapped to `400` by the global handler.

1. **Service.** Business logic executes in a `@Service`, scoped to the resolved `organizationId`.

1. **Repository → PostgreSQL.** Spring Data JPA repositories read/write entities. All operational queries are organization-scoped (e.g. `findAllByOrganizationIdAndStatus`).

1. **Redis (when applicable).** Live operational state is written to Redis rather than PostgreSQL — decking queue mutations (`DeckingService`), room/bed occupancy (`RoomOccupancyService`), JWT revocation, rate limiting, and MFA challenges. See [`backend.md`](./backend.md) §Redis.

1. **Real-time broadcast (when applicable).** Mutations that affect shared live views call `NotificationService` to publish a STOMP message to the relevant org-suffixed topic (Section 6).

1. **Audit (after completion).** `AuditInterceptor.afterCompletion` records an immutable audit row for successful mutating requests (`POST/PUT/PATCH/DELETE`, status `< 400`) asynchronously. See [`backend.md`](./backend.md) §Audit.

1. **HTTP response.** The controller returns a DTO; Jackson serializes it to JSON.

1. **Angular handler → signal → OnPush re-render.** The response returns through the interceptor to the calling service, which updates an Angular **signal** (e.g. a session signal, an unread-count signal, or a feature data signal). Because components use `OnPush` change detection, the signal write schedules a targeted re-render of the dependent view.

---

## 5. Public vs Protected Surfaces

`SecurityConfig.filterChain` declares the public allow-list explicitly; everything else is `authenticated()`:

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

`/api/public/**` serves the booking website; `/api/webhooks/**` receives the PayMongo webhook (which is authenticated by HMAC signature, not JWT); `/ws/**` is the STOMP/SockJS endpoint. All other API paths require a valid access token.

---

## 6. Real-Time Architecture

### Broker configuration

`com.sumicare.notification.config.WebSocketConfig` enables a STOMP broker over WebSocket:

````java
registry.enableSimpleBroker("/topic", "/user");
registry.setApplicationDestinationPrefixes("/app");
registry.setUserDestinationPrefix("/user");
// endpoint
registry.addEndpoint("/ws").setAllowedOriginPatterns("*").withSockJS();
registry.addEndpoint("/ws").setAllowedOriginPatterns("*");
````

* Broker destination prefixes: `/topic`, `/user`
* Application (inbound) prefix: `/app`
* Endpoint: `/ws`, with a plain WebSocket registration and a SockJS fallback

### The six org-suffixed topics

`NotificationService` (`com.sumicare.notification.service.NotificationService`) publishes via `SimpMessagingTemplate.convertAndSend`. Every topic is suffixed with the organization id so subscribers only receive their tenant's events:

|Topic destination|Publisher method|Triggered by|
|-----------------|----------------|------------|
|`/topic/decking-updates/{orgId}`|`broadcastDecking`|Any decking/lineup mutation in `DeckingService`|
|`/topic/room-updates/{orgId}`|`broadcastRoomUpdate`|Bed occupy/release in `RoomOccupancyService`; reconciler|
|`/topic/bookings/{orgId}`|`broadcastBookingEvent`|Booking create/update events|
|`/topic/orders/{orgId}`|`broadcastOrderEvent`|Cashier order events|
|`/topic/messages/{orgId}`|`broadcastMessageEvent`|Inbound contact messages|
|`/topic/feedback/{orgId}`|`broadcastFeedbackEvent`|New public feedback|

There is no `reservation-updates` topic. The booking/order/message/feedback payloads share a common shape (`event`, an id field, `summary`, ISO-8601 `at`, and `organizationId`); decking and room payloads carry their domain state directly.

### Frontend subscription

`StompService` (`apps/sumicare-web/src/app/core/realtime/stomp.service.ts`) wraps `@stomp/rx-stomp`. It connects to `environment.wsUrl` (normalizing `http`→`ws`), attaching the bearer token as a connect header, and exposes `watch<T>(destination)` which returns an `Observable<T>` of parsed message bodies.

`NotificationFeedService` (`apps/sumicare-web/src/app/core/notifications/notification-feed.service.ts`) subscribes to the four operational topics for the current organization and maintains unread-count **signals** (`unreadBookings`, `unreadOrders`, `unreadMessages`, `unreadFeedback`), seeded from REST unread-count endpoints and incremented on each incoming STOMP event:

````typescript
this.stomp.watch('/topic/' + key + '/' + organizationId).subscribe({
  next: (message) => { counter.set(counter() + 1); this.events$.next(...); }
});
````

Because the consuming components are `OnPush`, signal updates drive precise re-renders of badges and feeds.

### Session registry

`WebSocketSessionRegistry` listens for `SessionConnectedEvent`/`SessionDisconnectEvent` and tracks active STOMP session ids in Redis sets (`ws:sessions:decking`, `ws:sessions:roommap`).

---

## 7. Multi-Tenancy Model

SumiCare is multi-tenant by `organization_id`:

* **Per-tenant data.** Operational tables carry an `organization_id` column; repositories query with org-scoped finders.
* **Per-request org resolution.** The organization is not inferred from the URL or a header — it is carried in the JWT (`org` claim) and surfaced as `AuthenticatedPrincipal.organizationId()`. Controllers pass it explicitly into services, so a token issued for one tenant can never read or mutate another tenant's rows.
* **Per-tenant branding.** The `organizations` table stores branding (logo, `primary_color`, `secondary_color`, `accent_color`, `theme`, `font_family`), allowing the public site and internal UI to be themed per organization without code changes.
* **Tenant fan-out in jobs.** Scheduled jobs iterate `organizationRepository.findAll()` and process each tenant independently (see [`backend.md`](./backend.md) §Scheduled Jobs).

---

## 8. Design Patterns Present

|Pattern|Where it appears|
|-------|----------------|
|**Repository**|Spring Data JPA interfaces in every module's `repository` package|
|**Service Layer**|`@Service` classes hold all business logic; controllers and repositories hold none|
|**DTO / Immutable record**|Request/Response `record` types in each `dto` package; e.g. `AuthenticatedPrincipal`, `MfaService.Challenge`|
|**Observer (publish/subscribe)**|STOMP topics: `NotificationService` publishes, Angular `StompService`/`NotificationFeedService` subscribe; `WebSocketSessionRegistry` observes Spring application events|
|**Interceptor / Decorator**|`AuditInterceptor` (Spring `HandlerInterceptor`) decorates request handling with cross-cutting audit logging; `JwtAuthenticationFilter` decorates the chain with authentication|
|**Strategy**|`EmailSender` interface with two interchangeable implementations (`SmtpEmailSender`, `BrevoEmailSender`) selected by `sumicare.email.provider`; `PaymentGateway` interface implemented by `PayMongoGateway`|
|**Adapter**|`PayMongoGateway` adapts the PayMongo REST API to the internal `PaymentGateway` port|

---

## 9. Runtime & Startup Notes

`SumicareApiApplication` sets the default JVM time zone to **Asia/Manila** before the context starts, and enables scheduling (`@EnableScheduling`) and asynchronous execution (`@EnableAsync`):

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

On startup, Liquibase runs `db/changelog/db.changelog-master.xml` against PostgreSQL, then Hibernate validates the resulting schema (`ddl-auto: validate`). Redis (Lettuce) and the configured email/payment providers are wired during context initialization. Implementation specifics are in [`backend.md`](./backend.md).
