# SaaS architecture — Moving SmartPayroll to multi-tenant

**English** · [Français](./ARCHITECTURE-SAAS.fr.md)

> A condensed, showcase version of the internal architecture document. It covers the key decisions behind turning single-site on-premise software into a hosted multi-tenant SaaS platform.

## 1. Starting point and market constraint

SmartPayroll was distributed **on-premise**: one binary per customer, a local MySQL database, and an offline encrypted license controlling the active organization. Moving this model to a hosted SaaS raised one central question, specific to the target market: **how do you bill a recurring subscription in the DRC?**

The default Western answer (credit card + automatic debit) doesn't apply. Mobile money there works as a **push payment** (customers confirm each transaction themselves from their phone), with no reliable direct debit mandate. This constraint shaped the whole billing model, detailed in §4.

## 2. Architecture decisions

| Decision | Why | Consequence |
|---|---|---|
| **Keep a modular monolith**, no microservice extraction | One infrastructure to run, consistent with a small team | The new SaaS modules (subscriptions, wallet, entitlements, platform admin) live in the same repository, with the same conventions as the existing code |
| **Centralized hosting (Railway)**, no more binary distribution | Reverse engineering of a binary shipped to customers is a risk obfuscation alone can't control | GPS clock-in and every other feature now depend on real internet access on the customer side, not just a local network |
| **Prepaid wallet billing**, not recurring debit | Matches how local mobile money push payments work | Subscription renewal debits an internal balance that's already been topped up, never an external account or card |
| **Object storage (S3-compatible)** instead of local disk | A PaaS host doesn't guarantee local disk persistence across redeploys, and clock-in proofs have audit value | Files (HR documents, clock-in proofs) are referenced by object key, never by disk path |
| **Strict Platform Admin / Tenant separation**, at table, JWT and route level | Prevent an access-control bug from turning an organization admin into an admin of the whole platform | Every `/api/platform/*` route requires a separate JWT (different audience and secret), never the same token as a customer |
| **CDF as the wallet's reference currency**, USD only as a commercial indication | Mobile money in the DRC transacts in CDF; avoid the complexity of a multi-currency double ledger from day one | The exchange rate is only recorded for internal reporting, never used in debit logic |
| **Full removal of the local license system** once no active on-premise deployment remained | No more code shipped to customers means no offline artifact to protect | Access checks now happen in the database, in real time, on every request, with no legacy fallback |

## 3. Target system overview

```text
                         Internet (now required)
                                    │
                    ┌───────────────┴────────────────┐
                    │          A single project        │
                    │                                  │
   ┌────────────────┼──────────────────────────────────┼───────────────┐
   │                │                                  │               │
   ▼                ▼                                  ▼               ▼
 web             worker                              MySQL           Redis
 (Express API   (BullMQ: payroll,                   (managed)       (managed)
  + static      subscription                                            │
  React         renewal,                                                ├── Cache
  build)        billing webhooks,                                       ├── Rate limiting
                reconciliation)                                         └── Job queue
   │
   ├── /api/*                          → tenant routes (payroll, attendance, HR…)
   ├── /api/wallet/*                   → mobile money top-up, balance, history
   ├── /api/webhooks/billing/:provider → callbacks from the mobile money aggregator
   └── /api/platform/*                 → reserved for the platform operator
                    │
                    ▼
        External object storage (S3-compatible)
        employee documents, clock-in proofs
```

One deployment, many organizations. Isolation is **logical** (modules, tables, routes, `organizationId` propagated from the JWT), not infrastructural. That's a deliberate choice of operational simplicity for the target team size and traffic, with a clear path forward if needs change (see §8).

## 4. Billing: prepaid wallet and subscription lifecycle

Each organization has a **wallet** (internal balance in CDF) and a **subscription** tied to a plan. The wallet has a **single entry point** for any balance change: no other part of the code may write to it directly, which guarantees that every movement is recorded and consistent.

**Subscription state machine:**

```text
TRIAL ──────────► ACTIVE       (first sufficient top-up, successful debit)
TRIAL ──────────► CANCELED     (trial ends without payment, or cancellation)

ACTIVE ─────────► PAST_DUE     (insufficient balance at renewal)
PAST_DUE ───────► ACTIVE       (top-up received before the past-due window ends)
PAST_DUE ───────► GRACE_PERIOD (after an unresolved past-due delay)
GRACE_PERIOD ───► SUSPENDED    (after an unresolved cumulative grace delay)
SUSPENDED ──────► ACTIVE       (top-up + successful debit)

(any state) ────► CANCELED     (voluntary cancellation)
```

Read-only access is kept even in `SUSPENDED`: suspending an organization never blocks it from viewing its own data, only from creating or modifying it.

Debit and reactivation logic lives in a **single function** that locks the subscription, then the wallet, always in that order. This guarantees the absence of deadlocks *structurally*, instead of relying on discipline at every call site. A regular top-up (by far the most common case) never locks the subscription.

Integration with the mobile money aggregator goes through a **common interface** (`PaymentProvider`), decoupling business logic from the chosen provider. Every incoming webhook is signature-checked, then deduplicated by a unique event ID before any business processing, and handled asynchronously, never inline in the HTTP handler. A periodic reconciliation job completes the setup: webhooks sent from a mobile money aggregator to a server hosted outside the DRC aren't 100% reliable, so the system never depends on the webhook alone.

## 5. Platform Admin / Tenant separation

The platform operator (super-admin) has an account, a JWT and a signing secret **entirely separate** from those of customer organizations, not just a different role on the same account. Platform authentication requires a second factor (TOTP) from account creation, with no exceptions. Every write action through platform routes records an audit entry **before** responding to the client, never afterwards in a background task.

## 6. Offline attendance tracking

Clock-in happens in the field, where connectivity is intermittent. The principle: record the action **immediately on the client** (timestamp captured at the real moment), then sync later with no loss and no duplicates.

- **Idempotency**: every pending clock-in carries a client-generated ID, deduplicated on the server.
- **Tiered timestamps**: drift between the client clock and the server clock is handled in three levels (silently accepted / flagged for manual review / excluded from payroll until approved) rather than a binary threshold, so an employee who was actually present never loses a day's pay because of a simple clock bug.
- **Client-side geographic check is advisory only**: never blocking; the server remains the final authority at sync time.
- **Transient error vs. business rejection** during sync: a network error triggers a retry with increasing delay; a business rejection (out of zone, already clocked in) immediately becomes final, with no pointless retry.

## 7. Compliance and data retention

The platform processes employee personal data under the DRC's applicable data protection framework. The principle: personal data is kept only as long as its purpose requires, then anonymized or deleted, never kept indefinitely in identifiable form. A recurring job audits canceled organizations and applies the export → anonymization → final purge cycle according to configurable delays, with full traceability of each step.

## 8. Deliberately out of scope for now

Some improvements were identified but intentionally postponed, for lack of a proven need at the current scale: materialized daily aggregates for the platform dashboard, distributed tracing (aggregate metrics are enough for now), real-time presence per organization. A SaaS architecture isn't judged only by what it builds, but also by what it consciously chooses not to build until the need is proven.

---

*This document is a condensed, showcase version of the project's internal architecture documentation.*

---

← [Back to the README](../README.md)
