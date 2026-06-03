




















# PMS Project — Professional Engineering Audit

**Date:** 2026-06-03 | **Auditor:** Senior Architect Review | **Stack:** Spring Boot 3.4.9 + Next.js 15 + MySQL + ActiveMQ Artemis

---

# 1. Executive Summary

| Dimension | Score | Grade |
|---|---|---|
| **Overall Project Quality** | 74/100 | B |
| **Architecture** | 82/100 | B+ |
| **Security** | 61/100 | C+ |
| **Scalability** | 70/100 | B- |
| **Maintainability** | 78/100 | B+ |
| **Frontend Quality** | 72/100 | B |
| **Backend Quality** | 79/100 | B+ |

**Summary Statement:** This is a well-structured enterprise application with strong architectural consistency across 17 domain modules. The developer demonstrates genuine understanding of layered architecture, event-driven design, and OpenAPI-first development. The project is meaningfully above average for a solo-built full stack application. However, several critical data integrity gaps (missing FK constraints, broken notification recipient tracking), a near-complete absence of automated tests, and disabled CSRF protection create real production risk that must be resolved before deployment.

---

# 2. Architecture Review

## Current Architecture

```
┌────────────────────────────────────────────────────────────┐
│                        FRONTEND                             │
│  Next.js 15 / React 19 / Redux Toolkit / TypeScript        │
│  Axios API Layer → REST + STOMP WebSocket                  │
└──────────────────────┬─────────────────────────────────────┘
                       │ HTTP + WebSocket
┌──────────────────────▼─────────────────────────────────────┐
│                        BACKEND                              │
│  Spring Boot 3.4.9 — Modular Maven Multi-Module            │
│                                                             │
│  library/                                                   │
│    security-adapter (JWT, BCrypt, CORS, CSRF)               │
│    session-manager  (RequestContext per-thread)             │
│    sql-db-adapter   (AbstractQueryBuilder, JPA)             │
│    rest-adapter     (GlobalExceptionHandler, ErrorVTO)      │
│    mq-adapter       (ActiveMQ event publishing)             │
│    swagger          (OpenAPI generator plugin)              │
│    common           (BusinessException, InternalServerEx)   │
│                                                             │
│  modules/ (17 domains)                                      │
│    Each: entity → repository → service → controller        │
│           mapper → event → enum → liquibase                 │
└──────────────────────┬─────────────────────────────────────┘
                       │ JDBC / JPA
┌──────────────────────▼─────────────────────────────────────┐
│  MySQL 8 + Liquibase 4.27 migrations                        │
└────────────────────────────────────────────────────────────┘
                       │ JMS
┌──────────────────────▼─────────────────────────────────────┐
│  ActiveMQ Artemis — domain event bus                        │
└────────────────────────────────────────────────────────────┘
```

## Strengths

1. **OpenAPI-first design** enforced consistently — generated controller interfaces prevent implementation drift from the contract.
2. **AbstractQueryBuilder** in `library/sql-db-adapter` is a clean, reusable pattern that prevents repeated HQL construction across all 17 modules. Every module correctly inherits it.
3. **Library isolation** — cross-cutting concerns (JWT, CORS, error handling, logging, messaging) are extracted into library modules rather than scattered across business modules.
4. **Event-driven notifications** are consistently wired across domains: each module has its own `Events` enum, `EventData` class, and `@JmsListener` notification handler.
5. **Liquibase** manages schema evolution modularly per domain, with a master changelog per module.
6. **BCrypt strength=12** — correct and non-trivial password hashing.
7. **Role-based method security** via `@PreAuthorize`/`@Secured` at controller level.
8. **Redux Toolkit** with consistent `createAsyncThunk` + `pending/fulfilled/rejected` pattern across 18 slices.

## Weaknesses

1. **Denormalized FK fields without DB constraints** in 2 major modules (Ticket, Inspection). This is the largest structural gap.
2. **Notification recipient tracking is broken by design** — the `Notification` entity has no `userId` FK.
3. **Zero unit/integration test coverage** across 39+ service classes and 50+ controller endpoints.
4. **CSRF disabled** in `SecurityConfiguration` with no compensating control documented.
5. **Frontend Inspection module** violates the established Redux slice pattern.

## Coupling / Cohesion

- **Cohesion:** High per module. Each domain owns its full vertical slice.
- **Coupling:** Low between modules at the code level. However, notification and audit modules are coupled to all other domains through the event bus, which is appropriate. The `RequestContext` (session-manager) creates an implicit dependency that every service relies on for `userId` injection.
- **Concern:** `UserManagementServiceImpl` in the admin module has TODOs that depend on the file/storage module — this is a cross-module dependency that is currently incomplete.

---

# 3. Backend Audit

## APIs

**Strengths:**
- Every module has a generated `*Controller` interface from OpenAPI spec and a separate `*ControllerImpl` that delegates to the service. This is the correct pattern.
- Endpoints follow REST conventions consistently (`/api/{resource}`, proper HTTP verbs).

**Issues:**
- `WebSocketChatController` does **not** use a generated interface. It is the only controller that bypasses the Swagger-first pattern.
  - **File:** `modules/chat/chat-mgt/.../controller/implementation/WebSocketChatControllerImpl.java`
  - **Fix:** Define a WebSocket endpoint contract or at minimum add `@PreAuthorize` at the method level since WebSocket frames bypass Spring Security's HTTP filter chain.
- The `/auth/token/refresh` path is listed in the frontend's no-token-injection list but it's unclear whether the backend correctly validates the refresh token without the access token — verify `AuthController` refresh endpoint.

## Services

**Strengths:**
- Consistent `@Transactional` on writes and `@Transactional(readOnly=true)` on reads across all service implementations.
- Business validation raises typed `BusinessException` with domain-specific error enums.
- `RequestContext` injection for current user ID is consistent.

**Issues:**
1. **No `@Transactional(rollbackFor = Exception.class)`** — Spring's default only rolls back on unchecked exceptions. If a `BusinessException` (which is a checked exception) is thrown partway through a multi-step operation, the transaction will **not** roll back.
   - **Fix:** Add `rollbackFor = Exception.class` to all write `@Transactional` annotations or make `BusinessException` extend `RuntimeException`.
2. **`UserManagementServiceImpl`** has 3 unfilled TODOs:
   - `"TODO: Add storage quota and stats from storage module"`
   - `"TODO: Set storage quota if provided"`
   - `"TODO: Update storage quota if provided"`
   - **Impact:** Admin cannot set file storage limits per user/property. Feature is silently missing.

## Validation

**Strengths:**
- `@Valid` annotations on `@RequestBody` parameters.
- `MethodArgumentNotValidException` handled in `RESTGlobalExceptionHandler`.

**Issues:**
1. **No `@Validated` at service layer** — validation only fires at the HTTP boundary. Internal service-to-service calls skip validation entirely.
   - **Fix:** Add `@Validated` to service interfaces where appropriate for critical business rules (e.g., `LeaseService.create()`).
2. **No custom constraint annotations** for domain-specific rules (e.g., "lease start date must be before end date", "inspection date must be in the future"). These are likely validated procedurally inside the service but are invisible to the API consumer.

## JPA

**Critical Issues:**

1. **`Ticket` entity — denormalized FKs with no constraints:**
   - `buildingId` (Long), `assignedToId` (Long), `createdById` (Long) are stored as plain fields with no `@ManyToOne` mapping.
   - **File:** `modules/maintenance/maintenance-mgt/.../entity/Ticket.java`
   - **Why it matters:** A deleted building or user leaves orphaned tickets with no cascade or referential integrity. Queries filter by `buildingId` with no index.
   - **Fix:** Add `@ManyToOne(fetch=LAZY) @JoinColumn(name="building_id") Building building;` or at minimum add a Liquibase `<addForeignKeyConstraint>` + `<createIndex>`.

2. **`Inspection` entity — same pattern as Ticket:**
   - `buildingId`, `inspectorId` are denormalized.
   - **File:** `modules/inspection/inspection-mgt/.../entity/Inspection.java`

3. **`Inspection.itemsJson` stored as TEXT column:**
   - Inspection items serialized as JSON blob in a `columnDefinition=TEXT` field.
   - **Why it matters:** No querying, filtering, or indexing on individual item fields. Defeats the relational model.
   - **Current state is acceptable** if `InspectionItem` entity is used for individual querying and `itemsJson` is a denormalized read-optimization. Verify this is the intent.

4. **`Invoice.status` is `String`, not `@Enumerated`:**
   - **File:** `modules/billing/billing-mgt/.../entity/Invoice.java`
   - **Why it matters:** Any string can be stored; constraint is only at application layer.
   - **Fix:** Create `InvoiceStatus` enum, apply `@Enumerated(EnumType.STRING)`.

5. **Fetch type risk on `ChatMessage.room` and `ChatMessage.sender`:**
   - Both are `@ManyToOne(fetch=LAZY)` — correct. But if the mapper (`ChatMgtMapper`) accesses both in a non-transactional context, you will get `LazyInitializationException` at runtime.
   - **Fix:** Ensure all mapper calls that access lazy associations happen within a service `@Transactional` boundary.

## Transactions

- **Issue:** `BusinessException` is checked. Spring default `@Transactional` only rolls back on `RuntimeException`. A `BusinessException` thrown mid-transaction **will commit partial state**.
  - **Example scenario:** `LeaseService.create()` creates the lease entity, then fires a notification event, then throws `BusinessException` for a validation failure. The lease is committed but the notification is also sent.
  - **Fix:** Extend `BusinessException` from `RuntimeException`, or annotate all write transactions with `@Transactional(rollbackFor = Exception.class)`.

## Liquibase

**Strengths:**
- Modular per-domain master changelogs.
- Consistent use of `changeset` IDs with date-based naming convention.
- FKs and composite PKs correctly defined for core modules (user, property, lease).

**Issues:**

1. **No `<rollback>` elements in any changeset.** Every changeset must support rollback for production deployments.
   - **Fix:** Add `<rollback><dropTable tableName="..."/></rollback>` for `CREATE TABLE` changesets. For `ALTER TABLE` changes, add the inverse `<dropColumn>` or `<dropForeignKeyConstraint>`.

2. **`changeset-2026-05-31-fix-enum-case.xml` for inspection and vendor** — this was a correction changeset for enum case mismatches. This indicates that the DB schema and Java enums were out of sync, signaling a gap in the dev/test workflow.

3. **Missing indexes** across multiple modules:
   - `ticket.building_id` — no index, despite `getTicketsByBuilding()` filtering by it.
   - `inspection.building_id` — same.
   - `chat_message.room_id` — critical for message history pagination.
   - `rental_payment.tenant_id` — `getMyPayments()` filters by this.
   - **Fix:** Add `<createIndex>` changesets for all FK columns and any column used in `WHERE` clauses.

4. **Missing `NOT NULL` constraints** on some fields that are semantically required (e.g., `Ticket.buildingId` is Long but may be nullable in the schema without a constraint).

## MapStruct

**Strengths:**
- Consistent `@Mapper(componentModel = "spring", builder = @Builder(disableBuilder = true))` across all 15 mappers.
- Password encryption injected directly into `UserMgtMapper` via Spring-managed component — correct approach.
- `RequestContext` injection for audit fields (`createdById`, `lastModifiedById`).

**Issues:**
1. **Mappers injecting service dependencies** — when a mapper injects a Spring service (like `SecurityService` for password encoding), it becomes a Spring component with dependencies, making it harder to test in isolation. **Current approach is consistent with the project style — acceptable.**
2. **Ensure `@Mapping(target="...", ignore=true)` is used for fields that should not be mapped** (e.g., `id` during create operations). Missing it causes generated code to map `id` from the request DTO.

## ActiveMQ

**Strengths:**
- `mq-adapter` library encapsulates all JMS configuration.
- `MessagePublisher` abstraction used consistently across all services.
- Per-domain event enums with numeric IDs for routing.
- Typed `EventData` DTOs per domain event.

**Issues:**

1. **No dead-letter queue (DLQ) strategy documented or configured.** If an `@JmsListener` handler throws, the message behavior depends on broker configuration. Without explicit DLQ config, failed events may be silently dropped.
   - **Fix:** Configure `dead-letter-address` in `broker.xml` and add an explicit `@JmsListener` on the DLQ for alerting/retry.

2. **No idempotency guards on notification handlers.** If ActiveMQ redelivers an event (e.g., after consumer restart), the notification handler will insert duplicate notifications.
   - **Fix:** Add a processed-event tracking table or use message IDs for idempotency checks.

3. **WebSocket send also triggers REST save** in the chat module — this means a message may be published twice (once via WebSocket, once persisted via REST). Verify the persistence path is not duplicated.

## Security

**Critical Issues:**

1. **CSRF disabled in `SecurityConfiguration`.** Since the frontend uses cookies (`auth-token`, `auth-role`), CSRF is a real attack vector. Cookie-based auth without CSRF protection allows malicious sites to make authenticated requests on behalf of users.
   - **Fix:** Either switch to `Authorization: Bearer` header only (no cookies), or enable CSRF with the double-submit cookie pattern.

2. **Dual token storage: cookies AND localStorage:**
   - `pms-frontend/src/middleware.ts` reads from cookies. `pms-frontend/src/store/slices/authSlice.ts` stores to localStorage.
   - **Why it matters:** localStorage is XSS-accessible. Cookies with `httpOnly` are XSS-resistant but vulnerable to CSRF. Using both creates two separate attack surfaces.
   - **Fix:** Pick one. HttpOnly cookies with CSRF protection is the correct enterprise choice. Remove the localStorage path entirely.

3. **JWT secret minimum length** — verify that the HMAC-SHA256 key is at least 256 bits (32 bytes). Short secrets are brute-forceable.

4. **No rate limiting on `/auth/login`, `/auth/register`, `/auth/forgot-password`:** These are brute-force targets. Spring Security + Bucket4j or a gateway-level rate limiter is needed.

## Exception Handling

**Strengths:**
- Two-tier exception handler (`RESTGlobalExceptionHandler` + `SecurityGlobalExceptionHandler`) with correct `@Order(HIGHEST_PRECEDENCE)`.
- Typed `ErrorVTO` with bilingual `code` + `messageEn`.
- Catch-all `Exception` handler prevents stack trace leakage.

**Issues:**
1. **`BusinessException` extends `Exception` (checked).** Breaks `@Transactional` rollback — covered in the Transactions section.
2. **No logging inside the catch-all handler.** The generic `Exception` catch produces a 500 but may swallow the original exception without logging it, making production debugging very hard.
   - **Fix:** Add `log.error("Unhandled exception", e)` before returning the 500 response.
3. **`NoHandlerFoundException` returns 404** — correct. But verify `spring.web.mvc.throw-exception-if-no-handler-found=true` is configured, otherwise this handler never fires.

## Query Optimization

1. **`AbstractQueryBuilder` uses HQL string concatenation** — safe from SQL injection (uses named parameters), but HQL is inherently less optimizable than Criteria API or `@Query` with `JOIN FETCH`.
2. **No `JOIN FETCH` seen in any repository** — lazy loading is correct but means N+1 is possible if the mapper accesses lazy associations for each item in a result set list.
   - **Example:** `getTicketsByBuilding()` returns a list of `Ticket`. If the mapper accesses `ticket.comments` for each ticket, that is 1 query + N additional queries.
   - **Fix:** Where result set views include nested collections, use `JOIN FETCH` or a projection DTO that excludes the nested collection.

## Missing Indexes

- `ticket.building_id`
- `inspection.building_id`
- `chat_message.room_id`
- `rental_payment.tenant_id`
- `notification.*` (query shape unclear without userId)

## Missing Constraints

- `Ticket.buildingId` — no FK constraint
- `Ticket.assignedToId` — no FK constraint
- `Inspection.buildingId` — no FK constraint
- `Inspection.inspectorId` — no FK constraint
- `Invoice.status` — no enum constraint (plain String)

## Missing Test Coverage

**Critical finding:** Only 2 test files exist across the entire backend:
- `BootstrapTest.java` — context load test
- `ModuleIntegrationTest.java` — unknown scope

This means:
- Zero unit tests for 39+ service classes.
- Zero controller tests for 50+ endpoints.
- Zero repository tests for custom query implementations.
- Zero security tests for `@PreAuthorize` annotations.

---

# 4. Frontend Audit

## Next.js Structure

**Strengths:**
- Route groups `(auth)/`, `manager/`, `resident/`, `superintendent/` are clean.
- `middleware.ts` correctly handles role-based routing at the framework level before any page renders.
- `lib/api/`, `lib/hooks/`, `lib/utils/`, `lib/validators/`, `lib/websocket/` are well-separated concerns.

**Issues:**
1. **`app/manager/reports/page.tsx` imports mock data alongside real Redux thunks.** Some chart data may display fabricated numbers.
   - **Fix:** Remove all mock data imports; let loading/error states handle the empty case.
2. **Superintendent area is partially implemented** — only `dashboard` and `buildings` pages exist.

## State Management

**Strengths:**
- Consistent `createAsyncThunk` + tri-state (`pending/fulfilled/rejected`) pattern across all 18 slices.
- `extraReducers` pattern is uniform.

**Issues:**
1. **`manager/inspections/page.tsx` bypasses Redux entirely** — direct API calls with `useState` while every other module uses Redux slices.
   - **Fix:** Create `src/store/slices/inspectionSlice.ts` matching the pattern of `maintenanceSlice.ts`.

2. **`authSlice.ts` stores token in both Redux state AND localStorage.** Single source of truth is missing.
   - **Fix:** Store in Redux only; persist via Redux Persist if page refresh retention is needed.

3. **Chat optimistic UI + WebSocket race condition:** If the WebSocket echo arrives before the REST confirmation, the message appears twice.
   - **Fix:** Add a `localId` to optimistic messages and deduplicate on WebSocket echo.

## API Integration

**Strengths:**
- Centralized Axios client with request/response interceptors.
- Token refresh flow with 401 handling.
- `ApiError` class wrapping `ErrorVTO` for typed error handling.

**Issues:**
1. **`/auth/users/` is in the no-token-injection list** — this path pattern is too broad. Any user-related endpoint that actually requires auth will silently send unauthenticated requests.
2. **No request cancellation (AbortController)** — navigating away during a slow API call causes stale state updates on unmounted components.
   - **Fix:** Use `AbortController` in `useEffect` cleanup.
3. **`process.env.NEXT_PUBLIC_API_BASE_URL ?? '/api'` fallback** — silently fails in production if the env var is missing.
   - **Fix:** Add a runtime check that throws if the env var is absent in production builds.

## Error Handling

**Strengths:**
- `ApiError` class with `ErrorVTO` parsing.
- `rejectWithValue` in all `createAsyncThunk` handlers.
- Toast notification hook for user-facing error messages.

**Issues:**
1. **No React Error Boundary** anywhere in the component tree. An unhandled render error crashes the entire application.
   - **Fix:** Add `<ErrorBoundary>` at the layout level for each role (`manager/layout.tsx`, `resident/layout.tsx`).
2. **Redux `error` state is set but not consistently displayed** — some pages may not render the `state.error` string to the user, silently failing.

## UX Issues

1. **No pagination UI components verified** — backend returns `ResultSet { data: [], total: int }` but it is unclear if all frontend pages implement paginated loading or load all records in one call.
2. **Superintendent dashboard is sparse** — the role exists but the UI does not expose the full functionality a building superintendent would need.

## Missing Loading States

- If `isLoading` is never reset to `false` after an error (check each `rejected` reducer), the UI stays in an infinite loading state.

## Type Safety Issues

1. **`User.id` typed as `string`** but the backend returns `Long` (numeric). Comparisons like `user.id === ticket.createdById` will silently fail.
2. **Enum literals in TypeScript use kebab-case** (`'in-progress'`, `'fire-safety'`) while backend Java enums use UPPER_SNAKE_CASE (`IN_PROGRESS`, `FIRE_SAFETY`). Without a systematic mapping layer, future mismatches will recur.
   - **Fix:** Add a TypeScript enum mapping utility or ensure the backend serializes all enums to the exact literal the frontend expects (Jackson `@JsonProperty` or `@JsonNaming`).

---

# 5. Integration Audit

## Backend / Frontend Mismatches

| Module | Issue | Severity |
|---|---|---|
| **Inspection** | Frontend enum case (`'in-progress'`) vs backend enum serialization | High |
| **Vendor** | Same enum case mismatch (fixed in changeset but root cause remains) | High |
| **Staff** | Frontend `StaffMember.id` typed as `string`; backend `Long` | Medium |
| **Invoice** | Backend `status` is untyped `String`; frontend has typed `StaffInvoiceStatus` — no guarantee of alignment | Medium |
| **Chat** | Dual send paths: REST `POST /chat/rooms/{id}/messages` + WebSocket `/app/chat/{roomId}/send` — which one is authoritative for persistence? | High |
| **Notifications** | Frontend polls `GET /notifications` but backend `Notification` has no `userId` — every query returns all notifications or filters incorrectly | Critical |

## DTO Mismatches

1. **`Ticket.createdById`** — backend returns `Long`, frontend types it as `string?`. Comparisons will silently fail.
2. **`ChatMessage.isRead`** — backend stores `Boolean`, null check needed in frontend.
3. **`Notification` VTO** — without a `userId` FK, the backend response structure for notifications is ambiguous.

## Missing APIs

1. **Admin panel** — backend has `Admin`, `SystemSetting`, `FileTypePolicy` entities and services, but no frontend pages exist.
2. **Audit log viewer** — backend has `AuditLog` entity and service, but no frontend page.
3. **Superintendent staff management** — partial frontend under `superintendent/`, incomplete API wiring.
4. **Storage quota management** — admin module TODOs indicate this is unimplemented on the backend.
5. **Role management** — `RoleController` and `RoleService` exist on backend; no frontend UI.

---

# 6. Missing Features

## Unfinished Features

1. **Admin module** — `UserManagementServiceImpl` has 3 storage quota TODOs. No frontend pages.
2. **Audit log** — backend fully implemented; no UI to view or filter audit history.
3. **Superintendent portal** — only dashboard + buildings exist. Missing: ticket management, vendor view, inspection view, staff view.
4. **Role management UI** — backend service exists; no frontend screen.
5. **Storage quota enforcement** — `FileTypePolicy` and `DailyCounter` entities exist but the enforcement path is not implemented.

## Placeholder / Mock Implementations

- **`manager/reports/page.tsx`** — imports mock data alongside real API calls. Some chart data may be fabricated.
- **JWT qualifier in `RESTJWTFilterService.java`** — `TODO qualifier name should be removed` indicates an unresolved cleanup item.

## Dead Code

- The `DailyCounter` entity appears to be used for rate limiting and analytics but its actual integration path is unclear — verify it is actively written to and read from.

---

# 7. Performance Audit

## Backend Bottlenecks

1. **N+1 Query Risk in Result Set Mapping:**
   - `TicketVTO` includes `comments: TicketComment[]` and `attachments: TicketAttachment[]`.
   - If `MaintenanceMgtMapper.toVTO(Ticket)` accesses these lazy collections for a list of 100 tickets, that is 100 × 2 = 200 additional queries per page.
   - **Fix:** Use `JOIN FETCH` in the repository when fetching tickets for list views, or return a lean `TicketSummaryVTO` without comments/attachments for list endpoints.

2. **`AbstractQueryBuilder` builds HQL dynamically on every request** — no query result caching.
   - **Fix:** Add Spring `@Cacheable` on `ReportingService` methods with a short TTL.

3. **Chat message history** — `getMessages(roomId, params)` with no index on `chat_message.room_id` means full table scans for active rooms.

## Frontend Bottlenecks

1. **No request deduplication** — multiple components dispatching the same thunk simultaneously fire multiple API calls.
2. **WebSocket reconnection** (`reconnectDelay: 5000`) with no exponential backoff. Under network instability, clients hammer the server every 5 seconds.
   - **Fix:** Implement exponential backoff in `WebSocketService.connect()`.
3. **Recharts rendering** of large data sets with no data windowing or lazy loading.

## Caching Opportunities

- Report aggregations (`/reports/manager-summary`, `/reports/revenue-vs-cost`) — expensive, change infrequently. Cache with `@Cacheable` + 5-minute TTL.
- Lookup tables (`user_status`, `role`, `event_parameter`) — cache indefinitely with manual eviction.
- `getMyUnit()`, `getMyLease()` — tenant-specific data that changes rarely. Cache per-user with `@CacheEvict` on assignment/lease changes.

---

# 8. Security Audit

## JWT / Authentication

1. **CSRF disabled** (Critical) — Cookie-based auth + disabled CSRF = CSRF vulnerability.
2. **Dual token storage** (High) — localStorage is XSS-accessible.
3. **JWT secret length** — verify configured salt meets 256-bit minimum.
4. **Token refresh path** — must validate refresh token belongs to the requesting user.
5. **No token revocation mechanism** — if a user's account is disabled or password is changed, their existing token remains valid.
   - **Fix:** Maintain a token blacklist in Redis or use short-lived access tokens (5 minutes) with refresh tokens.

## Authorization

1. **`@PreAuthorize`/`@Secured` annotations are present** on most controllers — good.
2. **WebSocket controller lacks method-level security** — Spring Security's HTTP filter chain does not protect WebSocket frames by default. A user with a valid token for Room A could potentially send messages to Room B.
   - **Fix:** Validate room membership inside `WebSocketChatControllerImpl.sendMessage()`.
3. **Superintendent access scope** — verify `WorkerController` endpoints enforce that a superintendent can only access buildings they are assigned to.

## SQL Injection

- **No risk from JPA/HQL** — parameterized queries throughout.
- **`AbstractQueryBuilder`** uses named parameters, not string concatenation of user input.

## XSS

- **Backend outputs JSON** — no server-side HTML rendering, XSS not a concern at the backend level.
- **Frontend:** Next.js escapes JSX content by default. Risk only arises from `dangerouslySetInnerHTML`. Scan for this usage in notification/chat content rendering.
  - **Fix:** Use `DOMPurify` before rendering any user-generated content.

## Sensitive Data Exposure

1. **`LightUserVTO` exposes email and mobile number** — verify this is appropriate for all consumers.
2. **File download** — verify that `PPFile` download endpoints check that the requesting user owns or has access to the file. A tenant should not download another tenant's lease document by guessing a file ID.
3. **Stack traces in production** — the catch-all handler swallows the exception without logging. Ensure logging is added and `spring.mvc.log-resolved-exception=false` in production.

---

# 9. Technical Debt

## Critical

| # | Issue | File/Location | Impact |
|---|---|---|---|
| C1 | CSRF disabled, dual token storage (XSS + CSRF exposure) | `SecurityConfiguration.java`, `authSlice.ts`, `middleware.ts` | Security breach risk |
| C2 | `Notification` entity has no `userId` FK — notification system non-functional for user targeting | `modules/notification/.../entity/Notification.java` | Feature broken |
| C3 | `BusinessException` is checked; transactions don't rollback on business rule violations | `common/.../exception/BusinessException.java` | Data corruption risk |
| C4 | Zero automated tests across 39+ services and 50+ endpoints | Entire backend | No regression safety net |

## High

| # | Issue | File/Location | Impact |
|---|---|---|---|
| H1 | No FK constraints or indexes on `Ticket.buildingId`, `assignedToId` | `Ticket.java`, Liquibase changesets | Data integrity + query performance |
| H2 | No FK constraints on `Inspection.buildingId`, `inspectorId` | `Inspection.java`, Liquibase changesets | Same |
| H3 | No DLQ strategy for ActiveMQ — failed events silently dropped | `mq-adapter` config | Silent data loss |
| H4 | N+1 query risk in `TicketVTO` mapping (comments + attachments per ticket) | `MaintenanceMgtMapper.java` | Performance under load |
| H5 | Mock data in `manager/reports/page.tsx` alongside real API calls | Frontend reports page | Fabricated data shown to users |
| H6 | No rate limiting on auth endpoints | `SecurityConfiguration.java` | Brute force / credential stuffing |

## Medium

| # | Issue | File/Location | Impact |
|---|---|---|---|
| M1 | Inspection page bypasses Redux — no slice | `manager/inspections/page.tsx` | Pattern inconsistency, no caching |
| M2 | Enum case mismatch (TypeScript kebab-case vs Java UPPER_SNAKE_CASE) | `types/index.ts`, Java enums | Silent API failures on new enums |
| M3 | `Invoice.status` untyped String on backend | `Invoice.java` | Invalid status values possible |
| M4 | `Invoice` table named `staff_invoice` — semantic misnomer | `Invoice.java`, Liquibase | Confusion, future migration complexity |
| M5 | No rollback strategy in any Liquibase changeset | All `*-liquibase` modules | Cannot safely revert schema in prod |
| M6 | Chat dual send paths (REST + WebSocket) — potential duplicate persistence | `chatSlice.ts`, `WebSocketChatControllerImpl.java` | Duplicate messages |
| M7 | No `JOIN FETCH` for lazy associations in list result sets | All repositories | N+1 query risk |
| M8 | Admin module storage quota TODOs unimplemented | `UserManagementServiceImpl.java` | Silent feature gap |
| M9 | WebSocket lacks authorization validation per room | `WebSocketChatControllerImpl.java` | Cross-room message injection |

## Low

| # | Issue | File/Location | Impact |
|---|---|---|---|
| L1 | `RESTJWTFilterService.java` has a `TODO qualifier name should be removed` | `RESTJWTFilterService.java` | Code cleanliness |
| L2 | No exponential backoff on WebSocket reconnect | `WebSocketService.ts` | Server hammering under instability |
| L3 | No React Error Boundary | Frontend layouts | Crash = blank page |
| L4 | No request cancellation on unmount | All `useEffect` + API calls | Stale state updates |
| L5 | `DailyCounter` usage unclear — verify it is actively tracked | `modules/file/.../entity/DailyCounter.java` | Possibly unused entity |

---

# 10. Refactoring Plan

## Quick Wins (1–3 days)

1. **Fix `BusinessException` rollback:** Change `BusinessException extends Exception` → `extends RuntimeException`. Zero API changes needed.
2. **Remove mock data from reports page:** Replace hardcoded arrays with loading/empty states.
3. **Add `log.error("Unhandled exception", e)`** to catch-all handler in `RESTGlobalExceptionHandler`.
4. **Create `inspectionSlice.ts`** following the exact pattern of `maintenanceSlice.ts`. Move the inspection page's `useState` + direct API calls into Redux.
5. **Add missing Liquibase indexes** for `ticket.building_id`, `inspection.building_id`, `chat_message.room_id`, `rental_payment.tenant_id` in new changeset files.
6. **Create `InvoiceStatus` enum** and apply `@Enumerated(EnumType.STRING)` to `Invoice.status`.
7. **Add React Error Boundary** at `manager/layout.tsx` and `resident/layout.tsx`.

## Mid-Term Improvements (1–4 weeks)

1. **Fix notification architecture:** Add `user_id` FK to `notification` table. Alter `NotificationService` to always receive a `recipientId`. Add Liquibase migration. Update `NotificationRepository` to filter by user.
2. **Add FK constraints to Ticket and Inspection:** Liquibase `<addForeignKeyConstraint>` + `<createIndex>` changesets. Decide whether to use `@ManyToOne` or keep denormalized with DB-level FK.
3. **Fix CSRF + token storage:** Enable CSRF with cookie-based double-submit pattern OR remove cookie usage from `middleware.ts` and rely solely on `Authorization: Bearer` header. Remove localStorage storage in `authSlice.ts` after picking a strategy.
4. **Fix N+1 in ticket list:** Add a `TicketSummaryVTO` (no comments/attachments) for list endpoints. Use `JOIN FETCH` only on single-ticket detail endpoint.
5. **Implement chat idempotency:** Add `localId` to optimistic messages, deduplicate on WebSocket echo. Consolidate on single send path (WebSocket with REST as fallback).
6. **Add rate limiting:** Spring Boot + Bucket4j on `/auth/login`, `/auth/register`, `/auth/forgot-password`.
7. **Standardize enum serialization:** Add `@JsonValue` and `@JsonCreator` to all Java enums that map to TypeScript kebab-case literals. Update `types/index.ts` to match exactly.
8. **Implement rollback strategies:** Add `<rollback>` to all existing Liquibase changesets — this can be done module by module.

## Long-Term Architecture Improvements (1–3 months)

1. **Test coverage:** Write service unit tests for at least: `UserService`, `AuthService`, `TicketService`, `LeaseService`, `InspectionService`. Write controller integration tests for critical endpoints using `@SpringBootTest` + `MockMvc`. Target 60% line coverage minimum.
2. **DLQ and retry strategy for ActiveMQ:** Configure `dead-letter-address` + retry interceptors. Add a `failed_events` table for business-critical events that exhaust retries.
3. **Token revocation:** Implement a `revoked_tokens` table (or Redis set) for logout and password-change scenarios. Check on every JWT validation.
4. **File authorization:** Implement ownership validation in `FileControllerImpl.download()` — verify requesting user has access to the file domain before serving it.
5. **Superintendent portal completion:** Build full superintendent UI: inspections, vendors, staff, building-level ticket management.
6. **Admin portal:** Build admin UI: system settings, file type policies, user management, role management, storage quota.
7. **Caching layer:** Add `@Cacheable` to report aggregations and lookup tables. Consider Redis for session/token management.

---

# 11. Production Readiness

## Production-Ready (can deploy as-is)

- **Authentication flow** (login, register, OTP activation, forgot-password) — well implemented.
- **Property management** (properties, buildings, units, tenants, assignments) — complete, no critical issues.
- **Lease management** — complete stack, clean FK constraints.
- **Maintenance tickets** — functionally complete; denormalized FK is a data integrity risk but not a showstopper if the application enforces it.
- **Community module** (events, groups) — complete, consistent.
- **Vendor module** — complete.
- **Staff module** — complete.
- **WebSocket chat** — functional with the caveat about dual send paths.

## Risky (deploy with caution)

- **Billing/Payments** — `Invoice.status` untyped, table name misleading. Functional but low type safety.
- **Reports** — mock data contamination in the page; must be cleaned before production or reports are unreliable.
- **File management** — no download authorization validation confirmed.
- **Notification system** — UI exists but backend recipient tracking is broken; users may see wrong or all notifications.

## Must Fix Before Deployment

1. **CSRF protection + fix dual token storage** — active security vulnerability.
2. **`BusinessException` → `RuntimeException`** — prevents transaction rollback.
3. **Remove mock data from reports page** — users will see fabricated numbers.
4. **Fix `Notification.userId`** — the notification bell is functionally broken.
5. **Add minimum test coverage** — at least smoke tests for critical endpoints.
6. **Rate limiting on auth endpoints** — minimum security requirement.

---

# 12. Final Verdict

## Is the project enterprise-level?

**Partially.** The architectural foundations are enterprise-grade: modular multi-module Maven, OpenAPI-first design, library extraction, event-driven with ActiveMQ, Liquibase migrations, MapStruct, role-based security. These patterns are not accidental — they reflect deliberate architectural choices that are correct and consistent.

However, two things disqualify it from "production-ready enterprise" today: (1) near-zero test coverage, and (2) the CSRF/token storage security gap. Both are fixable within weeks.

## Developer Level

**Senior-leaning Mid-level / Junior Senior.** The developer demonstrates:
- Strong architectural pattern recognition and consistent application.
- Genuine understanding of JPA, Spring Security, event-driven design, and modular Maven.
- Ability to design and maintain a consistent project-wide style across 17 modules.
- **Gap:** Security depth (CSRF, token storage), testing culture, and database constraint rigor are below senior level. These are learned skills, not talent indicators.

## Strongest Engineering Areas

1. **Modular architecture** — the library + module separation is genuinely good engineering.
2. **OpenAPI-first consistency** — generated interfaces used uniformly.
3. **Event-driven design** — consistent event enums, event data, and listeners across all 17 domains.
4. **MapStruct mapping** — correct, consistent, Spring-integrated.
5. **Redux state management** — consistent tri-state pattern, proper error handling shape.

## Weakest Engineering Areas

1. **Testing** — near-zero automated test coverage is the single largest risk.
2. **Security depth** — CSRF disabled, dual token storage, no rate limiting.
3. **Database constraint enforcement** — missing FK constraints and indexes on multiple tables.
4. **Transaction safety** — checked exception + `@Transactional` anti-pattern.
5. **Feature completeness** — admin, audit UI, and superintendent portal are unfinished.

---

# Top 10 Critical Issues

| Priority | Issue | Fix |
|---|---|---|
| 1 | CSRF disabled + dual token storage (cookies + localStorage) | Enable CSRF or switch to header-only auth; remove localStorage token storage |
| 2 | `Notification` entity has no `userId` FK — recipient routing broken | Add `user_id` column + FK, refactor NotificationService |
| 3 | `BusinessException` (checked) — transactions do not rollback on business failures | Extend from `RuntimeException` |
| 4 | Zero automated tests across 39+ services and 50+ endpoints | Implement service unit tests + controller integration tests |
| 5 | No FK constraints/indexes on `Ticket.buildingId`, `assignedToId` | Add Liquibase `<addForeignKeyConstraint>` + `<createIndex>` |
| 6 | No FK constraints/indexes on `Inspection.buildingId`, `inspectorId` | Same as above |
| 7 | Mock data in `manager/reports/page.tsx` — fabricated numbers shown to users | Remove all mock imports, implement empty/loading states |
| 8 | No rate limiting on auth endpoints (brute-force attack surface) | Add Bucket4j or gateway rate limiter |
| 9 | N+1 query risk in ticket/inspection list endpoints that include nested collections | Add lean summary VTOs + `JOIN FETCH` for detail endpoint only |
| 10 | No DLQ strategy for ActiveMQ — business-critical events silently dropped on failure | Configure DLQ + retry policy in broker config |

---

# Top 10 Best Implemented Parts

| # | Area | Why It's Good |
|---|---|---|
| 1 | `AbstractQueryBuilder` in `sql-db-adapter` | Eliminates HQL duplication across all 17 modules with a clean inheritance pattern |
| 2 | OpenAPI-first controller generation | Generated interfaces prevent controller drift; Swagger UI is always accurate |
| 3 | Library module separation | Cross-cutting concerns (JWT, logging, MQ, error handling) are isolated and reusable |
| 4 | Event-driven domain design | Every domain has typed `Events` enum, `EventData`, `@JmsListener` — consistent and clean |
| 5 | BCrypt strength=12 | Above the minimum, not over-engineered |
| 6 | Redux Toolkit slice consistency | 18 slices follow identical shape: `createAsyncThunk` + tri-state reducers |
| 7 | MapStruct with Spring injection | Password encryption in mapper via injected service is the correct pattern |
| 8 | `RESTGlobalExceptionHandler` | Two-tier handler with `@Order`, typed `ErrorVTO`, catch-all prevents stack trace leakage |
| 9 | Liquibase modular master changelogs | Per-domain schema evolution is clean, prevents merge conflicts |
| 10 | WebSocket integration (STOMP + SockJS) | Real-time chat with heartbeat, reconnect, per-room subscriptions, and Redux integration |

---

# Summary Metrics

| Metric | Value |
|---|---|
| **Production Readiness** | 58% |
| **Technical Debt** | 34% |
| **Modules Fully Complete** | 9 / 17 |
| **Modules With Critical Issues** | 3 (Notification, Ticket, Inspection) |
| **Unfinished Modules** | 3 (Admin, Audit UI, Superintendent) |

---

# Stabilization Roadmap

```
Week 1 — Security & Data Integrity (Blocker)
  ✦ Fix CSRF + token storage
  ✦ Fix BusinessException rollback
  ✦ Add rate limiting to auth endpoints
  ✦ Add FK constraints + indexes to Ticket and Inspection
  ✦ Add userId to Notification entity

Week 2 — Correctness (Must-Have)
  ✦ Remove mock data from reports page
  ✦ Fix enum case mapping (TypeScript ↔ Java)
  ✦ Fix Chat dual send paths
  ✦ Create InvoiceStatus enum
  ✦ Implement inspectionSlice in Redux

Week 3 — Tests (Risk Reduction)
  ✦ Unit tests: UserService, AuthService, TicketService, InspectionService
  ✦ Controller integration tests: Auth, Tickets, Leases, Payments
  ✦ Security tests: @PreAuthorize annotations

Week 4 — Performance (Optimization)
  ✦ JOIN FETCH for list endpoints with nested collections
  ✦ @Cacheable on report aggregations
  ✦ WebSocket exponential backoff
  ✦ Request cancellation on unmount

Month 2 — Features (Completeness)
  ✦ Admin portal (system settings, user management, file policies)
  ✦ Audit log viewer
  ✦ Superintendent portal completion
  ✦ File download authorization
  ✦ DLQ + retry strategy for ActiveMQ
  ✦ Token revocation mechanism

Month 3 — Production Hardening
  ✦ Full test coverage to 60%+
  ✦ Rollback strategies in all Liquibase changesets
  ✦ Load testing with realistic data volumes
  ✦ Security penetration test
  ✦ Monitoring + alerting setup
```
