# PMS Web — Business & System Reference

This document summarizes **everything needed to reason about the product as a business idea** and as a **technical prototype** (`pms-web`). It pulls together marketing copy from the app, domain types, demo data, roles, navigation, stack, and how the wider workspace (`pms-backend`, `docs/`) is meant to align.

**Primary audience:** founders, product, investors, or partners evaluating a **Property Management System (PMS)** web client.

**Canonical code locations:** `pms-web/` (this SPA), `pms-backend/` (Spring Boot API), `docs/` (RBAC, C4 architecture, delivery phases).

---

## 1. Product positioning (from the live marketing surface)

The landing experience brands the solution as **Elm Technologies** (also referenced as **Technologies** in the dashboard shell) — a platform to **streamline operations, delight residents, and grow portfolios** for modern property management.

**Stated value themes (landing page feature grid):**

| Theme | User-facing promise |
|--------|---------------------|
| Maintenance tracking | Submit, assign, and track work orders from request to resolution with transparency for everyone involved. |
| Financial management | Track rental payments, property costs, staff salaries, and generate financial reports. |
| Staff coordination | Superintendents, time sheets, material invoices, and payroll-related views in one place. |
| Real-time reports | Revenue vs. cost, payment collection rates, category breakdowns. |
| Resident communication | Messaging between residents, staff, and management. |
| Community hub | Events, interest groups, and resident engagement. |

**Sample social proof (demo testimonials on landing):** faster maintenance resolution, mobile-friendly tickets and rent, simpler superintendent time sheets and invoices.

**Contact / sales inquiry topics** exposed in UI: Request a Demo, Pricing, Onboarding a Property, Partnership, Technical Support, Other.

---

## 2. Problem the system addresses

**Operational:** fragmented maintenance requests, unclear assignment and status, paper receipts and lost documentation for field staff.

**Financial:** rent and accounts-receivable visibility split across spreadsheets; building-level expenses vs. tenant payments hard to reconcile.

**People:** property managers, supers, and residents need **different** screens and permissions; staff (leasing, security, cleaning) may appear in directory-style UIs even before they become separate login roles.

**Community:** buildings want lightweight events, notices, and groups without a separate consumer social app.

---

## 3. Target users (personas) and roles

The app is explicitly **multi-role**. In code, `UserRole` is exactly:

- `manager` — **Property Manager** (portfolio / building operator).
- `super` — **Superintendent** (hands-on maintenance and staff workflows).
- `renter` — **Resident** (unit-scoped tenant experience).

**URL vs. code naming:** login routes use `/login/manager`, `/login/super`, and `/login/resident`. The **`resident` route maps to `renter`** in application state — this is intentional so marketing language (“Resident”) matches engineering (`renter`).

**What each persona does in the current prototype (high level):**

| Role | Primary jobs-to-be-done in `pms-web` |
|------|--------------------------------------|
| Property Manager | Portfolio costs, rental payments overview, staff directory, ticket triage, community/events, reports. |
| Superintendent | Assigned tickets, time sheets, expense invoices (materials), building-level work. |
| Resident | Unit maintenance tickets, rent payment view, community, messaging staff/manager. |

**Staff directory:** `StaffMember` models job titles (e.g. Leasing Agent, Security Guard, Cleaning Staff) for HR-style screens. These may later become **separate roles** or **fine-grained permissions** on top of `manager` / `super`; today they are primarily **UI / data** in mock seeds.

---

## 4. Feature inventory by role (actual routes)

Routes are defined in `src/App.tsx`. Authenticated areas use `DashboardLayout` with **role-specific side navigation** (`src/components/layout/DashboardLayout.tsx`).

### 4.1 Property Manager (`/manager/...`)

| Path | Nav label | Purpose (prototype) |
|------|-----------|---------------------|
| `/manager` | Dashboard | Overview KPIs and charts. |
| `/manager/costs` | Expenses | Building / portfolio cost lines (utilities, insurance, taxes, capex, etc.). |
| `/manager/payments` | Rental Payments | Tenant rent ledger / aging style views. |
| `/manager/staff` | Staff & Salaries | Staff roster with salary and pay frequency. |
| `/manager/tickets` | Tickets | Maintenance ticket triage and workflow. |
| `/manager/reports` | Reports | Analytics hub (Recharts-based). |
| `/manager/community` | Community | Events and groups management / participation. |

### 4.2 Superintendent (`/super/...`)

| Path | Nav label | Purpose (prototype) |
|------|-----------|---------------------|
| `/super` | Dashboard | Super landing metrics. |
| `/super/tickets` | Tickets | Work assigned to supers. |
| `/super/time` | Time Sheets | Hours and descriptions, link to tickets optional. |
| `/super/invoices` | Invoices | Staff expense invoices (line items, approval states). |

### 4.3 Resident (`/renter/...`)

| Path | Nav label | Purpose (prototype) |
|------|-----------|---------------------|
| `/renter` | Dashboard | Resident home summary. |
| `/renter/tickets` | My Tickets | Submit and track unit requests. |
| `/renter/payments` | Payments | Rent schedule / status. |
| `/renter/community` | Community | Events and groups. |
| `/renter/messages` | Messages | Conversations with manager/super. |

### 4.4 Public

| Path | Purpose |
|------|---------|
| `/` | Marketing landing + login entry. |
| `/login/:role` | Role-branded login (`manager` \| `super` \| `resident`). |
| `/contact` | Contact / inquiry. |

**Global UX:** floating **ChatWidget** is mounted from `DashboardLayout` for in-session assistance style UX (prototype).

---

## 5. Domain model (data the product understands)

Defined in `src/types/index.ts`. This is the **contract** the UI expects the backend to converge on.

**Core entities:**

- **User** — `id`, `name`, `email`, `role`, optional `avatar`, `unit` (for renters), `phone`.
- **Ticket** — maintenance work: `title`, `description`, `category`, `priority`, `status`, `createdBy`, optional `assignedTo`, `unitNumber`, timestamps, `comments[]`, optional `photos[]`.
  - Categories: plumbing, electrical, hvac, appliance, pest, general, structural, landscaping.
  - Status: open, in-progress, completed, closed.
  - Priority: low, medium, high, urgent.
- **RentalPayment** — tenant ledger row: amounts, due/paid dates, status (paid \| pending \| overdue \| partial), optional payment `method`, `note`.
- **Cost** — building expense: `description`, `category` (maintenance, utilities, insurance, taxes, supplies, renovation, other), `amount`, `date`, optional `vendor`, `recurring` + `frequency`.
- **StaffMember** — HR-style record: `role` (job title string), contact, `salary`, `payFrequency`, `startDate`, `status`.
- **TimeEntry** — superintendent labor: `staffId`, `date`, `hoursWorked`, `description`, optional `ticketId`, approval `status`.
- **Invoice** — staff reimbursement: `items[]` (description, quantity, unitPrice, total), `totalAmount`, `status`, optional `receiptUrl`, `notes`.
- **CommunityEvent** — scheduled building event with `attendees[]`, `category`.
- **CommunityGroup** — resident groups with `members[]`.
- **Message** / **Conversation** — simple 1:1 messaging model with participants, `unreadCount`, `lastMessage`.

---

## 6. Demo users and sample portfolio (mock seed)

**Source:** `src/context/mockData.ts` — all prototype data until real API wiring.

**Users:**

| Id | Name | Email | Role | Notes |
|----|------|-------|------|-------|
| u1 | Sarah Mitchell | sarah@propmanage.com | manager | Default manager demo. |
| u2 | Mike Rodriguez | mike@propmanage.com | super | Head Superintendent in staff list. |
| u7 | Carlos Diaz | carlos@propmanage.com | super | Second super. |
| u3–u6 | James, Emily, Alex, Lisa | *@tenant.com | renter | Units 101, 102, 201, 202. |

**Tickets:** mix of unit issues (faucet, AC, dishwasher, pests) and **common-area** work (parking pothole); assignments to supers; threaded comments on some tickets.

**Payments:** historical paid rows plus **overdue** and **pending** examples for storytelling (e.g. Emily Davis overdue on unit 102).

**Costs:** recurring utilities/insurance/taxes plus one-off renovation and repairs — suitable for **margin / NOI** style narratives in a pitch.

**Staff roster:** supers plus leasing, security, cleaning — shows path to **workforce** and **vendor spend** positioning.

**Time entries & invoices:** tie labor and materials to tickets (e.g. faucet repair, parking patch) — good for **audit trail** and **reimbursement** story.

**Community:** BBQ, town hall, yoga, maintenance shut-off notice — shows **operations + engagement** in one product.

**Conversations:** renter ↔ super (maintenance coordination), renter ↔ manager (lease renewal) — supports **retention / CRM** angle.

---

## 7. Authentication and state (prototype vs. production path)

**Today:** `AppContext` (`src/context/AppContext.tsx`) uses **mock login**. Choosing a role calls `login(role)` which picks the **first user in `mockData.users` with that role** and sets `currentUser`. No JWT, no server session — suitable for **demos and UX iteration**.

**Password fields on `LoginPage`:** pre-filled demo passwords (`manager123`, `super123`, `resident123`) for show; submission still uses mock `login()` after a short timeout.

**Planned alignment (workspace docs):** replace mock `login()` with **`pms-backend` `/auth/login`**, JWT claims (`USER_ID`, `ROLES_IDS`, optional string roles for web), and row-level **scope** (building / unit / assignment). See `docs/DELIVERY-PHASES-EPICS.md` Phase 0.

---

## 8. Authorization model (intended, documented in `docs/`)

**Layers:** authentication → **role** (`UserRole`) → future **permission strings** → **data scope** (BUILDING, UNIT, ASSIGNED).

**Capability matrix (summary from RBAC doc):**

- Manager: costs, all tenant payments, staff, ticket assignment, approve staff invoices, community management, reports, messaging.
- Super: assigned tickets and related time/invoice flows; participate in community; messaging; building-level ticket creation per policy.
- Renter: own tickets and payments, community participation, messaging to staff/manager.

**Backend module mapping (target):** `user`, `property`, `lease`, `billing`, `maintenance`, `notification`, `audit`, `admin`, `reporting` — each with role + scope checks on list/detail APIs. Full tables: `docs/RBAC-AND-DATA-SCOPE.md`.

**Role catalog alignment:** backend seeds / enums should use semantics matching **Property Manager / Superintendent / Resident** (web: `manager` / `super` / `renter`). Details: `docs/BACKEND-ROLE-ALIGNMENT.md`.

---

## 9. Technology stack (`pms-web`)

| Area | Choice |
|------|--------|
| Language | TypeScript |
| UI | React 19 |
| Build / dev | Vite 5 |
| Routing | React Router 7 |
| Styling | Tailwind CSS 3, tailwind-merge, tailwindcss-animate |
| Components | Radix UI primitives, shadcn-style `components/ui/*` |
| Charts | Recharts |
| Icons | Lucide React |
| Utilities | clsx, class-variance-authority, date-fns |
| Aliases | `@/` → `src/` (`vite.config.ts`) |

**npm package name:** `property-management` (private); folder name in repo: **`pms-web`**.

**Scripts:** `npm run dev` / `npm start` (Vite), `npm run build` (tsc + vite build), `npm run lint`, `npm run preview`.

---

## 10. Workspace architecture (how this SPA fits the platform)

**C4 summary (from `docs/ARCHITECTURE-C4.md`):**

- **Actors:** manager, super, renter (optional admin).
- **Primary kickoff client:** `pms-web` SPA talking **HTTPS + JSON + JWT** to Spring Boot API.
- **Data:** MySQL 8 (Liquibase migrations), Apache Artemis (JMS), filesystem for documents (`DOCUMENT_STORAGE_PATH`).

**Optional future:** `pms-frontend` is described in docs as a broader **institutional / multi-asset console**; **kickoff naming and RBAC prioritize `pms-web`**.

**Delivery dependency chain (phases 0→6):** Foundation (auth + roles) → Property hierarchy → Lease → Billing → Maintenance → Notifications & audit → Reporting. See `docs/DELIVERY-PHASES-EPICS.md` for epics and acceptance criteria.

---

## 11. Business model hooks (ideas grounded in the UI)

These are **not implemented as billing engines** in the repo but are natural extensions of what the screens already imply:

- **Per-unit / per-door SaaS** — seat-based pricing for managers; free or low-cost resident app driving adoption.
- **Payments rail** — rent display today; payment gateway marked as future in architecture docs.
- **Marketplace / referrals** — maintenance categories and vendor fields on costs suggest **preferred vendor** or **commission** models later.
- **Premium analytics** — `ManagerReports` and cost/rent charts map to **add-on reporting** tier.
- **White-label** — branding already uses logo + name slots (Elm Technologies / Technologies); easy to narrate **landlord-branded resident portal**.

---

## 12. Risks and honest prototype limits (useful for pitch Q&A)

- **No real backend integration yet** in the SPA context layer — data resets on refresh unless extended to localStorage or API.
- **Login is role-based demo**, not credential-based identity — must be replaced for production security.
- **Multi-building / org hierarchy** is assumed in docs (property → building → unit) but **MVP mock is single-building flavored** (e.g. unit numbers, “Common” as pseudo-unit).
- **Staff job titles ≠ login roles** — product policy decision pending (permissions vs. new roles).
- **Payment gateway, email SMTP, file uploads** are platform concerns on the backend side for a full go-to-market build.

---

## 13. Quick start (for demos)

1. Install dependencies: `npm install` in `pms-web/`.
2. Run: `npm run dev` and open the printed local URL (Vite default is typically `http://localhost:5173`).
3. From `/`, choose a role login or visit `/login/manager`, `/login/super`, `/login/resident`.
4. Use **Quick Demo** on the login page or submit the form — both use mock auth and navigate to `/manager`, `/super`, or `/renter`.

---

## 14. Source file index (for due diligence)

| Topic | Path |
|--------|------|
| Routes | `pms-web/src/App.tsx` |
| Global state & mutations | `pms-web/src/context/AppContext.tsx` |
| Seed data | `pms-web/src/context/mockData.ts` |
| Types | `pms-web/src/types/index.ts` |
| Navigation | `pms-web/src/components/layout/DashboardLayout.tsx` |
| Login / demo credentials UI | `pms-web/src/pages/login/LoginPage.tsx` |
| Marketing landing | `pms-web/src/pages/login/LandingPage.tsx` |
| RBAC & scope | `docs/RBAC-AND-DATA-SCOPE.md` |
| C4 architecture | `docs/ARCHITECTURE-C4.md` |
| Phased roadmap | `docs/DELIVERY-PHASES-EPICS.md` |
| Backend role alignment | `docs/BACKEND-ROLE-ALIGNMENT.md` |
| Backend runbook | `pms-backend/README.md` |

---

## 15. Document maintenance

When the product changes materially (new roles, real auth, API base URL, pricing page), update this file **or** replace sections with links to canonical product docs — this file is meant as a **single export** for business conversations derived from the repository state at authoring time.

---

*Generated for business and stakeholder use. Technical source of truth remains the code and `docs/` directory.*
