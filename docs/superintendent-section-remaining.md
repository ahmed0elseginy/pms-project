# Superintendent Section -- Implementation Status

This document tracks the implementation status of all superintendent section items across both backend and frontend. Items marked **DONE** have been implemented; items still marked **REMAINING** need future work.

---

## Table of Contents

1. [Backend Remaining Items](#backend-remaining-items)
2. [Frontend Remaining Items](#frontend-remaining-items)
3. [Priority Assessment](#priority-assessment)

---

## Backend Remaining Items

### Critical / High Severity

| # | Item | File(s) | Details |
|---|------|---------|---------|
| B1 | **No row-level data scope enforcement** | `RBAC-AND-DATA-SCOPE.md` defines building/assignment-level scoping for superintendents, but only `@Secured` role gates exist. No query-level filtering restricts a superintendent to their assigned buildings/units. | Any authenticated superintendent can potentially access data outside their assignment scope. |
| B2 | **InvoiceController is ad-hoc / technical debt** | `billing-mgt` `InvoiceControllerImpl.java` | Uses raw `Map<String, Object>` and direct JPA repository access instead of proper VTOs/services. Bypasses the swagger-first code generation pattern. Needs to be refactored to follow the standard controller/service/repository pattern. |
| B3 | **Role ID alignment not applied** | `CreateUserDTO.java:82` vs `CreateUserDTO.java:28` | Line 82 documents `Superintendent=4` (target per `BACKEND-ROLE-ALIGNMENT.md`), but the current DB has `Superintendent=2`. The planned Liquibase migration `changeset-2026-04-18-pms-roles.xml` was never created. This creates an inconsistent mapping. |

### Medium Severity

| # | Item | File(s) | Details |
|---|------|---------|---------|
| B4 | **Superintendent address/city validation not enforced** | `UserErrors.java` defines `ADDRESS_REQUIRED_FOR_SUPERINTENDENT` and `CITY_REQUIRED_FOR_SUPERINTENDENT` error codes, but no service class throws them. Superintendent-specific address and city validation is missing from registration/user-creation flows. | |
| B5 | **No `/me/staff` endpoint for superintendents** | `StaffControllerImpl.java` | All 7 staff endpoints use `@Secured({PROPERTY_MANAGER_ROLE})` exclusively. Superintendents cannot list or view their assigned staff via API. The RBAC doc indicates superintendents should see "assigned work." | |
| B6 | **`GET /buildings/{buildingId}/tickets` missing Secured annotation** | `TicketsControllerImpl.java` | Swagger says "manager/super view" but the implementation has no `@Secured` annotation -- any authenticated user can call this endpoint. Should be restricted to PM and superintendent roles. | |

### Low Severity

| # | Item | File(s) | Details |
|---|------|---------|---------|
| B7 | **`SystemStatisticsVTO.totalSuperintendents` hardcoded stub** | `AdminServiceImpl.java` | `getSystemStatistics()` always returns `0L` for superintendent count. `getDashboardStats()` correctly queries the DB. The stats endpoint needs to be wired to live data. | |
| B8 | **Superintendent KPI reporting not implemented** | `reporting-mgt` module | `RBAC-AND-DATA-SCOPE.md` mentions "superintendent KPIs optional" but the reporting module has no controller, service, or repository -- only a config class and an import annotation. `ManagerDashboardSummary` is defined in Swagger but has no Java implementation. | |

---

## Frontend Remaining Items

### Pages Using Mock Data (Need Real API Integration)

| # | Page | Route | Current State | Real API Available? |
|---|------|-------|---------------|---------------------|
| F1 | Buildings | `/superintendent/buildings` | Imports `buildings`, `units`, `tickets` from `mockData` | Yes -- `worker.getMyProperties()`, `buildings.getAllByProperty()`, `units.getAllByProperty()` |
| F2 | Calendar | `/superintendent/calendar` | Imports `tickets` from `mockData`; hardcoded `new Date(2026, 3, 1)` | Partially -- `tickets.getMyTickets()` exists |
| F3 | Time Sheets | `/superintendent/time-sheets` | Imports `timeEntries`, `tickets` from `mockData`; form submit is a no-op | Yes -- `timeEntries.getMyTimeEntries()`, `tickets.getMyTickets()` |
| F4 | Invoices | `/superintendent/invoices` | Imports `invoices` from `mockData`; form submit is a no-op | Yes -- `invoices.getMyInvoices()`, `invoices.createInvoice()` |

### Pages With No Backend (Entirely Static / Hardcoded)

| # | Page | Route | Details |
|---|------|-------|---------|
| F5 | Vendors | `/superintendent/vendors` | 8 hardcoded vendor entries inline. No `/vendors` API endpoint exists in backend. Both backend API and frontend integration need to be created. |
| F6 | Inspections | `/superintendent/inspections` | 5 hardcoded inspection entries inline. No `/inspections` API endpoint exists in backend. Create form submission is a no-op. Needs both backend API and frontend integration. |

### Structural / Routing Issues

| # | Item | Details |
|---|------|---------|
| F7 | **Empty `/superintendent/tickets/` directory** | Directory exists with no `page.tsx`. Route returns 404. Note: sidebar navigation links to `/superintendent/work-orders` instead, so this may be intentional, but the empty directory should be cleaned up or a redirect page added. |
| F8 | **No `/superintendent/settings` page** | Header component links to `${basePath}/settings` which resolves to `/superintendent/settings`, but no page exists. |
| F9 | **Messages page is a thin re-export** | `messages/page.tsx` simply re-exports the resident messages component. No superintendent-specific behavior (e.g., default chat rooms, different contexts). |

### Data / Configuration Issues

| # | Item | File | Details |
|---|------|------|---------|
| F10 | **Role label inconsistency** | `src/lib/utils/constants.ts:128` | `ROLE_LABELS` maps `super` to `"Super Admin"` but `Header.tsx:37` and `ProfilePage.tsx:38` use `"Superintendent"`. Should be unified to `"Superintendent"`. |
| F11 | **Permission map incomplete** | `src/lib/hooks/useAuth.ts` | `hasPermission` for `super` role lists: `dashboard, buildings, work-orders, calendar, time-sheets, invoices, vendors, profile`. Missing: `inspections` and `messages`. Sidebar shows these nav items but the permission check would deny access. |
| F12 | **Calendar hardcoded date** | `calendar/page.tsx` | Uses `new Date(2026, 3, 1)` instead of `new Date()`. Calendar always shows April 2026. |
| F13 | **Missing API endpoint definitions** | `src/lib/api/endpoints.ts` | No `vendors` or `inspections` frontend API definitions. `worker` section only has `getMyProperties()`. |

---

## Priority Assessment

### Phase 1: Must Have (Blocks core superintendent workflow)

1. **B1** -- Row-level data scope enforcement (superintendent should only see assigned buildings/units)
2. **F1-F4** -- Migrate buildings, calendar, time-sheets, invoices pages from mock data to real APIs
3. **F11** -- Add `inspections` and `messages` to `hasPermission` grants for `super` role

### Phase 2: Should Have (Important for feature completeness)

4. **B2** -- Refactor InvoiceController to follow standard pattern
5. **B5** -- Add `/me/staff` endpoint for superintendents (or at minimum a read-only staff list)
6. **B6** -- Add `@Secured` annotation to `GET /buildings/{buildingId}/tickets`
7. **F8** -- Create `/superintendent/settings` page
8. **F9** -- Make messages page superintendent-aware (distinct from resident)
9. **F10** -- Fix `ROLE_LABELS` from `"Super Admin"` to `"Superintendent"`
10. **F12** -- Fix calendar hardcoded date

### Phase 3: Nice to Have (Future enhancements)

11. **B3** -- Decide on role ID alignment (keep id=2 or migrate to id=4) and resolve `CreateUserDTO` inconsistency
12. **B4** -- Implement superintendent address/city validation using existing error codes
13. **B7** -- Wire `SystemStatisticsVTO.totalSuperintendents` to live data
14. **B8** -- Implement superintendent KPI reporting module
15. **F5-F6** -- Create backend APIs for vendors and inspections, then integrate in frontend
16. **F7** -- Clean up empty `tickets/` directory or add redirect
17. **F13** -- Add vendors and inspections endpoint definitions to `endpoints.ts`