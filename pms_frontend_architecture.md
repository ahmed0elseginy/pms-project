# PMS Frontend Architecture & Migration Plan

This document outlines the practical architecture for migrating the legacy PMS React application to a modern Next.js 16+ App Router foundation, introducing Redux Toolkit (RTK) for state management without disrupting the current UI themes, layout patterns, or route structures.

---

## 1) System Architecture (High-Level)

The system relies on Next.js as both a server-rendered delivery layer and an API proxy (BFF pattern) connecting a React client (powered by Redux Toolkit) to the backend microservices.

```text
+-------------------------------------------------------------------------+
|                              Client Browser                             |
|                                                                         |
|  +--------------------+  +--------------------+  +-------------------+  |
|  |   Next.js Router   |  |     Redux Store    |  | React Components  |  |
|  |  (App Directory)   |  | (RTK Query Cache)  |  |   (Local State)   |  |
|  +---------+----------+  +---------+----------+  +---------+---------+  |
+------------|-----------------------|-----------------------|------------+
             | Route Nav             | Data Fetch (Fetch API)| 
             v                       v                       v 
+-------------------------------------------------------------------------+
|                           Next.js Server Layer                          |
|                                                                         |
|  +--------------------+  +--------------------+  +-------------------+  |
|  | Server Components  |  |    Server Actions  |  |    API Routes     |  |
|  | (Init Data Fetch)  |  |  (Form Mutations)  |  |   (BFF / Proxy)   |  |
|  +--------------------+  +--------------------+  +---------+---------+  |
+------------------------------------------------------------|------------+
             |                                               |
             | TRUST BOUNDARY (JWT AuthZ Enforced)           |
             v                                               v
+-------------------------------------------------------------------------+
|                            Backend Services                             |
|                                                                         |
|  [ Auth Service ]  [ Users API ]  [ Tickets API ]  [ Payments API ]     |
+-------------------------------------------------------------------------+
             |                     |                       |
             v                     v                       v
+-------------------------------------------------------------------------+
|                               Data Stores                               |
|        [ PostgreSQL DB ]     [ Redis Cache ]     [ S3 Storage ]         |
+-------------------------------------------------------------------------+
```

---

## 2) Data Flow

Here is the end-to-end execution flow for three critical user journeys.

### 1. Login + Role Routing
- **Request Path**: User visits `/login/[role]` (e.g., `/login/manager`) and submits credentials.
- **Business Logic**: Credentials validated on the client, then sent to Next.js API route to protect against exposing backend endpoints.
- **Redux State**: Dispatches `authApi.endpoints.login.initiate()`. On success, updates `auth` slice with `{ isAuthenticated: true, user: {...} }`.
- **Data Access**: Next.js BFF calls Backend Auth service. Backend returns JWT. Next.js sets an HTTP-only secure cookie for the Refresh Token and returns a short-lived Access Token to the client.
- **Response Handling**: The application redirects the user to `/manager`. Next.js Middleware validates the HTTP-only cookie or token claims to ensure role authorization before rendering the target route.

### 2. Manager Reports/Tickets Flow
- **Request Path**: Manager navigates to `/manager/tickets`.
- **Business Logic**: View requires a list of active tickets assigned to the manager's properties.
- **Redux State**: React component mounts and calls `useGetTicketsQuery(propertyId)`. RTK Query checks cache; if empty, flags `isFetching: true`.
- **Data Access (Async)**: RTK Query performs a GET request to the Next.js API proxy with the Access Token. The proxy forwards it to the Backend.
- **Response Handling**: Backend returns JSON. RTK Query caches data under `['Ticket', 'LIST']` and updates Redux state. UI re-renders with the ticket list.

### 3. Renter Payments/Messages Flow
- **Request Path**: Renter navigates to `/renter/payments` and submits a new payment.
- **Business Logic**: Validate payment inputs locally.
- **Redux State**: Calls RTK Query mutation `useSubmitPaymentMutation()`.
- **Data Access**: POST request to the payment gateway integration via backend API.
- **Response Handling**: On HTTP 200, RTK Query automatically invalidates the `['Payment', 'LIST']` cache tag. The UI displays a success toast, and the payment history list is automatically re-fetched (Sync UI update resulting from Async mutation).

---

## 3) Frontend Module Structure

To maintain the exact existing route flow while introducing a scalable architecture, we adopt a **Feature-Sliced Design (FSD)** approach mixed with Next.js App Router conventions.

```text
src/
├── app/                        # ONLY Next.js routing and server layout components
│   ├── system/                 # Route: /system, /system/contact
│   ├── login/                  # Route: /login/[role]
│   ├── manager/                # Route: /manager/[[...slug]]
│   ├── super/                  # Route: /super/[[...slug]]
│   ├── renter/                 # Route: /renter/[[...slug]]
│   ├── api/                    # API proxy endpoints (BFF)
│   └── layout.tsx              # Root layout (Injects Redux Provider)
│
├── features/                   # Self-contained business domains
│   ├── auth/
│   │   ├── components/         # Login forms, specific UI
│   │   ├── authSlice.ts        # UI/sync state for auth (e.g. multi-step login)
│   │   └── authApi.ts          # RTK Query endpoints for login/logout
│   ├── tickets/                # Ticket creation, listings, detail views
│   ├── payments/               # Payment processing, history
│   ├── messages/               # Inbox, chat UI
│   ├── leases/                 # MF (Multi-Family) leases management
│   ├── documents/              # Document storage and signing
│   └── users/                  # Profile, staff management
│
├── store/                      # Redux setup
│   ├── store.ts                # configureStore setup
│   ├── rootReducer.ts          # Combine feature slices and RTK Query reducers
│   └── hooks.ts                # Typed useAppDispatch, useAppSelector
│
├── shared/                     # Reusable cross-feature assets
│   ├── components/             # Buttons, Modals, Inputs (Preserving current UI style)
│   ├── hooks/                  # Custom React hooks
│   ├── utils/                  # Formatters, generic logic
│   └── types/                  # Global TS interfaces, API schemas
│
└── lib/                        # Core infrastructural configs (e.g., API clients)
```

**Naming & Ownership Boundary:**
- `app/` owns **where** things render.
- `features/` owns **what** business logic runs and **how** domain data is structured.
- `shared/` owns generic elements devoid of business context.

---

## 4) Redux State Management Design

Redux Toolkit (RTK) is the core state container.

### Architecture Rules
- **RTK Query (Async Data)**: Used exclusively for all API interactions (fetching, caching, mutations). Replaces manual `fetch` / `axios` + `useEffect` blocks.
- **Redux Slices (Global Sync State)**: Reserved strictly for data that crosses component boundaries (e.g., currently authenticated user, global notification toasts, complex wizard states).
- **Local State (`useState`)**: Used for purely localized UI states (dropdown toggles, controlled form inputs).

### Slice Boundaries
1. **`auth` Slice**: Holds `{ isAuthenticated, userRole, userId, token }`.
2. **`ui` Slice**: Holds `{ themeTheme, sidebarOpen, activeModal }`.
3. **`api` Slice (RTK Query)**: Cache layers managing entities.
   - Tags: `User`, `Ticket`, `Payment`, `Message`, `Lease`, `Document`.

### Migration Strategy (Context to Redux)
1. **Coexistence**: Wrap the `<Provider store={store}>` inside existing Context providers.
2. **Incremental Adoption**: Move global states (like Auth) from Context to Redux first. Existing pages remain unchanged visually.
3. **Data Fetching Shift**: Gradually replace `useEffect` fetches in legacy pages with `useGetXQuery()` hooks.

---

## 5) Authentication & Authorization

- **Session Lifecycle**: 
  - User authenticates. Backend issues JWTs. 
  - Access Token is stored in memory (Redux `auth` slice) for CSR requests.
  - Refresh Token is stored as an `HttpOnly`, `Secure` cookie.
- **RBAC Strategy**: 
  - Three primary roles: `Manager`, `Superintendent`, `Renter`.
  - Enforced via **Next.js Middleware**. The middleware intercepts requests to `/manager/*`, `/super/*`, `/renter/*`. It checks the role claim in the token/cookie. If a Renter tries to access `/manager/tickets`, they are redirected to `/system/unauthorized` or `/login/manager`.
- **Token Handling**: Axios/Fetch interceptor in the API slice handles `401 Unauthorized` errors by automatically triggering a silent refresh against a `/api/refresh` endpoint using the `HttpOnly` cookie, before retrying the failed request.

---

## 6) Data Layer Design

**Core Entities:**
- **User**: `id`, `email`, `role`, `profileInfo`, `propertyIds[]`
- **Ticket**: `id`, `status` (Open/Closed/InProgress), `priority`, `creatorId`, `assigneeId`, `propertyId`, `logs[]`
- **Payment**: `id`, `amount`, `status`, `renterId`, `invoiceId`, `date`
- **Message**: `id`, `threadId`, `senderId`, `receiverId`, `content`, `readStatus`
- **Lease**: `id`, `unitId`, `renterIds[]`, `startDate`, `endDate`, `status`, `monthlyRent`
- **Document**: `id`, `type` (Lease/Invoice/Notice), `url`, `uploadedBy`, `createdAt`

**Frontend Caching Rules (RTK Query):**
- Data fetched is cached.
- **Invalidation**: Creating a ticket via `createTicket(data)` invalidates `['Ticket', 'LIST']`. Editing ticket `123` invalidates `['Ticket', { id: 123 }]`.

---

## 7) API Design

To ensure predictable frontend data handling, we standardize API interactions:

- **Endpoint Grouping**:
  - `/api/auth/*`
  - `/api/tickets/*`
  - `/api/payments/*`
  - `/api/users/*`
  - `/api/leases/*`
  - `/api/documents/*`
- **Response Envelope**:
  All frontend-bound responses follow a strict envelope:
  ```typescript
  interface ApiResponse<T> {
    data: T | null;
    error: { code: string; message: string; details?: any } | null;
    meta?: { page: number; totalPages: number; totalCount: number };
  }
  ```
- **Conventions**:
  - Pagination via query params: `?page=1&limit=20`
  - Filtering: `?status=OPEN&priority=HIGH`
  - Idempotency keys used for critical mutations (e.g., Payments) in HTTP headers.

---

## 8) Non-Functional Architecture

- **Security**: 
  - Next.js server actions / routes strictly validate inputs (using `zod`).
  - No secrets (API keys) exposed to client bundle (prefixed appropriately without `NEXT_PUBLIC_` unless required).
- **Performance**: 
  - Utilize Next.js route-level code splitting. 
  - Heavy selectors use `createSelector` from reselect to memoize derived state (e.g., complex ticket filtering logic).
- **Reliability**: 
  - RTK Query configured with automatic retries for transient 5XX errors.
  - Global `<ErrorBoundary>` wrapping layout layers to catch React crashes without white-screening the whole app.
- **Observability**: 
  - Structured console logs in dev.
  - Integration ready for Sentry/Datadog to catch client-side unhandled promise rejections and React render errors.

---

## 9) Environment Strategy

- **Development**: Local Node server. `.env.local` pointing to local DB or staging backend.
- **Staging**: Hosted environment matching Prod. Uses `.env.staging`. Used for QA and stakeholder sign-off.
- **Production**: Live environment (`.env.production`).
- **Test Data**: Utilize Mock Service Worker (MSW) during local development to simulate complex role-based data flows without relying on backend availability.

---

## 10) Deployment & DevOps

- **CI/CD Pipeline**:
  1. **Lint & Format**: Prettier / ESLint enforcing codebase rules.
  2. **Type Check**: Strict `tsc --noEmit`.
  3. **Test**: Run unit tests (Jest) for Redux logic and critical UI components.
  4. **Build**: `next build` command.
- **Release Strategy**: 
  - Immutable builds. 
  - Rollback achieved by reverting routing alias to previous successful build deployment via hosting provider (e.g., Vercel / AWS).
- **Migration Safety**: Feature flags to toggle between legacy React component and new Redux-integrated component during transition phases.

---

## 11) Migration / Roadmap

### Phase 1: Baseline Stabilization
- **Goal**: Establish foundation without breaking current flow.
- **Key Changes**: Setup Next.js App Router (already in progress), add Redux boilerplate (`configureStore`, `Provider`), migrate base layout.
- **Exit Criteria**: App runs exactly as it does today, but with Redux DevTools detecting an empty store.

### Phase 2: Redux Integration by Feature
- **Goal**: Move state and fetching to RTK.
- **Key Changes**: Introduce `authSlice`, `ticketApi`, `paymentApi`. Refactor `/manager` routes to use RTK Query hooks instead of legacy fetching.
- **Exit Criteria**: Core domains (Auth, Tickets, Payments) are fully driven by Redux.

### Phase 3: API Hardening & Performance
- **Goal**: Solidify the network layer.
- **Key Changes**: Implement Axios interceptors, silent token refresh, robust error boundaries, pagination standards.
- **Exit Criteria**: Resilient network handling, no forced logouts due to token expiry during active sessions.

### Phase 4: Scale & Readiness
- **Goal**: Prepare for broad production use.
- **Key Changes**: Memoization, bundle size audits, Sentry integration, removal of legacy context/state.
- **Exit Criteria**: Clean codebase, high performance metrics (Lighthouse), zero legacy fetch code.

---

## 12) Risks & Trade-offs

| Risk | Mitigation |
| :--- | :--- |
| **Breaking Existing Routes** | Next.js App Router allows gradual migration. We will strictly map legacy URLs (`/manager/[[...slug]]`) to App Router catch-alls initially. |
| **UI Regressions** | Strictly adhering to the "No UI redesign" constraint. Existing shared components are ported as-is. |
| **Redux Boilerplate Overhead** | Using Redux Toolkit drastically cuts boilerplate. |

**Trade-offs Chosen:**
- **RTK Query over Thunks**: We chose RTK Query because 90% of PMS frontend state is server-state (fetching tickets, payments). Writing manual Thunks for fetching is an anti-pattern in modern Redux.
- **Feature-Sliced over Route-Centric**: Grouping by feature (e.g., `features/tickets/`) rather than throwing everything in `app/` routes ensures that shared logic across Manager and Super roles isn't duplicated.
- **Next.js Server Components vs Client Components**: Due to the heavy interactive nature of the PMS dashboards and the Redux requirement, most inner route components will use `'use client'`. Server components will primarily handle initial layout, secure auth verification, and SEO/Metadata.

---

## Recommended Next 10 Tasks (Implementation Checklist)

- [ ] 1. Initialize `store/store.ts` with `configureStore` and wrap the root `app/layout.tsx` with `<Provider>`.
- [ ] 2. Setup `shared/types/` pulling from existing domain types to establish strict TypeScript bounds.
- [ ] 3. Create the `features/auth/authSlice.ts` to hold JWT and user role in sync state.
- [ ] 4. Create the base RTK Query API instance (`lib/api.ts` or `store/api/baseApi.ts`) with `fetchBaseQuery` including token injection interceptors.
- [ ] 5. Implement Next.js Middleware (`middleware.ts`) to securely protect `/manager`, `/super`, and `/renter` routes based on auth cookies.
- [ ] 6. Scaffold the `features/tickets/ticketsApi.ts` establishing `getTickets` and `createTicket` endpoints.
- [ ] 7. Identify one low-risk page (e.g., `/renter/messages`) and swap its legacy data fetching for the new RTK Query hook.
- [ ] 8. Review UI visual regression on the refactored page to ensure "No UI redesign" rule holds true.
- [ ] 9. Implement global API error handling (Toast notifications for RTK Query rejections).
- [ ] 10. Refactor the complex `/manager/tickets` view to utilize Redux cache tags for optimistic updates and invalidation.
