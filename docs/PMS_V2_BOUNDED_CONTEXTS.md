# PMS v2 — Spatial Property Platform: Bounded Contexts & Migration Plan

Source: `docs/Spatial Property Platform — SRS (EN AR).md` (Oct 6, 2026)
Baseline: PMS v1 (Spring Boot 3.4 / MySQL 8 / Liquibase / Artemis — Next.js 15 / Redux / Tailwind)
Date: 2026-10-18 · Author: Ahmed El-Seginy

---

## 1. What changes from v1 to v2

| Aspect | PMS v1 (today) | PMS v2 (SRS) |
|---|---|---|
| Product shape | Single-company PMS: leases, rent, tickets, staff, vendors, community | Multi-tenant SaaS: **3D property explorer** + **facility booking** |
| Tenancy | None (one company) | `Workspace` = management company; every row scoped by `workspace_id` |
| Property hierarchy | Property → Building → Unit (`floor` is an int) | Property → **Tower** → **Floor** → Unit, + **UnitLayout** |
| Unit status | VACANT, OCCUPIED, MAINTENANCE | OCCUPIED, VACANT, **RESERVED**, UNDER_MAINTENANCE |
| Roles | manager / super(intendent) / resident (global) | platform super-admin + per-workspace **owner, manager, staff, resident, viewer** with property scope |
| Main UI | Lists, tables, dashboards per role | Isometric 3D canvas + detail panel + card strip, deep-linkable |
| Out of v2 scope | — | leases, rent/payments, maintenance tickets, mobile app, 3D editor |

---

## 2. Bounded contexts

```
                      ┌──────────────────────────┐
                      │  Identity & Access (IAM) │  user-mgt (extended)
                      └────────────┬─────────────┘
                                   │ Membership / role / property scope (JWT claims)
                      ┌────────────▼─────────────┐
                      │   Workspace (Tenancy)    │  workspace-mgt (NEW, absorbs admin-mgt)
                      └──┬──────────┬─────────┬──┘
           workspace_id  │          │         │
     ┌───────────────────▼──┐  ┌────▼───────────────┐   ┌─────────────────────┐
     │ Property Portfolio   │◄─┤ Spatial Scene      │──►│ Asset Storage       │
     │ property-mgt (refac.)│  │ spatial-mgt (NEW)  │   │ file-mgt (extended) │
     └──────────┬───────────┘  └────────────────────┘   └─────────────────────┘
                │ Unit / Resident (read)          ▲ facility model assets
     ┌──────────▼───────────┐                     │
     │  Facility Booking    │─────────────────────┘
     │  booking-mgt (NEW)   │
     └──────────┬───────────┘
                │ domain events (Artemis)
   ┌────────────┼──────────────┬────────────────────┐
   ▼            ▼              ▼                    ▼
Notification  Audit        Search (read model)   Reporting / Export
(existing)    (existing)   search-mgt (NEW)      reporting-mgt (extended)
```

### BC-1 · Identity & Access — `user-mgt` (extend)

- **Owns:** User, Credential, OTP, Membership, Session/refresh token, 2FA secret.
- **SRS reqs:** S-05 (email+password, magic link, Google SSO, 2FA for managers), S-06 (RBAC per workspace and property).
- **Changes:**
  - New `Membership(user_id, workspace_id, role, property_scope)`; replaces global `UserRole` for workspace roles. `UserRole` stays only for **platform** roles (`SUPER_ADMIN`).
  - JWT claims: `USER_ID`, `WORKSPACE_ID` (active workspace), `WORKSPACE_ROLE` (same naming as the existing `USER_ID` / `ROLES_IDS` claims). Workspace **always** comes from the token (SRS §7). Property scope is stored on the membership; not yet in the token.
  - Add refresh tokens, magic-link (reuse `OTP`), Google OAuth2, TOTP 2FA.
  - Rate limiting on auth endpoints (N-07).

### BC-2 · Workspace (Tenancy) — `workspace-mgt` (NEW, absorbs `admin-mgt`)

- **Owns:** Workspace (name, slug, plan, currency, locale, timezone default, branding), WorkspaceSettings, Plan, demo-workspace reset.
- **SRS reqs:** S-02 workspace switcher, S-11 demo workspace, S-12 settings, super-admin role.
- **Cross-cutting piece (library):** `TenantContext` in `session-manager` + a Hibernate `@Filter("workspaceFilter")` / `@TenantId` on every tenant entity, so no repository can forget the `workspace_id` predicate. Tenant-isolation tests are mandatory (N-07).

### BC-3 · Property Portfolio — `property-mgt` (refactor)

- **Owns:** Property, Tower (rename of `Building`), Floor (NEW), UnitLayout (NEW), Unit, Resident (replaces `Tenant`).
- **SRS reqs:** A-01, A-02, A-09…A-18, A-22, A-23, S-09 (CRUD + CSV import).
- **Business rules (domain service `OccupancyService`):**
  - `occupancy% = round(occupied / total * 100)`; RESERVED is not occupied.
  - Tower/property totals are **computed** from units, never stored by hand (drop `Building.floors` as an input; derive from `Floor`).
  - UNDER_MAINTENANCE units are hidden for resident/viewer roles.
- **Read models:** tower occupancy breakdown, floor × unit status grid (A-11), unit directory (paged/sortable/filterable, A-23).
- **Events published:** `UNIT_STATUS_CHANGED`, `UNIT_CREATED`, `RESIDENT_MOVED_IN/OUT`.

### BC-4 · Spatial Scene — `spatial-mgt` (NEW)

- **Owns:** SceneManifest, NodeBinding (`model_node_key` → entity type + id), LabelAnchor, CameraPreset, LightingPreset, model-asset references per Property / Tower / Floor template / UnitLayout / Facility.
- **SRS reqs:** A-03…A-07, A-19 (fallback layout model), A-20, S-10, §10.2 pipeline.
- **Why its own BC:** the 3D pipeline changes on a different cadence than business data; Portfolio stays free of rendering concerns. It references Portfolio IDs only (Conformist downstream of Portfolio).
- **Key endpoint:** `GET /properties/{id}/scene` — one manifest the frontend loads, then recolours meshes by status at runtime (no new model on status change).

### BC-5 · Facility Booking — `booking-mgt` (NEW) — core domain

- **Owns:** Facility, Amenity, FacilityRule (opening hours, durations, capacity, price, requires_approval, booking_window_days, max_active_per_unit, cancel_cutoff), FacilityClosure, Reservation, ApprovalDecision.
- **SRS reqs:** B-01…B-33, §4.5.
- **Aggregate:** `Reservation` with state machine
  `REQUESTED → PENDING_APPROVAL → CONFIRMED → COMPLETED`, side exits `DECLINED`, `CANCELLED`, `NO_SHOW`. Free + shared facilities go `REQUESTED → CONFIRMED`.
- **Concurrency (N-09) on MySQL 8** — no Postgres exclusion constraint, so:
  1. `SELECT … FOR UPDATE` on the `facility` row inside the booking transaction (serialises bookings per facility), then
  2. check overlap (exclusive) or summed guests ≤ capacity (shared) for the slot, then insert.
  3. Safety net: a `reservation_slot(facility_id, slot_start)` table with a **unique key** for exclusive facilities, written in the same transaction.
  - Conflict → `409` with the conflicting slot.
- **Availability:** `GET /facilities/{id}/availability?date=&duration=` computed from rules − closures − reservations, in the **property time zone**.
- **Reads from Portfolio:** resident ↔ unit (Customer/Supplier; local cached projection fed by `RESIDENT_*` events).
- **Events published:** `RESERVATION_REQUESTED / CONFIRMED / APPROVED / DECLINED / CANCELLED / MOVED`, `RESERVATION_REMINDER_DUE` (scheduled, 1 h before).

### BC-6 · Asset Storage — `file-mgt` (extend)

- **Owns:** Asset (kind glb/image, url, size, checksum, lod_level).
- **SRS reqs:** S-10, N-03, N-07 (signed URLs).
- **Changes:** S3-compatible backend (MinIO locally, S3/R2 in prod) behind `FileStorageService`; `POST /assets` returns a pre-signed upload URL; GLB validation (size ≤ budget, checksum); CDN base URL config.

### BC-7 · Notification — `notification-mgt` (reuse)

- Consumes booking events → in-app bell + email (S-07). Add Arabic/English templates (N-11). Existing `Domain/Event/EventParameter` model fits as-is.

### BC-8 · Audit — `audit-mgt` (reuse)

- Consumes `UNIT_STATUS_CHANGED`, approvals, CRUD edits → `AuditLog(actor_id, entity, entity_id, action, diff JSON)` (S-08). Add `workspace_id`.

### BC-9 · Search — `search-mgt` (NEW, small)

- Read model over towers, units, facilities, residents (S-03, ⌘K). v2 = MySQL FULLTEXT on a denormalised `search_index` table fed by events; returns `{type, id, label, sceneNodeKey}` so the UI can fly the camera.

### BC-10 · Reporting & Export — `reporting-mgt` (extend)

- KPI endpoints (`/properties/{id}` KPIs, `/booking-stats`), `POST /exports` async jobs (CSV/XLSX via Artemis worker; PNG/PDF of the view is client-side). (A-24)

### Frozen in v2 (keep code, unplug from `main-execution`)

`lease-mgt`, `billing-mgt`, `maintenance-mgt`, `inspection-mgt`, `staff-mgt`, `vendor-mgt`, `chat-mgt`, `community-mgt`.
They are explicitly out of scope in SRS §1.2. Remove their `@Import…Mgt` and dependency from `main-execution` behind a Maven profile `v1-legacy` rather than deleting them; leases/payments/maintenance return in a later phase.

---

## 3. Context map (relationships)

| Upstream | Downstream | Pattern | Contract |
|---|---|---|---|
| IAM | all | Open Host | JWT claims `uid, wid, role, props[]` |
| Workspace | all | Shared Kernel (library) | `TenantContext`, `workspace_id` filter |
| Portfolio | Spatial Scene | Conformist | Property/Tower/Floor/Unit/Layout IDs |
| Portfolio | Booking | Customer/Supplier | `RESIDENT_*`, unit lookup API |
| Portfolio, Booking | Search, Audit, Reporting | Published Language | Artemis events |
| Booking | Notification | Published Language | `RESERVATION_*` events |
| Asset Storage | Spatial, Booking | Open Host | asset id → signed URL |

---

## 4. Data model (MySQL, following AGENTS.md conventions)

Keep project conventions over the SRS where they conflict: `bigint` auto-increment IDs (expose a `public_id` UUID column only where IDs appear in URLs), `created_on / created_by_id / last_modified_on / last_modified_by_id`, snake_case, `fk_{child}_{parent}`, Liquibase `changeset-YYYY-MM-DD.xml`, author `ahmed.el-seginy`.

| Table | New / changed | Notes |
|---|---|---|
| `workspace` | NEW | slug unique, currency, locale, branding JSON, is_demo |
| `membership` | NEW | (user_id, workspace_id) unique, role, property_scope JSON |
| `property` | CHANGE | + workspace_id, slug, country, status, timezone, lat, lng, community_type |
| `tower` | RENAME `building` | + code (unique per property), subtitle, property_type, model_node_key, sort_order |
| `floor` | NEW | tower_id, number (unique per tower), floor_template_asset_id |
| `unit_layout` | NEW | name, bedrooms, bathrooms, interior_asset_id |
| `unit` | CHANGE | floor_id (replaces int floor), layout_id, code, area_m2 (replaces sqft), facing, category, model_node_key; status enum + RESERVED; index (floor_id, status); drop rent_amount (out of scope) |
| `resident` | REPLACES `tenant` | user_id, unit_id, move_in, move_out, is_primary; lease fields dropped |
| `facility` | NEW | property_id, booking_type, capacity, open/close time, durations JSON, price, requires_approval, booking_window_days, max_active_per_unit, cancel_cutoff_hours, status, model_asset_id, thumbnail_asset_id |
| `amenity` | NEW | facility_id, label, icon |
| `facility_closure` | NEW | facility_id, start_at, end_at, reason |
| `reservation` | NEW | facility_id, unit_id, user_id, start_at, end_at (UTC), guests, note, fee, currency, status, decided_by_id, decided_at, decline_reason; index (facility_id, start_at, end_at) |
| `reservation_slot` | NEW | facility_id + slot_start unique (exclusive facilities only) |
| `scene_node_binding`, `camera_preset` | NEW | spatial-mgt |
| `asset` | CHANGE `pp_file` | + kind, lod_level, checksum, storage_key |
| `audit_log`, `notification` | CHANGE | + workspace_id |

Migration of v1 data: `building → tower` (code = generated from name), one `floor` row per distinct `unit.floor`, `tenant → resident`, everything assigned to a single seeded workspace.

---

## 5. API (swagger-first, per module)

Base path `/api/v1`. Errors keep the project `ErrorVTO { code, messageEn, reqBodyErrors }` (instead of RFC 7807) for consistency with the existing client; booking conflict = `409` + `conflictingSlot` field.

| Module swagger | Endpoints |
|---|---|
| workspace | `GET/POST /workspaces`, `GET/PUT /workspaces/{id}/settings`, `POST /workspaces/{id}/demo-reset` |
| user | + `/auth/refresh`, `/auth/magic-link`, `/auth/google`, `/auth/2fa/*`, `/memberships` |
| property | `/properties`, `/properties/{id}` (+KPIs), `/towers/{id}` (+breakdown, grid), `/towers/{id}/floors/{n}`, `/units` (directory), `PATCH /units/{id}`, `/unit-layouts`, `POST /imports/csv` |
| spatial | `GET /properties/{id}/scene`, admin CRUD for bindings and presets |
| booking | `/properties/{id}/facilities`, `/facilities/{id}`, `/facilities/{id}/availability`, `/facilities/{id}/closures`, `POST/GET /reservations`, `PATCH /reservations/{id}`, `POST /reservations/{id}/approve`, `/decline`, `GET /properties/{id}/booking-stats` |
| search | `GET /search?q=` |
| reporting | `POST /exports`, `GET /exports/{jobId}` |
| file | `POST /assets` (signed URL), `GET /assets/{id}` |

Realtime: existing STOMP over WebSocket (`@stomp/stompjs` already in frontend) on topics `/topic/ws/{wid}/units` and `/topic/ws/{wid}/reservations`.

---

## 6. Frontend v2

- **Add:** `three`, `@react-three/fiber`, `@react-three/drei`, `@react-three/postprocessing`.
- **State:** keep Redux Toolkit (project convention) — new `sceneSlice` (selected property/tower/floor/unit, hover, camera level) replaces the SRS's Zustand suggestion; `propertySlice`, `bookingSlice`, `workspaceSlice`.
- **Routes (App Router):**
  - `/[workspace]/p/[propertySlug]` → property level
  - `/[workspace]/p/[propertySlug]/t/[tower]/f/[floor]/u/[unit]` (optional catch-all, A-21)
  - `/[workspace]/p/[propertySlug]/units` → unit directory
  - `/[workspace]/bookings` → tabs: explorer / reservations / approvals
  - `/[workspace]/admin/*` → CRUD + CSV import + asset upload
  - `/platform/*` → super-admin
- **Components:** `SceneCanvas`, `TowerLabel` (drei `Html`), `KpiRow`, `DetailPanel`, `CardStrip`, `StatusChip`, `HeatGrid`, `DayStrip`, `SlotGrid`, `DaySchedule`, `ApprovalCard`, `CommandPalette` (⌘K), WebGL fallback image.
- **Design tokens:** replace the `brand` palette with the SRS sage/deep-green palette (`brand-600 ≈ #2F6B4F`, surface `#EEF2EC`, status occupied/vacant/reserved), Manrope font, 12–16 px radii.
- **i18n:** `next-intl` with `en` + `ar` (RTL) from day one (N-11).
- **Role keys:** replace `manager | super | resident` with `owner | manager | staff | resident | viewer` (+ `platform_admin`). Note: v1 `super` = superintendent, which collides with SRS "super admin" — retire the `super` key and the `/superintendent/*` routes.

---

## 7. Tech-stack decisions vs. SRS

The SRS recommends NestJS + PostgreSQL. Recommendation: **keep the existing Spring Boot / MySQL stack** — the module architecture, swagger generation, MQ, Liquibase and security already exist and match the BC split.

| SRS suggests | v2 decision | Consequence |
|---|---|---|
| NestJS | Spring Boot 3.4 (keep) | No rewrite; new BCs = new `*-mgt` modules |
| PostgreSQL + RLS + exclusion constraint | MySQL 8 (keep) + Hibernate tenant filter + row lock + unique slot table | Must test isolation and double-booking explicitly. Revisit if Postgres is preferred — this is the single biggest decision |
| UUID PKs | bigint + `public_id` UUID where exposed | Matches conventions |
| Redis + BullMQ | Artemis (keep) + Spring `@Scheduled`; add Redis only for availability cache if p95 > 300 ms | — |
| Zustand + TanStack Query | Redux Toolkit (keep) | — |
| RFC 7807 | `ErrorVTO` (keep) | — |
| Socket.IO / SSE | STOMP WebSocket (keep) | — |

---

## 8. Delivery plan mapped to the repo

| Phase | Scope (SRS) | Backend work | Frontend work |
|---|---|---|---|
| 0 · Foundations (2 w) | auth, workspaces, roles, admin CRUD, CSV import, design system | `workspace-mgt`, `Membership`, tenant filter in library, JWT claims, freeze legacy modules, Liquibase migration of v1 data | new shell (rail, top bar, breadcrumbs), tokens, role routing, i18n scaffold |
| 1 · 3D explorer MVP (4 w) | property + tower levels, KPIs, tower panel, card strip, unit directory | property refactor (Tower/Floor/Layout), `OccupancyService`, `spatial-mgt` scene manifest, `file-mgt` S3 assets | R3F canvas, labels, hover/select sync, KPI row, tower panel, unit directory table |
| 2 · Floors & units (3 w) | floor explorer, unit panel, status changes, interiors, deep links | floor/unit endpoints, `PATCH /units/{id}` + event + WS push, audit | floor plate view, unit interior, deep-link routing, live recolour |
| 3 · Facility booking (4 w) | explorer, availability, reservations, schedule, approvals, notifications | `booking-mgt` (rules, availability, concurrency-safe create, state machine), reminders, notification templates | facility explorer, slot grid, day schedule, kanban, approvals |
| 4 · Polish (2 w) | exports, audit log, demo workspace, perf, a11y, Arabic RTL | `search-mgt`, exports, demo reset, rate limiting, ≥70 % tests on booking/occupancy | ⌘K search, exports, RTL, WCAG pass, Playwright E2E |

---

## 9. Open decisions

1. **MySQL vs PostgreSQL** — keep MySQL (recommended, less churn) or switch to Postgres for RLS + exclusion constraints?
2. **Where v2 lives** — evolve `main` in place with legacy modules behind a Maven profile (recommended), or a `v2` branch / new repo?
3. **v1 data** — migrate existing properties/buildings/units into a default workspace, or start v2 with clean seed data?
4. **3D models** — who produces the Blender/GLB assets for the demo property (adds 3–6 weeks if in-house)?
5. **Currency/time zone of the first real customer** — SRS reference uses IDR/Jakarta; the team is in Egypt (EGP / Africa/Cairo).

---

## 10. Phase 0 progress (2026-10-18)

Done in `pms-backend` (not committed yet):

| Piece | Where | Notes |
|---|---|---|
| Workspace BC | `modules/workspace/workspace-mgt` (new) | `POST/GET /workspaces`, `GET /workspaces/{id}` (members only, 404 otherwise), `PUT /workspaces/{id}/settings` (owner of the *active* workspace only). Creator becomes OWNER. |
| Membership | `user-mgt` | `membership(user_id, workspace_id, role, property_scope JSON, is_default)`; `GET/POST /memberships`, `PUT/DELETE /memberships/{id}`. Rules: managers can't grant OWNER, last owner can't be demoted or removed, all lookups confined to the active workspace. |
| Workspace in the token | `user-mgt` `AuthServiceImpl` | Login uses the default membership; refresh keeps the token's workspace while still a member; `POST /auth/workspace/switch` re-issues tokens and makes the target the default. |
| Workspace authorities | `security-adapter` | Filter grants `WS_<ROLE>` → controllers use `@Secured(WorkspaceRoles.OWNER_AUTHORITY)` etc. Legacy role-id authorities (`"1"`, `"2"`, `"3"`) still work. |
| Tenant filter | `sql-db-adapter` `WorkspaceTenantIdentifierResolver` | Hibernate `@TenantId` discriminator. Context workspace → filter value; **no workspace → `0` (fail closed, sees nothing, inserts fail on FK)**. Applied to `property`, `building`, `unit`. Covers HQL, criteria **and** `findById`. Native SQL is *not* filtered. |
| Request-context hygiene | `rest-adapter`, `mq-adapter` | `ThreadContext` was never cleared between requests on pooled threads (possible user/workspace bleed). Now cleared per HTTP request and per MQ message. |
| 404s | `security-adapter` | `NotFoundException` used to fall into the catch-all → 500 everywhere. Now 404. |
| Migrations | `workspace`, `user`, `property` changelogs; `main/_liquibase` runs `workspace` first | Default workspace `id=1`; every v1 user gets a membership (Property_Manager→OWNER, Superintendent→STAFF, Resident→RESIDENT); all v1 properties/buildings/units move to workspace 1. |

Verified: upgrade migration on a copy of the v1 seed; 28/28 live API checks (isolation by list and by id, cross-workspace update/delete blocked, role rules, switch/refresh); new unit tests for resolver, filter, auth and membership rules.

Still open for Phase 0:

- `tenant` (v1 resident) is not workspace-scoped yet — it is created from an MQ listener at registration with no workspace; it is replaced by `resident` in Phase 1.
- Freeze the out-of-scope modules (`lease`, `billing`, `maintenance`, `inspection`, `staff`, `vendor`, `chat`, `community`). Note: `@SpringBootApplication` sits in `com.pms`, so it component-scans **every** module regardless of the `@Import…Mgt` annotations — removing imports alone does not freeze anything.
- Frontend: workspace switcher, role mapping `owner/manager/staff/resident/viewer`, new shell and tokens.
- Platform super-admin, magic link, Google SSO, 2FA, CSV import.
