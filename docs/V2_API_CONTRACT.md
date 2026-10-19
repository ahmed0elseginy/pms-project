# PMS v2 — API Contract (explorer + booking)

Shared contract between `pms-backend` (branch `v2/backend`) and `pms-frontend` (branch `v2/frontend`).
Source of truth for behaviour: `docs/Spatial Property Platform — SRS (EN AR).md` §3–§7. Architecture: `docs/PMS_V2_BOUNDED_CONTEXTS.md`.
Date: 2026-10-18. If either side needs to change a shape, edit this file in the same commit and say so in the commit message.

## Conventions

- All paths are under the existing backend context `/api` (frontend calls `/api/...` through the Next.js rewrite).
- JSON is camelCase. IDs are `int64` numbers. Dates `YYYY-MM-DD`, times `HH:mm`, instants ISO-8601 with offset.
- Workspace comes from the JWT (`WORKSPACE_ID`); never from the body. Every new table carries `workspace_id` and uses the Hibernate `@TenantId` filter added in Phase 0.
- Lists: `{ "data": [...], "total": n }` with `pageNum`, `pageSize`, `orderBy`, `orderDir` query params.
- Errors: `ErrorVTO` (`code`, `message`, `messageEn`, `messageAr`, `reqBodyErrors`). `message` is in the language of the request's `Accept-Language` header (`ar`, `ar-EG`, `ar-SA,ar;q=0.9` → Arabic; anything else → English); `messageEn`/`messageAr` are always both present. Booking conflicts → **409** with an `ErrorVTO`.
- Language: the frontend sends `Accept-Language: en` or `ar` on every call.
- Enums are UPPER_SNAKE on the wire. Frontend maps them to lower-case string unions.

## Enums

| Name | Values |
|---|---|
| UnitStatus | `OCCUPIED`, `VACANT`, `RESERVED`, `UNDER_MAINTENANCE` |
| UnitLayout | `STUDIO`, `ONE_BR`, `TWO_BR`, `THREE_BR`, `FOUR_BR` |
| FacilityBookingType | `SHARED`, `EXCLUSIVE` |
| FacilityStatus | `ACTIVE`, `MAINTENANCE` |
| ReservationStatus | `PENDING`, `CONFIRMED`, `COMPLETED`, `DECLINED`, `CANCELLED`, `NO_SHOW` |
| SearchResultType | `TOWER`, `UNIT`, `FACILITY`, `RESIDENT` |

## Module A — Property 3D explorer

| Method | Path | Response |
|---|---|---|
| GET | `/properties/{propertyId}/explorer` | `PropertyExplorerVTO` |
| GET | `/properties/{propertyId}/scene` | `SceneManifestVTO` |
| GET | `/towers/{towerId}` | `TowerDetailVTO` |
| GET | `/towers/{towerId}/floors/{floorNumber}` | `FloorVTO` |
| GET | `/units?propertyId=&towerId=&floor=&status=&layout=&q=` | `UnitResultSet` (`data: UnitSummaryVTO[]`) |
| GET | `/units/{unitId}` | `UnitDetailVTO` |
| PATCH | `/units/{unitId}/status` | body `{ "status": UnitStatus }` → `UnitDetailVTO` (manager/owner only) |
| GET | `/search?q=&limit=` | `{ "data": SearchResultVTO[] }` |

```jsonc
// PropertyExplorerVTO
{
  "property": { "id": 1, "name": "Meridian Park", "slug": "meridian-park", "status": "ACTIVE",
                "address": "Jl. Senopati No. 88, South Jakarta", "communityType": "Residential community" },
  "kpis": { "totalTowers": 4, "totalUnits": 624, "occupiedUnits": 512, "occupancyPct": 82, "vacantUnits": 74, "reservedUnits": 30 },
  "towers": [ /* TowerSummaryVTO */ ]
}

// TowerSummaryVTO
{ "id": 10, "code": "A", "buildingCode": "MDP-A", "name": "Tower A", "subtitle": "The Aspen",
  "floorsCount": 24, "totalUnits": 192, "occupied": 158, "vacant": 23, "reserved": 11, "underMaintenance": 0,
  "occupancyPct": 82, "status": "ACTIVE", "modelNodeKey": "tower_A", "sortOrder": 1 }

// TowerDetailVTO = TowerSummaryVTO +
{ "propertyType": "Residential", "unitsPerFloor": 8, "totalUnitAreaM2": 13728,
  "layoutRange": { "min": "STUDIO", "max": "THREE_BR" },
  "statusGrid": [ { "floor": 24, "units": [ { "id": 1, "code": "A-2401", "status": "OCCUPIED" } ] } ] } // highest floor first

// FloorVTO
{ "towerId": 10, "towerCode": "A", "number": 12, "units": [ /* UnitSummaryVTO */ ] }

// UnitSummaryVTO
{ "id": 1208, "code": "A-1208", "towerId": 10, "towerCode": "A", "floor": 12, "layout": "THREE_BR",
  "areaM2": 108, "status": "OCCUPIED", "modelNodeKey": "unit_A-1208" }

// UnitDetailVTO = UnitSummaryVTO +
{ "bedrooms": 3, "bathrooms": 1, "facing": "South · Garden view", "category": "Residential",
  "towerName": "Tower A", "towerSubtitle": "The Aspen", "propertyId": 1, "propertyName": "Meridian Park",
  "interiorAssetUrl": null }

// SceneManifestVTO — assets are optional; frontend renders procedural geometry when null
{ "propertyAssetUrl": null,
  "nodes": [ { "key": "tower_A", "entity": "TOWER", "id": 10 } ],
  "cameraPresets": [ { "name": "property", "position": [30, 30, 30], "target": [0, 0, 0] } ] }

// SearchResultVTO — towerCode / propertySlug added 2026-10-18 (frontend builds deep links from them)
{ "type": "UNIT", "id": 1208, "label": "A-1208", "subtitle": "Tower A · Floor 12 · 3BR",
  "propertyId": 1, "propertySlug": "meridian-park", "towerId": 10, "towerCode": "A", "floor": 12 }
// RESIDENT results: id = the resident's unit id, towerId/towerCode/floor = that unit's
```

`{propertyId}` in `/properties/{propertyId}/explorer`, `/scene`, `/facilities` and `/booking-stats` also accepts the property **slug** (e.g. `/properties/meridian-park/explorer`), because explorer deep links are `/manager/explorer/{slug}/t/{towerCode}/f/{floor}/u/{unitCode}`. Added 2026-10-18.

Occupancy % = occupied ÷ total, rounded to whole percent; reserved counts as not occupied. Tower totals are computed from units.

## Module B — Facility booking

| Method | Path | Response |
|---|---|---|
| GET | `/properties/{propertyId}/facilities` | `{ data: FacilityVTO[], total }` |
| GET | `/facilities/{facilityId}` | `FacilityVTO` |
| GET | `/facilities/{facilityId}/availability?date=&duration=` | `AvailabilityVTO` |
| GET | `/properties/{propertyId}/booking-stats?date=` | `BookingStatsVTO` |
| POST | `/reservations` | `CreateReservationDTO` → `ReservationVTO` (201; 409 on conflict) |
| GET | `/reservations?propertyId=&date=&facilityId=&status=` | `{ data: ReservationVTO[], total }` (residents only see their own) |
| GET | `/reservations/{reservationId}` | `ReservationVTO` |
| PATCH | `/reservations/{reservationId}` | `{ "date"?, "startTime"?, "endTime"?, "status"?: "CANCELLED" }` → `ReservationVTO` |
| POST | `/reservations/{reservationId}/approve` | → `ReservationVTO` (manager/owner) |
| POST | `/reservations/{reservationId}/decline` | `{ "reason"?: string }` → `ReservationVTO` (manager/owner) |

```jsonc
// FacilityVTO
{ "id": 3, "propertyId": 1, "name": "The Glasshouse", "tagline": "Make room for a moment.",
  "description": "A light-filled function hall for the celebrations, milestones, and gatherings that bring your community together.",
  "location": "Clubhouse · Level 01", "capacity": 60, "openTime": "08:00", "closeTime": "22:00",
  "bookingType": "EXCLUSIVE", "durations": [1, 2, 3], "price": 1500000, "currency": "IDR",
  "requiresApproval": true, "bookingWindowDays": 14, "status": "ACTIVE",
  "amenities": [ { "label": "60-guest capacity", "icon": "users" } ],
  "thumbnailUrl": null, "modelAssetUrl": null, "modelKind": "HALL", "timezone": "Asia/Jakarta" } // timezone = the property's IANA zone; modelKind picks the procedural model: POOL, COURT, HALL, GRILL, BOARDROOM, COWORK, STUDIO

// AvailabilityVTO
{ "facilityId": 3, "date": "2026-10-06", "duration": 1, "availableCount": 11,
  "slots": [ { "start": "08:00", "end": "09:00", "available": true, "remaining": 1, "reason": null } ],
  "timezone": "Asia/Jakarta", "today": "2026-10-06" } // reason: RESERVED | PAST | CLOSED | FULL; slots, PAST and `today` are computed in the property's time zone

// BookingStatsVTO
{ "communityFacilities": 7, "readyFacilities": 6, "todaysReservations": 6, "awaitingApproval": 3, "upcomingConfirmed": 6 }

// CreateReservationDTO — residents omit unitId/residentName (taken from their membership); managers and staff may book on behalf; residentRole: TENANT | OWNER | OCCUPANT
{ "facilityId": 3, "date": "2026-10-07", "startTime": "07:00", "endTime": "08:00", "guests": 2,
  "note": "Birthday", "unitId": 1201, "residentName": "Nadia Putri", "residentRole": "TENANT" }

// ReservationVTO
{ "id": 77, "facilityId": 3, "facilityName": "The Glasshouse", "unitId": 1208, "unitCode": "A-1208",
  "residentName": "Andi Pratama", "date": "2026-10-06", "startTime": "18:00", "endTime": "21:00",
  "guests": 40, "note": "Birthday gathering with family and friends.", "fee": 1500000, "currency": "IDR",
  "status": "PENDING", "decidedById": null, "decidedOn": null, "declineReason": null, "createdOn": "2026-10-01T10:00:00+07:00",
  "reference": "BK-2026-0007", "residentRole": "TENANT", "timezone": "Asia/Jakarta" } // date/startTime/endTime and the instants are in `timezone`; reference + residentRole (TENANT | OWNER | OCCUPANT) added 2026-10-18 for the approval cards
```

Booking rules (SRS §4.5): shared = capacity-based, exclusive = one group per slot; booking window, max active bookings per unit, cancellation cutoff; free shared facilities confirm instantly (`CONFIRMED`), others go to `PENDING` when `requiresApproval`. The server enforces overlap/capacity in a transaction (MySQL: row lock on the facility + slot check); two concurrent requests for the last slot → exactly one 201, the other 409.

## Seed data (both sides use the same demo property)

Workspace 2 "Meridian Park Co" → property **Meridian Park** (`meridian-park`), towers A The Aspen (24 floors × 8), B The Birch, C The Cedar (18 floors × 8), D The Dahlia; 624 units total, ~82 % occupied.
Facilities: The Lagoon Pool (shared, complimentary), Garden Court (exclusive, $15), The Glasshouse (exclusive, $150, approval), Gather & Grill (exclusive, $25, approval), The Boardroom (exclusive, $20, approval), The Commons (shared, complimentary), Motion Studio (maintenance). The workspace was seeded in IDR and moved to USD by booking changeset 015 (prices ÷ 10,000); its seeded reservations keep IDR.

Market demo workspaces (added 2026-10-18 for localization; looked up by slug, not id). Members of workspace 2 other than residents are also members of these:

| Workspace (slug) | Currency / locale / zone | Property (slug) | Towers | Units | Facilities |
|---|---|---|---|---|---|
| واحة التجمع للتطوير العقاري (`tagamoa-oasis-dev`) | EGP · ar-EG · Africa/Cairo | واحة التجمع, New Cairo (`tagamoa-oasis`) | A 12, B 10, C 10 floors × 6 | 192 | pool, padel EGP 400, hall EGP 5,000 (approval), grill EGP 750 (approval), co-working, meeting room EGP 600 (approval) |
| مجتمع النرجس السكني (`al-narjis-living`) | SAR · ar-SA · Asia/Riyadh | مجتمع النرجس, Riyadh (`al-narjis`) | A 14, B 14, C 10 floors × 6 | 228 | family pool, padel SAR 150 (open until 24:00), events hall SAR 1,500 (approval), grill SAR 250 (approval), meeting room SAR 200 (approval), fitness studio (maintenance) |
| Riverside Living Co (`riverside-living-co`) | USD · en-US · America/New_York | The Riverside, Jersey City (`the-riverside`) | A 20, B 16 floors × 6 | 216 | rooftop pool, pickleball $25, club room $250 (approval), grill terrace $40, conference room $30 |

Workspace currency is one of **USD, EGP, SAR** (`CreateWorkspaceDTO`/`WorkspaceSettingsDTO`, 400 `REST-0004` otherwise). `PUT /workspaces/{id}/settings` is open to OWNER and MANAGER of the active workspace; changing the currency moves every facility of the workspace to it (amounts unchanged), past reservations keep the currency they were made in, and `FacilityDTO.currency` is ignored - facilities always use the workspace currency. The manager UI is `/manager/settings`.

Workspace `locale` is `language-REGION`; the frontend takes the UI language from it (until the user picks one) and the market rules from the region: currency format, week start (Saturday EG, Sunday SA/US, Monday ID) and 12-hour clock for en-US only.

## Backend notes (`pms-backend` v2/backend, 2026-10-18)

Everything above is implemented. Additions the frontend may use (all optional supersets):

- `PropertyExplorerVTO.property` is the full `PropertyVTO` (adds `city`, `country`, `type`, `timezone`, `latitude`, `longitude`, `description`, `image`, `modelAssetUrl`); `kpis` adds `underMaintenanceUnits`. `TowerDetailVTO` adds `propertyId`.
- Scene `nodes` also contain `property_<slug>` (PROPERTY) and `unit_<code>` (UNIT) entries; `cameraPresets` = `property` + one per tower node key.
- `GET /units`: `orderBy` ∈ `code`, `tower`, `floor`, `layout`, `areaM2`, `status` (unknown keys fall back to `code`); `pageSize` up to 1000; `q` matches unit code, tower code or tower name.
- Residents and viewers never see `UNDER_MAINTENANCE` units (grid, floor, directory, scene, search, `GET /units/{id}` → 404); KPI totals still count them.
- `ReservationVTO` also has `propertyId` and `userId`; `reference` is `BK-<year>-<id, 4 digits>`. `GET /reservations` also takes `from`/`to` (inclusive days, used when `date` is absent) and `pageNum`/`pageSize` (default 200, max 500).
- `FacilityVTO` also has `maxActivePerUnit` and `cancelCutoffHours`. Availability `end` is `"24:00"` for slots that end at midnight.
- Errors: overlap → 409 `RESERVATION-0010`, capacity → 409 `RESERVATION-0011`; window/hours/duration/guests/closure/limit/cutoff → 400 (`RESERVATION-0002`…`0018`); facility under maintenance → 400 `FACILITY-0002`. Malformed JSON, bad enums or bad query types → 400 `REST-0004`.
- Moving a reservation (date/time PATCH) is owner/manager only; residents can cancel their own until `cancelCutoffHours` before start.
- Extra endpoints: `GET/POST /facilities/{id}/closures`, `DELETE /facilities/{id}/closures/{closureId}`; admin CRUD `POST/GET /properties`, `GET/PUT/DELETE /properties/{id}`, `POST/GET /properties/{id}/towers`, `PUT/DELETE /towers/{id}`, `POST /properties/{id}/units`, `PUT/DELETE /units/{id}`; `POST/PUT /tenants` accept `unitId` to link a resident to a unit.
- Seeded reservations are dated relative to the day the migration ran (Jakarta time), so "today" counts drift on later days.

## v1 ↔ v2 integration (2026-10-18)

The v1 pages (Properties, Buildings, Units, Leases, Inspections, superintendent Buildings, resident dashboard) now use the v2 model. "Building" in the UI is a **Tower**. There are no `/buildings` routes; use `/properties/{id}/towers` and `/towers/{id}`.

| Field / route | Change |
|---|---|
| `UnitSummaryVTO` (GET /units, GET /towers/{id}/floors/{n}) | add `propertyId`, `propertyName`, `towerName`, `bedrooms`, `bathrooms`, `rentAmount` (number or null), `currency` |
| `UnitDetailVTO` (GET /units/{id}, GET /tenant/unit) | add `rentAmount`, `currency`, `activeLeaseId`, `tenantId`, `tenantName` (null when vacant) |
| `UnitDTO` (POST /properties/{id}/units, PUT /units/{id}) | accept `rentAmount`; `code`, `towerId`, `floor` stay required; `layout` drives bedrooms when bedrooms is omitted |
| GET /units | supports `propertyId`, `towerId`, `floor`, `status`, `layout`, `q`, paging; the v1 pages pass `pageSize` (max 1000) |
| `PropertyVTO` (GET /properties/{id}) | add `towers: TowerSummaryVTO[]` and `unitsCount` |
| Tower CRUD | POST /properties/{id}/towers, GET/PUT/DELETE /towers/{id}; `TowerDTO { code, name, subtitle?, floorsCount, buildingCode?, sortOrder? }` |
| GET /worker/properties | each property includes `towers` (the staff member's assigned towers) |
| Unit status | `OCCUPIED`, `VACANT`, `RESERVED`, `UNDER_MAINTENANCE` everywhere (the v1 `MAINTENANCE` value is gone) |
| `rent_amount` | mapped again on `Unit`; demo units get rent seeded by layout |

All v1 module tables get `workspace_id` + `@TenantId`, backfilled from their tower/unit, so every v1 list, get-by-id and report is confined to the active workspace. Create endpoints check that the given `buildingId` (tower id) / `unitId` exists in the active workspace.

Backend details (`pms-backend` branch `v2/unit-api`, 2026-10-18):

- `currency` on units is the active workspace's currency (ISO code, e.g. `"USD"`); `rentAmount` is monthly, in that currency.
- `UnitDTO.rentAmount` omitted (or null) on PUT keeps the current rent; there is no way to clear it to null.
- `tenantId` is the tenant's **user id** (same as `LeaseVTO.tenantId`). It comes from the unit's ACTIVE lease (latest start date wins); when there is none and the unit is not `VACANT`, from the resident linked to the unit (`POST /tenants` with `unitId`), with `activeLeaseId: null`.
- `GET /properties` (list) also includes `towers` and `unitsCount` on each property. `unitsCount` counts all statuses. `PropertyVTO.type` is the enum (`RESIDENTIAL`), also on `/worker/properties`.
- `TowerDTO`: `code` (1–20 chars), `name` (1–100), `floorsCount` (0–200) are required → 400 `REST-0004` otherwise; floors 1..floorsCount are created, never removed. Duplicate code in the property → 400. `GET /towers/{id}` is the `TowerDetailVTO` above.
- `GET /worker/properties` → `[{ id: "1", name, type, address, city, unitsCount, towers: [{ id, code, name, subtitle, floorsCount, unitsCount }] }]` (`id` stays a string). With a staff record that has a tower (`building_id`) it is that one tower; a staff member without one (e.g. a v2 STAFF member with no staff record) gets every property of the active workspace with all its towers. `unitsCount` = units of the listed towers.
- Demo rents (Liquibase property 020, only where `rent_amount` is null), per layout STUDIO / 1BR / 2BR / 3BR / 4BR: IDR 6.5M / 9.5M / 14M / 21M / 30M, EGP 12,000 / 18,000 / 26,000 / 38,000 / 55,000, SAR 3,500 / 5,000 / 7,000 / 9,500 / 13,000, USD 2,300 / 2,950 / 3,800 / 4,900 / 6,400. The 12 v1 units keep their own rent. Changeset 019 fills `area_m2` from v1 `sqft` (÷ 10.764) where missing.
