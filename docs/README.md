# PMS documentation

Planning and architecture for this workspace. **Kickoff product roles and demo users are defined in `pms-web`** (`manager` / `super` / `renter`); backend work should align with that app, not legacy naming.

| Document | Description |
|----------|-------------|
| [RBAC-AND-DATA-SCOPE.md](./RBAC-AND-DATA-SCOPE.md) | Roles from `pms-web`, capabilities, module mapping, data scope |
| [BACKEND-ROLE-ALIGNMENT.md](./BACKEND-ROLE-ALIGNMENT.md) | Liquibase, enums, JWT, and seeds aligned with `UserRole` |
| [ARCHITECTURE-C4.md](./ARCHITECTURE-C4.md) | C4 context, container, and component diagrams |
| [DELIVERY-PHASES-EPICS.md](./DELIVERY-PHASES-EPICS.md) | Phases 0–6 with epics and acceptance criteria |

Primary SPA for this phase: [pms-web/README.md](../pms-web/README.md). Backend runbook: [pms-backend/README.md](../pms-backend/README.md).
