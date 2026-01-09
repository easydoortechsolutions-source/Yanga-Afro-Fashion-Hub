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
