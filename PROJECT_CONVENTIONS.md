# PROJECT_CONVENTIONS.md

A reusable guide for scaffolding a new project in the style of **PMS** (Property Management System).
Stack: **Spring Boot 3.4 multi-module Maven monolith** (swagger-first codegen) + **Next.js 15 / React 19 frontend**, in one repo via git submodules.

Every rule below is backed by a real file in this repo. When in doubt, open the cited file and copy its shape.

---

## Part A — Quick Start (bootstrap from zero)

### A.1 Repo skeleton
```
project-root/
  pms-backend/      # git submodule — Spring Boot reactor
  pms-frontend/     # git submodule — Next.js app
  docs/             # specs, plans, ADRs (markdown)
  docker-compose.yml
  AGENTS.md         # authoritative convention doc (this file's source of truth)
```
Submodules are declared in `.gitmodules`. The two halves are versioned independently and pinned by SHA from the root.

### A.2 Backend from zero
1. **Reactor pom** (`pms-backend/pom.xml`, `packaging=pom`): Java 17, Spring Boot 3.4.9, Spring Cloud 2024.0.2, Hibernate 6.6, MapStruct 1.6.3, Lombok 1.18.40, Liquibase 4.27, openapi-generator 7.19. Configure annotation-processor paths in `maven-compiler-plugin`: lombok → lombok-mapstruct-binding → mapstruct-processor → hibernate-jpamodelgen (in that order).
2. **Define profiles** in the reactor pom:
   - `service-swagger` → runs openapi-generator + formatter + impsort over `**/controller/generated/*` and `**/model/generated/*`.
   - `coverage` → JaCoCo with non-logic packages excluded (see `pom.xml:411-436`).
   - `default` (activeByDefault) → lists every `<module>`.
3. **Create the library layer** (`library/`): `lib-bom`, `common`, `swagger`, `unit-test`, plus adapters `rest-adapter`, `security-adapter`, `mq-adapter`, `logger-adapter`, `session-manager`, `sql-db-adapter`. `common` holds the shared `Errors`/`Domains`/`Events` interfaces, `AppException` hierarchy, `SearchFilter`/`PaginationInfo`/`SortingInfo`, and shared generated VTOs.
4. **For each business module** create `modules/{domain}/{domain}-mgt/` with the package layout in §B.2. Add the module to the reactor pom `default` profile.
5. **Write the API first**: author `modules/{domain}/_swagger/service/swagger.yaml`, then run `_swagger/service/generate.bat` (wraps `mvn generate-sources -Pservice-swagger`). This emits `controller/generated` interfaces + `model/generated` DTO/VTO. **Never edit generated output.**
6. **Hand-write** controller impls (`controller/implementation`), services (`api/service` + `core/service`), repositories (3 layers), entities, mappers, enums, filters, the `{Module}Config`, and the `Import{Module}Mgt` annotation.
7. **Wire it**: add `@Import{Module}Mgt` to `main/main-execution/.../Bootstrap.java`. `main-execution` is the only deployable Spring Boot app.
8. **Migrations**: add a Liquibase `changeset-YYYY-MM-DD.xml` under `_liquibase/modules/{module}-mgt/changes/` and include it from `db.changelog-master.xml`.

### A.3 Frontend from zero
1. `npx create-next-app` (App Router, TS). Pin React 19, Next 15.
2. Add deps: `@reduxjs/toolkit react-redux axios react-hook-form @hookform/resolvers zod clsx tailwind-merge framer-motion lucide-react @headlessui/react @heroicons/react @tanstack/react-table @dnd-kit/core recharts @stomp/stompjs sockjs-client date-fns`. Dev: `@playwright/test prettier prettier-plugin-tailwindcss eslint eslint-config-next tailwindcss`.
3. **Configs** — copy verbatim:
   - `.prettierrc.js`: `{ semi:true, trailingComma:'all', singleQuote:true, tabWidth:2, endOfLine:'lf', plugins:['prettier-plugin-tailwindcss'] }`
   - `.eslintrc.js`: `extends:['next/core-web-vitals']`
   - `tsconfig.json`: `strict:true`, `paths:{ "@/*":["./src/*"] }`
4. **Create `src/` tree** (§B.9). Build the axios `client.ts` first (it's the spine), then `store/index.ts` (calls `injectApiStore`), then slices, then `ui/` primitives, then pages.
5. `package.json` scripts: `dev`, `build`, `start`, `lint`, `format`, `typecheck` (`tsc --noEmit`), `test:e2e` (Playwright).

---

## Part B — Reference

### B.1 Backend module reactor
**Rule:** the build is a multi-module Maven reactor; `library/*` and `modules/*` are pom-managed libraries, only `main/main-execution` produces a runnable jar.
`pms-backend/pom.xml:478-509` lists all modules; `Bootstrap.java` is the single entry point.

### B.2 Backend package layout (per module)
**Rule:** every `{domain}-mgt` module uses the identical package tree under `com.pms.{module}`.
```
annotation/config/          # {Module}Config extends Abstract{Module}Config
annotation/imports/         # Import{Module}Mgt annotation
api/service/                # service INTERFACES
api/repository/             # repository INTERFACES
controller/generated/       # AUTO-GENERATED interfaces — do NOT edit
controller/implementation/  # {Tag}ControllerImpl
core/service/               # {Domain}ServiceImpl
core/mapper/                # {Module}Mapper (MapStruct)
core/proxy/                 # cross-module proxy services
model/entity/  entity/pk/   # JPA entities + composite keys
model/generated/            # AUTO-GENERATED DTO/VTO — do NOT edit
model/dto/ model/vto/       # hand-written DTO/VTO (when not generated)
model/filter/               # {Entity}SearchFilter extends SearchFilter
model/enums/                # {Module}Errors/Domains/Events/business enums
model/event/data/           # MQ event payloads
repository/jpa/             # Spring Data interfaces
repository/query/           # {Entity}QueryBuilder extends AbstractQueryBuilder
repository/implementation/  # {Entity}RepositoryImpl
```
Reference: full tree under `modules/property/property-mgt/src/main/java/com/pms/property/`.

### B.3 Swagger-first codegen
**Rule:** the API contract is `swagger.yaml`; controllers + DTO/VTO are generated, never hand-edited.
- Generator config: `pom.xml:109-179` (`generatorName=spring`, `interfaceOnly=true`, `delegatePattern=true`, `useTags=true`, models annotated `@lombok.Builder/@AllArgsConstructor/@NoArgsConstructor`).
- Trigger: `modules/property/_swagger/service/generate.bat` → `mvn generate-sources -Pservice-swagger`.
- Spec cross-references shared responses: `$ref: '../../../../library/_swagger/service/swagger.yaml#/components/responses/BadRequest'` (`modules/property/_swagger/service/swagger.yaml`).
- **Edge case:** the avatar endpoint's generated Java was hand-written to *mimic* the generator without re-running it — allowed only to match output exactly; prefer regenerating.

### B.4 Controllers
**Rule:** `{Tag}ControllerImpl implements {Tag}Controller`, `@RestController @AllArgsConstructor`, each method `@Override`, `@Secured({ROLE})`, name `_operationId`, returns `ResponseEntity`, delegates straight to a service.
```java
@RestController
@AllArgsConstructor
public class UnitControllerImpl implements UnitController {
    private final UnitService unitService;

    @Override
    @Secured({PROPERTY_MANAGER_ROLE})
    public ResponseEntity<Void> _addUnitToProperty(Long propertyId, UnitDTO unitDTO) {
        unitService.create(propertyId, unitDTO);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }
}
```
`controller/implementation/UnitControllerImpl.java`. Roles are static-imported from `com.pms.user.model.enums.UserRoles`.

### B.5 Services
**Rule:** interface in `api/service`, impl in `core/service` as `@Service @AllArgsConstructor`; build a `SearchFilter` from query params, use the repo's `selectAllByFilters`/`countAllByFilters`, map via the mapper; not-found → `orElseThrow(() -> new BusinessException(X_NOT_FOUND))`.
```java
public UnitVTO findById(Long id) {
    Unit unit = unitRepository.selectById(id)
            .orElseThrow(() -> new BusinessException(UNIT_NOT_FOUND));
    return propertyMgtMapper.toUnitVTO(unit);
}
```
`core/service/UnitServiceImpl.java`. **Always explicit types — never `var`.**

### B.6 Repositories (3-layer)
**Rule:** `api/repository` interface → `repository/implementation` (sets audit fields, delegates) → `repository/jpa` (Spring Data) + `repository/query` (`AbstractQueryBuilder`). Method names: `insert/update/selectById/selectAllByFilters/countAllByFilters/deleteById`.
```java
@Repository
@AllArgsConstructor
public class UnitRepositoryImpl implements UnitRepository {
    private final UnitJpaRepository unitJpaRepository;
    private final UnitQueryBuilder unitQueryBuilder;
    private final RequestContext requestContext;

    public Unit insert(Unit unit) {
        unit.setCreatedOn(LocalDateTime.now());
        unit.setCreatedById(requestContext.get(USER_ID, Long.class));
        return unitJpaRepository.save(unit);
    }
}
```
`repository/implementation/UnitRepositoryImpl.java`. The current user comes from `RequestContext`, not method args.

**QueryBuilder:** override `getEntityNameClass`, `evaluateWhereConditions`, `setParameters`, `joinQuery`. Aliases use `item.` as the root; LIKE filters lowercase both sides.
`repository/query/UnitQueryBuilder.java`.

### B.7 Entities
**Rule:** `@Data @Builder @AllArgsConstructor @NoArgsConstructor @Entity @Table(name="snake_case")`, `@GeneratedValue(IDENTITY)`, enums `@Enumerated(STRING)`, audit columns non-writable from app code.
```java
@Column(name = "created_on", insertable = false, updatable = false) private LocalDateTime createdOn;
@Column(name = "created_by_id", insertable = false) private Long createdById;
```
`model/entity/Unit.java`. Audit columns: `created_on/created_by_id/last_modified_on/last_modified_by_id`.

### B.8 Errors & exceptions
**Rule:** per-module `{Module}Errors` enum `implements Errors` (`domain/code/messageEn`); throw `new BusinessException(ENUM)` via static import; `AppException` is the abstract base.
- `library/common/.../interfaces/Errors.java` — interface (`getFullMessage()` = `[domain-code] [label]: messageEn`).
- `library/common/.../exception/AppException.java` — abstract base carrying `Errors error` + `args`.
- Hierarchy: `AppException → BusinessException(400) / InternalServerException(500) / AuthorizationException(403) / AuthenticationException(401)`.
- Domain ID ranges: Security=1xx, User/Common=2xxx, Property=3xxx.

### B.9 Mappers
**Rule:** abstract class, `@Mapper(componentModel="spring")`, `@Mapping(target=…, ignore=true)` for id + audit fields, hand-written `toResultSet(list,total)` builders.
`core/mapper/PropertyMgtMapper.java`. **Drift:** AGENTS.md documents a richer `@MapperConfig(...ADDER_PREFERRED)` variant (e.g. `UserMgtMapper`); `PropertyMgtMapper` uses the leaner `unmappedTargetPolicy=IGNORE`. Match the module you're editing.

### B.10 Wiring & config
**Rule:** each module exposes `Import{Module}Mgt` (`@Import` + `@ComponentScan("com.pms.{module}")`); `Bootstrap.java` composes adapters + all modules; config via `@ConfigurationProperties(prefix="pms.platform.{module}")`.
`main/main-execution/.../Bootstrap.java` (`@EnableScheduling`, `@SpringBootApplication`). Properties live in `config/service/pms-backend.properties` (e.g. `pms.platform.user-mgt.default-role-id=3`).

### B.11 Database / Liquibase
**Rule:** `changeset-YYYY-MM-DD.xml`, author `ahmed.el-seginy`, sequential IDs per file, snake_case tables/columns, `bigint` auto-increment ids, FK `fk_{child}_{parent}`, audit columns on every table. Master changelog includes each file explicitly.

### B.12 Frontend `src/` layout
```
src/app/{(auth),manager,resident,superintendent}/**/page.tsx
src/components/{ui,shared,layout,brand}/
src/lib/{api,hooks,utils,validators,websocket}/
src/store/{index.ts,slices/}
src/providers/  src/types/  src/styles/
```

### B.13 Frontend API client
**Rule:** one axios instance with request (JWT) + response (401→refresh→logout) interceptors; export generic `get/post/put/patch/del<T>`; wrap backend `ErrorVTO` in an `ApiError` class. The Redux store is injected (`injectApiStore`) to avoid an import cycle.
- `src/lib/api/client.ts`: `baseURL = NEXT_PUBLIC_API_BASE_URL ?? '/api'`; FormData requests drop `Content-Type`; `AUTH_PATHS_NO_TOKEN` skip the bearer token; 401 triggers one refresh-token retry (`_retry` guard) then logout + redirect.
- All endpoint functions are centralized in `src/lib/api/endpoints.ts` and consumed by thunks — pages never call axios directly.

### B.14 Redux Toolkit
**Rule:** 16 feature slices; `createAsyncThunk<Result, Arg, {rejectValue:string}>` with try/`rejectWithValue`; pending/fulfilled/rejected reducers; typed hooks `useAppDispatch`/`useAppSelector`; domain hooks (`useAuth`, `useChat`) wrap raw selectors/dispatch.
- `src/store/index.ts` registers reducers and calls `injectApiStore(store)`.
- `src/lib/hooks/useRedux.ts`: `useDispatch.withTypes<AppDispatch>()` / `useSelector.withTypes<RootState>()`.
- Initial `error` state is `null` (not `undefined`); rejected reducers use `action.payload ?? 'Failed'`.

### B.15 Components
**Rule:** all functional + `'use client'`; pages `export default`, shared components named-export with `displayName` + `forwardRef`; props `interface` above the component; variant→class maps via `Record<Variant,string>`; compose classes with `cn()`.
- `src/lib/utils/cn.ts`: `twMerge(clsx(inputs))`.
- Pattern reference: `AGENTS.md` §"Shared UI components" (`Button.tsx`).

### B.16 TypeScript style
**Rule:** strict mode; **no enums** — string-literal unions (`type UserRole = 'manager' | 'super' | 'resident'`); `interface` for object/props/state shapes, `type` for unions/helpers; status configs as `Record<Union, Config>`; `??` over `||`, `?.` everywhere; `import type` for types; `@/` alias only — never `../../`.

### B.17 Styling
**Rule:** Tailwind utilities only (no CSS modules); custom `brand` palette (50–950); cards `rounded-xl border border-gray-200 bg-white shadow-sm`; mobile-first (`sm:`/`lg:`). `prettier-plugin-tailwindcss` sorts class lists.

### B.18 Testing
**Rule:** Backend = JUnit 5 + Mockito unit tests co-located in each module's `src/test`; **integration tests centralized** in `main-execution/src/test` as `{Domain}IntegrationTest` extending `PmsBaseIntegrationTest` (Testcontainers MySQL + `@Sql`). JaCoCo `coverage` profile excludes entity/generated/config/repository-jpa/enums.
Frontend = **Playwright e2e only** (`e2e/`); no unit-test framework configured.

### B.19 Routing & auth (frontend)
**Rule:** `src/middleware.ts` does edge cookie-based route protection + role enforcement; `AuthProvider` holds client auth state; tokens stored in **both** cookies (middleware) and localStorage (client). Role routes: `/manager/*`, `/superintendent/*`, `/resident/*`.

---

## Part C — Decision Log

- **Swagger-first codegen vs hand-written controllers.** Chosen: generate interfaces + DTO/VTO from `swagger.yaml`, implement separately. Alternative: annotate hand-written controllers (springdoc reads them). Codegen wins because the spec is the single source of truth, drift between docs and code is impossible, and the frontend can be generated from the same spec.
- **Multi-module monolith vs microservices.** Chosen: one reactor, one deployable (`main-execution`), modules wired by `Import{Module}Mgt`. Alternative: per-domain services. Modules give clean boundaries and independent test scope without the ops cost of distributed services; can be split later because each module is already self-contained.
- **3-layer repository (interface / impl / jpa+query) vs plain Spring Data.** Chosen: explicit `AbstractQueryBuilder` for dynamic filtering + a thin impl that stamps audit fields from `RequestContext`. Alternative: Spring Data derived queries / Specifications. The builder gives readable HQL for many optional filters and one place to inject auditing.
- **Axios wrapper + `ApiError` vs raw fetch.** Chosen: single instance, interceptors, typed helpers, `ApiError` wrapping `ErrorVTO`. Interceptors centralize JWT injection and 401-refresh; the error class lets every catch block read `err.vto.messageEn`/`err.vto.code` uniformly.
- **Store injection (`injectApiStore`) vs importing the store in the client.** Chosen: inject. The client is imported by slices, so importing the store back would create a cycle; injection breaks it.
- **String-literal unions vs TS enums.** Chosen: unions. They serialize as plain strings (match backend JSON), erase at compile time, and combine with `Record<Union, Config>` for status maps.
- **Git submodules vs single repo / npm workspace.** Chosen: submodules. Backend and frontend have entirely different toolchains and release cadences; submodules let each version independently while the root pins a known-good pair.

---

## Part D — Don'ts

**Backend**
- Don't edit anything in `controller/generated/` or `model/generated/` — change `swagger.yaml` and regenerate.
- Don't use `@Autowired` field injection — constructor injection via `@AllArgsConstructor`.
- Don't use `var` — always write explicit Java types.
- Don't set audit fields by hand in services — the repository impl stamps them from `RequestContext`.
- Don't throw raw exceptions — `throw new BusinessException({Module}Errors.X)` with a static import.
- Don't put business logic in entities/generated/config — those are JaCoCo-excluded for a reason.
- Don't add a module without registering it in the reactor `default` profile **and** `Bootstrap.java`.

**Frontend**
- Don't call `axios`/`fetch` from components — go through `src/lib/api/endpoints.ts` → thunks.
- Don't use TypeScript `enum` — string-literal unions.
- Don't use relative parent imports (`../../`) — use the `@/` alias.
- Don't use CSS modules or inline style objects for theming — Tailwind utilities + `cn()`.
- Don't write class components — functional only, with `'use client'` on interactive ones.
- Don't use `||` where `null`/`0`/`''` is meaningful — use `??`.
- Don't `catch (e: any)` — catch `unknown`, check `instanceof ApiError`.
- Don't initialize Redux `error` to `undefined` — use `null`.
- Don't add comments unless asked.

---

## Known drift (inconsistencies — not rules)
- Duplicate `ErrorVTO` / `LightUserVTO` definitions under `library/common/.../model/{generated,vto,vto/vto}`.
- Mapper config varies between modules (`PropertyMgtMapper` lean vs `UserMgtMapper` `@MapperConfig`).
- Avatar endpoint's "generated" Java was hand-written, not regenerated.
- Frontend has no unit-test framework (e2e only).
