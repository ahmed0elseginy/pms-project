# Backend role alignment with `pms-web` (kickoff)

The product roles users see at login are **Property Manager**, **Superintendent**, and **Resident**, backed in code by `UserRole`: `manager`, `super`, `renter` ([`pms-web/src/types/index.ts`](../pms-web/src/types/index.ts)). The backend should use the **same names and semantics** in Liquibase seeds, the `role` table, enums, and JWT-facing documentation.

## 1. Target role catalog (suggested)

Stable numeric IDs are optional; what matters is **one clear mapping** from DB → JWT → web.

| `role.id` | `title_en` (API / DB) | Web `UserRole` | Notes |
|-----------|------------------------|----------------|--------|
| 1 | `Admin` | — | Platform / org admin |
| 2 | `Property_Manager` | `manager` | Portfolio operator |
| 3 | `Resident` | `renter` | Unit-scoped resident |
| 4 | `Superintendent` | `super` | Building maintenance lead |

Implemented in Liquibase [`changeset-2026-04-18-pms-roles.xml`](../pms-backend/modules/user/_liquibase/changes/changeset-2026-04-18-pms-roles.xml) (updates ids 2–3 titles + inserts id 4) and Java [`UserRoles.java`](../pms-backend/modules/user/user-mgt/src/main/java/com/pms/core/user/model/enums/UserRoles.java).

Add more rows later for **Leasing_Agent**, **Owner**, etc., when those become first-class logins—[`mockData.ts`](../pms-web/src/context/mockData.ts) already lists additional **staff** job titles for UI only.

## 2. Liquibase

- New changeset under `pms-backend/modules/user/_liquibase/changes/`:
  - Insert or update `role` rows to match the table above (use `update` if ids already exist in your environment).
- Keep `user_role.role_id` FKs valid for existing users if any; migrate rows to the new semantics rather than deleting historical ids without a data migration.

## 3. Java (`user-mgt`)

- Replace `UserRoles` enum values with **`ADMIN`, `PROPERTY_MANAGER`, `SUPERINTENDENT`, `RESIDENT`** (or shorter names matching `title_en`), with ids matching Liquibase.
- Update references in `AuthServiceImpl`, `AdminServiceImpl`, `UserManagementServiceImpl`, controllers—search for old identifiers and fix defaults (e.g. default onboarding role = `RESIDENT` if self-service residents).

## 4. JWT and API contract

- Keep **`ROLES_IDS`** as today, or add a claim **`app_roles`** with strings `manager` | `super` | `renter` so the web client can mirror `UserRole` without a hardcoded id map.
- OpenAPI: describe which operations require which role string (aligned with web).

## 5. Seed users (parity with web demo)

Mirror [`mockData.ts`](../pms-web/src/context/mockData.ts) users for integration tests:

- `sarah@propmanage.com` → `manager`
- `mike@propmanage.com`, `carlos@propmanage.com` → `super`
- `james@tenant.com`, etc. → `renter` with unit metadata on profile or lease link

## 6. Verification

- [ ] Login API returns tokens whose claims identify roles consistent with `UserRole`.
- [ ] No user-facing strings in admin APIs refer to non-PMS personas.
- [ ] `pms-web` can switch from mock `login()` to real API without renaming roles.
