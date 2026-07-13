---
title: "system_architecture"
date: 2026-07-13
categories: ["System Overview"]
---

## Overview

SumiCare is a web-based spa operations management platform designed to computerize the previously paper-based operational workflows of wellness enterprises such as New Lasema Spa Jjimjilbang. The platform encompasses two interconnected surfaces: a public-facing client booking website and a role-restricted internal operations system. Given this dual-surface structure, the complexity of its business rules (therapist decking, shift-based scheduling, multi-tier reporting, role-based access), and the constraints of an academic development timeline, SumiCare's architecture has been deliberately designed to balance domain expressiveness, developer ergonomics, security, and deployability.

This section presents the chosen architectural patterns, their rationale, the technology stack decisions, the database design philosophy, the security model, and the deployment strategy — each addressed in turn.

---

## 1. Recommended Architecture Pattern: Modular Monolith with Layered Architecture

### 1.1 Rationale

Three broad architectural styles were considered for SumiCare: a traditional layered monolith, a microservices architecture, and a modular monolith. Microservices were ruled out immediately for the following reasons: the development team is small (thesis-scale), inter-service network overhead introduces unnecessary complexity, and distributed system concerns (eventual consistency, service discovery, distributed tracing) are disproportionate to the project's scope and evaluation timeline. A pure layered monolith, on the other hand, risks becoming a tightly coupled "big ball of mud" as the domain grows — an especially notable risk given the many business subdomains present in SumiCare (booking, decking, reporting, recommendations, cashier operations, and more).

The **Modular Monolith** resolves this tension. It is a single deployable unit — preserving simplicity — but internally organized into well-defined, loosely coupled **bounded contexts** (modules), each with its own domain logic, service layer, and data access layer. Communication between modules occurs through explicit service interfaces rather than direct cross-module database queries. This enforces boundaries without the operational overhead of microservices, and critically, the modular structure means the system can be incrementally decomposed into independent services in a future production iteration, should the client eventually require it.

Within each module, a **four-layer Layered Architecture** (also called Clean/Onion Architecture in its spirit) is applied:

1. **Controller Layer** — REST API endpoint handlers. Responsible only for parsing HTTP requests, delegating to the service layer, and serializing responses. Contains request and response DTOs. Has no business logic.
1. **Service Layer** — The application's business logic lives here. Orchestrates domain operations, enforces business rules (e.g., the decking algorithm, the 15-minute buffer rule, commission calculations), and coordinates between repositories.
1. **Repository Layer** — Data access abstraction. Uses Spring Data JPA interfaces that translate domain operations into database queries. No business logic resides here.
1. **Domain/Model Layer** — JPA entities, value objects, and enumerations that represent the core domain concepts (Booking, Therapist, Shift, TreatmentSlip, Report, etc.).

This separation means that any layer can be modified — or tested independently — without cascading changes through the rest of the system, which directly supports the project's need for accurate, verifiable reports ("one mistake can create a big problem, the Domino effect," as stated by the client).

### 1.2 Backend Bounded Contexts (Modules)

The backend is organized into the following modules, each representing a distinct subdomain of SumiCare:

|Module|Responsibility|
|------|--------------|
|`auth`|Authentication, JWT issuance and validation, Spring Security configuration, CORS|
|`user`|User account management, role assignment, permission configuration, password resets|
|`booking`|Appointment scheduling, room assignment, session time tracking, walk-in logic, reservation types (hard/soft)|
|`therapist`|Therapist profiles, decking/lineup algorithm, skip management, requested-therapist flagging, backup therapist insertion|
|`shift`|Shift definitions, shift-therapist associations, biometrics clock-in integration interface|
|`transaction`|Treatment slip creation and digitization, session records, commission calculation|
|`report`|Cutoff reports, end-of-day reports, monthly reports, analytics, Excel export|
|`attendance`|Therapist attendance records, D.O. encoding, absence remarks, automated attendance report generation|
|`recommendation`|Personalized service recommendations based on client profile and massage category preferences|
|`client`|Optional client accounts (non-critical); tracks usage patterns, most requested services and therapists, voucher eligibility, and recommendation system profiles. Clients are never required to register — walk-ins and anonymous sessions remain the norm.|
|`notification`|Real-time WebSocket broker; broadcasts room occupancy and decking state changes to all connected receptionist terminals|
|`pos`|Point-of-sale operations: payment processing (cash, GCash, credit, debit), transaction recording, receipt generation, transaction ledger, cashier shift reconciliation|
|`audit`|System-wide audit logging for non-repudiation; per-action, per-user, per-role log entries|
|`content`|Editable public website content management (for Superadmin/Admin/Manager)|

Each module exposes its functionality through internal service interfaces. Cross-module dependencies are explicitly declared and flow in one direction to avoid circular coupling.

### 1.3 Frontend Architecture: Angular MVVM with Feature-Module Organization

Angular naturally implements the **Model-View-ViewModel (MVVM)** pattern:

* **View** — Angular component templates (HTML + Tailwind CSS). Declarative, reactive, and bound to the ViewModel.
* **ViewModel** — Angular component classes and injectable services. Services hold and transform state; components expose it to the template. State management for complex shared state (e.g., real-time decking updates, active session timers) is managed through a reactive store, using either NgRx or Angular's Signals API (preferred for newer Angular 17+ projects).
* **Model** — TypeScript interfaces and classes that mirror backend DTOs. Shared type definitions live in a dedicated library within the NX monorepo workspace.

The frontend is organized into **feature modules**, each lazy-loaded and route-guarded by role:

|Feature Module|Role Access|Description|
|--------------|-----------|-----------|
|`public`|Unauthenticated / Client|Public booking website, service catalogue, recommendation widget, feedback, consent forms; optional client account creation for pattern tracking and voucher eligibility|
|`auth`|All|Login page, session management|
|`receptionist`|Receptionist+|Booking interface, room map, treatment slip generation, decking view, POS/cashier|
|`manager`|Manager+|All receptionist views + reports, analytics, therapist performance monitoring|
|`admin`|Admin+|All manager views + user management, audit log viewer|
|`superadmin`|Superadmin only|All admin views + admin account management, full system control|
|`shared`|All|Common components (header, sidebar, modals, tables, form controls) using Shadcn/DaisyUI|

Routing is centralized in the `AppRoutingModule`. Route guards (`AuthGuard`, `RoleGuard`) enforce role-based access at the Angular routing level, while corresponding backend endpoint guards enforce it at the API level — a defense-in-depth approach.

### 1.4 Overall System Topology

````
┌──────────────────────────────────────────────────────────────────┐
│                        NX Monorepo                               │
│                                                                  │
│  ┌─────────────────────────┐   ┌──────────────────────────────┐  │
│  │   Angular Frontend App  │   │   Spring Boot Backend App    │  │
│  │  (apps/sumicare-web)    │   │   (apps/sumicare-api)        │  │
│  │                         │   │                              │  │
│  │  Feature Modules        │   │  Bounded Context Modules     │  │
│  │  - public               │◄──►  - auth                     │  │
│  │  - receptionist         │   │  - booking                   │  │
│  │  - manager              │   │  - therapist / shift         │  │
│  │  - admin                │   │  - transaction / pos         │  │
│  │  - superadmin           │   │  - report                    │  │
│  │  - auth                 │   │  - recommendation            │  │
│  └─────────────────────────┘   │  - audit / notification      │  │
│                                └──────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │               Shared Libraries (libs/)                   │    │
│  │   - shared-types (TS interfaces / DTOs)                  │    │
│  │   - ui-components (Shadcn/DaisyUI component wrappers)    │    │
│  │   - utils (formatting, date helpers, constants)          │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
                             │
                     REST API / WebSocket
                             │
              ┌──────────────────────────┐
              │   PostgreSQL Database    │
              │   (+ Liquibase Migrations│
              └──────────────────────────┘
````

---

## 2. Database Design: Relational Database (PostgreSQL)

### 2.1 Relational vs. Non-Relational

SumiCare's data is inherently **structured, relational, and transactional** in nature. The key data entities — clients, therapists, bookings, treatment slips, commissions, shifts, rooms, and reports — maintain explicit, well-defined relationships with one another. A booking record references a client, a therapist, a room, and a massage service type. A cutoff report aggregates transaction records within a defined time window. These relationships are not incidental; they are the operational backbone of the system and must be enforced at the database level.

For these reasons, a **relational database (PostgreSQL)** is unambiguously the correct choice. The arguments in its favor for SumiCare specifically are as follows:

**Referential Integrity.** The system's accuracy requirement is non-negotiable — the client stated that "one mistake can create a big problem, the Domino effect." PostgreSQL's foreign key constraints, cascading rules, and transaction isolation levels provide structural guarantees against data inconsistency that a non-relational database cannot offer.

**Complex Reporting Queries.** SumiCare generates cutoff, end-of-day, and monthly reports involving multi-table aggregations, date-range filtering, GROUP BY operations, and ranked summaries (e.g., top 10 most-requested therapists). SQL's native support for these operations — JOINs, window functions, CTEs — is precisely suited to this use case. Replicating this in a document store (e.g., MongoDB) would require complex application-level aggregation pipelines.

**ACID Compliance.** Transaction operations — assigning a therapist, recording a treatment slip, processing a payment — must be atomic. A partial failure (e.g., a therapist is assigned but the treatment slip is not created) must be rolled back entirely. PostgreSQL's full ACID compliance guarantees this.

**Auditability and Non-Repudiation.** The audit log requirements (per-action, per-user logging) benefit from the structured, queryable nature of relational records. Forensic queries across audit trails are trivial in SQL and cumbersome in unstructured stores.

Alongside PostgreSQL, **Redis** is a required component of SumiCare's data layer — not a replacement for the relational database, but a dedicated in-memory store for a class of data that is volatile, frequently accessed, and operationally critical to real-time correctness. The specific responsibilities assigned to Redis are detailed in Section 2.3 below.

### 2.2 Schema Design Highlights

The following are the principal entity groups within the PostgreSQL schema:

**Identity & Access**

* `users` — system accounts (id, username/nickname, email, hashed_password, role, is_active)
* `roles` — role definitions (SUPERADMIN, ADMIN, MANAGER, RECEPTIONIST, STAFF)
* `permissions` — granular permission flags, linked to roles
* `audit_logs` — immutable log entries (actor_id, action_type, target_entity, timestamp, metadata)

**Therapist & Shift Management**

* `therapists` — therapist profiles (nickname, gender, staff_number, is_backup)
* `shifts` — shift definitions (label, start_time, end_time, expected_therapist_count)
* `shift_assignments` — which therapists are assigned to which shifts
* `therapist_attendance` — clock-in records, absences, D.O. flags, remarks
* `decking_state` — snapshot of the lineup per shift persisted to PostgreSQL for durability; live working state is managed in Redis (see Section 2.3)

**Booking & Session**

* `rooms` — room definitions (room_number, floor, type: common/private/VIP, capacity, gender_lock)
* `beds` — individual bed/table records linked to rooms
* `clients` — optional client accounts (nickname UNIQUE, email UNIQUE nullable, fb_account nullable, locker_number, consent_given, data_tracking_consent, created_at); not required for a session to occur — exists solely to enable pattern tracking, voucher eligibility, and recommendation personalization for clients who opt to register
* `bookings` — reservation records (client_id, massage_type, reservation_type: hard/soft, scheduled_time, actual_start, actual_end, status)
* `sessions` — active session records (booking_id, therapist_id, room_id, bed_id, is_extension, is_requested)
* `treatment_slips` — digitized slip records (tsn, booking_id, session_id, client_nickname, locker_number, therapist_nickname, massage_type, timestamps, is_vip, signed_at)

**Transactions & Cashier**

* `transactions` — payment records (session_id, amount, payment_method, processed_by, timestamp)
* `transaction_ledger` — immutable financial log entries
* `commissions` — per-session commission records (therapist_id, session_id, commission_amount, is_backup)

**Reports**

* `cutoff_reports` — per-shift aggregated report records
* `day_reports` — end-of-day summaries
* `monthly_reports` — monthly summaries
* `report_therapist_stats` — per-therapist performance metrics per report period

**Public Website & Recommendations**

* `services_catalogue` — massage service definitions (name, duration_minutes, commission_rate, category: dry/oil/hard/soft, is_fixed_rate)
* `recommendations` — recommendation records (client_id, suggested_service_ids, generated_at, ai_used)
* `feedback` — client ratings (client_id, session_id, rating_stars, comment, submitted_at)
* `vouchers` — voucher definitions and redemption records
* `website_content` — editable CMS content blocks for the public website

**Schema Versioning with Liquibase.** All schema changes are tracked as ordered Liquibase changeset files. This ensures that database migrations are reproducible, version-controlled, and reversible — critical for a team-based thesis development workflow where schema changes occur frequently during iterative development.

### 2.3 Redis: Required In-Memory Data Layer

Redis is a required infrastructure component in SumiCare, integrated via **Spring Data Redis** (Lettuce driver). It is not a cache bolted on as an afterthought; it is the authoritative store for a specific, well-scoped class of data — volatile operational state that must be read and written with very low latency, often on every client interaction in the receptionist view. PostgreSQL remains the authoritative store for all durable, transactional records. Redis holds the live working state that PostgreSQL is too slow to serve at the required frequency for real-time UX.

The following responsibilities are assigned exclusively to Redis:

**Therapist Decking (Lineup) State** The decking queue is the operationally most active data structure in SumiCare. On every client assignment, every shift transition, every skip event, and every clock-in, the queue must be read and updated atomically. Redis **Sorted Sets** (`ZSET`) are the natural data structure for this: each therapist in the current lineup is a member with a numeric score representing their queue position. The "latest shift first" rule is enforced by assigning new shift therapists a score lower than all existing members (prepend). Atomic operations like `ZADD`, `ZREM`, `ZRANGE`, and `ZINCRBY` execute the decking algorithm without race conditions, without locking, and without database round-trips on every receptionist action. PostgreSQL's `decking_state` table receives a periodic durability snapshot (on each meaningful state change) so the queue can be reconstructed from the database in the event of a Redis restart.

Key naming convention: `decking:active` (global sorted set of therapist IDs by queue position), `decking:shift:{shiftId}:members` (set of therapist IDs belonging to a given shift currently in the queue).

**Room & Bed Occupancy State** The receptionist's room map must reflect live occupancy without polling. The live occupancy state of each bed is maintained as a Redis **Hash** (`HSET`): `room:{roomId}:bed:{bedId}` holds fields for `status`, `clientNickname`, `lockerNumber`, `therapistNickname`, `sessionStartedAt`, and `genderLock`. When a session starts or ends, the service layer updates both the Redis hash (immediately) and the PostgreSQL `sessions` table (durably). The `notification` module reads occupancy from Redis when pushing WebSocket updates to subscribers.

Key naming convention: `room:{roomId}:bed:{bedId}`.

**JWT Refresh Token Revocation (Deny-list)** When a user logs out or a refresh token is rotated, the old token's JTI (JWT ID) is written to a Redis **String** with a TTL equal to the token's remaining lifetime (`SET revoked:jti:{jti} 1 EX {ttlSeconds}`). The JWT filter checks this key on every refresh request. This is faster than a database lookup per request and self-cleaning — expired entries are evicted automatically by Redis TTL, requiring no periodic cleanup job.

Key naming convention: `revoked:jti:{jti}`.

**Rate Limiting (Authentication Endpoints)** Login and refresh endpoints are rate-limited using a Redis-backed sliding window counter to mitigate brute-force attacks. Each key tracks the number of attempts within a rolling window per IP or per username. Implementation uses Spring's `bucket4j-redis` or a custom `RedisTemplate`-based counter with `INCR` and `EXPIRE`.

Key naming convention: `ratelimit:login:{ipOrUsername}`.

**Active WebSocket Session Registry** The `notification` module tracks active WebSocket sessions per topic as Redis **Sets**. Session IDs are added on `SessionConnectedEvent` and removed on `SessionDisconnectedEvent`. This avoids in-memory maps inside the JVM and supports horizontal scaling.

Key naming convention: `ws:sessions:roommap`, `ws:sessions:decking`.

**Technology Integration**

* Spring Boot dependency: `spring-boot-starter-data-redis` (Lettuce as the non-blocking client)
* Local development: Redis runs as a service in Docker Compose alongside PostgreSQL
* Evaluation/production deployment: **Upstash** (serverless Redis; free tier: 10,000 commands/day, which is sufficient for evaluation-phase traffic; scales per-request with no idle cost)
* Configuration is injected via environment variable `REDIS_URL`, consumed by Spring's `RedisConnectionFactory`

---

## 3. Monorepo Strategy: NX Monorepo

### 3.1 Should a Monorepo Be Used for a Thesis Project?

For SumiCare specifically, the answer is **yes**, with caveats. The NX Monorepo approach is recommended for the following reasons:

**Single source of truth.** Both the Angular frontend and the Spring Boot backend reside in one repository. This eliminates version drift between frontend and backend type definitions, simplifies coordinating changes that span both (e.g., adding a new DTO), and makes the repository self-contained for academic submission and evaluation.

**Shared libraries.** NX's library system allows the definition of a `shared-types` library containing TypeScript interfaces that mirror the backend's DTOs. This eliminates the manual duplication of type definitions across the frontend and reduces the likelihood of type mismatch bugs.

**Developer experience.** NX provides a project graph (`nx graph`), affected-command optimization (only rebuild what changed), and generators for scaffolding new modules — all of which improve productivity for a small thesis team.

**The caveat** is that NX's tooling is primarily designed for JavaScript/TypeScript projects. Spring Boot integration is not first-class. The recommended approach is to house the Spring Boot application in `apps/sumicare-api/` as a standard Maven/Gradle project, managed via NX's `run-commands` executor (which wraps `./mvnw` or `./gradlew` commands). The NX workspace treats the Spring Boot app as an opaque executable target, handling it alongside the Angular app through shell command orchestration rather than deep NX plugin integration. This is an accepted and practical pattern.

### 3.2 Workspace Structure

````
sumicare/                          ← NX Workspace Root
├── apps/
│   ├── sumicare-web/              ← Angular Frontend
│   │   └── src/app/
│   │       ├── features/          ← Feature modules (lazy-loaded)
│   │       ├── core/              ← Auth interceptors, guards, services
│   │       └── shared/            ← Shared components, pipes, directives
│   └── sumicare-api/              ← Spring Boot Backend (Maven)
│       └── src/main/java/
│           └── com/sumicare/
│               ├── auth/
│               ├── booking/
│               ├── therapist/
│               ├── report/
│               └── ...            ← One package per bounded context
├── libs/
│   ├── shared-types/              ← TypeScript DTOs shared with frontend
│   └── ui/                        ← Reusable Angular UI components
├── nx.json
├── package.json
└── project.json
````

---

## 4. Security Architecture

Security in SumiCare is implemented at multiple layers — a defense-in-depth approach — with Spring Security as the backbone on the backend and Angular's built-in tools on the frontend.

### 4.1 Authentication: JWT with Refresh Token Rotation

SumiCare uses a **stateless JWT-based authentication** flow:

1. The client submits credentials (email + password) to `POST /api/auth/login`.
1. Spring Security's `AuthenticationManager` validates credentials against the hashed password stored in the database (BCrypt hashing).
1. Upon successful authentication, the server issues two tokens:
   * **Access Token** (short-lived, 15 minutes) — signed HS256 or RS256 JWT containing the user's ID, role, and permissions claim.
   * **Refresh Token** (long-lived, 7 days) — stored as an `httpOnly` cookie to prevent JavaScript access (XSS mitigation).
1. The client attaches the Access Token as a `Bearer` token in the `Authorization` header for all subsequent API requests.
1. When the Access Token expires, the client sends the Refresh Token cookie to `POST /api/auth/refresh` to obtain a new Access Token — implementing **refresh token rotation** (the old refresh token is invalidated on use).
1. Logout (`POST /api/auth/logout`) invalidates the refresh token server-side (stored in a `refresh_tokens` blacklist table or revoked via database flag).

### 4.2 Authorization: Role-Based Access Control (RBAC)

Spring Security's method-level security (`@PreAuthorize`) enforces authorization at the service layer, not merely at the controller level. Each endpoint and service method is annotated with the minimum required role. The role hierarchy is:

````
SUPERADMIN > ADMIN > MANAGER > RECEPTIONIST > STAFF
````

Custom permission flags (e.g., `CAN_EXPORT_REPORTS`, `CAN_MANAGE_USERS`) can be attached to roles via the `permissions` table and injected into the JWT claims, allowing granular access control beyond simple role matching.

### 4.3 CORS Configuration

Cross-Origin Resource Sharing is configured explicitly in Spring Security's `CorsConfigurationSource`. During development and the evaluation phase, origins are set to `*` (allow all) while no production domain has been finalized. Once the production domain is confirmed, `CORS_ALLOWED_ORIGINS` is updated to that specific origin and `*` is removed — this change requires no code modification, only an environment variable update.

* Allowed origins: `*` during development and evaluation; replaced with the exact production domain on go-live
* Allowed methods: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`
* Allowed headers: `Authorization`, `Content-Type`, `X-Requested-With`
* `allowCredentials`: set to `false` when using `*` origin (browsers disallow credentials with wildcard origin); switched to `true` once the origin is locked to a specific domain

On the Angular side, `withCredentials: true` is enabled only after the production domain is set. During the wildcard phase, the refresh token cookie mechanism is limited to same-origin local development.

### 4.4 Frontend Security

* **Route Guards** (`AuthGuard`, `RoleGuard`) prevent unauthorized navigation at the Angular routing level.
* **HTTP Interceptors** automatically attach the JWT Access Token to all outgoing API requests and handle 401 responses by initiating the refresh token flow transparently.
* **Content Security Policy (CSP)** headers are set server-side to mitigate XSS risks.
* Angular's built-in **DOM sanitization** protects against XSS injection in component templates.
* Sensitive data (e.g., access tokens in memory) is never written to `localStorage` — access tokens are kept in JavaScript memory and refresh tokens in `httpOnly` cookies.

### 4.5 Additional Security Measures

* All API endpoints are served over **HTTPS** in production.
* Passwords are hashed using **BCrypt** with a cost factor of 12.
* Input validation is enforced via **Jakarta Bean Validation** (`@NotBlank`, `@Email`, `@Size`, etc.) on all request DTOs, returning structured error responses on violation.
* **Audit logging** (the `audit` module) records every state-modifying action with the actor's identity and timestamp, supporting non-repudiation requirements for Admin and Superadmin oversight.
* **Rate limiting** on the public-facing authentication endpoints to mitigate brute-force attacks (implementable via a Spring filter or API gateway rule).

---

## 5. Real-Time Features: STOMP over WebSocket

SumiCare uses **STOMP over WebSocket** as its real-time transport, enabled via `@EnableWebSocketMessageBroker`. The bidirectional nature of WebSocket is required because the receptionist interface must both receive server-pushed state updates and send acknowledgments or mutations that propagate to other open terminals. Two receptionist terminals open simultaneously must remain in sync — a booking made on one terminal updates the room map on all others.

The real-time use cases in the general version are:

1. **Room Occupancy Map (Receptionist View)** — bed status, gender lock, elapsed time, and client/therapist details pushed to all connected receptionist sessions on any session mutation.
1. **Decking View (Receptionist View)** — queue order broadcast to all connected receptionist sessions after every assignment, skip, shift change, or clock-in event.
1. **Multi-terminal Sync** — all open receptionist terminals receive the same state update stream, eliminating stale UI states across concurrent users.

### 5.1 Spring Boot Configuration

`WebSocketMessageBrokerConfigurer` is implemented in the `notification` module:

* **Endpoint:** `/ws` with SockJS fallback (for corporate proxies and environments that block WebSocket upgrades)
* **Application destination prefix:** `/app` (client → server messages)
* **Topic prefix:** `/topic` (server → all subscribers on a topic)
* **User prefix:** `/user` (server → specific session, reserved for future use)

Topics in the general version:

|Topic|Direction|Trigger|
|-----|---------|-------|
|`/topic/room-updates`|Server → All receptionist clients|Any session start, end, extension, or bed state change|
|`/topic/decking-updates`|Server → All receptionist clients|Any decking queue mutation|

### 5.2 Angular Configuration

Angular uses `@stomp/rx-stomp`. A singleton `RxStompService` is provided at root and connects on application init. Feature modules subscribe to topics as RxJS observables:

````typescript
this.stompService.watch('/topic/room-updates').subscribe(msg => {
  const state: RoomStateDto = JSON.parse(msg.body);
  this.roomMapSignal.set(state);
});
````

### 5.3 Redis WebSocket Registry

Active WebSocket session IDs per topic are maintained in Redis Sets (`ws:sessions:roommap`, `ws:sessions:decking`). The `notification` module's `WebSocketSessionRegistry` registers sessions on `SessionConnectedEvent` and removes them on `SessionDisconnectedEvent`. The broadcast method skips serialization if the target set is empty.

---

## 6. Biometrics Integration

SumiCare does not replace the existing biometrics hardware — it integrates with it. The fingerprint biometrics system (already operational at La Sema for general employee attendance) records clock-in and clock-out events. A separate biometrics terminal dedicated to therapists is planned, and its events are the trigger for adding a therapist to the decking queue.

The `shift` module contains a `BiometricsAdapter` interface that abstracts the integration method. Three implementation options are provided in priority order:

### 6.1 Option A — Webhook (Preferred)

If the biometrics vendor's software supports outbound HTTP calls, it is configured to send a POST request to SumiCare's API on each clock-in event.

* **Endpoint:** `POST /api/biometrics/clock-in`
* **Payload:** `{ staffNumber: string, timestamp: ISO8601, deviceId: string }`
* **Processing:** `BiometricsWebhookAdapter` implements `BiometricsAdapter`; the endpoint is secured with a shared API key header (`X-Biometrics-Key`) rather than JWT, since the biometrics device is not a human user

### 6.2 Option B — Scheduled Polling (Fallback)

If the biometrics software exposes a REST API but does not support outbound webhooks, a `@Scheduled` task in `BiometricsPollingAdapter` calls the vendor API every 30 seconds and imports new events since the last poll. The `last_polled_at` timestamp is stored in Redis (`bio:last_poll`) to avoid reprocessing.

### 6.3 Option C — Shared Database (Last Resort)

If the biometrics system stores events to a local database and exposes no API, `BiometricsDatabaseAdapter` opens a read-only JDBC connection to the vendor's database schema. A `BiometricsAttendanceRepository` maps to the vendor's clock-in table. This is the most tightly coupled option and the least preferred.

### 6.4 Clock-In Event Processing

Regardless of the adapter used, every clock-in event is processed identically by `BiometricsService`:

1. Resolve `staffNumber` → `therapistId` via the `therapists` table
1. Call `AttendanceService.recordClockIn(therapistId, timestamp)` → writes to `therapist_attendance`
1. Check if the current time falls within the therapist's active shift via `ShiftService.getActiveShift(therapistId)`
1. If yes: call `DeckingService.addToQueue(therapistId, shiftId)` → updates the Redis Sorted Set and persists a snapshot to `decking_state`
1. Broadcast `/topic/decking-updates` via `NotificationService`

---

## 7. Recommendation System

The recommendation system is implemented as a **weighted scoring quiz** — a purely algorithmic approach requiring no machine learning. This satisfies the academic scope constraints while remaining accurate and explainable.

### 7.1 Quiz Structure

The client completes a short questionnaire (5 questions). Each question maps to one of the massage preference dimensions:

|Question|Options|
|--------|-------|
|Pressure preference|Light · Medium · Firm · Very firm|
|Focus area|Full body · Back and shoulders · Feet and legs · Head and neck|
|Texture preference|Dry · Oil-based|
|Available duration|30 min · 1 hour · 1.5 hours · 2 hours|
|Primary goal|Relaxation · Muscle tension relief · Fatigue recovery · Circulation|

### 7.2 Scoring Algorithm

A `RecommendationMatrix` maps every (question, answer) pair to a weight vector across all service types in the `services_catalogue`. The matrix is stored as a configuration table in the database (`recommendation_weights`) so administrators can tune it without code changes.

`RecommendationEngine.score(List<QuizAnswer> answers)` iterates over all answers, accumulates the weight vector per service type, and returns the services sorted by total score descending. The top result is the primary recommendation; the next two are alternatives.

Example weight assignments (illustrative):

* "Pressure: Very firm" → adds weight to Deep Tissue (+3), Thai Massage (+2), Ventosa (+2)
* "Focus: Feet and legs" → adds weight to Foot Reflex (+4), Swedish (+1)
* "Texture: Oil-based" → adds weight to Aromatherapy (+3), Lomi-Lomi (+2), Swedish (+1)
* "Duration: 1.5 hours" → adds weight to Aromatherapy w/ Reflex (+3), Ventosa (+2)

### 7.3 AI Enhancement (Optional)

If `ANTHROPIC_API_KEY` is configured, `RecommendationExplainerService` calls the Anthropic API after scoring is complete. It passes the top-ranked service and the client's answers, and asks the model to generate a short, friendly explanation of why that service was recommended — recreational framing only, no medical claims, no disease references. The explanation is appended to the recommendation response as a `rationale` string. If the API key is absent or the call fails, the system returns the recommendation without a rationale — the scoring result is unaffected.

The disclaimer ("SumiCare's recommendations are for relaxation purposes only and do not constitute medical advice") is rendered unconditionally on the frontend regardless of whether AI was used.

---

## 8. Deployment Strategy

### 8.1 Evaluation Phase — Free Cloud Deployment

For the academic evaluation phase, the following free-tier services are used. No evaluator needs to install anything locally.

|Component|Provider|Notes|
|---------|--------|-----|
|Angular frontend|**Vercel**|Static site; free tier; auto-deploys from GitHub `main` branch|
|Spring Boot backend|**Railway**|Detects Spring Boot via nixpacks; free starter tier; Dockerfile overrides nixpacks if needed|
|PostgreSQL|**Supabase**|Free tier: 500MB, daily backups, Supavisor connection pooling|
|Redis|**Upstash**|Free tier: 10,000 commands/day; serverless per-request billing|

**Docker:** Docker is used in both local development and production deployment. A multi-stage `Dockerfile` lives at `apps/sumicare-api/Dockerfile`. Stage 1 (`builder`) uses a Maven image to compile and package the JAR. Stage 2 uses a slim JRE base image to run it, keeping the final image small. The Angular frontend is built to static assets via `ng build --configuration=production` and deployed to Vercel — no Docker container required for the frontend in this phase, as Vercel handles static hosting natively.

`docker-compose.yml` at the workspace root defines three services for local development: `db` (PostgreSQL), `redis`, and `api` (Spring Boot). Developers run `docker compose up` to start the entire backend stack locally with a single command.

For Railway (cloud evaluation), the same `Dockerfile` is used. Railway builds the image from source on each push to `main`, runs the container, and manages environment variable injection through its dashboard.

**CORS in evaluation:** `CORS_ALLOWED_ORIGINS=*` is set in Railway's environment variables. Once a production domain is confirmed this single variable is updated.

### 8.2 Supabase Integration Details

Spring Boot connects to Supabase as a standard PostgreSQL database. The connection string targets Supabase's **Supavisor** connection pooler on port 6543 in transaction pooling mode, which is appropriate for a Spring Boot application using HikariCP:

````
jdbc:postgresql://aws-0-{region}.pooler.supabase.com:6543/{db-name}?user=postgres.{project-ref}&password={password}&sslmode=require
````

Key configuration decisions:

* **Row Level Security (RLS):** Disabled on all application tables. Spring Security owns all authorization; RLS would add a redundant and error-prone second layer.
* **`ddl-auto=validate`:** Hibernate validates schema on startup but does not modify it. Liquibase runs all DDL changes — Supabase's dashboard reflects the Liquibase-managed schema.
* **HikariCP pool size:** `maximum-pool-size=10`, `minimum-idle=2`. Supabase free tier supports up to 60 concurrent connections via Supavisor; these settings are conservative and appropriate for evaluation traffic.
* **Liquibase on deploy:** The Spring Boot startup sequence runs Liquibase's `update` command before the application becomes ready. This applies any pending migrations to the Supabase database automatically on each deploy.

### 8.3 Production Deployment Options

Three deployment models are available for production, listed by fit for La Sema's current infrastructure:

**Option A — On-Premise (Recommended for La Sema)** La Sema already has a server. The same `docker-compose.yml` used for development is the production deployment vehicle — with production environment variables. nginx runs as a reverse proxy container in the same Compose file, handling HTTPS termination (Let's Encrypt via Certbot), WebSocket upgrade headers, and optionally serving the Angular static build.

* PostgreSQL: the `db` service in Docker Compose, data persisted to a named volume
* Redis: the `redis` service in Docker Compose, append-only persistence enabled
* Spring Boot: the `api` service, image built from the multi-stage Dockerfile
* nginx: reverse proxy container; proxies `/api` and `/ws` to Spring Boot; serves Angular static files or forwards to Vercel
* Upgrades: `git pull && docker compose build api && docker compose up -d api` — zero-downtime with nginx buffering

**Option B — Managed Cloud** All components on managed services. Higher operational simplicity; ongoing cost.

* Frontend: Vercel Pro
* Backend: Railway Pro, Render paid, or a VPS (DigitalOcean Droplet / Vultr) running Docker
* PostgreSQL: Supabase Pro (point-in-time recovery, no pausing)
* Redis: Upstash Pro (no command limits)

**Option C — Hybrid (Most Likely)** Backend and databases on-premise; frontend on Vercel. This keeps sensitive operational data behind La Sema's network while the frontend is globally available.

* CORS: `CORS_ALLOWED_ORIGINS` = La Sema's registered domain (e.g., `https://sumicare.lasema.com.ph`)
* WebSocket: WSS (secure WebSocket) is required in production; nginx must be configured to proxy WebSocket upgrade headers

---

## 9. Architecture Decision Summary

|Decision|Choice|Primary Reason|
|--------|------|--------------|
|Overall Pattern|Modular Monolith|Team size, deployment simplicity, domain boundary enforcement without microservice overhead|
|Backend Architecture|Layered (Clean) per module|Separation of concerns, testability, maintainability|
|Frontend Architecture|Angular MVVM, Feature Modules|Angular's natural pattern; lazy loading for role-based routing|
|Database Type|Relational (PostgreSQL)|Structured, relational domain; ACID compliance; complex reporting queries|
|Schema Migrations|Liquibase|Version-controlled, reproducible schema changes|
|Monorepo|NX Monorepo|Single source of truth; shared types; developer ergonomics|
|Authentication|JWT (Access + Refresh Tokens)|Stateless scalability; httpOnly cookie for refresh once domain is fixed|
|Authorization|Spring Security RBAC + `@PreAuthorize`|Granular, method-level role enforcement|
|CORS|`*` (wildcard) now; locked to domain on go-live|No production domain confirmed yet; single env var change required|
|Real-Time Transport|STOMP over WebSocket + SockJS|Bidirectional; multi-terminal sync; decking and room map push|
|Real-Time Volatile State|Redis (required)|Decking ZSET, room occupancy Hash, JWT deny-list, rate limiting, WS session registry|
|Frontend State|Angular Signals / NgRx|Reactive, predictable state for room map and decking view|
|POS|Dedicated `pos` backend module + receptionist frontend view|SumiCare owns its own cashier operations; external POS not integrated|
|Recommendation Engine|Weighted scoring quiz|Deterministic, explainable, no ML; Anthropic API adds optional natural-language rationale|
|Biometrics Integration|`BiometricsAdapter` interface|Webhook-first; polling and direct DB fallbacks; decoupled from vendor|
|Staff TV Display|Out of scope (general version)|Deferred; architecture leaves room in `notification` module for future addition|
|Docker|Multi-stage Dockerfile + Docker Compose|Used for both local development and production deployment|
|Production DB|Supabase (PostgreSQL)|Managed PostgreSQL; Supavisor pooling; Liquibase-managed DDL; RLS disabled|
|Evaluation Deployment|Vercel + Railway + Supabase + Upstash|Free tier; zero-config; no local setup for evaluators|
|Production Deployment|On-premise Docker Compose (Option A)|La Sema has a server; same Compose file as dev; nginx reverse proxy|
|Styling|Tailwind CSS + Shadcn/DaisyUI|Utility-first CSS; accessible, unstyled component primitives|
