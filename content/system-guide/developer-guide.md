---
title: "developer-guide"
date: 2026-07-12
categories: ["System Guide"]
---

# SumiCare Developer Guide

This guide documents how to set up SumiCare locally, extend the backend and frontend, manage database  
migrations, and follow the project's conventions. It is grounded in the actual repository sources  
(`docker-compose.yml`, `.env.example`, `pom.xml`, `project.json`, the Liquibase changelog, `SecurityConfig`,  
the `voucher` and `shift` modules, and `app.routes.ts`). Where a detail could not be confirmed from the  
code, it is marked **not determinable from code**.

---

## 1. Local Setup

### Prerequisites

|Tool|Version|Notes|
|----|-------|-----|
|Node.js|20.19.0 or newer|Required by Angular 21 / NX 22 (`package.json` `engines.node >= 20.19.0`).|
|JDK|21|Backend targets `java.version` 21 (`pom.xml`).|
|Maven|3.9+|Globally installed. The backend has **no Maven wrapper** — use your system `mvn`.|
|Docker + Docker Compose|current|Used to run Redis (and, per the documented workflow, Postgres and the API).|

### Steps

````bash
# 1. Clone and enter the repository
git clone <repository-url> sumicare
cd sumicare

# 2. Create your local environment file from the template, then fill in required values
cp .env.example .env

# 3. Start the container-backed services
docker compose up

# 4. In a second terminal, install workspace dependencies
npm install

# 5. Start the Angular dev server
npm start
````

* `npm start` runs `nx serve sumicare-web` on **http://localhost:4200** (`project.json` `serve` target, `port: 4200`).
* The dev server proxies `/api/**`, `/ws/**`, and `/actuator/**` to **http://localhost:8080**  
  (`apps/sumicare-web/proxy.conf.json`). The frontend therefore makes same-origin calls in development,  
  avoiding CORS.
* The API listens on port **8080** (`SERVER_PORT` / `API_PORT` in `.env.example`).

### Default sign-in

On first run a default superadmin is seeded:

|Field|Value|
|-----|-----|
|Username|`superadmin`|
|Password|`ChangeMe!12345`|
|Organization slug|`lasema`|

Change this password immediately in any shared or deployed environment.

### Important note on `docker-compose.yml`

The checked-in `docker-compose.yml` currently defines **only the `redis` service** (`redis:7-alpine`,  
host port `6379`, append-only persistence, with a healthcheck). It does **not** define `db` (PostgreSQL)  
or `api` services, even though the README and onboarding notes describe `docker compose up` as bringing up  
Postgres 16, Redis 7, and the API on `:8080`.

In practice this means one of the following for local development:

* Run the Spring Boot API directly with `npm run api:start` (`nx run sumicare-api:serve`) or  
  `mvn spring-boot:run` from `apps/sumicare-api`, and point `DB_URL` at a Postgres instance you provide  
  (a managed Supabase URL is shown as the default in `.env.example`).
* Or run the API container from the multi-stage `apps/sumicare-api/Dockerfile` (which exists and is used  
  for both local and production builds) alongside your own Postgres.

The Angular proxy still expects the API at `localhost:8080` regardless of how it is started.

---

## 2. Adding a New Backend Module

Backend modules live under `com.sumicare.<module>` and follow a strict four-layer split:  
`controller -> service -> repository -> domain` (plus `dto` for request/response records). Use the  
existing **`voucher`** module as the template; its files are:

````
com/sumicare/voucher/
├── domain/Voucher.java                  @Entity
├── domain/VoucherRedemption.java
├── repository/VoucherRepository.java    extends JpaRepository
├── service/VoucherService.java          @Service, @PreAuthorize, @Transactional
├── controller/VoucherController.java     @RestController, /api/vouchers
└── controller/PublicVoucherController.java   public-facing endpoints
````

### Step 1 — Define the entity

Entities use field-level JPA annotations, UUID primary keys with `@GeneratedValue`, and an  
`organization_id` column for multi-tenancy. Plain getters/setters; **no comments, no `@JsonIgnore`** —  
serialize through dedicated DTOs when an entity is not appropriate to expose. Modeled on `Voucher.java`:

````java
package com.sumicare.example.domain;

import jakarta.persistence.*;
import java.util.UUID;

@Entity
@Table(name = "examples")
public class Example {

    @Id
    @GeneratedValue
    @Column(name = "id", columnDefinition = "uuid")
    private UUID id;

    @Column(name = "organization_id", nullable = false, columnDefinition = "uuid")
    private UUID organizationId;

    @Column(name = "name", nullable = false)
    private String name;

    @Column(name = "is_active")
    private boolean active = true;

    public UUID getId() { return id; }
    public void setId(UUID id) { this.id = id; }
    public UUID getOrganizationId() { return organizationId; }
    public void setOrganizationId(UUID organizationId) { this.organizationId = organizationId; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public boolean isActive() { return active; }
    public void setActive(boolean active) { this.active = active; }
}
````

### Step 2 — Write the Liquibase changeset

The schema runs with `hibernate.ddl-auto=validate`, so the table must exist before the entity can be  
mapped. Add a new changeset (see Section 5 for the exact format) creating the `examples` table with  
columns matching the entity, then register it in `db.changelog-master.xml`.

### Step 3 — Create the repository

````java
package com.sumicare.example.repository;

import com.sumicare.example.domain.Example;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;
import java.util.UUID;

public interface ExampleRepository extends JpaRepository<Example, UUID> {
    List<Example> findAllByOrganizationId(UUID organizationId);
}
````

### Step 4 — Create the service with `@PreAuthorize`

Authorization is enforced at the **service layer** with `@PreAuthorize` (method security is enabled  
globally via `@EnableMethodSecurity` in `SecurityConfig`). Mutating methods are `@Transactional`. Modeled  
on `VoucherService`:

````java
package com.sumicare.example.service;

import com.sumicare.example.domain.Example;
import com.sumicare.example.repository.ExampleRepository;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;
import java.util.UUID;

@Service
public class ExampleService {

    private final ExampleRepository repository;

    public ExampleService(ExampleRepository repository) {
        this.repository = repository;
    }

    @PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER')")
    public List<Example> list(UUID organizationId) {
        return repository.findAllByOrganizationId(organizationId);
    }

    @PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER')")
    @Transactional
    public Example create(UUID organizationId, Example example) {
        example.setOrganizationId(organizationId);
        example.setActive(true);
        return repository.save(example);
    }
}
````

### Step 5 — Create the controller and DTO records

Controllers are thin and resolve the caller's tenant from the JWT principal  
(`AuthenticatedPrincipal`, exposed via `@AuthenticationPrincipal`). DTOs are `record` types suffixed  
`Request` (incoming) / `Response` (outgoing). Modeled on `VoucherController`:

````java
package com.sumicare.example.controller;

import com.sumicare.auth.filter.JwtAuthenticationFilter.AuthenticatedPrincipal;
import com.sumicare.example.domain.Example;
import com.sumicare.example.service.ExampleService;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;
import java.util.List;
import java.util.UUID;

@RestController
@RequestMapping("/api/examples")
public class ExampleController {

    private final ExampleService exampleService;

    public ExampleController(ExampleService exampleService) {
        this.exampleService = exampleService;
    }

    @GetMapping
    public List<Example> list(@AuthenticationPrincipal AuthenticatedPrincipal principal) {
        return exampleService.list(UUID.fromString(principal.organizationId()));
    }

    @PostMapping
    public Example create(@AuthenticationPrincipal AuthenticatedPrincipal principal,
                          @RequestBody Example example) {
        return exampleService.create(UUID.fromString(principal.organizationId()), example);
    }

    public record ExampleResponse(UUID id, String name) {}
}
````

### Step 6 — Auditing is automatic

You do **not** add a per-module audit hook. The global `AuditInterceptor`  
(`com.sumicare.audit.interceptor.AuditInterceptor`, registered in `AuditWebConfig` for `/api/**`)  
records an audit log on every **successful** state-modifying request (`POST`, `PUT`, `PATCH`, `DELETE`  
with response status `< 400`) for authenticated principals. It derives the action/entity/target from the  
URI. For a brand-new top-level path it falls back to a generic `METHOD /uri` action; to produce a clean  
action label (e.g. `EXAMPLE.CREATE`) add a `case "examples"` branch to `AuditInterceptor.resolveTarget`  
(optional, cosmetic — auditing still works without it).

### Step 7 — Wire up security for public paths (only if applicable)

All non-public `/api/**` paths require authentication by default (`anyRequest().authenticated()` in  
`SecurityConfig`). If your module exposes **public** endpoints, place them under `/api/public/**`, which is  
already `permitAll`. Any other path that must be open (e.g. a webhook) needs an explicit  
`requestMatchers(...).permitAll()` entry in `SecurityConfig.filterChain`. Currently permitted prefixes  
include `/api/public/**`, `/api/webhooks/**`, `/uploads/**`, `/ws/**`, and the  
specific `/api/auth/*` flows.

---

## 3. Adding a New API Endpoint

To add an endpoint to an existing module:

1. **Add the controller method.** Keep it thin; map the route and delegate to the service. Resolve the  
   tenant from `@AuthenticationPrincipal AuthenticatedPrincipal principal`.

1. **Put business logic in the service**, annotated with `@PreAuthorize` for role enforcement and  
   `@Transactional` if it mutates. Example role expression from the codebase:  
   `@PreAuthorize("hasAnyRole('SUPERADMIN','ADMIN','MANAGER')")`.

1. **Throw domain-appropriate exceptions** rather than building error responses by hand. They are mapped  
   centrally by `GlobalExceptionHandler` (`@RestControllerAdvice` in `com.sumicare.common`):
   
   |Exception|HTTP status|
   |---------|-----------|
   |`IllegalArgumentException`|400 Bad Request|
   |`IllegalStateException`|400 Bad Request|
   |`MethodArgumentNotValidException`|400 (with field errors)|
   |`BadCredentialsException`|401 Unauthorized|
   |`AccessDeniedException`|403 Forbidden|
   |`LockedException`|429 Too Many Requests|
   |`EmptyResultDataAccessException` / `NoSuchElementException`|404 Not Found|
   |`DataIntegrityViolationException`|409 Conflict|
   |`RoomGenderConflictException`|409 Conflict|
   |`ResponseStatusException`|the status it carries|
   |any other `Exception`|500 Internal Server Error|
   
   All error responses are JSON of the form `{"message": "..."}` (validation errors also include a  
   `fields` map). To return a specific status from a controller, throw  
   `new ResponseStatusException(HttpStatus.NOT_FOUND, "…")` as `VoucherController.check` does.

1. **Test manually with curl.** First obtain an access token via the login flow, then call your endpoint  
   with a Bearer header:
   
   ````bash
   TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
     -H "Content-Type: application/json" \
     -d '{"username":"superadmin","password":"ChangeMe!12345","organizationSlug":"lasema"}' \
     | sed -E 's/.*"accessToken":"([^"]+)".*/\1/')
   
   curl -s http://localhost:8080/api/shifts \
     -H "Authorization: Bearer $TOKEN"
   ````
   
    > 
    > The exact login request body field names are **not determinable from code** in this guide's source  
    > set; adjust the JSON keys to match `AuthController` if the call returns 400. The access token is  
    > returned in the response body (the `Authorization` header is also exposed by CORS config).

---

## 4. Adding a New Angular Feature

Frontend features are **standalone, lazy-loaded, signals-first** components.

### Step 1 — Create the standalone component

Use `OnPush` change detection, inject `HttpClient`, and hold view state in signals. Model on  
`apps/sumicare-web/src/app/features/internal/admin/shifts.component.ts`:

````typescript
import { ChangeDetectionStrategy, Component, OnInit, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { environment } from '../../../../environments/environment';

interface Example {
  id: string;
  name: string;
}

@Component({
  selector: 'sumi-admin-examples',
  standalone: true,
  imports: [],
  templateUrl: './examples.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ExamplesAdminComponent implements OnInit {
  private http = inject(HttpClient);
  examples = signal<Example[]>([]);

  ngOnInit(): void {
    this.reload();
  }

  reload(): void {
    this.http.get<Example[]>(`${environment.apiBaseUrl}/api/examples`).subscribe({
      next: (rows) => this.examples.set(rows)
    });
  }
}
````

* `environment.apiBaseUrl` is empty/placeholder in development so requests are same-origin (`/api/**`)  
  and flow through the proxy (`environment.ts` defines `apiBaseUrl`, `wsUrl: '/ws'`,  
  `defaultOrganizationSlug: 'lasema'`).
* The `AuthInterceptor` attaches the Bearer token to outgoing requests, so service code does not set it  
  manually.
* Prefer signals over `BehaviorSubject`. Avoid `any`; pull shared DTOs from `libs/shared-types`.

### Step 2 — Add a lazy route in `app.routes.ts`

Internal routes live under `/sumicare/app/*` and are guarded. The shell itself is behind `authGuard`;  
individual children add `roleGuard(...)` with the appropriate role array. The route-level role constants  
in `app.routes.ts` are:

````typescript
const STAFF_ROLES = ['RECEPTIONIST', 'MANAGER', 'ADMIN', 'SUPERADMIN'];
const MANAGER_PLUS = ['MANAGER', 'ADMIN', 'SUPERADMIN'];
const ADMIN_PLUS = ['ADMIN', 'SUPERADMIN'];
````

Add a child under the `sumicare/app` route:

````typescript
{
  path: 'admin/examples',
  canActivate: [roleGuard(MANAGER_PLUS)],
  loadComponent: () =>
    import('./features/internal/admin/examples.component').then(m => m.ExamplesAdminComponent)
}
````

`authGuard` redirects unauthenticated users to `/sumicare/login`; `roleGuard(roles)` additionally checks  
`auth.hasRole(roles)` (`core/auth/auth.guard.ts`).

### Step 3 — Add it to the internal navigation (internal features only)

Navigation is data-driven in `internal-shell.component.ts` via the `groups: NavGroup[]` array. Add a  
`NavItem` to the appropriate group; the `roles` field controls which roles see the link  
(`visibleGroups` filters by the active session role):

````typescript
{ label: 'Examples', route: 'admin/examples', roles: MANAGER_PLUS }
````

Existing groups are **Overview**, **Operations**, **Finance**, **Configure**, and **Administration**.  
Public features are added under the root path tree in `app.routes.ts` instead and do **not** appear in  
internal nav. Do not add a staff sign-in link anywhere on the public surface — staff navigate directly to  
`/sumicare/login`.

---

## 5. Adding a Liquibase Migration

All schema changes are Liquibase changesets under  
`apps/sumicare-api/src/main/resources/db/changelog/changes/`, included in order by  
`db.changelog-master.xml`. The schema is never auto-generated: `hibernate.ddl-auto=validate` means the  
entity mapping must match the migrated schema exactly, so **every entity change needs a corresponding  
changeset**.

### Step 1 — Create the changeset file

Name it `NNNN-<thing>.xml` with the next sequential four-digit number (the latest in the repo is  
`0060-preferred-therapist.xml`). Use `author="sumicare"` and an idempotent precondition with  
`onFail="MARK_RAN"` so re-runs are safe. Changeset IDs follow `NNNN-<n>-<slug>`.

Column-add example (modeled on `0047-booking-remarks.xml`):

````xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <changeSet id="0061-1-example-name" author="sumicare">
        <preConditions onFail="MARK_RAN">
            <not>
                <columnExists tableName="examples" columnName="name"/>
            </not>
        </preConditions>
        <addColumn tableName="examples">
            <column name="name" type="varchar(255)">
                <constraints nullable="false"/>
            </column>
        </addColumn>
    </changeSet>

</databaseChangeLog>
````

Index example (modeled on `0057-session-status-index.xml`):

````xml
<changeSet id="0061-2-idx-examples-org" author="sumicare">
    <preConditions onFail="MARK_RAN">
        <not>
            <indexExists indexName="idx_examples_org"/>
        </not>
    </preConditions>
    <createIndex tableName="examples" indexName="idx_examples_org">
        <column name="organization_id"/>
    </createIndex>
</changeSet>
````

### Step 2 — Register it in the master changelog

Append an `<include>` to `db.changelog-master.xml` **in order**, after the previous entry:

````xml
<include file="changes/0061-example.xml" relativeToChangelogFile="true"/>
````

### Step 3 — Test locally

Liquibase runs on API startup. To apply a new migration, restart the API (e.g. restart the API container,  
or stop and re-run the Spring Boot process). For a clean re-migrate from an empty database — useful when  
iterating on a changeset before it ships — drop the volumes and bring services back up:

````bash
docker compose down -v
docker compose up
````

If the entity and the migrated schema disagree, startup fails fast with a Hibernate schema-validation  
error (because of `ddl-auto=validate`). Fix the changeset or the entity until they match.

---

## 6. Environment Variable Reference

All variables are defined in `.env.example`; copy it to `.env` and fill in secrets. Never commit real  
values. Defaults below are the values shown in `.env.example`.

|Variable|Purpose|Consumed by|Dev default|
|--------|-------|-----------|-----------|
|`SERVER_PORT`|Port the Spring Boot app binds to internally.|Backend|`8080`|
|`API_PORT`|Host port mapped to the API.|Backend|`8080`|
|`DB_URL`|JDBC URL of the PostgreSQL database.|Backend|`jdbc:postgresql://db.[YOUR-PROJECT-REF].supabase.co:5432/postgres?sslmode=require`|
|`DB_USERNAME`|Database username.|Backend|`postgres`|
|`DB_PASSWORD`|Database password.|Backend|*(empty)*|
|`HIBERNATE_TIME_ZONE`|JDBC/Hibernate time zone.|Backend|`Asia/Manila`|
|`POSTGRES_DB`|Database name for the local Docker Postgres.|Backend (container)|`sumicare`|
|`POSTGRES_USER`|Username for the local Docker Postgres.|Backend (container)|`sumicare`|
|`POSTGRES_PASSWORD`|Password for the local Docker Postgres.|Backend (container)|*(empty)*|
|`REDIS_URL`|Redis connection URL (decking, room state, JWT deny-list, rate limit, WS registry).|Backend|`redis://localhost:6379`|
|`JWT_SECRET`|Signing secret for access/refresh tokens (use ≥32 chars).|Backend|*(empty)*|
|`JWT_EXPIRY_MS`|Access-token lifetime in milliseconds.|Backend|`900000` (15 min)|
|`JWT_REFRESH_EXPIRY_MS`|Refresh-token lifetime in milliseconds.|Backend|`604800000` (7 days)|
|`CORS_ALLOWED_ORIGINS`|Allowed CORS origins; `*` in development.|Backend|`*`|
|`EMAIL_PROVIDER`|Email transport to use (`brevo` or `smtp`).|Backend|`brevo`|
|`EMAIL_FROM`|From address for outbound email.|Backend|`your-account@example.com`|
|`EMAIL_FROM_NAME`|Display name for outbound email.|Backend|`New Lasema Spa Jjimjilbang`|
|`BREVO_API_KEY`|Brevo API key (when `EMAIL_PROVIDER=brevo`).|Backend|*(empty)*|
|`SMTP_HOST`|SMTP host (when `EMAIL_PROVIDER=smtp`).|Backend|`smtp.gmail.com`|
|`SMTP_PORT`|SMTP port.|Backend|`587`|
|`SMTP_USERNAME`|SMTP username.|Backend|*(empty)*|
|`SMTP_PASSWORD`|SMTP password.|Backend|*(empty)*|
|`APP_PUBLIC_BASE_URL`|Public website base URL used for all generated email and QR links.|Backend|`https://newlasemaspa.up.railway.app`|
|`PAYMONGO_SECRET_KEY`|PayMongo secret key.|Backend|`sk_test_`|
|`PAYMONGO_PUBLIC_KEY`|PayMongo public key.|Backend|`pk_test_`|
|`PAYMONGO_WEBHOOK_SECRET`|Secret used to verify PayMongo webhook signatures (required in production).|Backend|*(empty)*|
|`PAYMONGO_MOCK_MODE`|When `true`, payments are mocked for local development.|Backend|`true`|

 > 
 > The frontend reads its own settings from `environment.ts` (`apiBaseUrl`, `wsUrl`,  
 > `defaultOrganizationSlug`), not from `.env`. The `apiBaseUrl` placeholder is substituted at build time  
 > (`scripts/set-env.js`).

---

## 7. Commit Conventions

Commits follow [Conventional Commits](https://www.conventionalcommits.org/): `type(scope): description`,  
with an imperative summary. Do **not** include `Claude` or `Co-Authored-By: Claude` trailers, and do not  
commit `CLAUDE.md`, `.env*`, or `.claude/`.

**Types in use**, with scopes such as `booking`, `cashier`, `web`, `db`, `auth`, `payments`,  
`notifications`, `reports`, `email`, `public`, `treatment-slip`:

|Type|Meaning|Example (drawn from / consistent with repo history)|
|----|-------|---------------------------------------------------|
|`feat`|New user-facing capability.|`feat(booking): add email-verified public booking cancellation`|
|`fix`|Bug fix.|`fix(payments): return to the frontend origin after PayMongo authorization`|
|`refactor`|Behavior-preserving restructuring.|`refactor(web): extract the booking form into a standalone component`|
|`perf`|Performance improvement.|`perf(db): add a composite index on sessions(organization_id, status)`|
|`chore`|Tooling, config, housekeeping.|`chore(web): replace the hardcoded API base URL with a placeholder`|
|`docs`|Documentation only.|`docs(db): document the Liquibase changeset workflow`|
|`security`|Security hardening.|`security(auth): add account lockout after repeated failed logins`|

A scope is optional but encouraged; it names the affected module or surface (e.g. `(web)` for frontend,  
`(db)` for migrations).

---

## 8. Common Gotchas

Business-rule and workflow pitfalls, drawn from the project context and observed in the code:

* **Never use the word "automate."** The product **computerizes** paper workflows. Apply this to code,  
  UI copy, comments-that-don't-exist, and docs.
* **No comments anywhere.** Java, TypeScript, HTML, XML, YAML — none. Names must carry meaning.
* **No emojis anywhere** in templates, strings, UI copy, or source.
* **Clients are identified by nickname only.** Real names are never written to operational tables  
  (treatment slips, sessions, bookings, transactions). The `clients` table holds nickname/email/Facebook  
  handle only. Client accounts are optional and non-critical — a session never requires one.
* **15-minute room prep buffer.** A booking's effective start is `scheduledAt + 15min`; the projected end  
  is `effectiveStart + service.durationMinutes`. Do not collapse this buffer.
* **Latest-shift-first decking.** Therapists form an ordered Redis sorted set keyed  
  `decking:active:{orgId}`; a new shift's therapists are prepended ahead of the remaining older-shift  
  therapists. After ordinary service a therapist rotates to the back; a specifically requested therapist  
  keeps their position; backups never auto-enter and are only inserted manually.
* **Gender-segregated common rooms.** Once a gender is admitted to a common room, remaining beds in that  
  room (or per row of three in multi-row rooms) must match. Violations surface as  
  `RoomGenderConflictException` → HTTP 409. The bed's `genderLock` drives the bed tint in the UI.
* **`ddl-auto=validate` is unforgiving.** Any entity change without a matching Liquibase changeset fails  
  startup with a schema-validation error. Always ship the migration with the entity change, and register  
  the `<include>` in `db.changelog-master.xml` in order.
* **CORS / credentialed requests.** The frontend sends `withCredentials: true` for the refresh cookie.  
  With `CORS_ALLOWED_ORIGINS=*` the API cannot emit `Access-Control-Allow-Credentials`, so browser logins  
  fail even though curl works. Use the dev proxy (so `/api` is same-origin) or lock  
  `CORS_ALLOWED_ORIGINS` to `http://localhost:4200`. Note `SecurityConfig.corsConfigurationSource`  
  hard-codes an allowlist (`http://localhost:4200` plus the Railway domains) with  
  `allowCredentials=true`; reconcile this with the `CORS_ALLOWED_ORIGINS` env var when changing origins.
* **TypeScript version.** Angular 21 requires TypeScript ≥5.9 (the repo pins `~5.9.0`). Don't downgrade.
* **`@angular/common/http` resolution.** Use `moduleResolution: "bundler"`; classic `node` resolution  
  does not resolve the package's `exports` subpaths.
* **The public site never links to login.** No "Staff sign-in" button or admin icon on the public  
  surface; staff know `/sumicare/login`.
* **No Maven wrapper.** Use a globally installed Maven 3.9+ for backend builds (`./mvnw` does not exist).
* **`docs/`, `CLAUDE.md`, `.env*`, and `.claude/` are gitignored.** This guide lives in the gitignored  
  `docs/` directory and will not be tracked unless `.gitignore` is changed. `CLAUDE.md` is the local  
  briefing and must never be committed.
* **`docker-compose.yml` only defines `redis`.** Despite the documented one-command workflow, the  
  compose file does not start Postgres or the API; run the API (and provide a Postgres) separately as  
  described in Section 1.
* **POS/cashier is internal-only.** SumiCare's `pos`/`cashier` module owns payments; it does not  
  integrate with the partner's external BIR-registered POS.
* **PRs into `dev`/`main`.** The default branch is `main`; branch first and open a PR rather than  
  committing to it directly. **Not determinable from code** whether server-side branch protection is  
  enforced — treat PRs as the required workflow.  
  </content>  
  </invoke>
