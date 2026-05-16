# Staff Section Implementation Guide

This document outlines the backend implementation required to support the Staff section frontend pages.

---

## Table of Contents

1. [Frontend Features Overview](#frontend-features-overview)
2. [API Endpoints Required](#api-endpoints-required)
3. [Database Schema](#database-schema)
4. [Module Structure](#module-structure)
5. [Related Features](#related-features)
6. [Implementation Notes](#implementation-notes)

---

## Frontend Features Overview

### Manager Staff Page (`/manager/staff`)

| Feature | Description |
|---------|-------------|
| Staff List | Data table with columns: Name (with avatar), Role, Email, Phone, Salary, Pay Frequency, Status |
| Search | Filter by name, role, or email |
| Status Filter | Filter by active/inactive |
| Add Staff | Modal form with validation for name, role, email, phone, salary, pay frequency, start date, building assignment |
| Staff Details | Drawer view showing full staff profile |
| Status Toggle | Activate/deactivate staff members |

### Add Staff Form Fields

| Field | Type | Validation | Required |
|-------|------|------------|----------|
| Full Name | text | min 1 char | Yes |
| Role | text | min 1 char | Yes |
| Email | email | valid email format | Yes |
| Phone | text | min 1 char | Yes |
| Salary | number | min 0 | Yes |
| Pay Frequency | select | weekly, bi-weekly, monthly | Yes |
| Start Date | date | required | Yes |
| Assigned Building | select | optional | No |

### Staff Details Drawer

- Avatar and name display
- Role badge
- Status badge (Active/Inactive)
- Email
- Phone
- Salary with pay frequency
- Start date
- Assigned building
- Activate/Deactivate button

---

## API Endpoints Required

### Staff Module

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/staff` | List staff with pagination, search, status filter, building filter |
| POST | `/staff` | Create new staff member |
| GET | `/staff/{id}` | Get staff by ID |
| PUT | `/staff/{id}` | Update staff member |
| PATCH | `/staff/{id}/status` | Toggle staff active/inactive status |
| DELETE | `/staff/{id}` | Delete staff member |

### Query Parameters for GET /staff

| Parameter | Type | Description |
|-----------|------|-------------|
| pageNum | integer | Page number (default: 0) |
| pageSize | integer | Page size (default: 20) |
| search | string | Search by name, role, or email |
| status | string | Filter by status (active, inactive) |
| buildingId | integer | Filter by assigned building |
| orderBy | string | Sort field (name, createdOn, etc.) |
| orderDir | string | Sort direction (ASC, DESC) |

### Request/Response Models

#### CreateStaffRequest

```json
{
  "name": "string",
  "role": "string",
  "email": "string",
  "phone": "string",
  "salary": 0.00,
  "payFrequency": "monthly",
  "startDate": "2026-01-15",
  "buildingId": 1
}
```

#### StaffVTO

```json
{
  "id": 1,
  "name": "John Smith",
  "role": "Maintenance Technician",
  "email": "john.smith@example.com",
  "phone": "+1 555-123-4567",
  "salary": 4500.00,
  "payFrequency": "monthly",
  "startDate": "2026-01-15",
  "status": "active",
  "buildingId": 1,
  "buildingName": "Building A",
  "createdOn": "2026-01-15T10:00:00Z",
  "lastModifiedOn": "2026-01-15T10:00:00Z"
}
```

---

## Database Schema

### Core Tables

```sql
-- staff table
CREATE TABLE staff (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT,
    name VARCHAR(255) NOT NULL,
    role VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    salary DECIMAL(12, 2),
    pay_frequency VARCHAR(20), -- weekly, bi-weekly, monthly
    start_date DATE,
    status VARCHAR(20) DEFAULT 'active', -- active, inactive
    building_id BIGINT,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_id BIGINT,
    last_modified_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_modified_by_id BIGINT,
    CONSTRAINT fk_staff_user FOREIGN KEY (user_id) REFERENCES users(id),
    CONSTRAINT fk_staff_building FOREIGN KEY (building_id) REFERENCES buildings(id),
    CONSTRAINT uk_staff_email UNIQUE (email)
);

-- Index for search performance
CREATE INDEX idx_staff_name ON staff(name);
CREATE INDEX idx_staff_email ON staff(email);
CREATE INDEX idx_staff_status ON staff(status);
CREATE INDEX idx_staff_building ON staff(building_id);
```

### Related Tables (for extended functionality)

```sql
-- time_entries table (for time tracking)
CREATE TABLE time_entries (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    staff_id BIGINT NOT NULL,
    date DATE NOT NULL,
    hours_worked DECIMAL(5, 2) NOT NULL,
    description VARCHAR(500),
    ticket_id BIGINT, -- optional link to maintenance ticket
    status VARCHAR(20) DEFAULT 'pending', -- pending, approved, rejected
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_id BIGINT,
    approved_by_id BIGINT,
    approved_on TIMESTAMP,
    CONSTRAINT fk_time_entry_staff FOREIGN KEY (staff_id) REFERENCES staff(id),
    CONSTRAINT fk_time_entry_ticket FOREIGN KEY (ticket_id) REFERENCES tickets(id),
    CONSTRAINT fk_time_entry_approver FOREIGN KEY (approved_by_id) REFERENCES users(id)
);

-- staff_invoices table (for contractor billing)
CREATE TABLE staff_invoices (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    staff_id BIGINT NOT NULL,
    date DATE NOT NULL,
    invoice_number VARCHAR(50) UNIQUE,
    items JSON, -- array of invoice line items
    total_amount DECIMAL(12, 2) NOT NULL,
    receipt_url VARCHAR(500),
    status VARCHAR(20) DEFAULT 'pending', -- pending, approved, rejected, paid
    notes TEXT,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_id BIGINT,
    approved_by_id BIGINT,
    approved_on TIMESTAMP,
    paid_on TIMESTAMP,
    CONSTRAINT fk_invoice_staff FOREIGN KEY (staff_id) REFERENCES staff(id),
    CONSTRAINT fk_invoice_approver FOREIGN KEY (approved_by_id) REFERENCES users(id)
);

-- work_order_assignments table (for task assignment)
CREATE TABLE work_order_assignments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    ticket_id BIGINT NOT NULL,
    staff_id BIGINT NOT NULL,
    assigned_by_id BIGINT NOT NULL,
    assigned_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    due_date DATE,
    status VARCHAR(20) DEFAULT 'assigned', -- assigned, in-progress, completed
    notes TEXT,
    CONSTRAINT fk_wo_assignment_ticket FOREIGN KEY (ticket_id) REFERENCES tickets(id),
    CONSTRAINT fk_wo_assignment_staff FOREIGN KEY (staff_id) REFERENCES staff(id),
    CONSTRAINT fk_wo_assignment_assigner FOREIGN KEY (assigned_by_id) REFERENCES users(id)
);
```

---

## Module Structure

### Staff Module (`staff-mgt`)

```
staff-mgt/src/main/java/com/pms/staff/
├── annotation/
│   ├── config/
│   │   └── StaffConfig.java
│   └── imports/
│       └── ImportStaffMgt.java
├── api/
│   ├── service/
│   │   └── StaffService.java
│   ├── repository/
│   │   └── StaffRepository.java
│   └── controller/
│       └── generated/
│           └── StaffController.java
├── controller/
│   └── implementation/
│       └── StaffControllerImpl.java
├── core/
│   └── service/
│       └── implementation/
│           └── StaffServiceImpl.java
├── core/
│   └── mapper/
│       └── StaffMgtMapper.java
├── model/
│   ├── entity/
│   │   ├── Staff.java
│   │   ├── TimeEntry.java
│   │   └── StaffInvoice.java
│   ├── entity/pk/
│   ├── filter/
│   │   ├── StaffSearchFilter.java
│   │   └── TimeEntrySearchFilter.java
│   ├── enums/
│   │   ├── StaffErrors.java
│   │   ├── StaffDomains.java
│   │   ├── StaffStatuses.java
│   │   └── StaffPayFrequencies.java
│   ├── generated/
│   │   ├── CreateStaffDTO.java
│   │   ├── UpdateStaffDTO.java
│   │   ├── StaffVTO.java
│   │   ├── StaffResultSet.java
│   │   ├── TimeEntryDTO.java
│   │   ├── TimeEntryVTO.java
│   │   ├── CreateTimeEntryDTO.java
│   │   ├── StaffInvoiceDTO.java
│   │   └── StaffInvoiceVTO.java
│   └── event/
│       └── data/
├── repository/
│   ├── jpa/
│   │   ├── StaffJpaRepository.java
│   │   ├── TimeEntryJpaRepository.java
│   │   └── StaffInvoiceJpaRepository.java
│   ├── query/
│   │   └── StaffQueryBuilder.java
│   └── implementation/
│       ├── StaffRepositoryImpl.java
│       ├── TimeEntryRepositoryImpl.java
│       └── StaffInvoiceRepositoryImpl.java
```

### Entity Classes

#### Staff.java

```java
@Data @Builder @AllArgsConstructor @NoArgsConstructor
@Entity @Table(name = "staff")
public class Staff {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String role;

    @Column(nullable = false, unique = true)
    private String email;

    private String phone;

    private BigDecimal salary;

    @Enumerated(EnumType.STRING)
    @Column(name = "pay_frequency")
    private StaffPayFrequency payFrequency;

    @Column(name = "start_date")
    private LocalDate startDate;

    @Enumerated(EnumType.STRING)
    @Builder.Default
    private StaffStatus status = StaffStatus.ACTIVE;

    @ManyToOne
    @JoinColumn(name = "building_id")
    private Building building;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @Column(name = "created_on", updatable = false)
    private LocalDateTime createdOn;

    @Column(name = "created_by_id")
    private Long createdById;

    @Column(name = "last_modified_on")
    private LocalDateTime lastModifiedOn;

    @Column(name = "last_modified_by_id")
    private Long lastModifiedById;
}
```

#### TimeEntry.java

```java
@Data @Builder @AllArgsConstructor @NoArgsConstructor
@Entity @Table(name = "time_entries")
public class TimeEntry {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "staff_id", nullable = false)
    private Staff staff;

    @Column(nullable = false)
    private LocalDate date;

    @Column(name = "hours_worked", nullable = false)
    private BigDecimal hoursWorked;

    private String description;

    @ManyToOne
    @JoinColumn(name = "ticket_id")
    private Ticket ticket;

    @Enumerated(EnumType.STRING)
    @Builder.Default
    private TimeEntryStatus status = TimeEntryStatus.PENDING;

    @Column(name = "approved_by_id")
    private Long approvedById;

    @Column(name = "approved_on")
    private LocalDateTime approvedOn;
}
```

---

## Related Features

### Time Sheets (Superintendent Section)

Related to staff management for tracking work hours:

| Feature | Description |
|---------|-------------|
| Staff Time Entries | Log daily working hours |
| Approval Workflow | Manager approves submitted timesheets |
| Ticket Linking | Link time to maintenance tickets |
| Reporting | Hours by staff, by date range, by ticket |

**API Endpoints:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/time-entries` | List time entries with filters |
| POST | `/time-entries` | Create time entry |
| GET | `/time-entries/{id}` | Get time entry by ID |
| PUT | `/time-entries/{id}` | Update time entry |
| PATCH | `/time-entries/{id}/approve` | Approve time entry |
| PATCH | `/time-entries/{id}/reject` | Reject time entry |

### Staff Invoices

For contractors and hourly staff who invoice for work:

| Feature | Description |
|---------|-------------|
| Create Invoice | Staff creates invoice for completed work |
| Line Items | Multiple items with description, quantity, rate |
| Approval | Manager approves pending invoices |
| Payment | Mark invoices as paid |

**API Endpoints:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/staff-invoices` | List staff invoices |
| POST | `/staff-invoices` | Create invoice |
| GET | `/staff-invoices/{id}` | Get invoice by ID |
| PATCH | `/staff-invoices/{id}/approve` | Approve invoice |
| PATCH | `/staff-invoices/{id}/pay` | Mark as paid |

### Work Order Assignment

Assign maintenance tickets to staff:

| Feature | Description |
|---------|-------------|
| Assign Staff | Link ticket to staff member |
| Track Progress | Update assignment status |
| Due Dates | Set deadline for completion |

---

## Implementation Notes

### Staff Types and Roles

Consider creating a standard list of staff roles:

| Role | Description |
|------|-------------|
| Maintenance Technician | General maintenance repairs |
| Electrician | Electrical systems |
| Plumber | Plumbing systems |
| HVAC Technician | Heating/cooling systems |
| Cleaner | Janitorial services |
| Security | Security personnel |
| Landscaper | Grounds maintenance |
| Property Manager | Property oversight |
| Superintendent | Building supervision |

### Salary Calculation

- **Hourly**: Store hourly rate, calculate based on time entries
- **Weekly/Bi-weekly**: Fixed amount per pay period
- **Monthly**: Fixed monthly salary

### Building Assignment

- Staff can be assigned to one or multiple buildings
- Use many-to-many relationship if staff can work across buildings
- Filter staff list by building in manager dashboard

### User Integration

Optionally link staff to system users:
- Staff with login access (for timesheet submission)
- Staff without login (managed entirely by admins)

### Validation Rules

1. Email must be unique across staff records
2. Start date cannot be in the future for new staff
3. Salary must be positive number
4. At least one of: user_id or (name + email + phone) required

---

## Swagger Definition Example

```yaml
paths:
  /staff:
    get:
      summary: List staff members
      parameters:
        - name: search
          in: query
          schema:
            type: string
        - name: status
          in: query
          schema:
            type: string
            enum: [active, inactive]
        - name: buildingId
          in: query
          schema:
            type: integer
      responses:
        '200':
          description: Staff list
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/StaffResultSet'
    post:
      summary: Create new staff member
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateStaffDTO'
      responses:
        '201':
          description: Staff created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/StaffVTO'

  /staff/{id}:
    get:
      summary: Get staff by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: Staff details
        '404':
          description: Staff not found

  /staff/{id}/status:
    patch:
      summary: Toggle staff status
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                status:
                  type: string
                  enum: [active, inactive]
      responses:
        '200':
          description: Status updated
```

---

## Notes

- All modules should follow the same package structure and naming conventions as specified in AGENTS.md
- Use constructor injection with `@AllArgsConstructor`
- Use `throw new BusinessException(STAFF_ERRORS)` for business errors
- Follow Liquibase conventions for database migrations with changeset author `ahmed.el-seginy`
- Generated controllers and DTOs go in `model/generated/` - do not edit manually
- Hand-written code goes in `controller/implementation/`, `entity/`, `filter/`, `enums/`, `repository/`