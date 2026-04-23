# Architecture (C4 views)

C4 model for the PMS workspace: **context**, **container**, and **component** (backend modules). **Primary client for kickoff:** [`pms-web`](../pms-web/) (roles `manager`, `super`, `renter` — see [`types/index.ts`](../pms-web/src/types/index.ts)). Maven layout: [pms-backend/README.md](../pms-backend/README.md).

## Level 1 — System context

Actors match the **`pms-web`** login personas (Property Manager, Superintendent, Resident).

```mermaid
flowchart LR
  subgraph actors [Actors_pms-web]
    mgr[manager_Property_Manager]
    sup[super_Superintendent]
    res[renter_Resident]
    adm[Admin_optional]
  end

  pms[PMS_Platform]

  mgr --> pms
  sup --> pms
  res --> pms
  adm --> pms

  subgraph external [External systems]
    email[Email_SMTP]
    pay[PaymentGateway_future]
  end

  pms --> email
  pms -.-> pay
```

**Notes**

- **Payment gateway** is optional later; the web prototype already models rent payments in UI state.
- **Resident** login URL uses `resident`; app role remains `renter` in [`LoginPage.tsx`](../pms-web/src/pages/login/LoginPage.tsx).

## Level 2 — Containers

Deployable units and data stores.

```mermaid
flowchart TB
  subgraph clients [User computers]
    fe[Web_SPA_pms-web_kickoff]
    fe2[Web_pms-frontend_future_console]
  end

  subgraph runtime [Server runtime]
    api[Spring_Boot_API_main-execution]
  end

  subgraph data [Data and messaging]
    db[(MySQL_8)]
    broker[Apache_Artemis_JMS]
    uploads[File_volume_DOCUMENT_STORAGE_PATH]
  end

  fe -->|HTTPS_JSON_JWT| api
  fe2 -.->|future| api
  api --> db
  api --> broker
  api --> uploads
```

**Technology choices** (from README)

| Container | Technology |
|-----------|------------|
| Web SPA | **`pms-web`** (kickoff); `pms-frontend` optional later console |
| API | Spring Boot 3.4.x, Java 17, `com.pms.platform` |
| Database | MySQL 8, schema via Liquibase |
| Async messaging | Spring JMS + Apache Artemis |
| Documents | Configurable filesystem path `DOCUMENT_STORAGE_PATH` |

**Runtime flow**

1. Browser obtains JWT from `/auth/login` (user module).
2. API validates JWT (`library/security-adapter` filters).
3. Domain services persist to MySQL; publish domain events to Artemis where modules use `MQClientService` (e.g. login event in `AuthServiceImpl`).
4. Notification and other consumers process messages asynchronously.

## Level 3 — Components (backend modules)

Logical modules inside the Spring Boot application (Maven modules under `pms-backend/modules/`).

```mermaid
flowchart TB
  subgraph libs [Shared libraries library]
    sec[security-adapter]
    rest[rest-adapter]
    mq[mq-adapter]
    common[common]
    swagger[swagger]
    session[session-manager]
    sql[sql-db-adapter]
  end

  subgraph modules [Domain modules]
    user[user-mgt]
    property[property-mgt]
    lease[lease-mgt]
    billing[billing-mgt]
    maintenance[maintenance-mgt]
    notification[notification-mgt]
    audit[audit-mgt]
    admin[admin-mgt]
    reporting[reporting-mgt]
  end

  db[(MySQL)]
  brokerNode[Artemis_JMS]

  user --> libs
  property --> libs
  lease --> libs
  billing --> libs
  maintenance --> libs
  notification --> libs
  audit --> libs
  admin --> libs
  reporting --> libs

  user --> db
  property --> db
  lease --> db
  billing --> db
  maintenance --> db
  notification --> db
  audit --> db
  admin --> db
  reporting --> db
  notification --> brokerNode
  maintenance --> brokerNode
```

**Per-module responsibility**

| Module | Responsibility |
|--------|----------------|
| `user` | Auth, profile, roles, activation |
| `property` | Property → building → unit hierarchy |
| `lease` | Lease lifecycle and linkage to units |
| `billing` | Charges, invoices, payments, aging |
| `maintenance` | Work orders and SLA-oriented workflow |
| `notification` | Templates, delivery, in-app + email |
| `audit` | Immutable activity trail |
| `admin` | Tenant configuration, user management orchestration |
| `reporting` | Aggregates and exports |

**Cross-cutting**

- **JMS**: use for decoupled notifications and future outbox-style processing; keep publishers idempotent where duplicates are possible.
- **Document storage**: lease PDFs, work-order photos; path from env, not committed secrets.

## Level 4 (optional sketch — not fully enumerated)

Within each `*-mgt` module: REST controllers (often generated from `_swagger`), services, JPA repositories, Liquibase changelogs in sibling `_liquibase` folders. Defer detailed class diagrams until a single module is refactored.

## Related documents

- [RBAC-AND-DATA-SCOPE.md](./RBAC-AND-DATA-SCOPE.md) — authorization and row scope (`pms-web` roles).
- [BACKEND-ROLE-ALIGNMENT.md](./BACKEND-ROLE-ALIGNMENT.md) — backend seeds and JWT aligned with web.
- [DELIVERY-PHASES-EPICS.md](./DELIVERY-PHASES-EPICS.md) — phased delivery order.
