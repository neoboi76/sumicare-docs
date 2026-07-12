---
title: "frontend"
date: 2026-07-12
categories: ["System Guide"]
---

# SumiCare Frontend Architecture

This document describes the architecture of the SumiCare Angular frontend (`apps/sumicare-web`). It is derived directly from the source in `apps/sumicare-web/src`, `libs/shared-types/src`, and `libs/ui/src`. Where a detail is not expressed in code, it is marked "not determinable from code".

---

## 1. Application Structure

The frontend is the `sumicare-web` application inside the NX monorepo. It is a fully standalone-component Angular app: there is no root `NgModule`. The app is bootstrapped from `main.ts` against `appConfig`, which is assembled in `app.config.ts`.

````typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(APP_ROUTES, withComponentInputBinding()),
    provideHttpClient(withFetch(), withInterceptors([authInterceptor, loadingInterceptor, errorInterceptor])),
    provideAnimations(),
    provideAppInitializer(() => inject(AuthService).bootstrapSession())
  ]
};
````

Key structural facts:

* **Standalone components throughout.** Every routed component is loaded via `loadComponent` (route-level lazy loading), so each feature is its own code-split chunk. There are no eager feature modules.
* **`withComponentInputBinding()`** is enabled, so route params (for example `:id`) bind directly to component inputs.
* **`provideAppInitializer`** runs `AuthService.bootstrapSession()` before the app renders, attempting a silent refresh so an existing session survives a page reload (see Section 3).
* **HTTP fetch backend** (`withFetch()`) with three functional interceptors registered in order: `authInterceptor`, `loadingInterceptor`, `errorInterceptor`.

### Two Surfaces

The app is split into two distinct surfaces, each with its own shell component rendered through a child `<router-outlet>`:

|Surface|Mount path|Shell component|Auth|
|-------|----------|---------------|----|
|Public booking site|`/` (root)|`PublicShellComponent`|None|
|Internal operations system|`/sumicare/app`|`InternalShellComponent`|`authGuard`|

The auth screens (`/sumicare/login`, `/sumicare/verify`, `/sumicare/reset-password`, `/sumicare/invite`) and the payment authorization screen (`/pay/authorize`) sit outside both shells as standalone routed components.

`PublicShellComponent` (`features/public/public-shell.component.ts`) renders the public navigation/footer, calls branding load on the public path, resets the menu and scroll position on each `NavigationEnd`, and hosts the global `ToastHostComponent`.

`InternalShellComponent` (`features/internal/internal-shell.component.ts`) renders the role-filtered sidebar, starts realtime + idle services, and hosts the notification toast and global toast host. Its navigation groups (`Overview`, `Operations`, `Finance`, `Configure`, `Administration`) are filtered per role via a `computed` signal (see Section 5).

---

## 2. Routing Architecture

The full route tree is defined in `app.routes.ts`. Role constants used by the guards are declared there:

````typescript
const STAFF_ROLES = ['RECEPTIONIST', 'MANAGER', 'ADMIN', 'SUPERADMIN'];
const MANAGER_PLUS = ['MANAGER', 'ADMIN', 'SUPERADMIN'];
const ADMIN_PLUS = ['ADMIN', 'SUPERADMIN'];
````

`STAFF_ROLES` is just the guard constant naming the four internal roles (`RECEPTIONIST`, `MANAGER`, `ADMIN`, `SUPERADMIN`) — it does **not** include a `STAFF` role; there is no `STAFF` role in the system. It is the "any internal operator" tier (RECEPTIONIST and above).

### Public routes (children of `PublicShellComponent`)

|Path|Component|Guard|Role|
|----|---------|-----|----|
|`/`|`LandingComponent`|none|—|
|`/about`|`AboutComponent`|none|—|
|`/packages`|`PackagesComponent`|none|—|
|`/services`|`ServicesComponent`|none|—|
|`/recommendation`|`RecommendationComponent`|none|—|
|`/book`|`BookComponent`|none|—|
|`/visit`|`VisitComponent`|none|—|
|`/feedback`|`FeedbackComponent`|none|—|
|`/contact`|`ContactComponent`|none|—|
|`/cancel`|`CancelComponent`|none|—|
|`/terms`|`TermsComponent`|none|—|

### Payment route (top-level, outside both shells)

|Path|Component|Guard|Role|
|----|---------|-----|----|
|`/pay/authorize`|`PaymongoAuthorizeComponent`|none|—|

### Auth routes (children of `/sumicare`)

|Path|Component|Guard|Role|
|----|---------|-----|----|
|`/sumicare`|redirect to `login`|none|—|
|`/sumicare/login`|`LoginComponent`|none|—|
|`/sumicare/verify`|`VerifyComponent`|none|—|
|`/sumicare/reset-password`|`ResetPasswordComponent`|none|—|
|`/sumicare/invite`|`InviteComponent`|none|—|

### Internal routes (children of `InternalShellComponent` under `/sumicare/app`)

The `app` parent route applies `canActivate: [authGuard]`; each child additionally applies a `roleGuard`. The default child redirects to `dashboard`.

|Path|Component|Guard|Allowed roles|
|----|---------|-----|-------------|
|`dashboard`|`DashboardComponent`|`authGuard` (inherited)|any authenticated|
|`bookings`|`BookingsComponent`|`roleGuard(STAFF_ROLES)`|RECEPTIONIST+|
|`reception`|`ReceptionComponent`|`roleGuard(STAFF_ROLES)`|RECEPTIONIST+|
|`lineup`|`LineupComponent`|`roleGuard(STAFF_ROLES)`|RECEPTIONIST+|
|`cashier`|`CashierComponent`|`roleGuard(STAFF_ROLES)`|RECEPTIONIST+|
|`pos`|redirect to `cashier`|—|—|
|`orders`|`OrdersListComponent`|`roleGuard(STAFF_ROLES)`|RECEPTIONIST+|
|`orders/:id`|`OrderDetailComponent`|`roleGuard(STAFF_ROLES)`|RECEPTIONIST+|
|`messages`|`MessagesComponent`|`roleGuard(STAFF_ROLES)`|RECEPTIONIST+|
|`registered-clients`|`RegisteredClientsComponent`|`roleGuard(STAFF_ROLES)`|RECEPTIONIST+|
|`treatment-slips`|`TreatmentSlipsComponent`|`roleGuard(STAFF_ROLES)`|RECEPTIONIST+|
|`treatment-slips/:id`|`TreatmentSlipDetailComponent`|`roleGuard(STAFF_ROLES)`|RECEPTIONIST+|
|`reports`|`ReportsComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|
|`attendance`|`AttendanceComponent`|`roleGuard(STAFF_ROLES)`|RECEPTIONIST+|
|`ledger`|`LedgerComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|
|`analytics`|`AnalyticsComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|
|`settings`|`SettingsComponent`|`authGuard` (inherited)|any authenticated|
|`users`|`UsersComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|
|`audit`|`AuditComponent`|`roleGuard(ADMIN_PLUS)`|ADMIN+|
|`branding`|`BrandingComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|
|`content`|`ContentComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|
|`admin/therapists`|`TherapistsAdminComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|
|`admin/shifts`|`ShiftsAdminComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|
|`admin/rooms`|`RoomsAdminComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|
|`admin/services`|`ServicesAdminComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|
|`admin/packages`|`PackagesAdminComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|
|`admin/vouchers`|`VouchersAdminComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|
|`admin/feedback`|`FeedbackAdminComponent`|`roleGuard(MANAGER_PLUS)`|MANAGER+|

A trailing wildcard route `{ path: '**', redirectTo: '' }` sends unknown paths to the public landing page.

### Guards

Both guards live in `core/auth/auth.guard.ts`.

````typescript
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  if (auth.isAuthenticated()) return true;
  router.navigate(['/sumicare/login']);
  return false;
};

export const roleGuard = (allowedRoles: string[]): CanActivateFn => {
  return () => {
    const auth = inject(AuthService);
    const router = inject(Router);
    if (auth.isAuthenticated() && auth.hasRole(allowedRoles)) return true;
    router.navigate(['/sumicare/login']);
    return false;
  };
};
````

`authGuard` checks only authentication; `roleGuard` is a factory that checks authentication **and** role membership. `isAuthenticated()` returns true only when a session exists and its `expiresAt` is still in the future, so an expired in-memory token fails the guard and redirects to login.

---

## 3. Authentication Flow

Authentication is implemented in `core/auth/auth.service.ts`. The defining characteristic is that the **access token is held only in memory** in a signal; it is never written to `localStorage` or `sessionStorage`. The refresh token lives in an `httpOnly` cookie managed by the server (every auth request uses `withCredentials: true`).

````typescript
export interface AuthSession {
  accessToken: string;
  role: string;
  expiresAt: number;
}

@Injectable({ providedIn: 'root' })
export class AuthService {
  private http = inject(HttpClient);
  readonly session = signal<AuthSession | null>(null);
  private refreshInFlight: Observable<TokenResponse> | null = null;
  ...
}
````

### Sequence

1. **Bootstrap.** On app start, `provideAppInitializer` invokes `bootstrapSession()`, which POSTs to `/api/auth/refresh` with credentials. On success the returned token is applied to the `session` signal; on failure the error is swallowed (`catchError(() => of(null))`) and the app loads unauthenticated.
1. **Login submit.** `LoginComponent.submit()` calls `auth.login(username, password)`. The service POSTs to `/api/auth/login`. The response may indicate `mfaRequired` with a `challengeId` and masked email — in that case the component switches to the MFA code step rather than navigating. When a token is returned (directly or after `verifyMfa`), `applyToken` populates the session and the component navigates to `/sumicare/app/dashboard`.
1. **Token application.** `applyToken` computes `expiresAt = Date.now() + expiresIn * 1000` and stores `{ accessToken, role, expiresAt }` in the signal.
1. **Refresh on 401** (see Section 4 for the interceptor). `refresh()` is de-duplicated: a single in-flight refresh is shared across concurrent callers via `shareReplay` and cleared on completion.

````typescript
refresh(): Observable<TokenResponse> {
  if (this.refreshInFlight) {
    return this.refreshInFlight;
  }
  this.refreshInFlight = this.http
    .post<TokenResponse>(`${environment.apiBaseUrl}/api/auth/refresh`, {}, { withCredentials: true })
    .pipe(
      tap((response) => this.applyToken(response)),
      finalize(() => { this.refreshInFlight = null; }),
      shareReplay({ bufferSize: 1, refCount: true })
    );
  return this.refreshInFlight;
}
````

5. **Logout.** `logout()` POSTs to `/api/auth/logout` (server clears the cookie) and sets the session signal to `null`. `InternalShellComponent.signOut()` confirms via a dialog, then navigates to `/sumicare/login` on both success and error.
5. **Organization scoping.** `organizationId()` decodes the JWT payload client-side (base64url) and returns the `org` claim, used to scope realtime topic subscriptions. It returns `null` if no token or the payload cannot be parsed.

An `IdleTimeoutService` (`core/auth/idle-timeout.service.ts`) is started by the internal shell. It tracks user-activity events, performs periodic background refreshes within a window before expiry, and on idle (one hour, per `IDLE_MS`) routes the user out with a reason flag the login screen reads as an idle banner.

---

## 4. HTTP Interceptors

Three functional interceptors are registered, applied in array order: `authInterceptor` → `loadingInterceptor` → `errorInterceptor`.

### `authInterceptor` (`core/auth/auth.interceptor.ts`)

Attaches the Bearer token to any request whose URL contains `/api/`, then handles 401 by refreshing and retrying once.

````typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  const router = inject(Router);
  const session = auth.session();
  const isApi = req.url.includes('/api/');
  const authedRequest = session && isApi
    ? req.clone({ setHeaders: { Authorization: `Bearer ${session.accessToken}` } })
    : req;
  return next(authedRequest).pipe(
    catchError((error) => {
      if (error.status === 401 && session && !req.url.endsWith('/api/auth/refresh')) {
        return auth.refresh().pipe(
          switchMap(() => {
            const refreshed = auth.session();
            const retried = refreshed
              ? req.clone({ setHeaders: { Authorization: `Bearer ${refreshed.accessToken}` } })
              : req;
            return next(retried);
          }),
          catchError(() => {
            router.navigate(['/sumicare/login']);
            return throwError(() => error);
          })
        );
      }
      return throwError(() => error);
    })
  );
};
````

Notes:

* The 401 branch is guarded so the refresh endpoint itself does not recurse (`!req.url.endsWith('/api/auth/refresh')`) and so unauthenticated requests are not retried (`session` must exist).
* On retry, the freshly applied token is read from the signal and a new `Authorization` header is set on the cloned request.
* If the refresh fails, the user is redirected to `/sumicare/login` and the original error is rethrown.

### `loadingInterceptor` (`core/loading/loading.interceptor.ts`)

Increments a global loading indicator on every request and decrements it on completion via `finalize`, backed by `LoadingService`.

### `errorInterceptor` (`core/errors/error.interceptor.ts`)

Surfaces a user-facing toast for transport-level and server-side failures, then rethrows so component-level handlers still run.

````typescript
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const toast = inject(ToastService);
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 0) {
        toast.error('Cannot reach the server. Check your connection and try again.');
      } else if (error.status === 408 || error.status === 504) {
        toast.error('The request timed out. Please try again.');
      } else if (error.status >= 500) {
        toast.error('Something went wrong on our end. Please try again in a moment.');
      }
      return throwError(() => error);
    })
  );
};
````

Status `0` (network/CORS), `408`/`504` (timeout), and `5xx` produce toasts. `4xx` errors other than the timeouts are passed through silently for the component to handle.

---

## 5. Signal-Based State Management

There is no NgRx store. Shared and component state is held in Angular signals, with `computed` for derived values and `effect`/lifecycle hooks for side effects. This keeps state colocated in the relevant root-provided service and makes change detection inexpensive (see Section 9).

Representative examples:

* **Auth session** — `AuthService.session = signal<AuthSession | null>(null)`. Components read it directly; the internal shell aliases it as `session = this.auth.session` and derives the visible navigation from it:

````typescript
visibleGroups = computed(() => {
  const role = this.session()?.role;
  if (!role) return [];
  return this.groups
    .map(g => ({ ...g, items: g.items.filter(i => i.roles.includes(role)) }))
    .filter(g => g.items.length > 0);
});
````

* **Branding** — `BrandingService.branding = signal<OrganizationBranding | null>(null)`, set when theme data loads and cleared on reset (Section 8). Shell templates bind to it for the display name, logo, and contact details.

* **Notification feed counters** — `NotificationFeedService` exposes one signal per channel:

````typescript
readonly unreadBookings = signal(0);
readonly unreadOrders = signal(0);
readonly unreadMessages = signal(0);
readonly unreadFeedback = signal(0);
readonly events$ = new Subject<NotificationEvent>();
````

Incoming realtime messages increment the relevant counter (`counter.set(counter() + 1)`); navigating to the corresponding route resets it to `0` via `markRead`. The sidebar reads these through `unreadFor(route)` to render unread badges.

Local component state (busy flags, error strings, form/modal toggles) is likewise expressed as signals, e.g. in `LoginComponent`: `busy = signal(false)`, `error = signal<string | null>(null)`, `mfaRequired = signal(false)`.

---

## 6. WebSocket / STOMP Client

Realtime updates use STOMP over WebSocket via `@stomp/rx-stomp`, wrapped in `core/realtime/stomp.service.ts`.

````typescript
@Injectable({ providedIn: 'root' })
export class StompService {
  private rx: RxStomp | null = null;

  connect(token: string | null): void {
    if (this.rx?.active) return;
    this.rx = new RxStomp();
    this.rx.configure({
      brokerURL: environment.wsUrl.startsWith('http')
        ? environment.wsUrl.replace('http', 'ws')
        : environment.wsUrl,
      connectHeaders: token ? { Authorization: `Bearer ${token}` } : {},
      reconnectDelay: 5000
    });
    this.rx.activate();
  }

  watch<T>(destination: string): Observable<T> {
    if (!this.rx) throw new Error('Stomp not connected');
    return this.rx.watch({ destination }).pipe(map((message) => JSON.parse(message.body) as T));
  }

  disconnect(): void {
    this.rx?.deactivate();
    this.rx = null;
  }
}
````

Configuration facts:

* **Broker URL** comes from `environment.wsUrl` (default `/ws`); if it begins with `http` it is rewritten to `ws`.
* **Connect headers** carry the access token as `Authorization: Bearer ...`.
* **Reconnect delay** is `5000` ms.
* `watch<T>(destination)` returns an `Observable<T>` that JSON-parses each frame body into the requested type.

### Lifecycle

`InternalShellComponent.ngOnInit` connects the socket and starts the feed; `ngOnDestroy` tears both down:

````typescript
this.stomp.connect(this.session()?.accessToken ?? null);
this.feed.start(this.auth.organizationId());
...
ngOnDestroy(): void {
  this.idleTimeout.stop();
  this.feed.stop();
  this.stomp.disconnect();
}
````

### Topics and subscribers

All topics are organization-scoped by appending the `organizationId` (from the JWT `org` claim). `NotificationFeedService.start(orgId)` subscribes to the four notification channels and seeds two unread counts over HTTP:

````typescript
this.stomp.watch<{ event: string; summary: string; at: string }>('/topic/' + key + '/' + organizationId)
````

|Subscriber|Topic(s)|Effect|
|----------|--------|------|
|`NotificationFeedService`|`/topic/bookings/{orgId}`, `/topic/orders/{orgId}`, `/topic/messages/{orgId}`, `/topic/feedback/{orgId}`|Increment per-channel unread signal; emit on `events$`|
|`DashboardComponent`|`/topic/bookings/{orgId}`, `/topic/orders/{orgId}`|Debounced reload (300 ms) of recent reservations|
|`BookingsComponent`|`/topic/bookings/{orgId}`, `/topic/room-updates/{orgId}`, `/topic/orders/{orgId}`, `/topic/decking-updates/{orgId}`|Refresh bookings, room map, orders, and lineup views|
|`ReceptionComponent` (room map)|`/topic/room-updates/{orgId}`|Reload room/bed occupancy|
|`RoomsAdminComponent`|`/topic/room-updates/{orgId}`|Reload occupancy|

### Deserialization and cleanup

Frames are deserialized in `StompService.watch` (`JSON.parse(message.body) as T`). The notification feed maps the parsed payload into a `NotificationEvent` and updates the signal counters. Subscriptions are cleaned up two ways depending on the consumer:

* **Manual `Subscription` handles** stored on the component and unsubscribed in `ngOnDestroy` (e.g. `ReceptionComponent.roomSubscription?.unsubscribe()`, `BookingsComponent`'s named subscriptions, the `feedSubscriptions` array in `DashboardComponent`). The feed service tears down its own list in `stop()`.
* **`takeUntilDestroyed(this.destroyRef)`** for router-event streams, e.g. the internal shell's `NavigationEnd` subscription that bumps the route-fade token and clears unread counts.

Each `watch` call site wraps the subscribe in a `try/catch` (or per-subscription error callbacks) so a not-yet-connected socket or a malformed frame does not crash the view.

---

## 7. Shared Libraries

Two NX libraries are exposed through TypeScript path aliases in `tsconfig.base.json`:

````json
"paths": {
  "@sumicare/shared-types": ["libs/shared-types/src/index.ts"],
  "@sumicare/ui": ["libs/ui/src/index.ts"]
}
````

### `@sumicare/shared-types`

A barrel of TypeScript-only DTOs and enums mirroring the backend contract, re-exported from `libs/shared-types/src/index.ts`:

|File|Notable exports|
|----|---------------|
|`auth.types.ts`|`Role` union (`'SUPERADMIN' \| 'ADMIN' \| 'MANAGER' \| 'RECEPTIONIST'`), `LoginRequest`, `TokenResponse`|
|`user.types.ts`|`UserResponse`, `CreateUserRequest`, `UpdateUserRequest`|
|`organization.types.ts`|`OrganizationBranding`, `UpdateBrandingRequest`|
|`therapist.types.ts`|`Gender`, `TherapistResponse`, `CreateTherapistRequest`|
|`booking.types.ts`|`ReservationType`, `BookingStatus`, `SessionStatus`, `CreateBookingRequest`, `BookingResponse`, `StartSessionRequest`, `SessionResponse`|
|`pos.types.ts`|`PaymentMethod`, `ProcessPaymentRequest`, `PaymentResponse`|
|`recommendation.types.ts`|`QuizAnswer`, `QuizSubmissionRequest`, `ServiceSummary`, `RecommendationResponse`|
|`report.types.ts`|`ReportSummary`|
|`decking.types.ts`|`DeckingFlag`, `DeckingEntry`|

### `@sumicare/ui`

Standalone presentational primitives re-exported from `libs/ui/src/index.ts`:

|Export|Selector|Purpose|
|------|--------|-------|
|`BadgeComponent`|`sumi-badge`|Pill badge with `variant` input (`neutral`/`primary`/`success`/`warn`/`danger`) mapped to Tailwind tone classes|
|`CardComponent`|`sumi-card`|Card container with optional `title`/`subtitle` and projected content|
|`EmptyStateComponent`|`sumi-empty-state`|Centered empty placeholder with `title`/`message` inputs|
|`DebounceClickDirective`|`[debounceClick]`|Debounced click emitter (default 500 ms) that suppresses double-submits|

These libraries are consumed by importing from the aliases, e.g. `import { BookingResponse } from '@sumicare/shared-types'`. Note: at the time of writing, no file under `apps/sumicare-web/src` imports from either alias — the web app currently declares equivalent interfaces inline and uses local shared components rather than the `libs/` packages. The libraries define the intended shared contract but are not yet wired into the web app's imports.

---

## 8. Tailwind and Theming

Styling is Tailwind utility classes in templates, with the base layer and design tokens defined in `src/styles.css`. The public theme is imported first (`@import "./styles/public-theme.css"`), followed by the three Tailwind layers.

Theme tokens are CSS custom properties declared in `:root` as fallbacks:

````css
:root {
    --sumi-primary: #eda0d8;
    --sumi-secondary: #e086c7;
    --sumi-accent: #b0d35d;
    --sumi-font-display: 'Anton', 'IBM Plex Sans', 'Inter', ui-sans-serif, system-ui, sans-serif;
    --sumi-font-body: 'Inter', ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
    --sumi-app-font-display: 'IBM Plex Sans', 'Inter', ui-sans-serif, system-ui, sans-serif;
    --sumi-app-font-body: 'IBM Plex Sans', 'Inter', ui-sans-serif, system-ui, sans-serif;
}
````

### Runtime theming via `BrandingService`

`core/branding/branding.service.ts` overrides these variables at runtime on `document.documentElement` so a single build can be re-skinned per organization without recompilation.

* **`loadPublicBranding(slug = environment.defaultOrganizationSlug)`** resets to defaults, then GETs `/api/public/branding/{slug}` and applies the result. Used on the public surface. Default slug is `lasema`.
* **`loadCurrentBranding()`** resets to defaults, then GETs `/api/organization/branding` for the authenticated org. Called by `InternalShellComponent.ngOnInit`.
* **`resetToDefault()`** clears the `branding` signal and removes the overriding inline properties so the `:root` fallbacks apply again.

`applyTheme` writes the variables that the service controls:

````typescript
if (branding.primaryColor) root.style.setProperty('--sumi-primary', branding.primaryColor);
if (branding.secondaryColor) root.style.setProperty('--sumi-secondary', branding.secondaryColor);
if (branding.accentColor) root.style.setProperty('--sumi-accent', branding.accentColor);
if (branding.fontFamily) {
  const family = branding.fontFamily.includes(',')
    ? branding.fontFamily
    : `'${branding.fontFamily}', 'IBM Plex Sans', 'Inter', sans-serif`;
  root.style.setProperty('--sumi-app-font-display', family);
  root.style.setProperty('--sumi-app-font-body', family);
} else {
  root.style.removeProperty('--sumi-app-font-display');
  root.style.removeProperty('--sumi-app-font-body');
}
if (branding.loginBackgroundUrl) {
  root.style.setProperty('--sumi-login-bg', `url('${branding.loginBackgroundUrl}')`);
} else {
  root.style.removeProperty('--sumi-login-bg');
}
this.applyFavicon(branding.faviconUrl);
````

Runtime-controlled variables: `--sumi-primary` (default `#eda0d8`), `--sumi-secondary` (`#e086c7`), `--sumi-accent` (`#b0d35d`), `--sumi-app-font-display`, `--sumi-app-font-body`, and `--sumi-login-bg`. A custom `fontFamily` without a comma is wrapped with a fallback stack; the login background is set as a CSS `url(...)`. `applyFavicon` swaps the `<link rel="icon">` href when a favicon URL is provided. When a load fails, the service falls back to defaults via `resetToDefault()`.

`styles.css` also defines the keyframe-based entrance animations (`sumi-fade-in`, `sumi-slide-up`, `anim-pop`, etc.), the `sumi-spinner` loading indicator, active-press affordances on buttons, a `prefers-reduced-motion` block that disables animation, and an extensive `@media print` block scoped to `.print-area-wrapper` (used for treatment slips and reports).

---

## 9. Change Detection

Routed and shell components declare `changeDetection: ChangeDetectionStrategy.OnPush` (e.g. `PublicShellComponent`, `InternalShellComponent`, `LoginComponent`). The application also enables zone event coalescing via `provideZoneChangeDetection({ eventCoalescing: true })`.

`OnPush` is safe here because the app stores shared and local state in signals. Reading a signal inside a template registers that template as a consumer, so when the signal's value changes Angular marks the component for check automatically — there is no reliance on mutable object identity or manual `markForCheck` for signal-driven state. Combined with `computed` derivations (such as `visibleGroups`), this gives fine-grained, push-based updates while avoiding the cost of default change detection across the large internal shell tree.

---

## 10. Error Handling

Error handling operates at two levels.

**Global (transport/server).** The `errorInterceptor` (Section 4) centralizes user feedback for failures that are not specific to one screen: network unreachable (`status === 0`), timeouts (`408`/`504`), and server faults (`>= 500`) each raise a toast through `ToastService`, rendered by the global `ToastHostComponent` mounted in both shells. The interceptor rethrows so per-call logic still runs.

**Component-level.** Individual components own their error state, typically as a signal, and degrade gracefully on failure rather than surfacing raw errors. Patterns observed:

* `LoginComponent` clears `busy` and sets an `error` signal in the `error` callback of `login()`.
* Data-loading components fall back to an empty collection on error, e.g. `ReceptionComponent.reload()` does `error: () => this.rooms.set([])`, and `DashboardComponent.loadRecentReservations()` does `error: () => this.recentReservations.set([])`.
* Realtime subscribers swallow stream errors (`error: () => undefined`) and wrap `watch` in `try/catch` so a socket problem never breaks the view.
* `NotificationFeedService.seedUnreadCounts()` ignores failures of the unread-count endpoints (`error: () => undefined`), leaving counters at zero.
* Empty result sets are communicated to the user through the shared `EmptyStateComponent` rather than error text.

The 401-driven refresh-and-retry path (Section 4) is the other half of error handling: it transparently recovers expired access tokens and only redirects to `/sumicare/login` when refresh itself fails.
