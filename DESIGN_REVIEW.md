# DESIGN_REVIEW.md

A critical companion to [PROJECT_CONVENTIONS.md](PROJECT_CONVENTIONS.md). Where that file documents *how the codebase works*, this one documents **what to add, what's missing, and what's risky**. Every item is backed by a file in the repo. Severity: 🔴 high · 🟠 medium · 🟡 low.

---

## 1. Red Flags (design risks)

### ~~1.1 `/users/` was exempt from the auth token on the frontend~~ FIXED
`AUTH_PATHS_NO_TOKEN` was broadened to `'/users/'`, stripping the bearer token from **all** `/users/...` routes. Narrowed to only `/avatars/` (the actual public endpoint).
`pms-frontend/src/lib/api/client.ts:102`

### 🔴 1.2 JWT + refresh token stored in `localStorage`
Tokens are read/written from `localStorage` and mirrored into cookies.
```ts
localStorage.setItem('token', token);
localStorage.removeItem('refreshToken');
```
`pms-frontend/src/lib/api/client.ts:129-156`. `localStorage` is readable by any injected script → XSS = full token theft, including the long-lived refresh token. **Fix:** keep the refresh token in an `HttpOnly`, `Secure`, `SameSite` cookie; keep only a short-lived access token in memory (not `localStorage`). The cookie half already exists for middleware — consolidate on it.

### 🔴 1.3 No data-scope authorization — only role checks
Controllers gate by role with `@Secured`, but services return data with no ownership/tenancy filter. `findAll` returns **every** unit in the system to any manager or superintendent:
```java
@Secured({PROPERTY_MANAGER_ROLE, SUPERINTENDENT_ROLE})
public ResponseEntity<UnitResultSet> _getAllUnits(...) { return ResponseEntity.ok(unitService.findAll(...)); }
```
`controller/implementation/UnitControllerImpl.java:47-51`; `UnitServiceImpl.findAll` builds a filter with no owner/building constraint. This is a horizontal-authorization (IDOR) gap — Manager A can read Manager B's properties. `docs/RBAC-AND-DATA-SCOPE.md` describes the intended scoping, but the query layer doesn't enforce it. **Fix:** inject the caller's scope (building/property IDs) from `RequestContext` into every `SearchFilter` and into `selectById` ownership checks.

### ~~1.4 Missing/inconsistent transaction boundaries~~ FIXED
Write methods across all service implementations now have `@Transactional` (`jakarta.transaction.Transactional`). 69 annotations across 30 service files covering create/update/delete/approve/reject/invite/sign operations. Read-only methods (`find*`, `get*`) are intentionally left non-transactional.

### ~~1.5 Every `BusinessException` returned HTTP 400 — including "not found"~~ FIXED
Created `NotFoundException extends AppException` at `library/common/.../exception/NotFoundException.java`. Updated `RESTGlobalExceptionHandler` to return HTTP 404 for `NotFoundException`. Migrated all 125 `BusinessException(X_NOT_FOUND)` throw sites across 32 files to `NotFoundException(X_NOT_FOUND)`. Non-NOT_FOUND business errors still correctly return 400 via `BusinessException`.

### 🟠 1.6 `n+1` and unbounded fan-out risk in mappers
`toVTO` maps entities one-by-one and entities hold `@ManyToOne` relations (`Unit.property`, `Unit.building`) that are lazily loaded per row. List endpoints map a page of entities, each touching its relations → n+1 queries. `PropertyMgtMapper.toVTOList` + `Unit.java:33-39`. **Fix:** fetch-join the relations the VTO needs in the `QueryBuilder.joinQuery()`, or project directly to VTOs.

### 🟡 1.7 Hand-written code masquerading as generated
The avatar endpoint's `generated/` Java was authored by hand to imitate the generator, and the Liquibase changeset that creates `avatar_fid` was intentionally excluded from the master changelog (per project memory / session notes). The next real `generate-sources` run will overwrite the hand-written file, and the column may be missing in fresh DBs. **Fix:** re-run the generator from the updated `swagger.yaml` and re-include the changeset.

---

## 2. Missing (gaps to close)

- 🔴 **Frontend has no unit/component tests.** Only Playwright e2e (`pms-frontend/e2e/`, `playwright.config.ts`). Slices, the `client.ts` error parser, and utils have zero coverage. **Add:** Vitest + React Testing Library; start with `parseApiError` and the auth slice.
- 🟠 **No rate limiting / lockout at the edge.** Backend has app-level login lockout (`pms.platform.user-mgt.max-failed-login-count=3`) but nothing throttles the API gateway/`/auth` endpoints. **Add:** bucket4j or gateway-level rate limits.
- 🟠 **No request/correlation ID propagation.** `RequestContext` carries `USER_ID` but there's no trace/correlation ID threaded through logs and MQ events, making cross-module debugging hard. **Add:** an MDC correlation id in the rest-adapter, propagated onto MQ event data.
- 🟠 **No API versioning.** Paths are `/users`, `/properties` with no `/v1`. Breaking changes to a generated DTO break clients silently. **Add:** version prefix in each `swagger.yaml` `servers`/paths.
- 🟠 **Secrets in `*.properties` files.** `config/service/*.properties` and `application-*.properties` hold config inline; Firebase/DocuSign/DB creds risk being committed. **Add:** externalize via env vars / a secrets manager; keep only non-secret defaults in-repo.
- 🟡 **No OpenAPI → frontend type generation.** The backend is swagger-first, but the frontend hand-maintains `src/types/index.ts` and `endpoints.ts`. **Add:** generate the TS client/types from the same specs to kill drift.
- 🟡 **No DB index strategy documented.** QueryBuilders filter on `LOWER(unitNumber) LIKE` and FK columns; without functional/FK indexes these table-scan. **Add:** indexes in the changesets for every filterable column.
- 🟡 **No health/readiness contract.** No Actuator config surfaced for liveness/readiness in `render.yaml`/compose. **Add:** Spring Boot Actuator endpoints + container healthchecks.

---

## 3. Enhancements (would raise the ceiling)

- **Generate the frontend API layer from swagger** (openapi-typescript or orval). Eliminates the dual maintenance of `endpoints.ts` + `types/index.ts` and the 1.1-style hand patches.
- **Introduce RTK Query** for server cache. The 16 hand-rolled `createAsyncThunk` slices reimplement loading/error/caching per feature (`store/slices/*`). RTK Query collapses that boilerplate and adds caching/invalidation for free.
- **Codify data-scope as an aspect.** A `@ScopedQuery`/interceptor that injects the caller's allowed IDs into every `SearchFilter` would fix 1.3 once instead of per-service.
- **Shared module archetype / generator.** All 17 modules repeat the identical package tree; a Maven archetype or scaffolding script would make new modules consistent by construction (and prevent the mapper-config drift noted in conventions §B.9).
- **Contract tests on the generated controllers.** Since the spec is the source of truth, add spec-driven contract tests (e.g. schemathesis against the running app) so swagger changes are verified end-to-end.
- **Centralize toast/error UX.** The catch-block pattern (`instanceof ApiError → addError(vto.messageEn, vto.code)`) is repeated everywhere; a single thunk-rejection→toast middleware would DRY it.
- **CSP + Subresource Integrity** on the Next.js app to reduce the blast radius of the `localStorage` token exposure until 1.2 is fixed.

---

## 4. What's genuinely good (keep)
- Swagger-first contract with strict generated/hand-written separation.
- Clean 3-layer repository with a reusable `AbstractQueryBuilder` (pagination + sorting handled centrally: `setFirstResult/setMaxResults` in `library/sql-db-adapter/.../AbstractQueryBuilder.java`).
- One global exception handler producing a uniform `ErrorVTO` consumed by a single frontend `ApiError` class — now with proper `NotFoundException` → 404 mapping.
- Modular monolith with explicit `Import{Module}Mgt` wiring — boundaries are real and splittable.
- Centralized integration tests on Testcontainers MySQL.
- Transaction boundaries on all write service methods via `@Transactional`.

---

### Summary of fixes applied

| Issue | Status | Scope |
|-------|--------|-------|
| 1.1 Token exemption (`/users/` → `/avatars/`) | **Fixed** | 1 file |
| 1.4 `@Transactional` on write methods | **Fixed** | 69 annotations across 30 service files |
| 1.5 `NotFoundException` → HTTP 404 | **Fixed** | 1 new class + 1 handler update + 125 throw sites across 32 files |
| 1.2 `localStorage` token storage | Open | Requires auth architecture rework |
| 1.3 Data-scope authorization (IDOR) | Open | Requires scoping strategy across all modules |
| 1.6 n+1 in mappers | Open | Per-module query optimization |
| 1.7 Hand-written generated code | Open | Needs generator re-run + changeset inclusion |

### Remaining priority order
1. 1.3 data scoping (IDOR) → 1.2 token storage — security.
2. 1.6 n+1 queries — performance.
3. 1.7 generator alignment — correctness.
4. §2 frontend tests + secrets externalization — durability.
