# PMS Project - AI Coding Style Guide & Conventions

> This file serves as the authoritative reference for coding conventions, patterns, and architecture decisions. Any AI or developer working on this project MUST follow these rules.

---

## Project Overview

- **Project:** Property Management System (PMS)
- **Backend:** Spring Boot 3.4.x / Java 17 / MySQL 8 / Liquibase / Apache Artemis (JMS)
- **Frontend:** Next.js 15 (App Router) / React 19 / TypeScript 5.7 / Redux Toolkit / Tailwind CSS 3
- **Monorepo Structure:** `pms-backend/` and `pms-frontend/` in the same repository

---

## Backend Conventions

### Module Structure

Each domain module follows this package layout under `com.pms.{module}`:

```
{module}-mgt/src/main/java/com/pms/{module}/
  annotation/config/          # Module config classes
  annotation/imports/         # Import annotations
  api/service/                # Service interfaces
  api/repository/             # Repository interfaces
  controller/generated/       # AUTO-GENERATED - do NOT edit
  controller/implementation/  # Hand-written controller impls
  core/service/implementation/ # Service implementations
  core/mapper/                # MapStruct mappers
  model/entity/               # JPA entity classes
  model/entity/pk/            # Composite key classes
  model/generated/            # AUTO-GENERATED DTOs/VTOs - do NOT edit
  model/filter/               # Search filter DTOs
  model/enums/                # Business enums (Errors, Domains, Events)
  model/event/data/           # MQ event data classes
  repository/jpa/             # Spring Data JPA interfaces
  repository/query/           # HQL query builders
  repository/implementation/  # Repository implementations
```

### Naming Conventions

| Category | Pattern | Example |
|----------|---------|---------|
| Entity | Simple noun, no suffix | `User`, `Role`, `Property`, `Unit` |
| Composite Key | `{Entity}Id` or `{Entity1}{Entity2}Id` | `UserRoleId` |
| JPA Repository | `{Entity}JPARepository` | `UserJpaRepository` |
| Repository Interface | `{Entity}Repository` | `UserRepository` |
| Repository Impl | `{Entity}RepositoryImpl` | `UserRepositoryImpl` |
| Query Builder | `{Entity}QueryBuilder` extends `AbstractQueryBuilder` | `UserQueryBuilder` |
| Service Interface | `{Domain}Service` | `UserService`, `AuthService` |
| Service Impl | `{Domain}ServiceImpl` | `UserServiceImpl` |
| Controller (generated) | `{Tag}Controller` (interface) | `UserController` |
| Controller (impl) | `{Tag}ControllerImpl` | `UserControllerImpl` |
| Mapper | `{Module}Mapper` | `UserMgtMapper` |
| Request DTO | `{Action}{Entity}DTO` | `CreateUserDTO`, `LoginUserDTO` |
| Response VTO | `{Entity}VTO` | `UserVTO`, `LightUserVTO` |
| List Response | `{Entity}ResultSet` | `UserResultSet` |
| Search Filter | `{Entity}SearchFilter` | `UserSearchFilter` |
| Error Enum | `{Module}Errors` implements `Errors` | `UserErrors` |
| Domain Enum | `{Module}Domains` implements `Domains` | `UserDomains` |
| Event Enum | `{Module}Events` implements `Events` | `UserEvents` |
| Config Class | `{Module}Config` extends `Abstract{Module}Config` | `UserConfig` |
| Import Annotation | `Import{Module}Mgt` | `ImportUserMgt` |

### Annotation Patterns

**Entities:**
```java
@Data @Builder @AllArgsConstructor @NoArgsConstructor
@Entity @Table(name = "table_name")
public class User { ... }
```

**Composite Keys:**
```java
@Data @NoArgsConstructor
public class UserRoleId implements Serializable { ... }
```

**Services:**
```java
@Service @AllArgsConstructor
public class UserServiceImpl implements UserService { ... }
```

**Controllers:**
```java
@RestController @AllArgsConstructor
public class UserControllerImpl implements UserController { ... }
```

**Search Filters:**
```java
@Data @EqualsAndHashCode(callSuper = true) @AllArgsConstructor @NoArgsConstructor @SuperBuilder
public class UserSearchFilter extends SearchFilter { ... }
```

**Mappers:**
```java
@Mapper(componentModel = "spring", builder = @Builder(disableBuilder = true),
        imports = {Collectors.class, Date.class, Long.class, CommonRequestContextKeys.class})
@MapperConfig(collectionMappingStrategy = CollectionMappingStrategy.ADDER_PREFERRED)
public abstract class UserMgtMapper { ... }
```

**Config:**
```java
@Configuration @ConfigurationProperties(prefix = "pms.platform.user-mgt")
@PropertySource(value = "classpath:config/service/pms-backend.properties", ignoreResourceNotFound = true)
public class UserConfig extends AbstractUserConfig { }
```

**Import Annotations:**
```java
@Target(ElementType.TYPE) @Retention(RetentionPolicy.RUNTIME) @Documented
@Import(value = {ImportUserMgt.Root.class})
public @interface ImportUserMgt {
    @Import({UserConfig.class})
    @ComponentScan(basePackages = {"com.pms.user"})
    class Root { }
}
```

### Swagger-First Code Generation

- Controllers and model classes in `generated/` packages are AUTO-GENERATED from `_swagger/service/swagger.yaml`
- **NEVER edit generated code** - modify the swagger.yaml and regenerate
- Generated controllers are interfaces only (delegate pattern)
- Hand-written implementations go in `controller/implementation/`
- Generated models use `@lombok.Builder @lombok.AllArgsConstructor @lombok.NoArgsConstructor`

### REST API Conventions

- **Plural nouns** for collections: `/users`, `/properties`, `/auth`
- **Path parameters** for resources: `/users/{userId}`, `/properties/{propertyId}`
- **HTTP methods:** GET = read, POST = create, PUT = update/replace, PATCH = partial update, DELETE = remove
- **Response wrapping:** `ResponseEntity<T>` for all responses (no generic envelope)
- **List responses:** `{Entity}ResultSet` with `data` list + `total` count
- **Error responses:** `ErrorVTO` with `code`, `messageEn`, `reqBodyErrors`
- **Pagination:** via query params `pageNum`, `pageSize`, `pageOffset`
- **Sorting:** via query params `orderBy`, `orderDir`
- **Authentication:** JWT Bearer token in Authorization header

### Exception Handling

**Hierarchy:**
```
AppException (abstract)
  -> BusinessException (400)
  -> InternalServerException (500)
  -> AuthorizationException (403)
  -> AuthenticationException (401)
```

**Error enum pattern:**
```java
@AllArgsConstructor
public enum UserErrors implements Errors {
    USER_NOT_FOUND(USER, "0001", "user not found"),
    ;
    private final Domains domain;
    private final String code;
    private final String messageEn;
    @Override public Domains domain() { return domain; }
    @Override public String code() { return code; }
    @Override public String messageEn() { return messageEn; }
}
```

**Use static imports:** `import static com.pms.user.model.enums.UserErrors.USER_NOT_FOUND;`

**Throw pattern:** `throw new BusinessException(USER_NOT_FOUND);`

**Domain ID ranges:** User=2xxx, Security=1xx, Common=2xxx, Property=3xxx

### Repository Pattern

Three-layer repository:
1. **Interface** (`api/repository/`) - contract
2. **JPA** (`repository/jpa/`) - Spring Data JPA interface extends `JpaRepository`
3. **Implementation** (`repository/implementation/`) - orchestrates JPA + QueryBuilder

**Repository method naming:** `insert`, `update`, `selectById`, `selectAllByFilters`, `countAllByFilters`

**Query Builder:** extends `AbstractQueryBuilder`, override `evaluateWhereConditions`, `setParameters`, `joinQuery`, `getSortingMap`, `getDefaultSorting`

### Service Patterns

- Constructor injection via `@AllArgsConstructor` (no `@Autowired` field injection)
- Use `orElseThrow(() -> new BusinessException(...))` for not-found cases
- Use mapper for entity <-> DTO/VTO conversions
- Filter objects built with builder pattern
- MQ event publishing: `mqClientService.sendMessage(EVENT_ENUM, eventData)`

### Database/Liquibase Conventions

- **File naming:** `changeset-YYYY-MM-DD.xml`
- **Changeset author:** `ahmed.el-seginy`
- **Sequential IDs** within each file
- **FK naming:** `fk_{child}_{parent}`
- **Table/column naming:** snake_case
- **Audit columns:** `created_on` (DEFAULT NOW()), `created_by_id`, `last_modified_on`, `last_modified_by_id`
- **IDs:** `bigint` auto-increment

---

## Frontend Conventions

### File Naming

| Category | Pattern | Example |
|----------|---------|---------|
| Page routes | `page.tsx` | `src/app/manager/dashboard/page.tsx` |
| Route groups | `(kebab-case)` | `src/app/(auth)/login/` |
| UI components | `PascalCase.tsx` | `Button.tsx`, `Modal.tsx` |
| Shared components | `PascalCase.tsx` | `StatCard.tsx`, `PageHeader.tsx` |
| Layout components | `PascalCase.tsx` | `DashboardLayout.tsx` |
| Hooks | `camelCase` with `use` prefix | `useAuth.ts`, `useRedux.ts` |
| Redux slices | `camelCase` + `Slice` suffix | `authSlice.ts`, `propertySlice.ts` |
| Utility modules | `camelCase` | `cn.ts`, `format.ts`, `auth.ts` |
| Type barrel files | `index.ts` | `src/types/index.ts` |
| Validators | `camelCase` by domain | `auth.ts` |

### Import Organization

Order imports as follows:
1. `'use client'` directive (if needed)
2. React / Next.js imports
3. Third-party libraries (lucide-react, framer-motion, etc.)
4. UI component libraries (@headlessui/react, etc.)
5. Redux / state imports
6. `@/` path-alias imports (components, hooks, store, types, API, utils)
7. Type-only imports last (`import type { ... }`)

**Always use `@/` path alias** - never use relative parent imports like `../../`
**Use `import type`** for type-only imports

### Component Structure

**Page components:**
```tsx
'use client';

import { ... } from '...';

// Module-level constants
const DASHBOARDS: Record<string, string> = { ... };
const containerVariants = { ... };

export default function ManagerDashboardPage() {
  // 1. Hooks (useState, useRouter, useAuth, etc.)
  // 2. Derived data / computations
  // 3. Event handlers
  // 4. Early returns (loading, auth guards)
  // 5. Return JSX
}
```

**Shared UI components:**
```tsx
'use client';

import { forwardRef, type ButtonHTMLAttributes, type ReactNode } from 'react';
import { cn } from '@/lib/utils/cn';

type ButtonVariant = 'primary' | 'secondary' | 'danger' | 'ghost' | 'outline';
type ButtonSize = 'sm' | 'md' | 'lg';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: ButtonVariant;
  size?: ButtonSize;
  loading?: boolean;
  leftIcon?: ReactNode;
  rightIcon?: ReactNode;
}

const variantStyles: Record<ButtonVariant, string> = { ... };
const sizeStyles: Record<ButtonSize, string> = { ... };

const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ variant = 'primary', size = 'md', ...props }, ref) => { ... }
);

Button.displayName = 'Button';

export { Button, type ButtonProps, type ButtonVariant, type ButtonSize };
```

- **All components are functional** - no class components
- **Pages use `export default`**; shared components use **named exports**
- **`displayName`** on every component
- **`forwardRef`** for components that need ref forwarding
- **Props interfaces** defined immediately above the component
- **Export types alongside values:** `export { Button, type ButtonProps }`

### TypeScript Patterns

- **No enums** - use string literal union types: `type UserRole = 'manager' | 'super' | 'resident'`
- **`interface`** for object shapes, component props, Redux state shapes, API request/response shapes
- **`type`** for union types, literal types, mapped/helper types, utility configs
- **Status configs** use `Record<EnumType, Config>` pattern:
  ```ts
  type StatusConfig = { label: string; color: string; bgColor: string };
  export const TICKET_STATUS_CONFIG: Record<TicketStatus, StatusConfig> = { ... };
  ```
- **Null coalescing** (`??`) preferred over OR (`||`)
- **Optional chaining** (`?.`) used extensively
- **TypeScript strict mode** is enabled
- **Generic types** for pagination: `PaginatedResponse<T>`

### Styling (Tailwind CSS)

- **Utility function:** `cn()` (clsx + tailwind-merge) for all class composition
- **Brand colors:** Custom `brand` palette (50-950 scale) - use `bg-brand-600`, `text-brand-600`, etc.
- **Common patterns:**
  - Cards: `rounded-xl border border-gray-200 bg-white shadow-sm`
  - Primary buttons: `bg-brand-600 text-white shadow-sm hover:bg-brand-700 focus:ring-brand-500`
  - Page containers: `mx-auto w-full max-w-7xl px-4 py-6 sm:px-6 lg:px-8`
  - Section headings: `text-2xl font-bold tracking-tight text-gray-900`
  - Error text: `text-sm text-red-700`
  - Responsive grid: `grid gap-4 sm:grid-cols-2 lg:grid-cols-4`
- **No CSS modules** - all styling is Tailwind utility classes
- **Mobile-first** responsive design with `sm:` and `lg:` breakpoints

### State Management (Redux Toolkit)

**Slice structure:**
```ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';
import type { FeatureState, Entity } from '@/types';
import * as api from '@/lib/api/endpoints';

const initialState: FeatureState = {
  entities: [],
  currentEntity: null,
  isLoading: false,
  error: null,
};

export const fetchEntities = createAsyncThunk<Entity[], void, { rejectValue: string }>(
  'feature/fetchEntities',
  async (arg, { rejectWithValue }) => {
    try {
      return await api.fetchEntities(arg);
    } catch (error: unknown) {
      return rejectWithValue(
        error instanceof Error ? error.message : 'Failed to fetch entities'
      );
    }
  }
);

const featureSlice = createSlice({
  name: 'feature',
  initialState,
  reducers: { clearError(state) { state.error = null; } },
  extraReducers: (builder) => {
    builder
      .addCase(fetchEntities.pending, (state) => { state.isLoading = true; state.error = null; })
      .addCase(fetchEntities.fulfilled, (state, action) => { state.isLoading = false; state.entities = action.payload; })
      .addCase(fetchEntities.rejected, (state, action) => { state.isLoading = false; state.error = action.payload ?? 'Failed'; });
  },
});

export default featureSlice.reducer;
export const { clearError } = featureSlice.actions;
```

**Typed hooks:** `useAppDispatch` and `useAppSelector` from `@/lib/hooks/useRedux`

**Custom hooks wrap Redux:** `useAuth()` instead of direct `useAppSelector`/`useAppDispatch`

### API Layer

- **Axios client** with request interceptor (JWT) and response interceptor (401 -> logout)
- **Base URL:** `NEXT_PUBLIC_API_BASE_URL` env var (defaults to `/api`)
- **Generic typed HTTP methods:** `get<T>`, `post<T>`, `put<T>`, `patch<T>`, `del<T>`
- **Error model:** Backend returns `ErrorVTO` (`{ code, messageEn, reqBodyErrors }`). Client parses this into `ApiError` class with `.vto` (typed as frontend `ErrorVTO` interface) and `.statusCode` properties.
- **Toast notifications:** Use `useToast()` hook for all user-facing messages. Methods: `addError(msg, code?)`, `addSuccess(msg)`, `addWarning(msg)`, `addInfo(msg)`, `addToast(msg, variant, duration)`.
- **Error handling pattern:** Catch errors as `unknown`, use `ApiError` class (check `instanceof ApiError`) to extract `vto.messageEn` and `vto.code`. Use `addError()` for error toasts, `addSuccess()` for success toasts.
- **Error display in catch blocks:**
  ```ts
  } catch (err: unknown) {
    if (err instanceof ApiError && err.vto) {
      addError(err.vto.messageEn, err.vto.code);
    } else {
      addError(err instanceof Error ? err.message : 'Default error message');
    }
  }
  ```

### Form Handling

- **react-hook-form** + **zod** for all form validation
- Zod schemas in `src/lib/validators/` by domain
- Schema naming: `{domain}Schema` (e.g., `loginSchema`, `registerSchema`)
- Inferred types: `export type LoginFormData = z.infer<typeof loginSchema>`
- UI state (like `showPassword`) uses `useState`, not form state

### Animation Patterns

**Framer Motion stagger pattern:**
```tsx
const containerVariants = {
  hidden: { opacity: 0 },
  show: { opacity: 1, transition: { staggerChildren: 0.06 } },
};
const itemVariants = {
  hidden: { opacity: 0, y: 16 },
  show: { opacity: 1, y: 0 },
};

<motion.div variants={containerVariants} initial="hidden" animate="show">
  <motion.div variants={itemVariants}>...</motion.div>
</motion.div>
```

### Routing & Auth

- **Middleware** (`src/middleware.ts`): Edge-level cookie-based route protection and role enforcement
- **AuthProvider** (`src/providers/AuthProvider.tsx`): Client-side auth state, auto-fetches profile
- **Dual storage:** Cookies (for middleware) + localStorage (for client)
- **Role routing:** `/manager/*`, `/superintendent/*`, `/resident/*`
- **Role map:**
  ```ts
  const DASHBOARDS: Record<string, string> = {
    manager: '/manager/dashboard',
    super: '/superintendent/dashboard',
    resident: '/resident/dashboard',
  };
  ```

---

## Cross-Cutting Rules

### DO
- Use `@/` path alias for all internal imports (frontend)
- Use `import type` for type-only imports
- Use `cn()` for className composition (frontend)
- Use constructor injection with `@AllArgsConstructor` (backend)
- Follow the swagger-first approach - define API in swagger.yaml, generate controllers/models, then implement
- Write hand-written code only in `implementation/`, `entity/`, `filter/`, `enums/`, `query/` packages
- Use `throw new BusinessException(ERROR_ENUM)` for business errors (backend)
- Use `orElseThrow(() -> new BusinessException(...))` for not-found cases (backend)
- Use string literal union types instead of TypeScript enums (frontend)
- Use `null` (not `undefined`) for Redux initial error states (frontend)
- Use `'use client'` directive on all interactive components (frontend)
- Error display pattern: `error && <motion.div>...</motion.div>` (frontend)

### DON'T
- Do NOT edit generated code in `controller/generated/` or `model/generated/` (backend)
- Do NOT use `@Autowired` field injection - use constructor injection (backend)
- Do NOT use TypeScript enums - use string literal union types (frontend)
- Do NOT use relative parent imports (`../../`) - use `@/` alias (frontend)
- Do NOT use CSS modules - use Tailwind utility classes (frontend)
- Do NOT use class components - always functional (frontend)
- Do NOT use `||` for fallback when `null`/`undefined` is meaningful - use `??` (frontend)
- Do NOT catch errors as `any` - always use `unknown` (frontend)
- Do NOT add comments unless explicitly asked

### Database Role IDs (seeded via Liquibase)
- ID 1: `Property_Manager` (maps to frontend `manager`)
- ID 2: `Superintendent` (maps to frontend `super`)
- ID 3: `Resident` (maps to frontend `renter`/`resident`)

### User Roles (Frontend)
- `manager` -> routes: `/manager/*`
- `super` -> routes: `/superintendent/*`
- `resident` -> routes: `/resident/*`

---

## Testing

- Backend: JUnit 5 with `pms-backend/library/unit-test/` test utilities
- Frontend: No test framework configured yet

## Lint & Format Commands
- Frontend: `npm run lint` (ESLint), `npm run format` (Prettier), `npm run typecheck` (TypeScript)
- Backend: Maven formatter plugin and impsort plugin for generated code only