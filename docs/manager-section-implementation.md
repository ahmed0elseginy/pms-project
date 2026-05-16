# Manager Section Implementation Guide

This document outlines the backend implementation required to support the Manager section frontend pages.

---

## Table of Contents

1. [Frontend Pages Overview](#frontend-pages-overview)
2. [API Endpoints Required](#api-endpoints-required)
3. [Database Schema](#database-schema)
4. [Module Structure](#module-structure)
5. [Implementation Priority](#implementation-priority)

---

## Frontend Pages Overview

| Page | Route | Description |
|------|-------|-------------|
| Dashboard | `/manager/dashboard` | Overview with stats cards, revenue/cost charts, recent tickets, recent payments, upcoming lease expirations, quick actions |
| Properties | `/manager/properties` | CRUD operations for properties with search and type filtering |
| Property Detail | `/manager/properties/[id]` | View property details with Buildings and Standalone Units tabs; add/edit/delete buildings and units |
| Building Detail | `/manager/buildings/[id]` | View building details, list units within building, add units |
| Units List | `/manager/units` | List all units across properties with property and status filters |
| Unit Detail | `/manager/units/[id]` | View unit details, edit unit information, delete unit |
| Tenants | `/manager/tenants` | List, search, invite by email, add existing user, view tenant details in drawer |
| Leases | `/manager/leases` | List, create, view details, update status, renew lease, upload documents |
| Tickets | `/manager/tickets` | Kanban and table views, create tickets, update status, add comments |
| Staff | `/manager/staff` | List staff, add new staff, view staff details in drawer |
| Billing | `/manager/billing` | Payments and Costs tabs; record payments, add costs |
| Reports | `/manager/reports` | Financial analytics with charts (revenue, costs, occupancy trends) |
| Community | `/manager/community` | Events and Groups tabs; create events/groups |
| Profile | `/manager/profile` | View and edit user profile |

---

## API Endpoints Required

### Property Module

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/properties` | List properties with pagination, search, type filter |
| POST | `/properties` | Create new property |
| GET | `/properties/{id}` | Get property by ID with buildings and units |
| PUT | `/properties/{id}` | Update property |
| DELETE | `/properties/{id}` | Delete property |
| POST | `/properties/{id}/buildings` | Add building to property |
| DELETE | `/properties/{id}/buildings/{buildingId}` | Delete building from property |

### Building Module

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/buildings/{id}` | Get building by ID with units |
| PUT | `/buildings/{id}` | Update building |
| DELETE | `/buildings/{id}` | Delete building |
| GET | `/buildings/{id}/units` | Get units within building |
| POST | `/buildings/{id}/units` | Add unit to building |

### Unit Module

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/units` | List all units with property and status filters |
| POST | `/units` | Create new unit (standalone or in building) |
| GET | `/units/{id}` | Get unit by ID |
| PUT | `/units/{id}` | Update unit |
| DELETE | `/units/{id}` | Delete unit |

### Tenant Module

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/tenants` | List tenants with fullName and lease status filters |
| POST | `/tenants` | Create tenant (link to existing user) |
| POST | `/tenants/invite` | Invite new tenant via email (creates user with Resident role) |
| GET | `/tenants/{id}` | Get tenant by ID with lease and payment info |
| DELETE | `/tenants/{id}` | Delete tenant |

### Lease Module

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/leases` | List leases with status and search filters |
| POST | `/leases` | Create new lease |
| GET | `/leases/{id}` | Get lease by ID with documents |
| PUT | `/leases/{id}/status` | Update lease status (DRAFT, ACTIVE, EXPIRED, TERMINATED) |
| POST | `/leases/{id}/renew` | Renew lease with new dates and rent |
| POST | `/leases/{id}/documents` | Upload lease document |
| DELETE | `/leases/{id}/documents/{docId}` | Delete lease document |

### Ticket Module

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/tickets` | List tickets with status, priority, category filters |
| POST | `/tickets` | Create new ticket |
| GET | `/tickets/{id}` | Get ticket by ID with comments |
| PATCH | `/tickets/{id}` | Update ticket (status, assignedTo, priority, etc.) |
| POST | `/tickets/{id}/comments` | Add comment to ticket |

### Staff Module

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/staff` | List staff with search and status filters |
| POST | `/staff` | Create new staff member |
| GET | `/staff/{id}` | Get staff by ID |
| PUT | `/staff/{id}` | Update staff member |
| PATCH | `/staff/{id}/status` | Toggle staff active/inactive status |

### Billing Module

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/payments` | List rental payments with status filter |
| POST | `/payments` | Record new payment |
| GET | `/payments/summary` | Get payment summary (collected, outstanding, overdue) |
| GET | `/costs` | List costs with category filter |
| POST | `/costs` | Add new cost |

### Community Module

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/community/events` | List events with category filter |
| POST | `/community/events` | Create new event |
| GET | `/community/events/{id}` | Get event by ID |
| POST | `/community/events/{id}/attendees` | Join/leave event |
| GET | `/community/groups` | List groups |
| POST | `/community/groups` | Create new group |
| GET | `/community/groups/{id}` | Get group by ID |
| POST | `/community/groups/{id}/members` | Join/leave group |

### Dashboard Module

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/dashboard/summary` | Get manager dashboard summary (open tickets, overdue payments, monthly costs, collected rent) |
| GET | `/dashboard/revenue-chart` | Get monthly revenue data for charts |
| GET | `/dashboard/cost-chart` | Get monthly cost data for charts |
| GET | `/dashboard/occupancy-trend` | Get occupancy trend data |

---

## Database Schema

### Core Tables

```sql
-- properties table
CREATE TABLE properties (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    address VARCHAR(500) NOT NULL,
    city VARCHAR(100),
    type VARCHAR(50) NOT NULL, -- RESIDENTIAL, COMMERCIAL, MIXED_USE
    description TEXT,
    image VARCHAR(500),
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_id BIGINT,
    last_modified_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_modified_by_id BIGINT
);

-- buildings table
CREATE TABLE buildings (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    address VARCHAR(500) NOT NULL,
    floors INT,
    property_id BIGINT NOT NULL,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_id BIGINT,
    last_modified_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_modified_by_id BIGINT,
    CONSTRAINT fk_building_property FOREIGN KEY (property_id) REFERENCES properties(id)
);

-- units table
CREATE TABLE units (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    unit_number VARCHAR(50) NOT NULL,
    floor INT,
    type VARCHAR(50), -- APARTMENT, OFFICE, RETAIL, STUDIO, LOFT, TOWNHOUSE
    status VARCHAR(50) NOT NULL, -- VACANT, OCCUPIED, MAINTENANCE
    bedrooms INT,
    bathrooms INT,
    sqft INT,
    rent_amount DECIMAL(12, 2),
    property_id BIGINT NOT NULL,
    building_id BIGINT,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_id BIGINT,
    last_modified_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_modified_by_id BIGINT,
    CONSTRAINT fk_unit_property FOREIGN KEY (property_id) REFERENCES properties(id),
    CONSTRAINT fk_unit_building FOREIGN KEY (building_id) REFERENCES buildings(id)
);

-- tenants table
CREATE TABLE tenants (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    unit_id BIGINT,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_id BIGINT,
    CONSTRAINT fk_tenant_user FOREIGN KEY (user_id) REFERENCES users(id),
    CONSTRAINT fk_tenant_unit FOREIGN KEY (unit_id) REFERENCES units(id)
);

-- staff table
CREATE TABLE staff (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT,
    name VARCHAR(255) NOT NULL,
    role VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    salary DECIMAL(12, 2),
    pay_frequency VARCHAR(50), -- weekly, bi-weekly, monthly
    start_date DATE,
    status VARCHAR(50) DEFAULT 'active', -- active, inactive
    building_id BIGINT,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_id BIGINT,
    last_modified_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_modified_by_id BIGINT,
    CONSTRAINT fk_staff_user FOREIGN KEY (user_id) REFERENCES users(id),
    CONSTRAINT fk_staff_building FOREIGN KEY (building_id) REFERENCES buildings(id)
);

-- tickets table
CREATE TABLE tickets (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(50) NOT NULL, -- plumbing, electrical, hvac, appliance, pest, general, structural, landscaping
    priority VARCHAR(50) NOT NULL, -- low, medium, high, urgent
    status VARCHAR(50) NOT NULL, -- open, in-progress, completed, closed
    created_by_id BIGINT NOT NULL,
    assigned_to_id BIGINT,
    unit_id BIGINT,
    building_id BIGINT,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_ticket_creator FOREIGN KEY (created_by_id) REFERENCES users(id),
    CONSTRAINT fk_ticket_assignee FOREIGN KEY (assigned_to_id) REFERENCES users(id),
    CONSTRAINT fk_ticket_unit FOREIGN KEY (unit_id) REFERENCES units(id),
    CONSTRAINT fk_ticket_building FOREIGN KEY (building_id) REFERENCES buildings(id)
);

-- ticket_comments table
CREATE TABLE ticket_comments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    ticket_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    text TEXT NOT NULL,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_comment_ticket FOREIGN KEY (ticket_id) REFERENCES tickets(id),
    CONSTRAINT fk_comment_user FOREIGN KEY (user_id) REFERENCES users(id)
);

-- rental_payments table
CREATE TABLE rental_payments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    unit_id BIGINT,
    lease_id BIGINT,
    amount DECIMAL(12, 2) NOT NULL,
    due_date DATE NOT NULL,
    paid_date DATE,
    status VARCHAR(50) NOT NULL, -- paid, pending, overdue, partial
    method VARCHAR(50), -- check, bank-transfer, credit-card, cash
    note TEXT,
    building_id BIGINT,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_id BIGINT,
    CONSTRAINT fk_payment_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id),
    CONSTRAINT fk_payment_unit FOREIGN KEY (unit_id) REFERENCES units(id),
    CONSTRAINT fk_payment_lease FOREIGN KEY (lease_id) REFERENCES leases(id)
);

-- costs table
CREATE TABLE costs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    description VARCHAR(500) NOT NULL,
    category VARCHAR(50) NOT NULL, -- maintenance, utilities, insurance, taxes, supplies, renovation, other
    amount DECIMAL(12, 2) NOT NULL,
    date DATE NOT NULL,
    vendor VARCHAR(255),
    recurring BOOLEAN DEFAULT FALSE,
    frequency VARCHAR(50), -- monthly, quarterly, annually
    building_id BIGINT,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_id BIGINT,
    CONSTRAINT fk_cost_building FOREIGN KEY (building_id) REFERENCES buildings(id)
);

-- community_events table
CREATE TABLE community_events (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    date DATE NOT NULL,
    time VARCHAR(50),
    location VARCHAR(255),
    category VARCHAR(50) NOT NULL, -- social, meeting, maintenance-notice, fitness, education, other
    created_by_id BIGINT NOT NULL,
    building_id BIGINT,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_event_creator FOREIGN KEY (created_by_id) REFERENCES users(id),
    CONSTRAINT fk_event_building FOREIGN KEY (building_id) REFERENCES buildings(id)
);

-- event_attendees table
CREATE TABLE event_attendees (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    event_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_attendee_event FOREIGN KEY (event_id) REFERENCES community_events(id),
    CONSTRAINT fk_attendee_user FOREIGN KEY (user_id) REFERENCES users(id)
);

-- community_groups table
CREATE TABLE community_groups (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(50) NOT NULL,
    created_by_id BIGINT NOT NULL,
    building_id BIGINT,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_group_creator FOREIGN KEY (created_by_id) REFERENCES users(id),
    CONSTRAINT fk_group_building FOREIGN KEY (building_id) REFERENCES buildings(id)
);

-- group_members table
CREATE TABLE group_members (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    group_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_member_group FOREIGN KEY (group_id) REFERENCES community_groups(id),
    CONSTRAINT fk_member_user FOREIGN KEY (user_id) REFERENCES users(id)
);
```

---

## Module Structure

### Property Module (`property-mgt`)

```
property-mgt/src/main/java/com/pms/property/
├── annotation/
│   ├── config/PropertyConfig.java
│   └── imports/ImportPropertyMgt.java
├── api/
│   ├── service/PropertyService.java
│   ├── repository/PropertyRepository.java
│   └── controller/generated/PropertyController.java
├── controller/implementation/
│   └── PropertyControllerImpl.java
├── core/service/implementation/
│   ├── PropertyServiceImpl.java
│   ├── BuildingServiceImpl.java
│   └── UnitServiceImpl.java
├── core/mapper/
│   └── PropertyMgtMapper.java
├── model/
│   ├── entity/
│   │   ├── Property.java
│   │   ├── Building.java
│   │   └── Unit.java
│   ├── entity/pk/
│   ├── filter/
│   │   ├── PropertySearchFilter.java
│   │   ├── UnitSearchFilter.java
│   │   └── BuildingSearchFilter.java
│   ├── enums/
│   │   ├── PropertyErrors.java
│   │   ├── PropertyDomains.java
│   │   ├── UnitStatuses.java
│   │   └── PropertyTypes.java
│   ├── generated/
│   │   └── (auto-generated from swagger.yaml)
│   └── event/data/
├── repository/
│   ├── jpa/
│   │   ├── PropertyJpaRepository.java
│   │   ├── BuildingJpaRepository.java
│   │   └── UnitJpaRepository.java
│   ├── query/
│   │   ├── PropertyQueryBuilder.java
│   │   ├── BuildingQueryBuilder.java
│   │   └── UnitQueryBuilder.java
│   └── implementation/
│       ├── PropertyRepositoryImpl.java
│       ├── BuildingRepositoryImpl.java
│       └── UnitRepositoryImpl.java
```

### Tenant Module (`tenant-mgt`)

```
tenant-mgt/src/main/java/com/pms/tenant/
├── annotation/config/TenantConfig.java
├── api/service/TenantService.java
├── api/repository/TenantRepository.java
├── controller/implementation/TenantControllerImpl.java
├── core/service/implementation/TenantServiceImpl.java
├── model/
│   ├── entity/Tenant.java
│   ├── filter/TenantSearchFilter.java
│   └── enums/TenantErrors.java, TenantDomains.java
├── repository/
│   ├── jpa/TenantJpaRepository.java
│   ├── query/TenantQueryBuilder.java
│   └── implementation/TenantRepositoryImpl.java
```

### Ticket Module (`ticket-mgt`)

```
ticket-mgt/src/main/java/com/pms/ticket/
├── annotation/config/TicketConfig.java
├── api/service/TicketService.java
├── api/repository/TicketRepository.java
├── controller/implementation/TicketControllerImpl.java
├── core/service/implementation/TicketServiceImpl.java
├── model/
│   ├── entity/Ticket.java
│   ├── entity/TicketComment.java
│   ├── filter/TicketSearchFilter.java
│   └── enums/TicketErrors.java, TicketDomains.java, TicketStatuses.java, TicketPriorities.java, TicketCategories.java
├── repository/
│   ├── jpa/TicketJpaRepository.java, TicketCommentJpaRepository.java
│   ├── query/TicketQueryBuilder.java
│   └── implementation/TicketRepositoryImpl.java
```

### Staff Module (`staff-mgt`)

```
staff-mgt/src/main/java/com/pms/staff/
├── annotation/config/StaffConfig.java
├── api/service/StaffService.java
├── api/repository/StaffRepository.java
├── controller/implementation/StaffControllerImpl.java
├── core/service/implementation/StaffServiceImpl.java
├── model/
│   ├── entity/Staff.java
│   ├── filter/StaffSearchFilter.java
│   └── enums/StaffErrors.java, StaffDomains.java, StaffStatuses.java
├── repository/
│   ├── jpa/StaffJpaRepository.java
│   ├── query/StaffQueryBuilder.java
│   └── implementation/StaffRepositoryImpl.java
```

### Billing Module (`billing-mgt`)

```
billing-mgt/src/main/java/com/pms/billing/
├── annotation/config/BillingConfig.java
├── api/service/BillingService.java
├── api/repository/PaymentRepository.java, CostRepository.java
├── controller/implementation/BillingControllerImpl.java
├── core/service/implementation/BillingServiceImpl.java
├── model/
│   ├── entity/RentalPayment.java
│   ├── entity/Cost.java
│   ├── filter/PaymentSearchFilter.java
│   ├── filter/CostSearchFilter.java
│   └── enums/BillingErrors.java, BillingDomains.java, PaymentStatuses.java, CostCategories.java
├── repository/
│   ├── jpa/PaymentJpaRepository.java, CostJpaRepository.java
│   ├── query/PaymentQueryBuilder.java, CostQueryBuilder.java
│   └── implementation/PaymentRepositoryImpl.java, CostRepositoryImpl.java
```

### Community Module (`community-mgt`)

```
community-mgt/src/main/java/com/pms/community/
├── annotation/config/CommunityConfig.java
├── api/service/CommunityService.java
├── api/repository/EventRepository.java, GroupRepository.java
├── controller/implementation/CommunityControllerImpl.java
├── core/service/implementation/CommunityServiceImpl.java
├── model/
│   ├── entity/CommunityEvent.java
│   ├── entity/CommunityGroup.java
│   ├── entity/EventAttendee.java
│   ├── entity/GroupMember.java
│   ├── filter/EventSearchFilter.java
│   ├── filter/GroupSearchFilter.java
│   └── enums/CommunityErrors.java, CommunityDomains.java, EventCategories.java
├── repository/
│   ├── jpa/EventJpaRepository.java, GroupJpaRepository.java
│   └── implementation/EventRepositoryImpl.java, GroupRepositoryImpl.java
```

### Dashboard Module (`dashboard-mgt`)

```
dashboard-mgt/src/main/java/com/pms/dashboard/
├── annotation/config/DashboardConfig.java
├── api/service/DashboardService.java
├── controller/implementation/DashboardControllerImpl.java
├── core/service/implementation/DashboardServiceImpl.java
├── model/
│   ├── generated/ManagerDashboardSummaryVTO.java
│   ├── generated/ChartDataVTO.java
│   └── enums/DashboardErrors.java
```

---

## Implementation Priority

### Phase 1: Core Entities (High Priority)

1. **Property Module** - CRUD for properties, buildings, units
2. **Unit Module** - Standalone unit management
3. **Tenant Module** - Tenant management with user linking

### Phase 2: Lease Integration (High Priority)

4. **Lease Integration** - Link tenants to leases (extends existing lease-mgt)
5. **Dashboard Module** - Summary statistics

### Phase 3: Operations (Medium Priority)

6. **Ticket Module** - Maintenance request management
7. **Staff Module** - Staff management

### Phase 4: Financials (Medium Priority)

8. **Billing Module** - Payments and costs tracking
9. **Reports** - Financial analytics

### Phase 5: Community (Low Priority)

10. **Community Module** - Events and groups

---

## Swagger Integration

All REST APIs should be defined in `swagger.yaml` following the swagger-first code generation approach:

```yaml
paths:
  /properties:
    get:
      summary: List properties
      parameters:
        - name: pageNum
          in: query
          schema:
            type: integer
        - name: pageSize
          in: query
          schema:
            type: integer
        - name: search
          in: query
          schema:
            type: string
        - name: type
          in: query
          schema:
            type: string
            enum: [RESIDENTIAL, COMMERCIAL, MIXED_USE]
      responses:
        '200':
          description: Property list
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PropertyResultSet'
```

---

## Notes

- All modules should follow the same package structure and naming conventions
- Use constructor injection with `@AllArgsConstructor`
- Use `throw new BusinessException(ERROR_ENUM)` for business errors
- Use `orElseThrow(() -> new BusinessException(...))` for not-found cases
- Follow Liquibase conventions for database migrations
- Generated controllers and DTOs go in `model/generated/` - do not edit manually
- Hand-written code goes in `controller/implementation/`, `entity/`, `filter/`, `enums/`, `repository/`