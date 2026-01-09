# Yanggah – Payments & Escrow (MVP)

## Purpose of this document
Define how money moves through Yanggah in a way that:
- Protects buyers
- Is fair to vendors
- Is legally and technically defensible
- Works with imperfect African payment infrastructure
- Scales to multiple providers later

This document defines **internal truth**.  
Payment providers are implementation details.

---

## Core principle

Yanggah is not a simple “pass-through” marketplace.

Yanggah operates an **escrow-like flow**:
- Buyer pays
- Money is considered “held”
- Goods are delivered
- Buyer confirms (or dispute resolves)
- Funds are released to vendor

Even if the payment provider does not support true escrow,  
**Yanggah’s internal ledger enforces escrow logic.**

---

## Two separate systems must exist

### 1) External payment providers
Examples (TBD):
- Card processors
- Bank transfer rails
- Wallets

They do:
- Collect money from buyers
- Store or move money
- Send webhooks

They are **not** the source of truth.

---

### 2) Yanggah internal payment ledger (authoritative)

Yanggah maintains:
- What should be held
- What should be released
- What should be refunded
- Why

This lives in:
- `payment_intents`
- `payment_events`
- `escrow_ledger_entries`

If the provider is wrong or delayed, Yanggah’s ledger is still correct.

---

## Payment lifecycle (MVP)

### 1) Order created
Order status:




Internal:
- A `payment_intent` is created
- No money yet

---

### 2) Buyer pays
Provider returns:
- success
- or requires_action
- or failed

Pending_Payments
Internal:
- payment_intent.status = succeeded (if paid)
- order.status = paid_held
- escrow_ledger_entries:
    - entry_type = hold
    - amount = order.total_amount

This means:
> “Yanga now owes this amount either to the vendor or to the buyer.”

---

### 3) Vendor fulfills
Order status:

in_fulfilment

No money moves.

---

### 4) Delivery
Order status:

delivered_pending_confirmation


Buyer can:
- Confirm delivery
- Or open dispute

---

### 5A) Buyer confirms
Internal:
- escrow_ledger_entries:
    - entry_type = release
    - amount = vendor_amount
- order.status = completed

Provider:
- Transfer / payout to vendor initiated

---

### 5B) Buyer disputes
Order status:
disputed


Admin reviews evidence.

Admin chooses:
- Refund buyer
- Release to vendor
- Split (optional later)

Ledger records:
- escrow_ledger_entries:
    - refund or release or adjustment

Provider:
- Refund or partial transfer executed

---

## Platform fees

Yanga takes fees via:
- A `fee` ledger entry
- Subtracted from vendor payout

This ensures:
- Gross amount
- Platform revenue
- Vendor earnings
are all explicit.

---

## Why the ledger matters

At any time, Yanga must be able to answer:

| Question | Source |
|--------|------|
| Who paid what? | payment_intents |
| What happened at the provider? | payment_events |
| Who should have the money? | escrow_ledger_entries |
| Why? | admin_actions + disputes |

This protects:
- Against provider bugs
- Against fraud
- Against legal disputes
- Against internal mistakes

---

## Provider independence

Yanga must be able to support:
- Stripe
- Paystack
- Flutterwave
- Bank transfers
- Future wallets

Therefore:
- Providers only write events
- Yanga decides truth

---

## Refunds

Refunds are:
- Business decisions
- Recorded in ledger
- Then executed via provider

Never let providers issue refunds without matching ledger entries.

---

## Compliance & auditability (MVP)

At minimum:
- Every money movement has:
    - amount
    - order
    - actor
    - reason
    - timestamp

This is what keeps Yanga safe when:
- A vendor complains
- A buyer threatens chargeback
- A regulator asks questions

---

## Open decisions (TBD)
- First payment provider(s)
- Whether funds are authorized or captured immediately
- Whether vendor payouts are instant or batched
- Whether auto-release is enabled after X days
- Chargeback handling policy


