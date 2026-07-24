---
name: haiti-payments
description: >-
  Integrate Haitian mobile-money payments — MonCash (Digicel's native API) and
  Pay'm (aggregator for MonCash, NatCash, and Kashpaw) — covering cash-in
  (collecting payments) and cash-out (payouts) in web, mobile, or backend apps.
  Use this whenever the user is building or debugging a payment, checkout,
  top-up, recharge, wallet, subscription, voucher, or payout flow that touches
  MonCash, NatCash, Kashpaw, Pay'm, Digicel money, or accepting/sending money in
  Haiti or in gourdes (HTG) — even if they don't name the exact provider. Covers
  OAuth, create-payment + redirect, payment verification/polling, HMAC-signed
  withdrawals, prepaid balances, webhooks, idempotency, and the money-safety
  patterns plus real-world API quirks the official docs get wrong.
license: MIT
---

# Haitian Payments (MonCash + Pay'm)

Integrate mobile money in Haiti. Two providers are covered because they solve
different problems:

- **MonCash native** — Digicel's own API. MonCash only. Official, direct, lower
  fees, but you need a Digicel merchant/business account and API credentials.
  → full reference: [references/moncash-native.md](references/moncash-native.md)
- **Pay'm** — an aggregator that exposes **MonCash + NatCash + Kashpaw** through
  one API, with a signed payout flow. Easier onboarding, one integration for
  three rails, but ~3% fee, no sandbox, and no webhooks (you poll).
  → full reference: [references/paym.md](references/paym.md)

Read the relevant reference file **before** writing integration code — both APIs
have quirks (wrong auth headers, decimal rules, misspelled fields) that will
cost you hours if you guess. The details live in the reference files to keep this
overview short; this page gives you the shared mental model and the safety rules
that apply to **both**.

---

## Choosing a provider

| Question | Lean MonCash native | Lean Pay'm |
|---|---|---|
| Which rails do you need? | MonCash only | MonCash **and** NatCash (and/or Kashpaw) |
| Do you have a Digicel API contract? | Yes | No / not yet |
| Priority | Lowest fees, official | Fastest to launch, one API for all rails |
| Need a testing sandbox? | Yes (MonCash has one) | No sandbox exists — test tiny amounts live |

You can also run **both**: MonCash native for MonCash, Pay'm for NatCash. Keep a
provider-agnostic `PaymentAdapter` interface (see "Architecture" below) so the
rest of your app doesn't care which rail settled a payment.

---

## The mental model (both providers)

Every mobile-money payment has the same shape. Internalize this and the two APIs
stop feeling different:

**Cash-in (you collect money):**
1. **Create** a payment for `amount` + your unique `reference`/`orderId`.
2. **Redirect** the customer to the provider's URL; they approve on MonCash/NatCash.
3. **Confirm** it was actually paid — never trust the redirect back alone.
   - MonCash native: call `RetrieveOrderPayment` (the source of truth).
   - Pay'm: **poll** `paiement-verify` (no webhook exists).
4. **Deliver** the goods exactly once (atomic claim on your order).

**Cash-out (you send money):**
1. Ensure a **prefunded balance** exists (both providers debit a prepaid pool).
2. Execute a payout to a phone number with a unique `reference`.
3. On an ambiguous failure (timeout), **check status by reference before
   refunding** so a payout that went through isn't paid twice.

---

## Money-safety rules (non-negotiable — this is real money)

These are the rules that separate a toy from a payment system. They apply to
both providers and are the most common source of real financial loss.

- **Confirm server-side, never trust the client redirect.** The user landing
  back on your `return_url` does **not** mean they paid. Always verify against
  the provider (`RetrieveOrderPayment` / `paiement-verify`) before delivering.
- **One `reference` per order, never reused.** This is your idempotency key. It
  lets you retry safely and lets the provider reject duplicates.
- **Deliver exactly once with an atomic claim.** Before handing over goods:
  `UPDATE orders SET status='PAID' WHERE id=? AND status='PENDING'` — if it
  updates 0 rows, someone already delivered it, so stop. This is what makes
  client-polling + a server cron safe to run at the same time.
- **Check the amount actually paid ≥ the price** before delivering (guards
  against underpayment).
- **Reconcile with a server-side cron**, not just client polling. Customers
  close the tab, lose signal, or never return. A cron that re-verifies pending
  orders every ~2 min is what makes collection reliable.
- **Keep secrets and signatures server-side only.** OAuth secrets, `client_secret`,
  and HMAC computation must never touch client-side JS.
- **For payouts, reconcile before refunding.** A network timeout is not proof of
  failure. Query the payout status by `reference` first; only refund/cancel if
  the provider confirms it did not go through.
- **Log the provider's full error** (`error_code` + `message`). A generic
  "payment failed" that hides the real reason makes production undebuggable.

---

## Suggested architecture

Put a thin, provider-agnostic interface in front of both providers so your app
logic (checkout, wallet credit, voucher issuance) never branches on the rail:

```ts
interface PaymentAdapter {
  createPayment(input: { reference: string; amountHtg: number; method: string }):
    Promise<{ redirectUrl: string; providerRef: string }>;
  verifyPayment(reference: string):
    Promise<{ paid: boolean; amountHtg?: number }>;
  // Optional, only if you pay people out:
  payout?(input: { reference: string; amountHtg: number; recipient: string; method: string }):
    Promise<{ status: "success" | "failed"; providerRef?: string }>;
}
```

Implement `MoncashAdapter` and `PaymAdapter` against this. Ship a **stub adapter**
for local dev/tests too — but make the stub **fail closed in production** (a stub
payout that "succeeds" in prod would mark money sent without moving it). Gate
which adapter is live behind env flags / kill switches so you can turn a rail off
instantly without a deploy.

Typical flow for something like a store checkout or a Wi-Fi voucher:

```
Customer → your backend: createPayment(reference, amount, method)
your backend → provider: create → store order PENDING → return redirectUrl
Customer → provider URL → approves on MonCash/NatCash
your backend: verifyPayment (client poll + cron) → paid?
  → atomic claim PENDING→PAID → deliver goods (credit wallet / reveal voucher)
```

---

## Quick gotchas (read the reference for the rest)

**MonCash native**
- OAuth is HTTP **Basic** auth to get a Bearer token; the token expires — cache
  and refresh it.
- The **redirect-back is not confirmation** — `RetrieveOrderPayment` is the
  source of truth (`payment.message === "successful"`).
- Payouts use the **prefunded** `Transfert` endpoint and need a funded balance.

**Pay'm**
- Auth header is **`x-access-token`, not `Authorization: Bearer`** (docs are wrong → 401).
- **NatCash rejects decimal amounts** — always send whole gourdes (`Math.ceil`), min 20 HTG.
- **No webhook** — poll `paiement-verify`.
- The cash-in reference field is misspelled **`refference_id`** (double `f`).
- Payout is a **3-step HMAC** flow; the prepaid payout account must be
  **activated in the dashboard and funded** or you get `NO_PREPAID_ACCOUNT` / `empty`.

---

## Contributing

This skill is open source (MIT). If a provider changes its API or you discover a
new quirk, update the relevant reference file and note the date. Real, tested
behavior beats the official docs — when they disagree, trust what the live API
does and document it here.
