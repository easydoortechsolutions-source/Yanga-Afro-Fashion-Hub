#data-model# Yanggah – Data Model (MVP)

## Goal of this document
Define a minimal, normalized data model that supports:
- Roles (buyer/vendor/admin)
- Vendor verification
- Products and listings
- Orders and order items
- Payments ledger (internal source of truth)
- Disputes + evidence
With a clean upgrade path.

This is a conceptual schema; implementation may add indexes and constraints.

---

## Core entities

### users
Represents all authenticated accounts.

Fields:
- id (PK)
- email (unique)
- phone (optional; TBD requirement)
- name (optional)
- role (buyer | vendor | admin)  // or separate role table
- created_at
- updated_at
- status (active | disabled)

Notes:
- If a user becomes a vendor, they still remain a user; vendor details live in vendors.

---

### vendors
Represents vendor/business profile and verification status.

Fields:
- id (PK)
- user_id (FK → users.id, unique)
- display_name
- bio (optional)
- location (optional)
- verification_status (applied | under_review | approved | rejected | suspended)
- verification_submitted_at
- verification_reviewed_at
- reviewed_by (FK → users.id; admin)
- rejection_reason (optional)
- created_at
- updated_at

---

### vendor_verification_documents (optional; minimize)
Store references to verification artifacts only if necessary.

Fields:
- id (PK)
- vendor_id (FK → vendors.id)
- doc_type (TBD list)
- storage_url (or storage key)
- uploaded_at
- access_level (admin_only)
- deleted_at (optional)

Security:
- Avoid storing raw sensitive data where possible.

---

### products
Products listed by vendors.

Fields:
- id (PK)
- vendor_id (FK → vendors.id)
- title
- description
- category (TBD enum: agbada, aso-ebi, kaftan, bridal, etc.)
- price_amount
- price_currency (TBD; likely NGN/GBP/etc.)
- status (draft | active | paused | removed)
- created_at
- updated_at

---

### product_media
Images/videos for a product.

Fields:
- id (PK)
- product_id (FK → products.id)
- storage_key / url
- sort_order
- created_at

---

### inventory (MVP optional)
If you need basic stock tracking.

Fields:
- id (PK)
- product_id (FK → products.id, unique)
- quantity_available (int)
- updated_at

Note:
- If inventory is not needed for MVP, omit and treat products as “made to order”.

---

## Orders

### orders
Represents a purchase from a buyer.

Fields:
- id (PK)
- buyer_id (FK → users.id)
- vendor_id (FK → vendors.id)  // MVP assumption: one vendor per order
- status (
    pending_payment |
    paid_held |
    in_fulfillment |
    delivered_pending_confirmation |
    completed |
    cancelled |
    disputed |
    refunded
  )
- subtotal_amount
- fees_amount (platform fee; optional)
- shipping_amount (optional; TBD)
- total_amount
- currency
- delivery_address_id (FK → addresses.id; optional)
- created_at
- updated_at

MVP decision:
- Keep orders single-vendor to simplify escrow + fulfillment. Multi-vendor carts are later.

---

### order_items
Line items in an order.

Fields:
- id (PK)
- order_id (FK → orders.id)
- product_id (FK → products.id)
- product_title_snapshot (string)
- unit_price_snapshot
- quantity
- line_total
- created_at

Principle:
Snapshot key product fields at purchase time to prevent disputes when listings change.

---

### addresses (optional)
If you collect delivery addresses.

Fields:
- id (PK)
- user_id (FK → users.id)
- name (optional)
- line1, line2, city, state, postcode, country
- phone (optional)
- created_at

---

## Payments (internal ledger)

### payment_intents
Represents intent to collect payment for an order.

Fields:
- id (PK)
- order_id (FK → orders.id, unique)
- provider (TBD: stripe/paystack/etc.)
- provider_intent_id (unique; nullable until created)
- amount
- currency
- status (created | requires_action | succeeded | failed | cancelled)
- created_at
- updated_at

---

### payment_events
Immutable log of provider events + internal actions.

Fields:
- id (PK)
- payment_intent_id (FK → payment_intents.id)
- event_type (provider_webhook | internal_action)
- provider_event_id (unique; nullable)
- payload_json (raw; optional but useful)
- recorded_at

Rule:
Never delete events. This is your audit trail.

---

### escrow_ledger_entries
Explicit internal ledger of escrow state transitions.

Fields:
- id (PK)
- order_id (FK → orders.id)
- payment_intent_id (FK → payment_intents.id)
- entry_type (hold | release | refund | fee | adjustment)
- amount
- currency
- actor_user_id (FK → users.id; admin/system)
- reason (text)
- created_at

Principle:
Orders tell you “status”. Ledger tells you “money truth”.

---

## Disputes

### disputes
Represents an issue raised by buyer (or vendor later).

Fields:
- id (PK)
- order_id (FK → orders.id, unique)
- opened_by (FK → users.id)
- status (opened | under_review | resolved_buyer_refund | resolved_vendor_release | resolved_split)
- reason_code (TBD enum)
- description
- opened_at
- resolved_at
- resolved_by (FK → users.id; admin)
- resolution_notes (text)

---

### dispute_evidence
Attachments/notes for a dispute.

Fields:
- id (PK)
- dispute_id (FK → disputes.id)
- submitted_by (FK → users.id)
- evidence_type (image | text | file; TBD)
- storage_key / url (nullable if text)
- text (nullable)
- created_at

---

## Admin decisions (optional but recommended)

### admin_actions
Captures any sensitive admin operation (approve vendor, remove listing, resolve dispute).

Fields:
- id (PK)
- admin_user_id (FK → users.id)
- action_type (approve_vendor | reject_vendor | remove_product | resolve_dispute | release_funds | refund | etc.)
- target_type (vendor | product | order | dispute | payment_intent)
- target_id (string/uuid)
- reason (text)
- created_at

Principle:
If an admin can do it, it should be logged.

---

## Relationships summary (MVP)
- users 1—1 vendors (optional)
- vendors 1—N products
- products 1—N product_media
- users (buyers) 1—N orders
- vendors 1—N orders
- orders 1—N order_items
- orders 1—1 payment_intent
- payment_intents 1—N payment_events
- orders 1—N escrow_ledger_entries
- orders 0/1—1 disputes
- disputes 1—N dispute_evidence

---

## Open decisions (explicit TBD)
- Whether inventory is tracked or made-to-order
- Whether shipping is calculated/charged in MVP
- Currency strategy for MVP (single-currency vs multi-currency)
- Whether orders can contain multiple vendors (defer)
- Required vendor verification fields and document types

