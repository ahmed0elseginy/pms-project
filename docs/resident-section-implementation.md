# Resident Section Implementation Guide

This document outlines the backend implementation required to support the Resident section frontend pages.

---

## Table of Contents

1. [Frontend Features Overview](#frontend-features-overview)
2. [API Endpoints Required](#api-endpoints-required)
3. [Database Schema](#database-schema)
4. [Module Structure](#module-structure)
5. [Implementation Priority](#implementation-priority)
6. [Key Implementation Notes](#key-implementation-notes)

---

## Frontend Features Overview

### Pages and Routes

| Page | Route | Description |
|------|-------|-------------|
| Dashboard | `/resident/dashboard` | Quick action cards, upcoming payment, active tickets, recent messages, community events |
| Pay Rent | `/resident/pay-rent` | Current balance, payment history, upcoming schedule, auto-pay toggle, make payment modal |
| Maintenance | `/resident/maintenance` | Stats (open/in-progress/completed), create request modal, ticket cards with status timeline |
| Lease | `/resident/lease` | Lease details, timeline progress bar, documents list, renewal request button |
| Messages | `/resident/messages` | Conversation list, chat thread, send messages, new conversation modal |
| Documents | `/resident/documents` | Categories (Lease, Move-in, Notices, Receipts), search, download |
| Community | `/resident/community` | Events tab (upcoming/past, RSVP), Groups tab (join/leave) |
| Profile | `/resident/profile` | View and edit user profile (uses shared component) |

---

## API Endpoints Required

### Payment Module (Resident)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/payments/mine` | Get current resident's payment history |
| POST | `/payments` | Record new payment |
| GET | `/payments/mine/summary` | Get payment summary (amount due, paid count, overdue count) |
| GET | `/payments/mine/scheduled` | Get upcoming scheduled payments |
| POST | `/payments/autopay` | Enable/disable auto-pay |
| GET | `/payments/autopay/status` | Get auto-pay status |

### Ticket Module (Resident)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/tickets/mine` | Get current resident's tickets |
| GET | `/tickets/mine/{id}` | Get ticket by ID |
| POST | `/tickets` | Create new maintenance request |
| PATCH | `/tickets/{id}` | Update ticket (title, description, etc.) |
| POST | `/tickets/{id}/comments` | Add comment to ticket |
| POST | `/tickets/{id}/attachments` | Upload attachment to ticket |

### Lease Module (Resident)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/leases/mine` | Get current resident's active lease |
| GET | `/leases/mine/documents` | Get lease documents |
| POST | `/leases/mine/renewal-request` | Request lease renewal |

### Documents Module (Resident)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/documents/mine` | Get resident's documents by category |
| GET | `/documents/mine/{id}` | Get document download URL |

### Community Module (Resident)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/community/events` | List community events (filter by upcoming/past) |
| GET | `/community/events/{id}` | Get event details |
| POST | `/community/events/{id}/rsvp` | RSVP to event |
| DELETE | `/community/events/{id}/rsvp` | Cancel RSVP |
| GET | `/community/groups` | List community groups |
| POST | `/community/groups/{id}/join` | Join group |
| DELETE | `/community/groups/{id}/leave` | Leave group |

### Messaging Module (Resident)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/conversations` | List resident's conversations |
| POST | `/conversations` | Start new conversation |
| GET | `/conversations/{id}` | Get conversation details |
| GET | `/conversations/{id}/messages` | Get messages in conversation |
| POST | `/conversations/{id}/messages` | Send message |
| PATCH | `/conversations/{id}/read` | Mark conversation as read |

### Dashboard Module (Resident)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/dashboard/resident-summary` | Get dashboard data (upcoming payment, open tickets, recent messages, upcoming events) |

---

## Database Schema

### Additional Tables for Resident Features

```sql
-- payment_autopay settings table
CREATE TABLE payment_autopay (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    enabled BOOLEAN DEFAULT FALSE,
    payment_method VARCHAR(50),
    day_of_month INT, -- 1-28 for monthly autopay
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_autopay_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id),
    CONSTRAINT uk_autopay_tenant UNIQUE (tenant_id)
);

-- ticket_attachments table
CREATE TABLE ticket_attachments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    ticket_id BIGINT NOT NULL,
    file_fid VARCHAR(100) NOT NULL,
    file_name VARCHAR(255),
    file_size BIGINT,
    mime_type VARCHAR(100),
    uploaded_by_id BIGINT,
    uploaded_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_attachment_ticket FOREIGN KEY (ticket_id) REFERENCES tickets(id),
    CONSTRAINT fk_attachment_uploader FOREIGN KEY (uploaded_by_id) REFERENCES users(id)
);

-- resident_documents table (for building notices and receipts)
CREATE TABLE resident_documents (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    document_type VARCHAR(50) NOT NULL, -- lease, move-in, notice, receipt
    title VARCHAR(255) NOT NULL,
    file_fid VARCHAR(100) NOT NULL,
    file_name VARCHAR(255),
    file_size BIGINT,
    mime_type VARCHAR(100),
    building_id BIGINT,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_resident_doc_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id),
    CONSTRAINT fk_resident_doc_building FOREIGN KEY (building_id) REFERENCES buildings(id)
);

-- lease_renewal_requests table
CREATE TABLE lease_renewal_requests (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    lease_id BIGINT NOT NULL,
    requested_by_id BIGINT NOT NULL,
    status VARCHAR(50) DEFAULT 'pending', -- pending, approved, rejected
    requested_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_by_id BIGINT,
    processed_on TIMESTAMP,
    notes TEXT,
    CONSTRAINT fk_renewal_lease FOREIGN KEY (lease_id) REFERENCES leases(id),
    CONSTRAINT fk_renewal_requester FOREIGN KEY (requested_by_id) REFERENCES users(id),
    CONSTRAINT fk_renewal_processor FOREIGN KEY (processed_by_id) REFERENCES users(id)
);
```

---

## Module Structure

### Payment Module (`billing-mgt` - Extended)

```
billing-mgt/src/main/java/com/pms/billing/
├── controller/implementation/
│   └── ResidentBillingControllerImpl.java
├── core/service/implementation/
│   └── ResidentBillingServiceImpl.java
├── model/
│   ├── filter/
│   │   └── ResidentPaymentFilter.java
│   ├── generated/
│   │   ├── ResidentPaymentSummaryVTO.java
│   │   ├── ScheduledPaymentVTO.java
│   │   └── AutopaySettingsDTO.java
│   └── enums/
│       └── PaymentErrors.java (extended)
```

### Ticket Module (`ticket-mgt` - Extended)

```
ticket-mgt/src/main/java/com/pms/ticket/
├── controller/implementation/
│   └── ResidentTicketControllerImpl.java
├── core/service/implementation/
│   └── ResidentTicketServiceImpl.java
├── model/
│   ├── entity/
│   │   └── TicketAttachment.java
│   ├── filter/
│   │   └── ResidentTicketFilter.java
│   └── enums/
│       └── TicketErrors.java (extended)
├── repository/
│   ├── jpa/
│   │   └── TicketAttachmentJpaRepository.java
│   └── implementation/
│       └── TicketAttachmentRepositoryImpl.java
```

### Lease Module (`lease-mgt` - Extended)

```
lease-mgt/src/main/java/com/pms/lease/
├── controller/implementation/
│   └── ResidentLeaseControllerImpl.java
├── core/service/implementation/
│   └── ResidentLeaseServiceImpl.java
├── model/
│   ├── entity/
│   │   └── LeaseRenewalRequest.java
│   └── enums/
│       └── LeaseErrors.java (extended)
```

### Documents Module (`document-mgt`)

```
document-mgt/src/main/java/com/pms/document/
├── annotation/
│   ├── config/
│   │   └── DocumentConfig.java
│   └── imports/
│       └── ImportDocumentMgt.java
├── api/
│   ├── service/
│   │   └── DocumentService.java
│   └── repository/
│       └── DocumentRepository.java
├── controller/implementation/
│   └── ResidentDocumentControllerImpl.java
├── core/service/implementation/
│   └── DocumentServiceImpl.java
├── model/
│   ├── entity/
│   │   └── ResidentDocument.java
│   ├── filter/
│   │   └── DocumentSearchFilter.java
│   └── enums/
│       ├── DocumentErrors.java
│       ├── DocumentDomains.java
│       └── DocumentTypes.java
├── repository/
│   ├── jpa/
│   │   └── ResidentDocumentJpaRepository.java
│   └── implementation/
│       └── DocumentRepositoryImpl.java
```

### Community Module (`community-mgt` - Extended)

```
community-mgt/src/main/java/com/pms/community/
├── controller/implementation/
│   └── ResidentCommunityControllerImpl.java
├── core/service/implementation/
│   └── ResidentCommunityServiceImpl.java
```

### Messaging Module (`chat-mgt` - Extended)

```
chat-mgt/src/main/java/com/pms/chat/
├── controller/implementation/
│   └── ResidentChatControllerImpl.java
├── core/service/implementation/
│   └── ResidentChatServiceImpl.java
```

---

## Implementation Priority

### Phase 1: Core Features (High Priority)

1. **Payment Module (Resident)** - View payment history, make payments, auto-pay toggle
2. **Lease Module (Resident)** - View lease details and documents
3. **Ticket Module (Resident)** - Create and view maintenance requests

### Phase 2: Communication (Medium Priority)

4. **Messaging Module (Resident)** - Chat with property manager and neighbors
5. **Ticket Comments** - Add comments to maintenance requests

### Phase 3: Additional Features (Medium Priority)

6. **Documents Module** - View and download documents by category
7. **Community Module (Resident)** - RSVP to events, join/leave groups
8. **Dashboard Aggregator** - Combine data for dashboard display

---

## Key Implementation Notes

### Resident Context

- All endpoints filter data by the authenticated user's tenant record
- Use tenant ID from authentication context
- Handle edge case: user is not a tenant (no lease assigned)

### Payment Features

- Auto-pay should charge on specified day of month (default: 1st)
- Late fees calculation: configurable amount per overdue payment
- Payment methods: bank-transfer, credit-card, check, cash
- Receipt generation after successful payment

### Maintenance Request Features

- Categories: plumbing, electrical, hvac, appliance, pest, general, structural, landscaping
- Priorities: low, medium, high, urgent
- Status workflow: open → in-progress → completed → closed
- Photo attachments: max 5 files, 5MB each, image types
- Comments: text only, timestamped

### Lease Features

- Show lease timeline with progress percentage
- Days remaining calculation
- Document types: lease agreement, addendum, move-in checklist, inspection report
- Renewal request creates pending request for manager approval

### Community Features

- Event RSVP adds user to event_attendees table
- Group join/leave updates group_members table
- Filter events by upcoming (future) or past
- Show attendee count and user avatars

### Messaging Features

- Real-time not required for MVP (polling acceptable)
- Unread count tracking per conversation
- Message display: sent messages right-aligned, received left-aligned
- Start new conversation: search users by name

---

## Swagger Definition Example

```yaml
paths:
  /payments/mine:
    get:
      summary: Get resident's payment history
      security:
        - bearerAuth: []
      responses:
        '200':
          description: Payment list
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaymentResultSet'

  /tickets/mine:
    get:
      summary: Get resident's maintenance tickets
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [open, in-progress, completed, closed]
        - name: category
          in: query
          schema:
            type: string
      responses:
        '200':
          description: Ticket list

  /tickets:
    post:
      summary: Create maintenance request
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateTicketDTO'
      responses:
        '201':
          description: Ticket created

  /conversations:
    get:
      summary: List conversations
      security:
        - bearerAuth: []
      responses:
        '200':
          description: Conversation list

  /conversations/{id}/messages:
    post:
      summary: Send message
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                text:
                  type: string
      responses:
        '201':
          description: Message sent
```

---

## Notes

- Reuse existing ticket, payment, lease, and community tables from other modules
- Add new tables only where needed (autopay, attachments, documents)
- All modules follow the same package structure and naming conventions as specified in AGENTS.md
- Use constructor injection with `@AllArgsConstructor`
- Use `throw new BusinessException(ERROR_ENUM)` for business errors
- Generated controllers and DTOs go in `model/generated/` - do not edit manually
- Hand-written code goes in `controller/implementation/`, `entity/`, `filter/`, `enums/`, `repository/`