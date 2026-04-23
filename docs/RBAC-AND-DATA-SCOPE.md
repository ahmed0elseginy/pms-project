# RBAC and data scope (PMS)

This document defines **who can do what** for kickoff, using **`pms-web` as the canonical product surface** for roles and demo users. It maps those roles to backend **modules** and to **row-level scope** (building → unit) so the API matches what the web app already models.

## 1. Canonical roles (`pms-web`)

**Source of truth*

- Type: [`UserRole`](../pms-web/src/types/index.ts) — `'manager' | 'super' | 'renter'`.
- Demo users and behaviour: [`mockData.ts`](../pms-web/src/context/mockData.ts) (e.g. Sarah Mitchell `manager`, supers `super`, residents `renter` with `unit`).
- Login labels and routes: [`LoginPage.tsx`](../pms-web/src/pages/login/LoginPage.tsx) — URL uses `manager` | `super` | `resident`; **`resident` maps to app role `renter`** (same `UserRole` in context).

| `UserRole` | User-facing title (login) | What they do in the current web prototype |
|------------|---------------------------|-------------------------------------------|
| `manager` | Property Manager | Portfolio costs, payments overview, staff, tickets triage, community/events, reports |
| `super` | Superintendent | Assigned tickets, time sheets, expense invoices, building-level work |
| `renter` | Resident | Own unit tickets, rent payments view, community, messaging staff/manager |

**Staff directory** in mock data ([`StaffMember`](../pms-web/src/types/index.ts)) describes job titles (e.g. Leasing Agent, Security) for HR-style screens; those users are not necessarily separate `UserRole` values yet. Plan either **additional roles** later or **permissions** on top of `manager` / `super`.

## 2. Goals (backend + web together)

- **Same vocabulary everywhere**: JWT / `role` table / OpenAPI docs use names aligned with `manager`, `super`, `renter` (and optional `admin` for platform operators).
- **Consistent authorization**: mutating APIs check role plus **scope** (which building or unit).
- **Least privilege**: a `renter` only sees their lease, unit, tickets, and payments; a `super` sees assigned work and relevant building context; a `manager` sees the managed estate (or assigned properties when you add org scoping).

## 3. Authorization model

### 3.1 Layers

1. **Authentication**: JWT after real login (today `pms-web` uses mock login; backend already has `/auth/login` etc.).
2. **Role**: coarse gate aligned with `UserRole` (plus optional `admin` for `user` / `admin` modules).
3. **Permission** (next refinement): strings such as `ticket:assign`, `payment:record`, `cost:write` keyed off the same personas.
4. **Data scope**:
   - **BUILDING** — default scope for this product slice (single building MVP in mock data).
   - **UNIT** — `renter`: only rows for `unit` linked to their user / lease.
   - **ASSIGNED** — `super`: tickets/time entries/invoices where they are assignee or staff record.

### 3.2 Role × capability (aligned with current `pms-web` features)

| Capability | manager | super | renter |
|------------|:-------:|:-----:|:------:|
| View / manage building costs | Yes | Read / execute per policy | No |
| View all tenant payments / aging | Yes | No | Own only |
| Staff directory / HR-style data | Yes | No | No |
| Create building-level ticket | Yes | Yes | No |
| Create unit ticket | Via policy | Yes | Yes |
| Assign / own ticket workflow | Yes | Own assigned | Own submitted |
| Approve staff expense invoices | Yes | Submit own | No |
| Community events / groups | Create / manage | Participate | Participate |
| Messaging | Yes | Yes | Yes (to manager/super) |
| Reports | Yes | Limited / none | No |

### 3.3 Role × backend module (target mapping)

| Module | manager | super | renter |
|--------|:-------:|:-----:|:------:|
| `user` | invite users for building (future) | profile | profile |
| `admin` | building settings (future) | — | — |
| `property` | full CRUD scoped to managed assets | read unit/common areas | read own unit |
| `lease` | lifecycle | read | read own |
| `billing` | charges, AR, record payments | staff reimbursements if modeled | own ledger |
| `maintenance` | tickets CRUD, assign | update assigned tickets | create/read own tickets |
| `notification` | templates (admin) + receive | receive | receive |
| `audit` | read for managed scope | — | — |
| `reporting` | manager dashboards | superintendent KPIs optional | — |

## 4. API conventions

- Apply **scope** on every list/detail for `property`, `lease`, `billing`, `maintenance`: intersect query filters with the caller’s building/unit/assignment.
- **403** when role forbids; **404** vs empty list is a product choice for cross-tenant safety.
- Document required roles in OpenAPI per operation.

## 5. Backend alignment (kickoff)

Implement `role` seeds, JWT claims, and Java enums so they **match `pms-web`**, not a legacy domain. Concrete steps: [BACKEND-ROLE-ALIGNMENT.md](./BACKEND-ROLE-ALIGNMENT.md).

## 6. Optional: `pms-frontend` console

[`pms-frontend/config/business.ts`](../pms-frontend/config/business.ts) describes a broader multi-asset **PropManage** console (institutional ICP). Treat it as a **longer-term** or **enterprise** surface; **kickoff planning and RBAC naming in this repo prioritize `pms-web`.**

## 7. References

- JWT claims today: `AuthServiceImpl` — `USER_ID`, `ROLES_IDS`, `USER_FULL_NAME`.
- Role hierarchy fields: `Role` entity — `level`, `parent_role_id`.
- Nav by role: `pms-web` — `DashboardLayout.tsx` (`manager` | `super` | `renter`).
