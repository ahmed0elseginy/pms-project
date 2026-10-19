# Spatial Property Platform — Software Requirements Specification

Oct 6, 2026 · @Ahmed El-Seginy

## 1. Introduction

The product is a web SaaS for residential property managers that replaces spreadsheets and floor-plan PDFs with an interactive isometric 3D model of the property, plus a booking module for shared community facilities. It is reverse-engineered from two reference demos (a 3D property explorer and a facility-booking screen) and written as a buildable spec.

### 1.1 Purpose

Define every screen, behaviour, data entity and quality requirement needed to build an MVP and a v1 of the platform, so a solo developer or a small team can estimate, build and test it.

### 1.2 Scope

In scope for v1:

- **Module A — Property 3D Explorer:** drill down Property → Tower → Floor → Unit → Furnished interior, with live occupancy and unit specs.
- **Module B — Facility Booking:** explore shared facilities in 3D, book time slots, view a day schedule, and approve or decline requests.
- **Shared shell:** workspaces, global search, KPI cards, export, roles and authentication.

Out of scope for v1: leasing contracts, rent collection and payments, maintenance tickets, a resident mobile app, and an in-browser 3D editor (models are prepared offline).

### 1.3 Definitions

| Term | Meaning |
| --- | --- |
| Workspace | A tenant (management company) that owns one or more properties |
| Property | A residential community, e.g. "Meridian Park", with an address and status |
| Tower | A building inside a property (code e.g. MDP-A, floors, units per floor) |
| Unit | A residence on a floor (e.g. A-1208), with area, bedrooms, layout, status |
| Facility | A bookable shared space (pool, court, BBQ area, meeting room, co-working, gym) |
| Slot | A bookable start time of a fixed duration (1, 2 or 3 hours) |
| Reservation | A booking of one facility for one slot by one resident |
| Hotspot | A clickable 3D object (tower, unit, room) linked to a record |
| Isometric view | The fixed-angle 3D camera style used across the product |

## 2. Overall Description

The system is a multi-tenant web application with a 3D viewer embedded in a dashboard layout: left icon rail, top bar, KPI row, 3D canvas on the left, contextual detail panel on the right, and a selectable card strip under the canvas.

### 2.1 User roles

| Role | Main goals | Key permissions |
| --- | --- | --- |
| Super admin (platform) | Run the SaaS | Manage workspaces, plans, global settings |
| Workspace owner | Run a management company | Manage properties, users, billing, branding |
| Property manager | Day-to-day operations | Edit towers/units, statuses, facilities; approve or decline bookings; export |
| Front desk / staff | Handle residents | View everything, create bookings on behalf of residents |
| Resident | Use the community | Browse facilities, book slots, see own reservations |
| Viewer / prospect (optional) | Explore available units | Read-only 3D explorer, vacant units only |

### 2.2 Operating environment

- Desktop browsers first (latest Chrome, Edge, Safari, Firefox) at 1280 px and wider; tablet supported; phone shows a simplified layout with the 3D canvas stacked above the panel.
- WebGL 2 required for 3D; a static isometric image fallback is shown when WebGL is unavailable.
- Cloud-hosted backend; 3D assets served from object storage behind a CDN.

### 2.3 Assumptions and dependencies

- 3D models (towers, floor plates, furnished interiors, facilities) are produced in Blender and exported as glTF/GLB; the app does not generate geometry from floor plans in v1.
- Each workspace supplies its own property data, or imports it from CSV.
- Time zone is per property (the reference shows Jakarta); currency is per workspace (reference shows IDR, e.g. Rp150.000).
- The UI language is English in v1, with i18n ready for Arabic (RTL) and others.

## 3. Module A — Property 3D Explorer

The explorer lets a manager go from the whole property to a single furnished unit in four clicks, with the right panel and the bottom card strip always showing the record behind the current 3D selection.

### 3.1 Navigation levels

1. **Property level:** isometric model of all towers on a landscaped site. Each tower has a floating label (name + subtitle, e.g. "Tower A · The Aspen").
2. **Tower level:** the selected tower is highlighted (label turns solid green, others stay white); the right panel shows tower details.
3. **Floor level:** an exploded floor plate with every unit drawn as a room box and labelled with code, layout and area (e.g. "A-1205 · 2BR · 96 m²").
4. **Unit level:** a furnished cut-away interior with room labels (Primary bedroom, Bedroom 2, Living & dining, Kitchen, Balcony).

A "Back to property" / "Back to floor plan" button and the breadcrumb (Portfolio › Property) move up one level.

### 3.2 Functional requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| A-01 | Show property header: name, status badge (Active), address, community type | Must |
| A-02 | Show 4 KPI cards: total towers, total units, occupancy % (with occupied count), vacant units | Must |
| A-03 | Render the property isometric 3D model with orbit (drag), zoom (scroll and +/−), reset view, fit-to-screen and fullscreen controls | Must |
| A-04 | Show floating 3D labels anchored to each tower; clicking a label or the mesh selects the tower | Must |
| A-05 | Hovering a tower highlights it (outline or tint) and shows a pointer cursor | Must |
| A-06 | Lighting preset selector (Daylight; later Dusk, Night) | Should |
| A-07 | Legend under the canvas: Occupied, Vacant, Reserved colours | Must |
| A-08 | "Your towers" card strip: one card per tower with thumbnail, name, floors, units, occupancy bar; selecting a card selects the tower in 3D and vice versa | Must |
| A-09 | Tower panel: name, subtitle, status (Active building), total units, floors, occupancy %; occupancy breakdown bar with Occupied/Vacant/Reserved counts | Must |
| A-10 | Tower panel "Building details": building code, property type, units per floor, total unit area (m²), unit layouts range (Studio – 3 bedroom) | Must |
| A-11 | "A closer look": a mini heat-grid of floors × units coloured by status; clicking a cell opens that floor/unit | Should |
| A-12 | "Explore floors" button switches to the floor explorer for the selected tower | Must |
| A-13 | Floor explorer panel: "Find your floor" grid of floor numbers (highest first), selected floor highlighted | Must |
| A-14 | Floor list: every unit with code, layout, area and status chip (Occupied, Vacant, Reserved) | Must |
| A-15 | Floor 3D view: exploded floor plate, unit boxes coloured/tagged by status, labels per unit; click a unit to open it | Must |
| A-16 | Bottom strip at floor level: unit cards for the floor (A-1201…A-1208) with status dot | Must |
| A-17 | Unit panel: code, layout + tower name, status chip, area m², bedrooms, bathrooms | Must |
| A-18 | Unit specifications: property, tower/floor, category, layout, facing (e.g. South · Garden view), unit status | Must |
| A-19 | Unit 3D view: furnished interior model with room labels; falls back to a generic layout model when no custom model exists | Must |
| A-20 | Camera transitions between levels are animated (≤ 800 ms ease-in-out) and keep the selected object centred | Should |
| A-21 | Every level is deep-linkable by URL (e.g. /p/meridian-park/t/A/f/12/u/A-1208) | Must |
| A-22 | Managers can change a unit's status from the panel; the change updates KPIs, colours and counts live | Must |
| A-23 | "Unit directory" tab: the same data as a sortable, filterable table (tower, floor, layout, area, status) | Must |
| A-24 | Export current property as CSV/XLSX and the current view as PNG/PDF | Should |

### 3.3 Business rules

- Occupancy % = occupied units ÷ total units, rounded to whole percent; reserved units count as not occupied.
- Unit status is one of: Occupied, Vacant, Reserved, Under maintenance (hidden from residents).
- Tower totals are always computed from units, never typed in by hand.

## 4. Module B — Facility Booking

Residents pick a shared space, a day, a duration and a start time in under 30 seconds; paid or private spaces go to a manager for approval, free shared spaces confirm instantly. The page header reads "More space. More possibilities." with a property selector and a "New reservation" button, and three tabs: Facility explorer, Reservations, Approvals (with a pending count badge).

### 4.1 KPI row

Four cards: Community facilities (count + "N ready to enjoy"), Today's reservations, Awaiting approval, Upcoming & confirmed.

### 4.2 Facility explorer — functional requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| B-01 | Hero area with facility tagline (e.g. "Your everyday escape."), short description and the facility's isometric 3D model with zoom/orbit/reset/fullscreen | Must |
| B-02 | Floating facility card on the canvas: icon, name, location (e.g. Clubhouse · Level 01), status chip (Ready to enjoy / Being refreshed) | Must |
| B-03 | "A space for every part of your day" strip: one card per facility with 3D thumbnail, name and price label (Complimentary or fee); selecting it swaps the 3D model and the booking panel | Must |
| B-04 | Booking panel header: facility name, location, capacity (Up to N people), opening hours | Must |
| B-05 | "Pick your day": 7-day date strip starting today, with weekday letter, date and a dot when bookings exist; a calendar icon opens a full date picker | Must |
| B-06 | "A little time for you": duration toggle 1 / 2 / 3 hours | Must |
| B-07 | "Choose a start time": grid of hourly slots inside opening hours, each showing availability; taken or past slots are disabled; header shows "N available" | Must |
| B-08 | Info note by booking type: "A shared space for everyone" (others may book the same time, up to capacity) vs "This space is just for your group" (exclusive) | Must |
| B-09 | Price row: "Included in your community · Complimentary" or "Reservation fee · Rp150.000" | Must |
| B-10 | "Reserve this space" button; disabled until a slot is chosen; shows a confirmation with the result (Confirmed / Pending approval) | Must |
| B-11 | "Thoughtfully equipped — It's all in the details": amenity checklist per facility (e.g. 25 m lap pool, sun loungers, changing rooms, outdoor showers) | Must |
| B-12 | "On the calendar" card: the selected day's bookings for this facility (time, resident) or an empty state | Should |
| B-13 | Maintenance mode: a facility under refresh shows "A little refresh is underway", hides slots and disables booking | Must |
| B-14 | Footer hint: approval and cancellation policy line under the button | Should |

### 4.3 Reservations tab

| ID | Requirement | Priority |
| --- | --- | --- |
| B-20 | Date navigator (previous / date picker / next / Today) | Must |
| B-21 | Day schedule: rows = facilities, columns = hours (06:00–22:00); reservations drawn as blocks with resident name and time; colour by status (Confirmed, Pending) | Must |
| B-22 | Facilities under maintenance show a striped row with "Reservations paused" | Must |
| B-23 | Toggle to a status board (Kanban: Pending, Confirmed, Completed, Cancelled) | Should |
| B-24 | Click a block to open reservation details; managers can cancel or move it | Must |

### 4.4 Approvals tab

| ID | Requirement | Priority |
| --- | --- | --- |
| B-30 | Header "Good plans, waiting for a yes." with the count of requests awaiting a decision | Must |
| B-31 | One card per pending request: facility thumbnail and name, Pending badge, resident name + unit, guest count, date, time range, the resident's note, and the suggested fee | Must |
| B-32 | "Approve reservation" and "Decline" actions; decline asks for an optional reason; both notify the resident | Must |
| B-33 | Dismiss (×) hides a card from the view without deciding | Could |

### 4.5 Booking rules

- A facility is either **shared** (capacity-based, many residents per slot) or **exclusive** (one group per slot).
- Each facility has: opening hours, allowed durations, capacity, price (0 = complimentary), approval required (yes/no), booking window (e.g. up to 14 days ahead), max active bookings per unit, and a cancellation cutoff.
- A booking cannot overlap an exclusive booking or push a shared slot above capacity; the server enforces this with a transactional check, not only the UI.
- Status lifecycle: Requested → Pending approval → Confirmed → Completed; side exits Declined, Cancelled, No-show. Free shared spaces skip Pending.
- Fees are shown and recorded in v1; online payment is a later phase.

## 5. Shared Shell

Every screen sits inside the same frame, so modules feel like one product.

| ID | Requirement | Priority |
| --- | --- | --- |
| S-01 | Left icon rail: logo, Dashboard/Explorer, Properties, Units grid, Bookings (with notification badge), Help, user avatar; vertical product label | Must |
| S-02 | Top bar: logo + product tag, workspace switcher ("My workspace"), global search "Search towers or units…" with ⌘K shortcut, demo-workspace indicator, help, user name and role | Must |
| S-03 | Global search returns towers, units, facilities and residents; selecting a result flies the 3D camera to it | Must |
| S-04 | Breadcrumbs on every page (Portfolio › Property › …) | Must |
| S-05 | Auth: email + password, magic link, Google SSO; optional 2FA for managers | Must |
| S-06 | Role-based access control per workspace and per property (section 2.1) | Must |
| S-07 | Notifications: in-app bell plus email for booking created, approved, declined, cancelled, reminder 1 h before | Must |
| S-08 | Audit log of status changes, approvals and edits (who, what, when) | Should |
| S-09 | Admin CRUD for properties, towers, floors, units, facilities, amenities, and CSV import | Must |
| S-10 | Upload and attach GLB models and thumbnails per property, tower, floor template, unit layout and facility | Must |
| S-11 | Demo workspace with seeded data, resettable, for sales demos | Should |
| S-12 | Settings: workspace branding (logo, accent colour), currency, time zone, language | Should |

## 6. Data Model

Every table carries `workspace_id` for tenant isolation (row-level security), plus `id` (UUID), `created_at`, `updated_at`.

| Entity | Key fields | Relations |
| --- | --- | --- |
| Workspace | name, slug, plan, currency, locale, branding | has many Users, Properties |
| User | name, email, password\_hash, avatar | many-to-many Workspace via Membership |
| Membership | role (owner, manager, staff, resident, viewer), property\_scope\[\] | User ↔ Workspace |
| Property | name, slug, address, city, country, type, status, timezone, model\_asset\_id, lat/lng | has many Towers, Facilities |
| Tower | code (MDP-A), name, subtitle, floors\_count, property\_type, model\_node\_key, sort\_order | belongs to Property; has many Floors |
| Floor | number, floor\_template\_asset\_id | belongs to Tower; has many Units |
| UnitLayout | name (Studio, 1BR, 2BR, 3BR), bedrooms, bathrooms, interior\_asset\_id | used by Units |
| Unit | code (A-1208), area\_m2, facing, category, status, model\_node\_key | belongs to Floor, UnitLayout; has Residents |
| Resident | user\_id, unit\_id, move\_in, move\_out, is\_primary | User ↔ Unit |
| Facility | name, tagline, description, location, capacity, open\_time, close\_time, booking\_type (shared/exclusive), durations\[\], price, requires\_approval, booking\_window\_days, status, model\_asset\_id, thumbnail | belongs to Property; has Amenities, Reservations |
| Amenity | label, icon | belongs to Facility |
| Reservation | facility\_id, unit\_id, user\_id, start\_at, end\_at, guests, note, fee, status, decided\_by, decided\_at, decline\_reason | belongs to Facility, Unit, User |
| FacilityClosure | facility\_id, start\_at, end\_at, reason | blocks booking |
| Asset | kind (glb, image), url, size, checksum, lod\_level | referenced by Property/Tower/Floor/Layout/Facility |
| AuditLog | actor\_id, entity, entity\_id, action, diff (JSON) | — |
| Notification | user\_id, type, payload, read\_at | — |

Indexes that matter: `unit(floor_id, status)`, `reservation(facility_id, start_at, end_at)` plus a Postgres exclusion constraint (`tstzrange` with `&&`) for exclusive facilities to make double-booking impossible at the database level.

## 7. API Overview

REST over HTTPS, JSON, versioned under `/api/v1`, JWT access token + refresh token; the workspace comes from the token, never from the request body. Real-time updates (status changes, new bookings) go over WebSocket or Server-Sent Events.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | /properties | List properties in the workspace |
| GET | /properties/{id} | Property with KPIs (towers, units, occupancy, vacant) |
| GET | /properties/{id}/scene | Scene manifest: asset URLs, node keys → record ids, label anchors, camera presets |
| GET | /towers/{id} | Tower details + occupancy breakdown + floor×unit status grid |
| GET | /towers/{id}/floors/{n} | Floor with units |
| GET | /units?property=&tower=&floor=&status=&layout= | Unit directory (paged, sortable) |
| GET / PATCH | /units/{id} | Unit specs; PATCH changes status |
| GET | /search?q= | Global search across towers, units, facilities, residents |
| GET | /properties/{id}/facilities | Facility cards with amenities and price |
| GET | /facilities/{id}/availability?date=&duration= | Slot grid with remaining capacity per slot |
| POST | /reservations | Create a booking (validates window, overlap, capacity, limits) |
| GET | /reservations?date=&facility=&status= | Day schedule and status board |
| PATCH | /reservations/{id} | Move or cancel |
| POST | /reservations/{id}/approve · /decline | Approval decision (manager only) |
| GET | /properties/{id}/booking-stats | Booking KPI cards |
| POST | /exports | Start CSV/XLSX/PDF export; returns a job id |
| POST | /assets | Signed upload URL for GLB/image |

Errors use RFC 7807 problem+json; booking conflicts return 409 with the conflicting slot.

## 8. Non-Functional Requirements

| ID | Category | Requirement |
| --- | --- | --- |
| N-01 | Performance | Dashboard first contentful paint < 1.5 s on broadband; property 3D scene interactive < 3 s with a placeholder shown at once |
| N-02 | Performance | 3D view holds ≥ 50 fps on a mid-range laptop (integrated GPU) and ≥ 30 fps on tablets |
| N-03 | 3D budget | Property scene ≤ 8 MB GLB (Draco/Meshopt compressed, KTX2 textures), ≤ 150k triangles; unit interior ≤ 3 MB; lazy-load each level |
| N-04 | API | p95 latency < 300 ms for reads, < 500 ms for booking creation |
| N-05 | Scalability | 1,000 workspaces, 500 properties per workspace, 5,000 units per property without redesign |
| N-06 | Availability | 99.5% monthly uptime for v1; daily backups, 30-day retention, tested restore |
| N-07 | Security | OWASP Top 10, tenant isolation tested, bcrypt/argon2 passwords, rate limiting on auth and booking, signed asset URLs |
| N-08 | Privacy | Residents see only their own bookings and first-name + unit of others; data export and deletion on request |
| N-09 | Concurrency | Two residents booking the last slot at the same time: exactly one succeeds, the other gets a clear 409 message |
| N-10 | Accessibility | WCAG 2.1 AA for all non-3D UI; every 3D action is also reachable through lists, cards and keyboard |
| N-11 | Localisation | All strings externalised; RTL layout for Arabic; locale-aware dates, numbers and currency |
| N-12 | Observability | Structured logs, error tracking, metrics for booking failures and 3D load time |
| N-13 | Maintainability | ≥ 70% unit-test coverage on booking and occupancy logic; E2E tests for the main flows |

## 9. UI/UX & Design System

The look is calm and premium: a soft sage-green surface, white cards with large radii, one deep green accent, and warm-toned isometric models with trees and soft shadows.

### 9.1 Layout

- 12-column grid; left rail 64 px; content max width \~1440 px.
- Explorer pages: canvas ≈ 65% width on the left, detail panel ≈ 35% on the right (sticky, scrolls independently), card strip under the canvas.
- KPI row of 4 equal cards above the canvas on every module.

### 9.2 Tokens (starting values)

| Token | Value | Use |
| --- | --- | --- |
| Primary | deep green ≈ #2F6B4F | Buttons, selected tower label, selected date |
| Surface | sage ≈ #EEF2EC | Page background, canvas background |
| Card | #FFFFFF | Cards, panels |
| Occupied | green ≈ #3E7C5A | Status |
| Vacant | light grey-green ≈ #CFD8CF | Status |
| Reserved | amber ≈ #E3A857 | Status, Pending badge |
| Radius | 12–16 px cards, 8 px chips | All surfaces |
| Type | Geometric sans (e.g. Manrope or Plus Jakarta Sans); 32–40 px headings, 13–14 px body | All text |

### 9.3 Interaction details

- Selected state = green border + light green fill on cards; solid green on 3D labels.
- Copy tone is warm and human ("A little time for you", "Good plans, waiting for a yes").
- Empty, loading (skeleton + model placeholder) and error states for every panel.
- Micro-interactions: hover lift on cards, 150–250 ms transitions, animated camera between levels.
- Status is never shown by colour alone; every chip also has a text label.

## 10. Tech Stack, 3D Pipeline & Phases

The recommended stack is Next.js + React Three Fiber on the front end and NestJS + PostgreSQL on the back end; the 3D pipeline, not the code, is the hardest part of the project.

### 10.1 Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Web app | Next.js (App Router), TypeScript, Tailwind, shadcn/ui | Fast dashboard UI, SSR for non-3D pages |
| 3D | three.js via React Three Fiber + drei (OrbitControls, Html labels, Bounds, useGLTF), postprocessing outline | Declarative 3D inside React; `Html` gives the floating labels |
| State / data | TanStack Query, Zustand for scene selection state | Sync panel ↔ 3D ↔ card strip from one store |
| Backend | NestJS, Prisma or TypeORM, class-validator | Modular, matches the API in section 7 |
| Database | PostgreSQL 16 (+ row-level security, exclusion constraints) | Tenant isolation and safe bookings |
| Cache / jobs | Redis + BullMQ | Availability cache, exports, reminders, emails |
| Storage | S3-compatible (S3, R2, MinIO) + CDN | GLB and image assets |
| Realtime | Socket.IO or SSE | Live status and booking updates |
| DevOps | Docker, GitHub Actions, preview deployments, Sentry | CI/CD and monitoring |

### 10.2 3D asset pipeline

1. Model the site, towers, a floor-plate template per tower, one interior per unit layout, and one model per facility in Blender (isometric-friendly, low-poly, baked lighting).
2. Name every clickable mesh with a stable key (e.g. `tower_A`, `unit_A-1208`, `room_kitchen`); the scene manifest maps keys to database ids.
3. Bake ambient occlusion and soft shadows into textures; use an orthographic or narrow-FOV camera for the isometric look.
4. Export GLB, compress with gltf-transform (Draco or Meshopt + KTX2), generate thumbnails by rendering each model once.
5. Upload through the admin; the app recolours unit meshes at runtime by status, so status changes never need a new model.

### 10.3 Delivery phases

1. **Phase 0 — Foundations (2 weeks):** auth, workspaces, roles, admin CRUD, CSV import, design system.
2. **Phase 1 — 3D explorer MVP (4 weeks):** property and tower levels, KPIs, tower panel, card strip, unit directory table.
3. **Phase 2 — Floors and units (3 weeks):** floor explorer, unit panel, status changes, interior models, deep links.
4. **Phase 3 — Facility booking (4 weeks):** explorer, availability, reservations, day schedule, approvals, notifications.
5. **Phase 4 — Polish (2 weeks):** exports, audit log, demo workspace, performance budget, accessibility, Arabic RTL.

Estimates assume one full-stack developer working full time with 3D models supplied or bought; add 3–6 weeks if you model everything yourself.

## النسخة العربية — وثيقة مواصفات متطلبات البرمجيات (SRS)

### 1. المقدمة

المنتج منصة SaaS على الويب لمديري العقارات السكنية، بتستبدل الإكسل وملفات المساقط الـPDF بنموذج ثلاثي الأبعاد تفاعلي (Isometric) للمشروع، ومعاه موديول لحجز المرافق المشتركة. الوثيقة مبنية على تحليل فيديوهين مرجعيين (مستكشف عقار 3D وشاشة حجز مرافق) ومكتوبة بحيث تتنفذ مباشرة.

**الهدف:** تحديد كل شاشة وسلوك وكيان بيانات ومتطلب جودة لازم لبناء MVP ثم الإصدار الأول، علشان مطور لوحده أو فريق صغير يقدر يقدّر الوقت ويبني ويختبر.

**النطاق في الإصدار الأول:**

- **موديول A — مستكشف العقار 3D:** التنقل من المشروع ← البرج ← الدور ← الوحدة ← التصميم الداخلي المفروش، مع نسب الإشغال ومواصفات الوحدة لحظيًا.
- **موديول B — حجز المرافق:** استكشاف المرافق 3D، حجز مواعيد، جدول اليوم، وقبول أو رفض الطلبات.
- **الإطار المشترك:** مساحات العمل، البحث العام، كروت المؤشرات، التصدير، الصلاحيات وتسجيل الدخول.

**خارج النطاق:** عقود الإيجار، تحصيل الإيجار والدفع، تذاكر الصيانة، تطبيق موبايل للسكان، ومحرر 3D داخل المتصفح (النماذج بتتجهز برا السيستم).

| المصطلح | المعنى |
| --- | --- |
| Workspace (مساحة عمل) | شركة إدارة (مستأجر في السيستم) تمتلك عقار أو أكتر |
| Property (مشروع) | مجمع سكني مثل "Meridian Park" له عنوان وحالة |
| Tower (برج) | مبنى داخل المشروع، له كود (MDP-A) وعدد أدوار ووحدات لكل دور |
| Unit (وحدة) | شقة في دور (مثل A-1208) لها مساحة وغرف وتصميم وحالة |
| Facility (مرفق) | مكان مشترك قابل للحجز: حمام سباحة، ملعب، منطقة شوي، قاعة اجتماعات، Co-working، جيم |
| Slot (فترة) | وقت بداية متاح لمدة ثابتة (ساعة، ساعتين، 3 ساعات) |
| Reservation (حجز) | حجز مرفق واحد لفترة واحدة بواسطة ساكن واحد |
| Hotspot | عنصر 3D قابل للضغط (برج، وحدة، غرفة) مربوط بسجل في الداتابيز |

### 2. الوصف العام

السيستم ويب متعدد المستأجرين (Multi-tenant)، والعارض الـ3D جوه داشبورد: شريط أيقونات يسار، شريط علوي، صف مؤشرات KPI، الكانفاس الـ3D على اليسار، لوحة تفاصيل على اليمين، وشريط كروت تحت الكانفاس.

| الدور | الهدف | الصلاحيات الأساسية |
| --- | --- | --- |
| Super admin | تشغيل المنصة | إدارة مساحات العمل والباقات والإعدادات العامة |
| مالك مساحة العمل | إدارة شركته | إدارة المشاريع والمستخدمين والفوترة والهوية البصرية |
| مدير العقار | التشغيل اليومي | تعديل الأبراج والوحدات والحالات والمرافق، قبول ورفض الحجوزات، التصدير |
| الاستقبال / الموظفين | خدمة السكان | عرض كل شيء وعمل حجوزات نيابة عن السكان |
| الساكن | استخدام المرافق | تصفح المرافق، الحجز، متابعة حجوزاته |
| زائر / عميل محتمل (اختياري) | استكشاف الوحدات المتاحة | عرض 3D للقراءة فقط للوحدات الفاضية |

**بيئة التشغيل:** متصفحات الديسكتوب أولًا (Chrome, Edge, Safari, Firefox) من عرض 1280px، التابلت مدعوم، والموبايل بتصميم مبسّط. لازم WebGL 2، ولو مش متاح تظهر صورة Isometric ثابتة بديلة. الباك إند على السحابة والنماذج الـ3D على Object Storage وراه CDN.

**الافتراضات:** النماذج بتتعمل على Blender وتتصدّر GLB؛ كل شركة بتدخل بياناتها أو تستوردها CSV؛ المنطقة الزمنية لكل مشروع والعملة لكل مساحة عمل؛ الواجهة إنجليزي في البداية ومجهزة للعربي (RTL).

### 3. موديول A — مستكشف العقار 3D

المدير يقدر يوصل من المشروع كله لشقة مفروشة واحدة في 4 ضغطات، واللوحة اليمين وشريط الكروت دايمًا بيعرضوا السجل الخاص بالاختيار الحالي في الـ3D.

**مستويات التنقل:**

1. **المشروع:** نموذج لكل الأبراج على موقع فيه زرع، وفوق كل برج ليبل عائم (الاسم + وصف، مثل "Tower A · The Aspen").
2. **البرج:** البرج المختار يتعلّم (الليبل يبقى أخضر والباقي أبيض) واللوحة تعرض تفاصيله.
3. **الدور:** مسقط الدور مفكوك، كل وحدة صندوق عليه الكود والتصميم والمساحة (مثل "A-1205 · 2BR · 96 m²").
4. **الوحدة:** مقطع داخلي مفروش بأسماء الغرف (غرفة النوم الرئيسية، غرفة 2، المعيشة والسفرة، المطبخ، البلكونة).

| ID | المتطلب | الأولوية |
| --- | --- | --- |
| A-01 | هيدر المشروع: الاسم، شارة الحالة (Active)، العنوان، نوع المجتمع | أساسي |
| A-02 | 4 كروت KPI: عدد الأبراج، عدد الوحدات، نسبة الإشغال (وعدد المشغول)، الوحدات الفاضية | أساسي |
| A-03 | عرض نموذج المشروع 3D مع تدوير بالسحب، زوم (سكرول و+/−)، Reset، ملء الشاشة | أساسي |
| A-04 | ليبلات عائمة مثبتة على كل برج؛ الضغط على الليبل أو المبنى يختاره | أساسي |
| A-05 | عند المرور على البرج يتعلّم (Outline) والماوس يتغير | أساسي |
| A-06 | اختيار الإضاءة (نهار؛ لاحقًا غروب وليل) | مهم |
| A-07 | مفتاح ألوان: مشغول، فاضي، محجوز | أساسي |
| A-08 | شريط "أبراجك": كارت لكل برج فيه صورة واسم وأدوار ووحدات وشريط إشغال، والاختيار متزامن مع الـ3D في الاتجاهين | أساسي |
| A-09 | لوحة البرج: الاسم، الحالة، إجمالي الوحدات، الأدوار، نسبة الإشغال، وشريط تفصيل الإشغال بالأعداد | أساسي |
| A-10 | تفاصيل المبنى: الكود، نوع العقار، وحدات لكل دور، إجمالي المساحة م²، مدى التصاميم (ستوديو – 3 غرف) | أساسي |
| A-11 | "نظرة أقرب": شبكة مصغّرة أدوار × وحدات ملوّنة بالحالة، والضغط على خانة يفتحها | مهم |
| A-12 | زرار "استكشف الأدوار" يفتح مستكشف الأدوار للبرج | أساسي |
| A-13 | شبكة "اختار دورك" بأرقام الأدوار (الأعلى أولًا) والدور المختار متعلّم | أساسي |
| A-14 | قائمة وحدات الدور: الكود، التصميم، المساحة، شارة الحالة | أساسي |
| A-15 | عرض الدور 3D مع تلوين الوحدات بالحالة وليبل لكل وحدة؛ الضغط يفتح الوحدة | أساسي |
| A-16 | شريط كروت وحدات الدور (A-1201…A-1208) بنقطة حالة | أساسي |
| A-17 | لوحة الوحدة: الكود، التصميم واسم البرج، الحالة، المساحة، عدد الغرف والحمامات | أساسي |
| A-18 | مواصفات الوحدة: المشروع، البرج/الدور، الفئة، التصميم، الاتجاه (جنوب · إطلالة حديقة)، الحالة | أساسي |
| A-19 | الوحدة 3D مفروشة بأسماء الغرف؛ ولو مفيش نموذج خاص يظهر نموذج التصميم العام | أساسي |
| A-20 | انتقال الكاميرا بين المستويات متحرك (≤ 800ms) والعنصر المختار في النص | مهم |
| A-21 | كل مستوى له رابط مباشر (مثل /p/meridian-park/t/A/f/12/u/A-1208) | أساسي |
| A-22 | المدير يغيّر حالة الوحدة من اللوحة، والمؤشرات والألوان تتحدث لحظيًا | أساسي |
| A-23 | تبويب "دليل الوحدات": نفس البيانات في جدول قابل للترتيب والفلترة | أساسي |
| A-24 | تصدير المشروع CSV/XLSX والعرض الحالي PNG/PDF | مهم |

**قواعد العمل:** نسبة الإشغال = الوحدات المشغولة ÷ الإجمالي (المحجوز لا يُحسب مشغول). حالات الوحدة: مشغولة، فاضية، محجوزة، تحت الصيانة (مخفية عن السكان). إجماليات البرج دايمًا محسوبة من الوحدات ومش بتتكتب يدوي.

### 4. موديول B — حجز المرافق

الساكن يختار المكان واليوم والمدة ووقت البداية في أقل من 30 ثانية؛ المرافق المدفوعة أو الخاصة تروح للمدير للموافقة، والمرافق المشتركة المجانية تتأكد فورًا. الهيدر فيه "More space. More possibilities." مع اختيار المشروع وزرار "حجز جديد"، و3 تبويبات: مستكشف المرافق، الحجوزات، الموافقات (بعداد الطلبات المعلقة).

**صف المؤشرات:** عدد المرافق، حجوزات النهارده، في انتظار الموافقة، القادمة والمؤكدة.

| ID | المتطلب | الأولوية |
| --- | --- | --- |
| B-01 | منطقة رئيسية فيها شعار المرفق (مثل "Your everyday escape.") ووصف ونموذج 3D مع زوم وتدوير وReset وملء الشاشة | أساسي |
| B-02 | كارت عائم على الكانفاس: أيقونة، اسم، مكان (Clubhouse · Level 01)، حالة (جاهز / تحت التجديد) | أساسي |
| B-03 | شريط "مكان لكل جزء من يومك": كارت لكل مرفق بصورة 3D واسم وسعر (مجاني أو رسوم)، والاختيار يغيّر النموذج ولوحة الحجز | أساسي |
| B-04 | هيدر لوحة الحجز: الاسم، المكان، السعة (حتى N أشخاص)، مواعيد العمل | أساسي |
| B-05 | "اختار يومك": شريط 7 أيام من النهارده بحرف اليوم والتاريخ ونقطة لو فيه حجوزات، وأيقونة تقويم كامل | أساسي |
| B-06 | "وقت ليك": اختيار المدة ساعة / ساعتين / 3 ساعات | أساسي |
| B-07 | "اختار وقت البداية": شبكة فترات بالساعة داخل مواعيد العمل، المحجوز والفايت معطّل، والعنوان يعرض "N متاح" | أساسي |
| B-08 | ملاحظة حسب النوع: "مكان مشترك للكل" (أكتر من حد في نفس الوقت لحد السعة) أو "المكان ده لمجموعتك بس" (حصري) | أساسي |
| B-09 | سطر السعر: "مشمول في مجتمعك · مجاني" أو "رسوم الحجز · Rp150.000" | أساسي |
| B-10 | زرار "احجز المكان" معطّل لحد اختيار فترة، ويعرض النتيجة (مؤكد / في انتظار الموافقة) | أساسي |
| B-11 | "مجهز بعناية": قائمة تجهيزات لكل مرفق (حمام 25م، كراسي شمس، غرف تغيير، دُش خارجي) | أساسي |
| B-12 | كارت "على التقويم": حجوزات اليوم المختار للمرفق أو حالة فاضية | مهم |
| B-13 | وضع الصيانة: "تجديد بسيط جاري" ويخفي الفترات ويقفل الحجز | أساسي |
| B-14 | سطر سياسة الموافقة والإلغاء تحت الزرار | مهم |
| B-20 | تبويب الحجوزات: تنقل بالتاريخ (السابق / التقويم / التالي / النهارده) | أساسي |
| B-21 | جدول اليوم: صفوف = المرافق، أعمدة = الساعات (06:00–22:00)، الحجوزات بلوكات باسم الساكن والوقت وملوّنة بالحالة | أساسي |
| B-22 | المرافق تحت الصيانة صف مخطط مكتوب عليه "الحجوزات متوقفة" | أساسي |
| B-23 | تبديل لعرض لوحة حالات (Kanban: معلق، مؤكد، مكتمل، ملغي) | مهم |
| B-24 | الضغط على بلوك يفتح التفاصيل، والمدير يقدر يلغي أو ينقل | أساسي |
| B-30 | تبويب الموافقات: "خطط حلوة مستنية موافقة" وعدد الطلبات | أساسي |
| B-31 | كارت لكل طلب: صورة واسم المرفق، شارة معلق، اسم الساكن ووحدته، عدد الضيوف، التاريخ والوقت، ملاحظة الساكن، الرسوم المقترحة | أساسي |
| B-32 | زرار "موافقة" و"رفض" (مع سبب اختياري) والاتنين يبعتوا إشعار للساكن | أساسي |
| B-33 | إخفاء الكارت (×) من غير قرار | اختياري |

**قواعد الحجز:**

- المرفق إما **مشترك** (بسعة، أكتر من ساكن في نفس الفترة) أو **حصري** (مجموعة واحدة).
- لكل مرفق: مواعيد عمل، مدد مسموحة، سعة، سعر (0 = مجاني)، يحتاج موافقة أو لا، نافذة حجز (مثلًا 14 يوم قدام)، أقصى عدد حجوزات نشطة لكل وحدة، ومهلة إلغاء.
- ممنوع التداخل مع حجز حصري أو تخطي السعة، والسيرفر بيتحقق ده داخل Transaction مش الواجهة بس.
- دورة الحالة: مطلوب ← في انتظار الموافقة ← مؤكد ← مكتمل؛ ومخارج جانبية: مرفوض، ملغي، لم يحضر. المرافق المشتركة المجانية بتعدي مرحلة الانتظار.
- الرسوم بتتعرض وتتسجل في الإصدار الأول، والدفع أونلاين مرحلة لاحقة.

### 5. الإطار المشترك

| ID | المتطلب | الأولوية |
| --- | --- | --- |
| S-01 | شريط أيقونات يسار: اللوجو، الداشبورد، المشاريع، الوحدات، الحجوزات (بعداد إشعارات)، المساعدة، صورة المستخدم | أساسي |
| S-02 | شريط علوي: اللوجو، اختيار مساحة العمل، بحث عام "ابحث عن برج أو وحدة" باختصار ⌘K، مؤشر الديمو، اسم المستخدم ودوره | أساسي |
| S-03 | البحث يرجّع أبراج ووحدات ومرافق وسكان، والاختيار يحرّك الكاميرا للعنصر | أساسي |
| S-04 | Breadcrumbs في كل صفحة | أساسي |
| S-05 | تسجيل دخول بإيميل وباسورد، Magic link، Google SSO، و2FA اختياري للمديرين | أساسي |
| S-06 | صلاحيات حسب الدور لكل مساحة عمل ولكل مشروع | أساسي |
| S-07 | إشعارات داخل التطبيق وإيميل: حجز جديد، موافقة، رفض، إلغاء، تذكير قبلها بساعة | أساسي |
| S-08 | سجل تدقيق (Audit log) لتغييرات الحالة والموافقات والتعديلات | مهم |
| S-09 | لوحة إدارة CRUD للمشاريع والأبراج والأدوار والوحدات والمرافق والتجهيزات، واستيراد CSV | أساسي |
| S-10 | رفع نماذج GLB وصور لكل مشروع وبرج وقالب دور وتصميم وحدة ومرفق | أساسي |
| S-11 | مساحة عمل ديمو ببيانات جاهزة قابلة لإعادة الضبط | مهم |
| S-12 | الإعدادات: الهوية (لوجو ولون)، العملة، المنطقة الزمنية، اللغة | مهم |

### 6. نموذج البيانات

كل جدول فيه `workspace_id` لعزل بيانات كل شركة (Row-Level Security)، بالإضافة لـ`id` (UUID) و`created_at` و`updated_at`.

| الكيان | أهم الحقول | العلاقات |
| --- | --- | --- |
| Workspace | الاسم، slug، الباقة، العملة، اللغة، الهوية | له Users وProperties |
| User | الاسم، الإيميل، الباسورد المشفر، الصورة | مرتبط بـWorkspace عن طريق Membership |
| Membership | الدور، نطاق المشاريع المسموح | User ↔ Workspace |
| Property | الاسم، العنوان، المدينة، النوع، الحالة، المنطقة الزمنية، النموذج 3D، الإحداثيات | له Towers وFacilities |
| Tower | الكود، الاسم، الوصف، عدد الأدوار، نوع العقار، مفتاح العنصر في النموذج، الترتيب | يتبع Property؛ له Floors |
| Floor | الرقم، قالب الدور 3D | يتبع Tower؛ له Units |
| UnitLayout | الاسم (ستوديو، 1-3 غرف)، الغرف، الحمامات، النموذج الداخلي | تستخدمه Units |
| Unit | الكود، المساحة م²، الاتجاه، الفئة، الحالة، مفتاح العنصر | يتبع Floor وUnitLayout؛ له Residents |
| Resident | المستخدم، الوحدة، تاريخ الدخول والخروج، ساكن أساسي | User ↔ Unit |
| Facility | الاسم، الشعار، الوصف، المكان، السعة، مواعيد الفتح والقفل، النوع (مشترك/حصري)، المدد، السعر، يحتاج موافقة، نافذة الحجز، الحالة، النموذج، الصورة | يتبع Property؛ له Amenities وReservations |
| Amenity | النص، الأيقونة | يتبع Facility |
| Reservation | المرفق، الوحدة، المستخدم، البداية، النهاية، الضيوف، الملاحظة، الرسوم، الحالة، مين قرر ووقته، سبب الرفض | يتبع Facility وUnit وUser |
| FacilityClosure | المرفق، البداية، النهاية، السبب | يقفل الحجز |
| Asset | النوع (glb/صورة)، الرابط، الحجم، checksum، مستوى التفاصيل | مرتبط بالمشروع والبرج والدور والتصميم والمرفق |
| AuditLog | الفاعل، الكيان، العملية، الفرق (JSON) | — |
| Notification | المستخدم، النوع، المحتوى، وقت القراءة | — |

فهارس مهمة: `unit(floor_id, status)` و`reservation(facility_id, start_at, end_at)` مع Exclusion constraint في Postgres على `tstzrange` للمرافق الحصرية علشان الحجز المزدوج يبقى مستحيل من الداتابيز نفسها.

### 7. واجهات الـAPI

REST على HTTPS بصيغة JSON تحت `/api/v1`، مع JWT وRefresh token؛ مساحة العمل بتتاخد من التوكن مش من الطلب. التحديثات اللحظية عن طريق WebSocket أو SSE.

| Method | Endpoint | الغرض |
| --- | --- | --- |
| GET | /properties | قائمة المشاريع |
| GET | /properties/{id} | المشروع مع المؤشرات |
| GET | /properties/{id}/scene | ملف المشهد: روابط النماذج، ربط مفاتيح العناصر بالسجلات، أماكن الليبلات، زوايا الكاميرا |
| GET | /towers/{id} | تفاصيل البرج وتوزيع الإشغال وشبكة الأدوار × الوحدات |
| GET | /towers/{id}/floors/{n} | الدور ووحداته |
| GET | /units?… | دليل الوحدات بفلترة وترتيب وصفحات |
| GET / PATCH | /units/{id} | مواصفات الوحدة؛ PATCH لتغيير الحالة |
| GET | /search?q= | البحث العام |
| GET | /properties/{id}/facilities | كروت المرافق بالتجهيزات والسعر |
| GET | /facilities/{id}/availability?date=&duration= | الفترات المتاحة والسعة المتبقية |
| POST | /reservations | إنشاء حجز مع كل التحققات |
| GET | /reservations?… | جدول اليوم ولوحة الحالات |
| PATCH | /reservations/{id} | نقل أو إلغاء |
| POST | /reservations/{id}/approve · /decline | قرار الموافقة (للمدير فقط) |
| GET | /properties/{id}/booking-stats | مؤشرات الحجز |
| POST | /exports | بدء تصدير وإرجاع رقم المهمة |
| POST | /assets | رابط رفع موقّع للنماذج والصور |

الأخطاء بصيغة RFC 7807، وتعارض الحجز يرجّع 409 مع الفترة المتعارضة.

### 8. المتطلبات غير الوظيفية

| ID | الفئة | المتطلب |
| --- | --- | --- |
| N-01 | الأداء | الداشبورد يظهر في أقل من 1.5 ثانية، والمشهد 3D تفاعلي في أقل من 3 ثواني مع Placeholder فوري |
| N-02 | الأداء | 50 إطار/ثانية على الأقل في لابتوب متوسط و30 على التابلت |
| N-03 | ميزانية الـ3D | مشهد المشروع ≤ 8MB GLB مضغوط (Draco/Meshopt + KTX2) و≤ 150 ألف مثلث؛ الوحدة ≤ 3MB؛ تحميل كل مستوى عند الحاجة |
| N-04 | الـAPI | p95 أقل من 300ms للقراءة و500ms لإنشاء الحجز |
| N-05 | التوسع | 1000 مساحة عمل، 500 مشروع لكل منها، 5000 وحدة لكل مشروع |
| N-06 | الإتاحة | 99.5% شهريًا، نسخ احتياطي يومي لمدة 30 يوم مع اختبار الاسترجاع |
| N-07 | الأمان | OWASP Top 10، اختبار عزل المستأجرين، تشفير الباسوردات bcrypt/argon2، Rate limiting، روابط نماذج موقّعة |
| N-08 | الخصوصية | الساكن يشوف حجوزاته بس، وللآخرين الاسم الأول والوحدة فقط؛ تصدير ومسح البيانات عند الطلب |
| N-09 | التزامن | لو اتنين حجزوا آخر فترة في نفس اللحظة، واحد بس ينجح والتاني ياخد رسالة 409 واضحة |
| N-10 | الإتاحة لذوي الإعاقة | WCAG 2.1 AA لكل الواجهة غير الـ3D، وكل فعل في الـ3D متاح من القوائم والكيبورد |
| N-11 | التعريب | كل النصوص خارج الكود، تصميم RTL للعربي، تواريخ وأرقام وعملة حسب اللغة |
| N-12 | المراقبة | Logs منظمة، تتبع أخطاء، مقاييس لفشل الحجز ووقت تحميل الـ3D |
| N-13 | الصيانة | تغطية اختبارات ≥ 70% لمنطق الحجز والإشغال، واختبارات E2E للمسارات الرئيسية |

### 9. تجربة المستخدم ونظام التصميم

الشكل هادي وفخم: خلفية أخضر مريمي فاتح، كروت بيضا بحواف دائرية كبيرة، لون مميز واحد أخضر غامق، ونماذج Isometric بألوان دافية وشجر وظلال ناعمة.

- **التخطيط:** شبكة 12 عمود، شريط يسار 64px، عرض أقصى \~1440px؛ الكانفاس ≈ 65% يسار واللوحة ≈ 35% يمين (ثابتة وبتعمل سكرول لوحدها)، وشريط الكروت تحت الكانفاس، و4 كروت مؤشرات فوق في كل موديول.
- **الألوان:** الأساسي أخضر غامق ≈ #2F6B4F، الخلفية ≈ #EEF2EC، الكروت أبيض، مشغول ≈ #3E7C5A، فاضي ≈ #CFD8CF، محجوز/معلق ≈ #E3A857.
- **الحواف والخط:** 12–16px للكروت و8px للشارات؛ خط Geometric sans (Manrope أو Plus Jakarta Sans)، عناوين 32–40px ونص 13–14px.
- **الاختيار:** حدود خضرا وخلفية خضرا فاتحة للكروت، وليبل أخضر مصمت في الـ3D.
- **نبرة الكلام:** دافية وإنسانية ("وقت ليك"، "خطط حلوة مستنية موافقة").
- **الحالات:** فاضي وتحميل (Skeleton) وخطأ لكل لوحة؛ حركات 150–250ms؛ الكاميرا بتتحرك بين المستويات؛ الحالة عمرها ما تتعرض باللون بس، لازم نص معاه.

### 10. التقنيات وخط إنتاج الـ3D والمراحل

المقترح Next.js مع React Three Fiber في الواجهة، وNestJS مع PostgreSQL في الباك إند؛ وأصعب جزء في المشروع هو تجهيز النماذج الـ3D مش الكود.

| الطبقة | الاختيار | السبب |
| --- | --- | --- |
| الواجهة | Next.js (App Router)، TypeScript، Tailwind، shadcn/ui | داشبورد سريع وSSR للصفحات العادية |
| الـ3D | three.js عن طريق React Three Fiber + drei (OrbitControls، Html، Bounds، useGLTF) + Outline | 3D جوه React، و`Html` بيعمل الليبلات العائمة |
| الحالة والبيانات | TanStack Query وZustand لحالة الاختيار | مزامنة اللوحة والـ3D والكروت من مكان واحد |
| الباك إند | NestJS مع Prisma أو TypeORM وclass-validator | موديولر ومطابق للـAPI |
| الداتابيز | PostgreSQL 16 مع RLS وExclusion constraints | عزل المستأجرين وحجز آمن |
| الكاش والمهام | Redis + BullMQ | كاش الإتاحة، التصدير، التذكيرات، الإيميلات |
| التخزين | S3 أو R2 أو MinIO + CDN | النماذج والصور |
| اللحظي | Socket.IO أو SSE | تحديثات الحالة والحجوزات |
| DevOps | Docker، GitHub Actions، Preview deployments، Sentry | CI/CD والمراقبة |

**خط إنتاج الـ3D:**

1. نمذجة الموقع والأبراج وقالب دور لكل برج وداخلية لكل تصميم وحدة ونموذج لكل مرفق على Blender (Low-poly وإضاءة مخبوزة).
2. تسمية كل عنصر قابل للضغط بمفتاح ثابت (`tower_A`، `unit_A-1208`، `room_kitchen`) وملف المشهد بيربط المفاتيح بسجلات الداتابيز.
3. خَبز الـAmbient occlusion والظلال في التكستشر، واستخدام كاميرا Orthographic أو زاوية ضيقة للشكل الـIsometric.
4. التصدير GLB والضغط بـgltf-transform (Draco أو Meshopt + KTX2) وتوليد صور مصغرة.
5. الرفع من لوحة الإدارة؛ التطبيق بيلوّن الوحدات حسب الحالة وقت التشغيل، فتغيير الحالة مش محتاج نموذج جديد.

**مراحل التنفيذ:**

1. **المرحلة 0 — الأساسيات (أسبوعين):** تسجيل الدخول، مساحات العمل، الصلاحيات، لوحة الإدارة، استيراد CSV، نظام التصميم.
2. **المرحلة 1 — MVP المستكشف 3D (4 أسابيع):** مستوى المشروع والبرج، المؤشرات، لوحة البرج، شريط الكروت، جدول الوحدات.
3. **المرحلة 2 — الأدوار والوحدات (3 أسابيع):** مستكشف الأدوار، لوحة الوحدة، تغيير الحالة، النماذج الداخلية، الروابط المباشرة.
4. **المرحلة 3 — حجز المرافق (4 أسابيع):** المستكشف، الإتاحة، الحجوزات، جدول اليوم، الموافقات، الإشعارات.
5. **المرحلة 4 — التشطيب (أسبوعين):** التصدير، سجل التدقيق، الديمو، ميزانية الأداء، الإتاحة، العربي RTL.

التقديرات على أساس مطور Full-stack واحد بدوام كامل والنماذج جاهزة أو مشتراة؛ زوّد 3–6 أسابيع لو هتعمل النماذج بنفسك.
