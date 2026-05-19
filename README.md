# Enterprise Authentication & Authorization Architecture
*A production-ready, highly scalable authentication ecosystem built with Angular (Signals), NestJS, and PostgreSQL.*

## Architectural Overview
This repository serves as a technical showcase of an enterprise-grade authentication module. Designed for a multitenant SaaS environment (B2B), it implements strict Role-Based Access Control (RBAC), stateless session management, and robust security protocols.

**Core Stack:**
* **Client:** Angular 21 (Strictly Typed, Reactive Signals)
* **Gateway/API:** NestJS (TypeScript, Decorator-driven)
* **Database:** PostgreSQL with Prisma ORM
* **Security:** JWT, OAuth (Google), bcrypt

---

## Key Engineering Decisions & Patterns

### 1. Fine-Grained Role Hierarchy (RBAC)
Instead of simple booleans, the system implements a hierarchical numeric role system. This ensures that a `SUPER_USER` (Level 100) automatically inherits permissions from a `MANAGER` (Level 90) or `SELLER` (Level 60). 
Guards in both NestJS and Angular dynamically evaluate these hierarchies to prevent unauthorized component rendering and API execution.

### 2. "Zero-DB" Password Recovery Strategy
To minimize database writes and avoid cluttering schemas with temporary tokens, the "Forgot Password" flow relies entirely on **Stateless JWTs**.
* The backend generates a highly restricted JWT with a `purpose: 'PASSWORD_RESET'` payload and a strict 15-minute expiration.
* The Angular client extracts this token from the URL and securely dispatches it alongside the new payload.
* This prevents database enumeration attacks and ensures token integrity without adding temporary columns to the User table.

### 3. Reactive UI with Angular Signals
The frontend abandons traditional `RxJS` BehaviorSubjects for state management in favor of **Angular Signals**. 
* **Signal-Driven Forms:** Inputs are tightly coupled with pure computed validation functions (e.g., cross-field password matching validators).
* **DOM Efficiency:** State changes (like toggling UI elements based on the `PENDING_APPROVAL` or `SUSPENDED` account status) trigger pinpoint DOM updates without relying on Zone.js overhead.

### 4. Silent Token Renewal (HttpInterceptor)
To ensure zero friction for active users, the Angular client implements an advanced HTTP Interceptor using a queuing mechanism:
1. If a 401 error occurs during an active session, the interceptor pauses all outgoing HTTP requests.
2. It silently triggers a token renewal request to the `/check-status` NestJS endpoint.
3. Upon receiving a fresh JWT, it releases the queued requests and applies the new Authorization header, preventing data loss during long form submissions.

---

## 🔬 Code Snippets Showcase

### The Silent Refresh Interceptor (Angular)
Handling concurrency during token expiration:
```typescript
// Queueing mechanism to prevent multiple refresh calls simultaneously
if (!isRefreshing) {
    isRefreshing = true;
    refreshTokenSubject.next(null);

    return authService.checkAuthStatus().pipe(
        switchMap((isAuthenticated) => {
            isRefreshing = false;
            if (isAuthenticated) {
                const newToken = authService.token();
                refreshTokenSubject.next(newToken);
                // Retry the original failed request
                return next(request.clone({ setHeaders: { Authorization: `Bearer ${newToken}` } }));
            }
            authService.logout();
            return throwError(() => error);
        })
    );
} else {
    // Put overlapping requests on hold until the new token arrives
    return refreshTokenSubject.pipe(
        filter(t => t !== null),
        take(1),
        switchMap(jwt => next(request.clone({ setHeaders: { Authorization: `Bearer ${jwt}` } })))
    );
}
