# Yanga – Workflows (MVP Operating Playbook)

## Purpose
This document defines the **human-operable workflows** for Yanga’s MVP:
- Buyer journey (browse → pay → receive → confirm/dispute)
- Vendor journey (apply → list → fulfill → payout)
- Admin operations (approve vendors, moderate products, resolve disputes, control funds)

These workflows must align with:
- `architecture.md` (states + responsibilities)
- `data-model.md` (entities + relations)
- `payments.md` (internal ledger + escrow logic)

No implementation should invent new flows beyond what is written here.

---

## Global rules (apply to all workflows)

### R1 — Escrow principle
Buyer funds are treated as **held** until:
- Buyer confirms delivery, OR
- Admin resolves a dispute, OR
- Auto-release triggers (see R4)

### R2 — Audit trail is mandatory
Any admin action that affects:
- vendor status
- product status
- disputes
- refunds/releases/splits
must create an auditable record (e.g., `admin_actions`) with:
- actor (admin)
- timestamp
- decision
- reason/notes

### R3 — Money truth is the ledger
All fund movements (hold/release/refund/fees/adjustments) must be recorded in the internal ledger
(e.g., `escrow_ledger_entries`) before/alongside provider operations.

### R4 — Buyer confirmation must be time-bound
To prevent indefinite holds, every order in `delivered_pending_confirmation` has an **auto-release date**.
- Buyer UI must show the date clearly.
- If the buyer takes no action before the deadline, the order auto-completes and funds release to vendor.
- Exact duration is TBD (policy), but the existence of a timer is not optional.

### R5 — Product visibility is gated
Products do **not** go live immediately on vendor submission.
They follow:
- `draft` → `pending_review` → `active` (or `rejected` / `paused`)

### R6 — Payouts only from released funds
Vendor “earnings” must separate:
- Held (escrow)
- Released (available)
- Paid out (withdrawn)
Vendors can only withdraw from **released** balances.

### R7 — Shipping is vendor-to-buyer (Phase 1)
Yanga does not book couriers or route deliveries.
Yanga records:
- shipping provider name (optional)
- tracking number (recommended)
- shipped timestamp

---

## Status glossary (minimum set)

### Vendor verification status
- applied
- under_review
- approved
- rejected
- suspended

### Product status
- draft
- pending_review
- active
- paused
- rejected
- removed

### Order status (MVP)
- pending_payment
-pending_vendor_quote
- paid_held
- accepted
- in_progress
- shipped
- delivered_pending_confirmation
- completed
- cancelled
- disputed
- refunded

### Dispute status
- opened
- under_review
- resolved_buyer_refund
- resolved_vendor_release
- resolved_split (optional; later)

---

# Workflow 1 — Buyer (MVP)

Source: Buyer MVP flow. :contentReference[oaicite:3]{index=3}

## B1. Discover & browse
1. Buyer lands on homepage
2. Browses categories / uses search + filters
3. Opens a product detail page (PDP)
4. Reviews:
   - images
   - description
   - price
   - delivery estimate
   - “Protected by Yanga Escrow” messaging
5. Optionally views vendor profile for trust signals

## B2. Customize & checkout
6. Buyer selects size OR enters measurements (if product supports made-to-measure)
7. Buyer chooses variations (color/fabric/etc.)
8. Buyer adds optional note to vendor
9. Buyer adds to cart and proceeds to checkout
10. Buyer enters delivery details
11. Buyer reviews order summary and escrow explanation
12. Buyer selects payment method and pays

System outcome:
- If payment fails → order remains unpaid; buyer can retry/cancel
- If payment succeeds:
  - order status = `paid_held`
  - ledger entry = `hold` for total amount
  - buyer receives confirmation + order ID
 
## B2a – Quote-based checkout (custom / fabric items)

If the product is marked as “made-to-measure” or “fabric-based”:

1. Buyer selects product, enters measurements, and submits order request.
2. Order is created with:
   - status = `pending_vendor_quote`
   - no payment captured yet

3. Buyer sees:
   “Waiting for vendor to confirm final price.”

4. Vendor submits a final quote (price + delivery time).

5. Buyer receives notification and sees:
   - Final price
   - Delivery estimate
   - Accept or Cancel buttons

Decision:
- If buyer accepts:
  - Full amount is charged
  - Escrow ledger entry = `hold`
  - Order status becomes `paid_held`
  - Flow continues to normal fulfillment
- If buyer rejects:
  - Order status = `cancelled`
  - No money moves
  - Both parties are notified


## B3. Track order
13. Buyer sees order status updates:
- pending vendor acceptance
- in progress
- shipped
- delivered_pending_confirmation

When vendor marks shipped:
- buyer receives notification
- buyer can view tracking info (if provided)

## B4. Delivery decision (confirm vs dispute)
14. Upon delivery, buyer chooses:
- Confirm received → funds release
- Raise dispute → funds remain held; admin reviews

System rule:
- When status becomes `delivered_pending_confirmation`, show:
  - “Confirm or dispute by [AUTO-RELEASE DATE]”
  - After that date, funds auto-release to vendor (policy window TBD)

---

# Workflow 2 — Vendor (MVP)

Source: Vendor MVP flow. :contentReference[oaicite:4]{index=4}

## V1. Vendor onboarding & verification
1. Vendor clicks “Become a Vendor”
2. Creates account (email/password/phone)
3. Completes vendor profile (business name, full name, city/country, contact)
4. Uploads verification documents (ID + business proof)
5. Submits application

System outcome:
- vendor verification status = `applied` (or `under_review`)

Admin decision:
- If rejected → vendor notified; cannot sell
- If approved → vendor status = `approved`; vendor gets dashboard access

## V2. Product listing (with moderation gate)
6. Vendor clicks “Add Product”
7. Enters product details:
- title, images, description, category, price
- variations (size/color/fabric)
- made-to-measure toggle (optional)
- production time estimate
- shipping/delivery timeframe estimate

Vendor submits product.

System outcome (corrected):
- product status = `pending_review`
- product is NOT visible to buyers yet

Admin decision:
- If approved → product status = `active` (visible)
- If rejected → product status = `rejected`; vendor must edit/resubmit

## V3. Order management
8. Vendor receives new order notification
9. Vendor opens order and reviews selections/measurements/notes
10. submit_final_quote

When a new order arrives:

If order.status = `pending_vendor_quote`:
1. Vendor reviews:
   - Buyer measurements
   - Selected options
   - Notes and delivery deadline
2. Vendor enters:
   - Final price
   - Estimated production & delivery time
3. Vendor clicks “Submit Final Quote”

System:
- Stores quote
- Notifies buyer
- Order remains `pending_vendor_quote` until buyer responds

  If buyer accepts the quote:
- Order becomes `paid_held`
- Vendor can then accept and start production

If buyer rejects:
- Order becomes `cancelled`
- No escrow exists


Decision: Accept order?
- If NO:
  - order status = `cancelled`
  - ledger action = refund buyer (full amount) with reason "vendor_declined"
  - buyer notified
- If YES:
  - order status = `accepted`
  - vendor begins production

## V4. Production updates
10. Vendor updates:
- `in_progress`
- optional notes/messages (MVP optional)

## V5. Shipping & tracking
11. Vendor completes item and marks order as `shipped`
12. Vendor enters:
- shipping provider (optional)
- tracking number (recommended)
- shipped date/time (system captured)

System:
- buyer notified
- order shows tracking info (if available)

## V6. Completion & payout eligibility
13. Buyer confirms received OR auto-release triggers OR dispute resolved.

System:
- If released to vendor:
  - ledger entry = `release` (vendor net amount)
  - order status = `completed`
  - vendor “available balance” increases

Vendor withdrawal:
- Vendor may request payout only from released/available balance
- “Escrow held” is not withdrawable

---

# Workflow 3 — Admin (MVP)

Source: Admin MVP flow. :contentReference[oaicite:5]{index=5}

## A1. Vendor management (verification)
1. Admin views pending vendor applications
2. Admin reviews:
- business name, contact info
- ID docs, business proof

Decision:
- Approve → vendor status = `approved`
- Reject → vendor status = `rejected` + rejection reason

System:
- Vendor notified (email/SMS)

Audit requirement:
- Every approve/reject creates `admin_action` with reason.

## A2. Product moderation (QC gate)
1. Admin views products in `pending_review`
2. Admin checks:
- image quality
- correct category
- prohibited items
- reasonable pricing
- appropriate description

Decision:
- Approve → product status = `active`
- Reject → product status = `rejected` + reason
- Pause/remove (later) → `paused` or `removed`

Audit requirement:
- Every moderation decision creates `admin_action` with reason.

## A3. Order monitoring
Admin can view all orders and their:
- buyer details
- vendor details
- order notes
- escrow held amount
- shipping/tracking info (if provided)

Admin should not “edit” orders arbitrarily.
Admin actions must be explicit and logged.

## A4. Escrow oversight (ledger-first)
Admin views:
- funds held
- funds released
- pending disputes
- payout queue

Admin money actions must:
1) Write ledger entry (hold/release/refund/adjustment/fee)
2) Execute provider action (transfer/refund)
3) Record provider event
4) Update order/payment status accordingly

## A5. Dispute resolution (critical)
Trigger:
- Buyer clicks “Raise dispute”
- Order status becomes `disputed`
- Funds remain held

Admin reviews:
- buyer complaint text
- photos/evidence (optional)
- vendor response (optional for MVP)
- product listing snapshots
- shipping/tracking info

Decision outcomes:
1) Refund buyer
   - dispute status = `resolved_buyer_refund`
   - order status = `refunded`
   - ledger entry = `refund` (full or partial) + reason
2) Release to vendor
   - dispute status = `resolved_vendor_release`
   - order status = `completed`
   - ledger entry = `release` + reason
3) Split (optional later)
   - dispute status = `resolved_split`
   - ledger entries reflect split amounts + reason

Admin UI requirement (must-have):
- Admin must input a decision reason + optional notes before submitting resolution.
- Resolution must create:
  - `admin_action`
  - ledger entries
  - notifications to both parties

## A6. Payout management
Admin views payout requests and vendor bank details.

Decision:
- Approve payout:
  - payout is processed from released balance
  - payout recorded and vendor notified
- Hold payout:
  - reason required
  - vendor notified to correct details
 
Admins must not release or refund funds for orders in `pending_vendor_quote`,
because no escrow exists yet.

Admins may only intervene if:
- Vendor is abusing the quote system
- Buyer reports a problem


Audit requirement:
- Payout approvals/holds create `admin_action`.
- Payout execution creates ledger/provider event records.

## A7. User & risk controls (basic MVP)
Admin can suspend:
- vendors (fraud, repeated disputes, bad behavior)
- buyers (fraud/abuse)

Suspension must:
- set status = suspended/disabled
- prevent new orders/listings
- be logged with reason

---

## Special policy notes (MVP)

### Custom-made / made-to-measure items
- Disputes must consider that sizing/customization can be subjective.
- Policy TBD, but the workflow must allow admins to request evidence:
  - measurements submitted
  - product description
  - photos upon delivery

Recommendation for MVP policy placeholder:
- “Custom items are only refundable if materially misdescribed or clearly defective.” (final wording TBD)

### Tracking info policy
- Tracking is strongly recommended for vendors.
- If no tracking is provided, admins should weight disputes accordingly (policy TBD).
- The system should still allow shipping without tracking in MVP (to avoid blocking vendors), but log it.

---

## UI/Implementation checklist (for frontend + app build)

Buyer:
- Escrow badge on PDP & checkout
- Tracking display (if present)
- Confirm vs dispute CTA
- Auto-release date visible on delivered orders

Vendor:
- Vendor verification status UI
- Product status UI (draft/pending_review/active/rejected)
- Shipping form includes provider + tracking number
- Earnings split: Held vs Released vs Paid out

Admin:
- Vendor approval screens with reason field
- Product moderation queue with reason field
- Dispute queue with decision + reason + evidence viewer
- Money actions always create audit + ledger records
