# PMS Production-Readiness Plan — Billing Focus

**Date:** 2026-06-10
**Author:** Claude (stabilization review)
**Status:** Proposed — awaiting review

This document is the pre-work analysis requested before any implementation. It covers the current
state of the system, concrete defects found in the billing module, proposed changes with reasoning,
and a phased implementation plan. Nothing described here has been implemented yet.

---

## 1. Current-State Assessment

The system is in better shape than the "mock data everywhere" framing suggests:

- **Frontend:** no `TODO`/`FIXME` markers remain in `pms-frontend/src`. `src/data/mockData.ts` is
  dead code — nothing imports it. Manager billing, invoices, time-sheets, and resident pages are
  wired to real APIs through Redux thunks (`billingSlice`, `tenantSlice`, etc.).
- **Backend:** only 3 TODOs remain, all in `admin-mgt` `UserManagementServiceImpl` (storage quota
  integration — minor). All 15 modules follow the same controller/service/repository/mapper
  convention with `@Secured` role checks on controller implementations and MQ event publishing.
- **Security:** JWT + stateless Spring Security; all non-public paths require authentication;
  role enforcement via `@Secured` on controller impls (verified present across 22 controllers,
  including all 3 billing controllers).
- **Tests:** integration test infrastructure exists (`main-execution/src/test`), including
  `PaymentIntegrationTest` and `InvoiceIntegrationTest`; unit test suites exist for billing
  services, mappers, repositories, and query builders.

The remaining production risk is concentrated in **billing correctness**, a few **broken
end-to-end workflows**, and **missing financial automation**. Details below.

---

## 2. Findings

### P0 — Financial correctness / data integrity (must fix before client use)

#### F-1. `getInvoicesByBuilding` ignores `buildingId` — cross-building data leak
`InvoiceServiceImpl.getInvoicesByBuilding(...)` builds an `InvoiceSearchFilter` **without the
buildingId**, so `GET /buildings/{id}/invoices` returns every staff invoice in the system
regardless of building. Root cause: the billing `Invoice` entity does not map the `building_id`
column at all — even though the column exists in the `staff_invoice` table (the staff-mgt
`StaffInvoice` entity maps it). Consequently `createInvoice` also persists `building_id = NULL`.

**Fix:** add `buildingId` to billing's `Invoice` entity, `InvoiceSearchFilter`, and
`InvoiceQueryBuilder`; populate it on create (resolve the superintendent's assigned building via
the staff proxy); apply the filter in `getInvoicesByBuilding`. Backfill existing NULL rows via a
guarded Liquibase changeset.

#### F-2. Resident "Pay Rent" submit is a no-op
`pms-frontend/src/app/resident/pay-rent/page.tsx` — `handleSubmitPayment` only shows a success
toast ("submitted to the manager") and closes the modal. No API call is made; the manager is never
notified; nothing is recorded. Additional problems on the same page:
- **Invented late fee:** `lateFees = overduePayments.length * 50` — a hardcoded $50/overdue charge
  that exists nowhere in the backend. A resident could be shown (and asked to "pay") money the
  business never billed.
- **Fabricated "scheduled payments"** list (next 3 months synthesized client-side).
- **Auto-pay toggle** is pure UI state, persisted nowhere.

**Fix (proposed workflow — no real money movement, consistent with the system's record-keeping
model):** resident submits a *payment claim* against a specific pending/overdue payment —
`PATCH`-like endpoint `POST /me/payments/{paymentId}/payment-request` carrying method + note. This
sets a new `payment_requested` flag/status, publishes a billing MQ event (notification to the
manager, audit entry), and the manager confirms by marking the payment PAID through the existing
manager flow. Remove the hardcoded late fee and the fabricated schedule (or derive the schedule
from actual pending payments); remove or persist the auto-pay toggle (recommend remove for v1).

#### F-3. Monetary values stored as `Double`
`RentalPayment.amount`, `Invoice.totalAmount`, `Cost.amount`, `Lease.monthlyRent`,
`Lease.securityDeposit`, `Unit.rentAmount`, `Tenant.leaseMonthlyRent`, `StaffInvoice.totalAmount`
are all `Double` (only `Staff.salary` uses `BigDecimal`). Floating-point money accumulates
rounding error in sums (the manager dashboard already sums these client-side) and is the classic
source of cent-level discrepancies.

**Fix:** migrate all monetary fields to `BigDecimal` with `DECIMAL(12,2)` columns (Liquibase
`modifyDataType` changesets are lossless for this direction), update VTOs/mappers/OpenAPI specs
(`number` → `format: decimal` handled as BigDecimal), and frontend types stay `number` (JSON
unchanged). This touches billing, lease, property, staff modules — mechanical but wide; isolated
in its own phase with full regression run.

#### F-4. PAID payments remain mutable
`RentalPaymentServiceImpl.patchPayment` only guards against modifying a payment when
`request.getStatus() != null`. A request that sends **only** `amount` (or method/note/paidDate)
freely rewrites a PAID payment's amount — corrupting collected-revenue figures after the fact.

**Fix:** reject any mutation of a PAID payment except note edits (business call — proposal:
amount/dueDate/status immutable once PAID; allow note). Emit audit event on any post-PAID edit.

#### F-5. No validation on monetary inputs
Billing create/patch requests have no `@DecimalMin`/`@Positive` constraints — negative or zero
amounts are accepted for payments, costs, and invoices. Invoice `totalAmount` is also never
reconciled against the sum of its line items, and a JSON serialization failure of the items list
is silently swallowed (`itemsJson = "[]"`) while keeping the stated total.

**Fix:** add `minimum: 0.01` to amounts in the OpenAPI specs (regenerate) or bean validation on
the request models; in `createInvoice`, compute the items sum server-side and reject when it
differs from `totalAmount` by more than 0.005; replace the silent JSON catch with a
`BusinessException` (an invoice with lost line items must not be created).

### P1 — Missing core functionality

#### F-6. No billing automation (scheduler)
There is **no `@Scheduled` job anywhere in the backend**. Consequences:
- `RentalPaymentStatus.OVERDUE` exists but nothing ever sets it. Frontend stats ("overdue" cards,
  filters, resident warnings) can never trigger from real data.
- Monthly rent payments must be hand-created by the manager per tenant per month — the
  `recurring`/`frequency` fields on `Cost` are stored but never materialize new cost rows.

**Fix:** add a `billing-scheduler` (Spring `@EnableScheduling`, guarded by config flag +
`ShedLock`-style guard or single-instance assumption documented):
1. Nightly job: `pending` payments past `dueDate` → `overdue`, publishing an MQ event
   (notification to tenant + manager, audit).
2. Monthly job: for each ACTIVE lease, generate next month's rental payment from
   `lease.monthlyRent` if one doesn't already exist (idempotent by tenant+dueDate).
3. Recurring cost materialization per `frequency` (monthly/quarterly/yearly), idempotent.

#### F-7. Duplicate ownership of `staff_invoice`
Both `billing.model.entity.Invoice` and `staff.model.entity.StaffInvoice` map the same table with
divergent column sets (billing's misses `building_id`). Two write paths to one financial table is
a drift hazard.

**Fix (minimal, consistent with module boundaries):** keep both short-term but align the mappings
(F-1 adds the missing column); document staff-mgt as the manager-approval read/write path and
billing as the superintendent submit path. Longer-term recommendation: move invoice approval fully
into billing-mgt and have staff-mgt consume it via proxy — recorded here as a refactor candidate,
not scheduled.

### P2 — Cleanup / polish

- **F-8.** Delete dead `pms-frontend/src/data/mockData.ts` (unused, 200+ lines).
- **F-9.** Resolve the 3 admin storage-quota TODOs (wire to file-mgt storage stats or remove the
  fields from the admin API).
- **F-10.** Frontend computes billing stats by summing the **current page** of payments only
  (pagination makes "collected/outstanding" cards wrong once data exceeds a page). Add a small
  backend summary endpoint (`GET /buildings/{id}/payments/summary`) or document page-scope.
- **F-11.** E2E coverage gaps: resident has 10 pages / 2 specs; superintendent settings none.
  Add Playwright specs for pay-rent (after F-2), billing, invoices approval loop.

### Explicit non-goals (decisions for the client/owner)

- **Real payment processing (Stripe/ACH):** out of scope for this plan. The system is a
  record-keeping PMS today; F-2 makes the resident flow honest without moving money. Gateway
  integration is a separate project (PCI, webhooks, reconciliation).
- **Late-fee policy:** no backend concept exists. The $50 frontend constant is being removed
  (F-2). If the client wants automatic late fees, that's a new feature (config per building,
  applied by the F-6 scheduler) — needs a business decision first.

---

## 3. Implementation Plan

Each phase compiles, passes the full test suite, and is independently committable.

**Phase 1 — Billing correctness (F-1, F-4, F-5)**
Entity/filter/query changes + Liquibase changeset for invoice building scoping; PAID immutability
guard; amount validation + invoice total reconciliation + JSON failure hard-stop. Extend
`PaymentIntegrationTest`/`InvoiceIntegrationTest` with: cross-building isolation, paid-payment
mutation rejection, negative amount rejection, mismatched invoice total rejection. Unit tests per
existing Mockito pattern.

**Phase 2 — Resident payment request flow (F-2)**
OpenAPI spec addition → regenerate controller/models; `payment_requested` state + MQ events +
notification listener entries + audit; frontend: wire pay-rent modal to the new endpoint, surface
"requested" state on manager billing page (approve = existing mark-paid), remove late-fee constant,
fabricated schedule, and dead auto-pay toggle. Playwright spec for the loop.

**Phase 3 — Billing automation (F-6)**
Scheduler module/config + the three idempotent jobs + events/notifications + integration tests
(time-controlled via injected Clock).

**Phase 4 — Money type migration (F-3)**
`Double` → `BigDecimal`/`DECIMAL(12,2)` across billing/lease/property/staff + mappers/VTOs/specs +
full regression. Done after Phases 1–3 so functional fixes aren't entangled with a wide mechanical
change.

**Phase 5 — Cleanup & coverage (F-8…F-11)**
Dead code removal, admin TODOs, payments summary endpoint + frontend stat wiring, e2e specs.

**Verification per phase:** `mvn verify` (unit + integration, Docker MySQL), JaCoCo on billing
module, `npm run build` + `npx playwright test` for frontend phases.

---

## 4. Risk Notes

- Liquibase changesets must be guarded (`preConditions`) — prior incident with `staff_invoice`
  missing migration (changeset-2026-06-07) sets the pattern.
- F-1 backfill: existing invoices have NULL `building_id`; backfill from the staff member's
  building assignment where resolvable, leave NULL otherwise and have the building-scoped query
  exclude them (manager "all invoices" endpoint still shows them).
- F-3 changes JSON number serialization only in precision edge cases; frontend `formatCurrency`
  already rounds to 2dp — low UI risk.
- Phase 2 adds a new status value; frontend `PAYMENT_STATUS_CONFIG` must be extended in the same
  change to avoid unstyled badges.
