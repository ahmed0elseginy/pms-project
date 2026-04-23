# PMS Web — API specification (with models)

This document lists **REST APIs and JSON models** needed to replace mock state in `pms-web` (`AppContext`, `mockData`). Shapes align with **`src/types/index.ts`**. Paths are **proposed** for PMS domain resources unless noted as **already defined** in `pms-backend` user swagger.

**Conventions**

- **Base URL:** e.g. `https://api.example.com` (mount paths as your gateway defines; user module today uses root-relative paths like `/auth/login`).
- **Auth:** `Authorization: Bearer <accessToken>` on all endpoints except auth and public marketing.
- **Correlation / tenancy:** `X-Building-Id` or `buildingId` query (MVP single-building) — align with `docs/RBAC-AND-DATA-SCOPE.md` (BUILDING / UNIT / ASSIGNED scope).
- **IDs:** `pms-web` types use `string` ids in the prototype; backend may use `int64` or UUID strings — map in the client or standardize on **UUID strings** in JSON for new APIs.
- **Dates:** ISO-8601 `date` (`YYYY-MM-DD`) or `date-time` (`...Z`) as noted per field.
- **Pagination (lists):** `page`, `size` or `pageOffset`, `pageSize` — match `library/_swagger` shared params if you merge into OpenAPI.
- **Errors:** JSON body with `code`, `message`, optional `formErrors[]` (same spirit as `pms-backend` `ErrorVTO`).

**Existing backend (reference only)**

| Area | Swagger file | Today |
|------|----------------|--------|
| Auth, profile, users, roles | `pms-backend/modules/user/_swagger/service/swagger.yaml` | `/auth/login`, `/auth/token/refresh`, `/profile`, `/users`, role assign, activation, password flows |
| Notifications | `pms-backend/modules/notification/_swagger/service/swagger.yaml` | Web/mobile push list + mark read |
| Chat (legacy shape) | `pms-backend/modules/chat/_swagger/service/swagger.yaml` | Case/lawyer rooms — **not PMS-shaped**; PMS messaging should be new resources or adapted |
| Maintenance / billing / property / reporting | respective `*/_swagger/service/swagger.yaml` | Mostly **health** stubs — **domain CRUD below is to be implemented** |

---

## 1. Shared enums (JSON)

These mirror `src/types/index.ts`.

### `UserRole`

`"manager"` | `"super"` | `"renter"`

*(JWT may expose numeric role ids; API responses for PMS should include `appRole: UserRole` for the web client.)*

### `TicketStatus`

`"open"` | `"in-progress"` | `"completed"` | `"closed"`

### `TicketPriority`

`"low"` | `"medium"` | `"high"` | `"urgent"`

### `TicketCategory`

`"plumbing"` | `"electrical"` | `"hvac"` | `"appliance"` | `"pest"` | `"general"` | `"structural"` | `"landscaping"`

### `RentalPaymentStatus`

`"paid"` | `"pending"` | `"overdue"` | `"partial"`

### `PaymentMethod`

`"check"` | `"bank-transfer"` | `"credit-card"` | `"cash"`

### `CostCategory`

`"maintenance"` | `"utilities"` | `"insurance"` | `"taxes"` | `"supplies"` | `"renovation"` | `"other"`

### `CostFrequency`

`"monthly"` | `"quarterly"` | `"annually"`

### `StaffPayFrequency`

`"weekly"` | `"bi-weekly"` | `"monthly"`

### `StaffStatus`

`"active"` | `"inactive"`

### `TimeEntryStatus`

`"pending"` | `"approved"` | `"rejected"`

### `StaffInvoiceStatus`

`"pending"` | `"approved"` | `"rejected"` | `"paid"`

### `CommunityEventCategory`

`"social"` | `"meeting"` | `"maintenance-notice"` | `"fitness"` | `"education"` | `"other"`

### `FileAttachmentKind` (server classification)

`"image"` | `"document"` | `"video"` | `"audio"` | `"other"` — derived from `mimeType` or client hint; used for UI (thumbnails vs download link).

### `UploadPurpose` (multipart / staged upload)

`"ticket"` | `"ticket_comment"` | `"message"` | `"invoice_receipt"` | `"avatar"` — drives validation rules (size, allowed MIME) and audit metadata.

---

## 2. Core JSON models

Field lists match the web types; optional fields are marked **optional**.

### `User` (PMS-facing profile + directory row)

| Field | Type | Notes |
|-------|------|--------|
| `id` | string | Stable user id |
| `name` | string | Display name (or compose from `firstName` + `lastName` server-side) |
| `email` | string | |
| `role` | `UserRole` | |
| `avatar` | string (uri) | **optional** |
| `unit` | string | **optional** — unit label for renters (e.g. `"101"`) |
| `phone` | string | **optional** |

**Create / update DTOs** for admin flows can split `firstName`, `lastName` to match `UserProfileVTO` in user module.

### `FileAttachment` (tickets, comments, messages)

| Field | Type | Notes |
|-------|------|--------|
| `id` | string | Attachment row id |
| `url` | string (uri) | **GET** display/download; may be time-limited signed URL |
| `originalName` | string | Filename from uploader |
| `mimeType` | string | e.g. `image/jpeg`, `application/pdf` |
| `sizeBytes` | integer | |
| `kind` | `FileAttachmentKind` | **optional** — server may set |
| `uploadedBy` | string | User id |
| `uploadedAt` | string (`date-time`) | |

### `TicketComment`

| Field | Type |
|-------|------|
| `id` | string |
| `userId` | string |
| `userName` | string |
| `text` | string |
| `createdAt` | string (`date-time`) |
| `attachments` | `FileAttachment[]` | **optional** — photos, PDFs, videos for support |

### `Ticket`

| Field | Type |
|-------|------|
| `id` | string |
| `title` | string |
| `description` | string |
| `category` | `TicketCategory` |
| `priority` | `TicketPriority` |
| `status` | `TicketStatus` |
| `createdBy` | string (user id) |
| `assignedTo` | string | **optional** |
| `unitNumber` | string | e.g. `"101"` or `"Common"` |
| `buildingId` | string | **recommended** for scope |
| `createdAt` | string (`date-time`) |
| `updatedAt` | string (`date-time`) |
| `comments` | `TicketComment[]` |
| `photos` | string[] (uris) | **optional** — convenience: image URLs only (can mirror image `attachments`) |
| `attachments` | `FileAttachment[]` | **optional** — **all** support files on the ticket (images + documents) |

### `TicketCreateRequest` (subset for POST)

Required: `title`, `description`, `category`, `priority`, `unitNumber` (or `unitId`). Server sets `createdBy`, `status` default `open`, timestamps.

**Optional:** `attachmentIds: string[]` — files already uploaded via §16.1 **staged upload** (recommended for large files). Alternatively **multipart** create: `POST /pms/buildings/{buildingId}/tickets` with `metadata` JSON part + `files[]` parts (see §6.1).

### `TicketPatchRequest`

Partial `Ticket` (no `id` in body); use for status transitions, assignee, priority. May include `attachmentIds` to append staged files.

### `TicketCommentCreateRequest`

| Field | Type |
|-------|------|
| `text` | string |
| `attachmentIds` | string[] | **optional** — staged file ids to attach to this comment |

Server sets `userId`, `userName`, `createdAt`, resolves `attachmentIds` to `FileAttachment` rows.

**Multipart variant:** `POST .../comments` with `text` form field + `files[]` (see §6.1).

### `RentalPayment`

| Field | Type |
|-------|------|
| `id` | string |
| `tenantId` | string |
| `tenantName` | string |
| `unitNumber` | string |
| `amount` | number (decimal) |
| `dueDate` | string (`date`) |
| `paidDate` | string (`date`) | **optional** |
| `status` | `RentalPaymentStatus` |
| `method` | `PaymentMethod` | **optional** |
| `note` | string | **optional** |
| `buildingId` | string | **recommended** |
| `leaseId` | string | **optional** — when lease module exists |

### `RentalPaymentCreateRequest` / `RentalPaymentPatchRequest`

Manager **record payment** flows: `paidDate`, `status`, `method`, `note`, `amount` (partial payments).

### `Cost`

| Field | Type |
|-------|------|
| `id` | string |
| `description` | string |
| `category` | `CostCategory` |
| `amount` | number |
| `date` | string (`date`) |
| `vendor` | string | **optional** |
| `recurring` | boolean |
| `frequency` | `CostFrequency` | **optional** if not recurring |
| `buildingId` | string | **recommended** |

### `StaffMember`

| Field | Type |
|-------|------|
| `id` | string |
| `name` | string |
| `role` | string | Job title |
| `email` | string |
| `phone` | string |
| `salary` | number |
| `payFrequency` | `StaffPayFrequency` |
| `startDate` | string (`date`) |
| `status` | `StaffStatus` |
| `userId` | string | **optional** — link to login user if supers |

### `TimeEntry`

| Field | Type |
|-------|------|
| `id` | string |
| `staffId` | string |
| `staffName` | string |
| `date` | string (`date`) |
| `hoursWorked` | number |
| `description` | string |
| `ticketId` | string | **optional** |
| `status` | `TimeEntryStatus` |

### `InvoiceItem`

| Field | Type |
|-------|------|
| `description` | string |
| `quantity` | number |
| `unitPrice` | number |
| `total` | number |

### `Invoice` (staff expense)

| Field | Type |
|-------|------|
| `id` | string |
| `staffId` | string |
| `staffName` | string |
| `date` | string (`date`) |
| `items` | `InvoiceItem[]` |
| `totalAmount` | number |
| `receiptUrl` | string (uri) | **optional** |
| `status` | `StaffInvoiceStatus` |
| `notes` | string | **optional** |

### `CommunityEvent`

| Field | Type |
|-------|------|
| `id` | string |
| `title` | string |
| `description` | string |
| `date` | string (`date`) |
| `time` | string | Display time (e.g. `"7:00 PM"`) |
| `location` | string |
| `createdBy` | string (user id) |
| `attendees` | string[] (user ids) |
| `category` | `CommunityEventCategory` |
| `buildingId` | string | **recommended** |

### `CommunityGroup`

| Field | Type |
|-------|------|
| `id` | string |
| `name` | string |
| `description` | string |
| `members` | string[] (user ids) |
| `createdBy` | string |
| `category` | string |
| `buildingId` | string | **recommended** |

### `Message`

| Field | Type |
|-------|------|
| `id` | string |
| `senderId` | string |
| `senderName` | string |
| `receiverId` | string |
| `receiverName` | string |
| `text` | string |
| `createdAt` | string (`date-time`) |
| `read` | boolean |
| `attachments` | `FileAttachment[]` | **optional** — same model as ticket support files |

### `MessageCreateRequest`

| Field | Type |
|-------|------|
| `text` | string |
| `attachmentIds` | string[] | **optional** — staged uploads (§16.1) |

Multipart: `files[]` + `text` field (§14.1).

### `ConversationParticipant`

| Field | Type |
|-------|------|
| `id` | string |
| `name` | string |
| `role` | `UserRole` |

### `Conversation`

| Field | Type |
|-------|------|
| `id` | string |
| `participants` | `ConversationParticipant[]` (length 2 for MVP DM) |
| `messages` | `Message[]` |
| `lastMessage` | string | **optional** |
| `lastMessageAt` | string (`date-time`) | **optional** |
| `unreadCount` | integer |

### Reporting aggregates (for dashboards — extend as needed)

### `ManagerDashboardSummary`

| Field | Type |
|-------|------|
| `openTickets` | integer |
| `overduePayments` | integer |
| `monthlyRecurringCosts` | number |
| `collectedRentThisMonth` | number |
| `periodStart` | string (`date`) |
| `periodEnd` | string (`date`) |

### `ChartSeriesPoint`

| Field | Type |
|-------|------|
| `label` | string |
| `value` | number |

*(Manager reports pages can call dedicated endpoints returning arrays of `ChartSeriesPoint` per chart.)*

---

## 3. Auth & session (wire `LoginPage` → backend)

| Method | Path | Description | Request body | Response |
|--------|------|-------------|----------------|----------|
| POST | `/auth/login` | **Existing** user module | `{ "username": string, "password": string }` | `{ "accessToken", "refreshToken?", "status" }` per `LoginUserVTO` |
| POST | `/auth/token/refresh` | **Existing** | `{ "refreshToken": string }` | Same as login |
| POST | `/auth/register` | **Existing** (if self-serve) | `CreateUserDTO` | 201 |
| GET | `/profile` | **Existing** — current user | — | `UserProfileVTO` — **extend** or add `/pms/me` to include `appRole`, `unit`, `buildingId` |
| PUT | `/profile` | **Existing** | `UserProfileDTO` | 204 |

**PMS extension (recommended):**

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| GET | `/pms/me` | Current user + roles + building/unit scope | `{ "user": User, "buildings": [{ "id", "name" }], "defaultBuildingId": string }` |

---

## 4. Users & directory

| Method | Path | Roles | Description | Query | Response |
|--------|------|-------|-------------|-------|----------|
| GET | `/pms/buildings/{buildingId}/users` | manager, super | Directory for messaging / assignment | `role`, `q`, pagination | `{ "data": User[], "total": number }` |
| GET | `/pms/users/{userId}` | manager, super, renter | Public-safe profile for participants | — | `User` |

*(Optional: reuse `GET /users` from user module with filters if you prefer one user service — map to `User`.)*

---

## 5. Property context (minimal for `pms-web`)

| Method | Path | Roles | Response |
|--------|------|-------|----------|
| GET | `/pms/buildings` | manager, super, renter | `{ "data": [{ "id", "name", "address" }] }` |
| GET | `/pms/buildings/{buildingId}/units` | manager, super | For staff/ticket unit pickers | `{ "data": [{ "id", "unitNumber", "tenantUserId?" }] }` |
| GET | `/pms/me/unit` | renter | Current lease unit | `{ "unitNumber", "unitId", "leaseId" }` |

---

## 6. Tickets (maintenance / work orders) — including support files

Maps to: `tickets`, `addTicket`, `updateTicket`, `addTicketComment`; pages: manager/super/renter ticket UIs. **Support files** = images, PDFs, short videos residents or staff attach to **the ticket** or to a **comment** (evidence, manuals, vendor quotes).

### 6.1 Ticket & comment file flows

| Pattern | When to use |
|---------|-------------|
| **Staged upload** | Client `POST` §16.1 with `purpose=ticket` or `ticket_comment`, gets `attachmentId` + `url`; then `POST` ticket or comment JSON with `attachmentIds[]`. Best for large files, retries, mobile. |
| **Multipart on create** | Single request: `multipart/form-data` with part `metadata` = JSON `TicketCreateRequest` and parts `files` (repeatable). Server creates ticket + links attachments atomically. |
| **Multipart on comment** | `POST .../comments` with `text` + optional `files[]`. |

**Recommended limits (product policy):** e.g. 10 files per ticket, 25 MB per file (images smaller); block executables; scan in **file** module.

| Method | Path | Roles | Body | Response |
|--------|------|-------|------|----------|
| GET | `/pms/buildings/{buildingId}/tickets` | manager | List + filters | query: `status`, `category`, `priority`, `assignedTo`, `unitNumber`, pagination | `{ "data": Ticket[], "total" }` |
| GET | `/pms/buildings/{buildingId}/tickets` | super | Scoped to **assigned** or building | same | same |
| GET | `/pms/me/tickets` | renter | Own unit / own created | pagination | `{ "data": Ticket[], "total" }` |
| GET | `/pms/tickets/{ticketId}` | manager, super (scoped), renter (own) | — | `Ticket` (includes `comments[].attachments`, ticket-level `attachments`) |
| POST | `/pms/buildings/{buildingId}/tickets` | manager, super, renter | `application/json` `TicketCreateRequest` **or** `multipart/form-data` (see §6.1) | `Ticket` (201) |
| PATCH | `/pms/tickets/{ticketId}` | manager (full), super (assigned fields), renter (limited) | `TicketPatchRequest` | `Ticket` |
| POST | `/pms/tickets/{ticketId}/comments` | manager, super, renter (if participant) | JSON `TicketCommentCreateRequest` **or** `multipart` (`text` + `files[]`) | `TicketComment` (201) |
| POST | `/pms/tickets/{ticketId}/attachments` | same as comment participants | `multipart` one or more `file` **or** JSON `{ "attachmentIds": string[] }` linking staged files | `{ "data": FileAttachment[] }` (201) |
| DELETE | `/pms/tickets/{ticketId}/attachments/{attachmentId}` | uploader (policy), manager | — | 204 |

`GET /pms/tickets/{ticketId}/attachments/{attachmentId}/download` — optional **redirect** to signed S3/blob URL if `url` on `FileAttachment` is not directly public.

---

## 7. Rental payments (AR / resident ledger)

Maps to: `payments`, `addPayment`, `updatePayment`; pages: `ManagerPayments`, `RenterPayments`.

| Method | Path | Roles | Response |
|--------|------|-------|----------|
| GET | `/pms/buildings/{buildingId}/payments` | manager | query: `status`, `tenantId`, `dueBefore`, pagination | `{ "data": RentalPayment[], "total" }` |
| GET | `/pms/me/payments` | renter | Own ledger | `{ "data": RentalPayment[], "total" }` |
| GET | `/pms/payments/{paymentId}` | manager, renter (own) | — | `RentalPayment` |
| POST | `/pms/buildings/{buildingId}/payments` | manager | `RentalPaymentCreateRequest` (scheduled charge) | `RentalPayment` |
| PATCH | `/pms/payments/{paymentId}` | manager | `RentalPaymentPatchRequest` (record payment, waive, adjust) | `RentalPayment` |

---

## 8. Building costs (expenses)

Maps to: `costs`, `addCost`, `deleteCost`; page: `ManagerCosts`.

| Method | Path | Roles | Response |
|--------|------|-------|----------|
| GET | `/pms/buildings/{buildingId}/costs` | manager | query: `category`, `from`, `to`, pagination | `{ "data": Cost[], "total" }` |
| POST | `/pms/buildings/{buildingId}/costs` | manager | `Cost` without `id` | `Cost` (201) |
| PATCH | `/pms/costs/{costId}` | manager | partial `Cost` | `Cost` |
| DELETE | `/pms/costs/{costId}` | manager | — | 204 |

---

## 9. Staff & salaries

Maps to: `staff`, `addStaff`, `updateStaff`; page: `ManagerStaff`.

| Method | Path | Roles | Response |
|--------|------|-------|----------|
| GET | `/pms/buildings/{buildingId}/staff` | manager | pagination | `{ "data": StaffMember[], "total" }` |
| POST | `/pms/buildings/{buildingId}/staff` | manager | `StaffMember` without `id` | `StaffMember` |
| PATCH | `/pms/staff/{staffId}` | manager | partial | `StaffMember` |

---

## 10. Time sheets (superintendent)

Maps to: `timeEntries`, `addTimeEntry`, `updateTimeEntry`; page: `SuperTimeSheets`.

| Method | Path | Roles | Response |
|--------|------|-------|----------|
| GET | `/pms/me/time-entries` | super | query: `from`, `to`, `status`, pagination | `{ "data": TimeEntry[], "total" }` |
| GET | `/pms/buildings/{buildingId}/time-entries` | manager | Approve queue | same |
| POST | `/pms/me/time-entries` | super | `TimeEntry` without `id` / server sets `staffId` | `TimeEntry` |
| PATCH | `/pms/time-entries/{entryId}` | super (own pending), manager (approve) | partial | `TimeEntry` |

---

## 11. Staff expense invoices

Maps to: `invoices`, `addInvoice`, `updateInvoice`; page: `SuperInvoices`.

| Method | Path | Roles | Response |
|--------|------|-------|----------|
| GET | `/pms/me/invoices` | super | pagination | `{ "data": Invoice[], "total" }` |
| GET | `/pms/buildings/{buildingId}/invoices` | manager | All staff invoices | same |
| POST | `/pms/me/invoices` | super | `Invoice` without `id` | `Invoice` |
| PATCH | `/pms/invoices/{invoiceId}` | super (pending), manager (approve/reject/paid) | partial `Invoice` | `Invoice` |

---

## 12. Community — events

Maps to: `events`, `addEvent`, `updateEvent`, `joinEvent`, `leaveEvent`; pages: `ManagerCommunity`, `RenterCommunity`.

| Method | Path | Roles | Response |
|--------|------|-------|----------|
| GET | `/pms/buildings/{buildingId}/events` | manager, super, renter | pagination | `{ "data": CommunityEvent[], "total" }` |
| POST | `/pms/buildings/{buildingId}/events` | manager | `CommunityEvent` without `id` | `CommunityEvent` |
| PATCH | `/pms/events/{eventId}` | manager | partial | `CommunityEvent` |
| POST | `/pms/events/{eventId}/attendees/me` | renter, super | join | `CommunityEvent` |
| DELETE | `/pms/events/{eventId}/attendees/me` | renter, super | leave | `CommunityEvent` |

---

## 13. Community — groups

Maps to: `groups`, `addGroup`, `updateGroup`, `deleteGroup`, `joinGroup`, `leaveGroup`.

| Method | Path | Roles | Response |
|--------|------|-------|----------|
| GET | `/pms/buildings/{buildingId}/groups` | manager, super, renter | pagination | `{ "data": CommunityGroup[], "total" }` |
| POST | `/pms/buildings/{buildingId}/groups` | manager, renter (policy) | `CommunityGroup` without `id` | `CommunityGroup` |
| PATCH | `/pms/groups/{groupId}` | manager (or creator) | partial | `CommunityGroup` |
| DELETE | `/pms/groups/{groupId}` | manager (or creator) | — | 204 |
| POST | `/pms/groups/{groupId}/members/me` | renter, super | join | `CommunityGroup` |
| DELETE | `/pms/groups/{groupId}/members/me` | renter, super | leave | `CommunityGroup` |

---

## 14. Messaging (conversations) — including file attachments

Maps to: `conversations`, `sendMessage`, `getConversation`; page: `RenterMessages` (and any future manager inbox). Same **support file** model as tickets: `Message.attachments` / staged `purpose=message` uploads.

### 14.1 Send message with files

| Method | Path | Roles | Body | Response |
|--------|------|-------|------|----------|
| GET | `/pms/me/conversations` | manager, super, renter | pagination | `{ "data": Conversation[], "total" }` |
| GET | `/pms/conversations/{conversationId}` | participants | query: `messagesPage`, `messagesSize` | `Conversation` (messages include `attachments`) |
| POST | `/pms/conversations` | manager, super, renter | `{ "participantUserId": string }` — idempotent get-or-create DM | `Conversation` (200/201) |
| POST | `/pms/conversations/{conversationId}/messages` | participants | JSON `MessageCreateRequest` **or** `multipart/form-data`: field `text` + repeatable `files` | `Message` (201) |
| PUT | `/pms/conversations/{conversationId}/read` | participant | mark read | 204 |

**Staged flow:** `POST` §16.1 with `purpose=message` → `attachmentIds` in JSON body for `MessageCreateRequest`.

*(Alternative: WebSocket channel `ws://.../pms/chat` for realtime text; **file sends** often stay REST POST for reliability.)*

---

## 15. Reporting & dashboards

Maps to: `ManagerDashboard`, `ManagerReports`, super/renter dashboard widgets.

| Method | Path | Roles | Response |
|--------|------|-------|----------|
| GET | `/pms/buildings/{buildingId}/reports/manager-summary` | manager | optional `?month=YYYY-MM` | `ManagerDashboardSummary` |
| GET | `/pms/buildings/{buildingId}/reports/revenue-vs-cost` | manager | `from`, `to` | `{ "series": ChartSeriesPoint[] }` |
| GET | `/pms/buildings/{buildingId}/reports/payments-collection` | manager | period | `{ "series": ChartSeriesPoint[] }` |
| GET | `/pms/buildings/{buildingId}/reports/costs-by-category` | manager | period | `{ "series": ChartSeriesPoint[] }` |
| GET | `/pms/me/dashboard` | super, renter | Role-specific small summary | object (define per role) |

---

## 16. Files (`file` module) — shared uploads for tickets, messages, invoices, avatars

Central place for **binary storage** (local path or object storage behind **`DOCUMENT_STORAGE_PATH`** / cloud). All features that need blobs should go through these contracts.

### 16.1 Staged upload (recommended for tickets & messages)

| Method | Path | Roles | Request | Response |
|--------|------|---------|---------|----------|
| POST | `/pms/uploads` or `/files/upload` | authenticated | `multipart/form-data`: field `file` (required); optional fields `purpose` (`UploadPurpose`), `buildingId` | **201** `{ "attachmentId": string, "fileId": string, "url": string, "originalName": string, "mimeType": string, "sizeBytes": integer }` |

Client keeps `attachmentId` until ticket/comment/message is submitted; server garbage-collects orphans after TTL (e.g. 24h) if never linked.

### 16.2 Download / metadata

| Method | Path | Roles | Response |
|--------|------|-------|----------|
| GET | `/pms/files/{fileId}` or `/pms/attachments/{attachmentId}` | scoped participant | 302 to signed URL **or** stream with `Content-Disposition` | binary |
| GET | `/pms/files/{fileId}/metadata` | scoped | `FileAttachment` without sensitive internal paths | 200 JSON |

### 16.3 Delete

| Method | Path | Roles | Notes |
|--------|------|-------|--------|
| DELETE | `/pms/attachments/{attachmentId}` | uploader (time-limited), manager | Or only via ticket/message delete — product choice |

**Use returned `url` / `attachmentId` in:** `Ticket.attachments`, `TicketComment.attachments`, `Message.attachments`, `Invoice.receiptUrl`, user `avatar`.

---

## 17. Notifications (optional for bell UI)

Reuse **`GET /notifications/web`** etc. from `notification` swagger, or add:

| Method | Path | Response |
|--------|------|----------|
| GET | `/pms/me/notifications` | In-app feed aligned to ticket/payment/lease events |

---

## 18. Contact / marketing form (landing)

If persisted server-side:

| Method | Path | Body | Response |
|--------|------|------|----------|
| POST | `/pms/public/contact` | `{ "name", "email", "subject", "message" }` | 202 / 204 |

(No auth; rate-limit.)

---

## 19. Traceability: `AppContext` → APIs

| Client method | HTTP mapping |
|----------------|--------------|
| `login` / `logout` | POST `/auth/login`; clear tokens client-side; optional POST revoke |
| `addTicket` | POST `/pms/buildings/{id}/tickets` (optional `attachmentIds` or multipart `files`) |
| `updateTicket` | PATCH `/pms/tickets/{id}` |
| `addTicketComment` | POST `/pms/tickets/{id}/comments` (optional `attachmentIds` or multipart) |
| *(ticket files)* | POST `/pms/uploads` then link ids, or POST `/pms/tickets/{id}/attachments` |
| `addPayment` / `updatePayment` | POST/PATCH `/pms/.../payments` |
| `addCost` / `deleteCost` | POST / DELETE `/pms/.../costs` |
| `addStaff` / `updateStaff` | POST / PATCH `/pms/.../staff` |
| `addTimeEntry` / `updateTimeEntry` | POST / PATCH `/pms/.../time-entries` |
| `addInvoice` / `updateInvoice` | POST / PATCH `/pms/.../invoices` |
| `addEvent` / `updateEvent` / `joinEvent` / `leaveEvent` | POST/PATCH events + POST/DELETE attendees |
| `addGroup` / `updateGroup` / `deleteGroup` / `joinGroup` / `leaveGroup` | CRUD groups + members |
| `sendMessage` | POST `/pms/conversations/{id}/messages` (JSON or multipart with `files`) |
| *(message files)* | POST `/pms/uploads` with `purpose=message`, then `attachmentIds` on send |
| `getConversation` | GET or POST `/pms/conversations` (get-or-create) |

---

## 20. Example JSON payloads

### Create ticket (resident)

```json
{
  "title": "AC not cooling",
  "description": "Thermostat at 78F, AC running continuously.",
  "category": "hvac",
  "priority": "high",
  "unitNumber": "102"
}
```

### Create ticket with staged photos (after `POST /pms/uploads`)

```json
{
  "title": "Leak under sink",
  "description": "See attached photos.",
  "category": "plumbing",
  "priority": "medium",
  "unitNumber": "101",
  "attachmentIds": ["att-550e8400-e29b-41d4-a716-446655440000"]
}
```

### Post comment with PDF / image ids

```json
{
  "text": "Vendor quote attached.",
  "attachmentIds": ["att-6ba7b810-9dad-11d1-80b4-00c04fd430c8"]
}
```

### Patch ticket (manager assigns)

```json
{
  "status": "in-progress",
  "assignedTo": "u2"
}
```

### Post comment (text only)

```json
{
  "text": "I'll stop by after 3pm."
}
```

### Send message with attachment ids

```json
{
  "text": "Here is the lease addendum.",
  "attachmentIds": ["att-7c9e6679-7425-40de-944b-e07fc1f90ae7"]
}
```

### Record rent payment (manager)

```json
{
  "status": "paid",
  "paidDate": "2026-04-10",
  "method": "bank-transfer",
  "note": "ACH confirmation #99821"
}
```

### Create staff invoice (super)

```json
{
  "date": "2026-04-06",
  "notes": "Materials for unit 101",
  "items": [
    { "description": "Faucet washer kit", "quantity": 1, "unitPrice": 12.99, "total": 12.99 }
  ],
  "totalAmount": 12.99
}
```

### Send message

```json
{
  "text": "Hi, when can you review my lease renewal?"
}
```

---

## 21. HTTP status summary

| Code | Use |
|------|-----|
| 200 | OK with body |
| 201 | Created |
| 204 | No content (delete, mark read) |
| 400 | Validation |
| 401 | Missing/invalid token |
| 403 | Role or scope denied |
| 404 | Hidden cross-tenant or missing id |
| 409 | Conflict (duplicate room, double payment idempotency) |

---

## 22. Backend modules — coverage vs. `pms-web`

Maven modules under `pms-backend/modules/` vs this spec:

| Module | What `pms-web` needs from it | In this spec today? | Typical extra APIs (beyond kickoff UI) |
|--------|------------------------------|---------------------|----------------------------------------|
| **user** | Login, refresh, profile, roles, user list for directory | Yes (§3, §4) | Register, activation, password reset — already in user swagger |
| **property** | Buildings, units, scope for tickets/payments | Partially (§5) | Org/property CRUD, unit amenities, floor plans |
| **lease** | Link renter to unit; rent schedule source; renewal context (messages in mock) | **Implicit** via `RentalPayment.leaseId` / `/pms/me/unit` | `GET/PATCH /leases`, renewals, parties, documents |
| **billing** | Rental charges, payments, aging | Yes (§7) | Invoices to *tenants*, credits, idempotent payment POST |
| **maintenance** | Full ticket lifecycle | Yes (§6) | SLA fields, vendors, scheduling |
| **notification** | Bell / push (optional) | Mentioned (§17) | Template-driven events from billing/maintenance |
| **audit** | Not in mock UI | **Not** | `GET /pms/.../audit-events` for manager compliance views |
| **admin** | Not in mock UI | **Not** | Building settings, feature flags, integrations |
| **reporting** | Manager charts / KPIs | Yes (§15) | Heavy aggregates, CSV export, scheduled reports |
| **file** | Ticket & comment & **message** attachments; invoice receipts; avatars | Yes — **§16** (+ §6.1, §14.1) | Virus scan, MIME/size policy, signed URLs, orphan TTL |
| **chat** | Resident ↔ staff messaging | §14 (new **PMS** conversation API) | Existing chat swagger is **case-centric** — implement PMS paths or adapt |

**Community** (events/groups) has **no dedicated Maven module** yet; options: new `community-mgt` module, or nest under `property` / `admin` until bounded context is clear.

### Do you need more in the spec?

- **For current `pms-web` screens only:** Sections **1–21** are enough to implement REST + models; **lease** should get explicit endpoints when you stop hard-coding tenant rent rows.
- **For full parity with all backend modules:** Add OpenAPI sections for **lease** (read/update for renter + manager), **audit** (read list), **admin** (building config), and wire **file** upload contracts to ticket/invoice flows.

*This spec remains the target contract for **`pms-web`** integration; extend §5–§7 and add lease/audit/admin appendices as those UIs ship.*
