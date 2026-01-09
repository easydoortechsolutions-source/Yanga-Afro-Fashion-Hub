# Yanggah – Architecture (MVP)

## Goal of this document
Define a clean, minimal architecture that supports the MVP:
- Marketplace listings
- Verified vendors
- Escrow-style payment flow (hold → confirm → release)
- Orders + disputes
Without overbuilding.

This document is a living reference for implementation decisions and system boundaries.

---

## System boundaries (what Yanggah is / isn’t)

### In scope for MVP
- User accounts (buyers, vendors, admin)
- Vendor verification workflow
- Product listings + media
- Order creation and order state tracking
- Payment intents + “held” funds representation (escrow logic)
- Release/refund decisions (admin-controlled for MVP)
- Disputes (basic)

### Out of scope for MVP
- Owning logistics operations
- Automated fraud/ML scoring
- Complex returns management
- Real-time chat
- Multi-warehouse inventory management
- Custom tailoring workflows

---

## Core services (logical components)

### 1) Client (Web App)
Responsibilities:
- Browse products
- View vendor profiles
- Checkout initiation
- View order status
- Confirm delivery
- Open disputes

Notes:
- UX must emphasize trust: verification, escrow, clear status.

### 2) API / Backend (Application layer)
Responsibilities:
- Auth + role checks
- CRUD for products (vendor-limited)
- Order state machine enforcement
- Payment orchestration (create/confirm/release/refund)
- Dispute workflow
- Admin moderation endpoints

### 3) Database (System of record)
Responsibilities:
- Users, roles
- Vendors + verification status
- Products + inventory basics
- Orders + order items
- Payments ledger (authoritative internal record of “what should be true”)
- Disputes + evidence

Principle:
The database is the source of truth. External payment providers are “effectors” and must be reconciled.

### 4) Storage (Media)
Responsibilities:
- Product images
- Verification documents (if collected; ideally minimized)
- Dispute evidence attachments

Security note:
Sensitive documents must be access-controlled and auditable.

### 5) Background Jobs / Webhooks
Responsibilities:
- Payment provider webhooks → update internal payment records
- Time-based events (optional): auto-cancel unpaid orders, reminders, etc.
- Reconciliation tasks (ensure internal ledger matches provider events)

---

## Roles & permissions (MVP)
- Buyer: browse, purchase, confirm delivery, dispute
- Vendor: manage own products, see own orders, fulfill orders
- Admin: approve vendors, moderate listings, decide disputes, release/refund payments

Rule:
Every request is authorized by role + ownership checks.

---

## State machines (the heart of the MVP)

### Vendor verification states
- applied
- under_review
- approved
- rejected
- suspended

Only `approved` vendors can publish products.

### Order states (MVP minimal)
- draft (optional; pre-payment)
- pending_payment
- paid_held  (payment captured/authorized and considered “held” by Yanggah logic)
- in_fulfillment
- delivered_pending_confirmation
- completed (funds released)
- cancelled
- disputed
- refunded

Notes:
- The exact payment provider mechanics (authorize vs capture vs transfer) is implementation detail.
- Internally we represent “held” and “released” explicitly regardless of provider.

### Dispute states
- opened
- under_review
- resolved_buyer_refund
- resolved_vendor_release
- resolved_split (optional; TBD)

---

## Payment design (MVP principle)
We maintain an internal ledger that records:
- what we intended to happen (intent)
- what the provider says happened (events)
- what decision was made (release/refund)
- immutable audit history

MVP payment policy (decision):
- Admin-controlled release/refund while learning.
- Automations (auto-release after X days) is TBD.

---

## Reliability & auditability requirements
- Idempotency: payment operations must be safe to retry
- Audit trail: all admin decisions recorded with actor + timestamp + reason
- Webhook verification: provider webhooks validated
- Observability: errors logged with correlation IDs (TBD exact tooling)

---

## Non-functional constraints (MVP)
- Keep architecture boring and maintainable
- Prefer fewer moving parts
- Minimize sensitive data collection
- Ensure future migration paths (e.g., adding logistics providers, expanding payment rails)

---

## Open decisions (explicit TBD)
- Tech stack finalization (e.g., Next.js + Supabase vs other)
- Payment provider(s) for MVP and exact escrow mechanism
- Whether delivery confirmation is buyer-only or includes proof-of-delivery
- Auto-release policy (time-based release) and dispute window length
