# MonCash Native API (Digicel) — Integration Reference

Digicel's own MonCash Business/Gateway API. MonCash only. You need a Digicel
merchant account and API credentials (`clientId` + `clientSecret`). MonCash has a
real **sandbox**, so develop there first.

## Table of contents
1. Environments & credentials
2. OAuth (get a Bearer token)
3. Cash-in: CreatePayment → redirect → confirm
4. Verifying a payment (source of truth)
5. Webhook / redirect-back notes
6. Cash-out: prefunded Transfert
7. Prefunded status & balance
8. Error handling
9. Reference Node.js adapter

---

## 1. Environments & credentials

| Item | Value |
|---|---|
| Sandbox base | `https://sandbox.moncashbutton.digicelgroup.com` |
| Production base | `https://moncashbutton.digicelgroup.com` |
| Credentials | `MONCASH_CLIENT_ID`, `MONCASH_CLIENT_SECRET` (from the MonCash Business portal) |
| Currency | HTG, **decimals allowed** (e.g. `500.00`) |

Amounts in the API are **whole/decimal HTG** (not centimes). Store money as
integer centimes internally and convert (`amount = cents / 100`) at the API edge.

---

## 2. OAuth — `POST /Api/oauth/token`

HTTP **Basic** auth (base64 of `clientId:clientSecret`), form-encoded body:

```
POST /Api/oauth/token
Authorization: Basic base64(clientId:clientSecret)
Content-Type: application/x-www-form-urlencoded
Accept: application/json

scope=read,write&grant_type=client_credentials
```

Response: `{ "access_token": "...", "expires_in": 59 }`. Tokens are short-lived —
**cache the token and refresh ~30 s before expiry**. Use it as
`Authorization: Bearer <access_token>` on all other calls.

---

## 3. Cash-in — `POST /Api/v1/CreatePayment`

```
POST /Api/v1/CreatePayment
Authorization: Bearer <token>
Content-Type: application/json

{ "amount": 500.00, "orderId": "ORDER-2026-000123" }
```

- `amount` — HTG (decimal).
- `orderId` — **your** unique id (idempotency key). Reuse your order id.

Response:
```json
{ "payment_token": { "token": "abc123...", "expired": "..." }, "mode": "..." }
```

**Redirect the customer** to:
```
{BASE}/Moncash-middleware/Payment/Redirect?token={payment_token.token}
```
The token is valid ~15 min. The customer approves in the MonCash app / gateway.

---

## 4. Verify a payment — `POST /Api/v1/RetrieveOrderPayment` (source of truth)

**This is the only reliable confirmation.** After the customer returns (or from a
reconciliation cron), call:

```
POST /Api/v1/RetrieveOrderPayment
Authorization: Bearer <token>
Content-Type: application/json

{ "orderId": "ORDER-2026-000123" }
```

Response:
```json
{ "payment": {
    "transaction_id": "...", "cost": 500, "message": "successful",
    "payer": "509XXXXXXXX", "reference": "ORDER-2026-000123" } }
```

- Paid ⟺ `payment.message === "successful"`.
- `cost` = amount actually paid (HTG) → check `cost >= expectedHtg`.
- `payer` = customer's MonCash number; `transaction_id` = MonCash's id (store it).

There is also `RetrieveTransactionPayment` (lookup by `transactionId`), but for
order-driven flows `RetrieveOrderPayment` is what you want.

---

## 5. Webhook / redirect-back notes

- MonCash redirects the customer back to your configured return URL after
  payment. **Treat the redirect as "go verify now", not as proof of payment.**
  Always call `RetrieveOrderPayment` before delivering.
- If you expose a server notification endpoint, **verify its authenticity** and
  still re-confirm via `RetrieveOrderPayment`. A common hardening pattern is to
  require an HMAC-SHA256 signature over the raw request body using a shared
  secret and compare with `timingSafeEqual` (constant-time). Never trust an
  unauthenticated "successful" callback — anyone could forge it to self-credit.
- Because a customer can always abandon the redirect, a **reconciliation cron**
  that re-verifies pending orders is mandatory for reliability.

---

## 6. Cash-out — `POST /Api/v1/Transfert` (prefunded)

Sends HTG to a customer's MonCash account. Debits your **prefunded** MonCash
balance, so that balance must be funded first.

```
POST /Api/v1/Transfert
Authorization: Bearer <token>
Content-Type: application/json

{ "amount": 500.00, "receiver": "509XXXXXXXX", "desc": "Payout ORDER-123",
  "reference": "PAYOUT-0001" }
```

- `reference` — unique idempotency key for the payout.
- Success ⟺ `transfer.message === "successful"` and `transfer.transaction_id` present:
  ```json
  { "transfer": { "transaction_id": "...", "message": "successful" }, "status": 202 }
  ```
- **Ambiguous failure (timeout / lost response):** do NOT immediately refund.
  Call `PrefundedTransactionStatus` by `reference` (§7) first; only refund if it
  confirms the payout did not succeed. Otherwise you risk paying twice.

---

## 7. Prefunded status & balance

**Status by reference** — `POST /Api/v1/PrefundedTransactionStatus`
```json
{ "reference": "PAYOUT-0001" }
```
→ `{ "transStatus": "successful" }`. A **404** means MonCash has no record of that
reference (i.e. the payout never registered → safe to refund).

**Balance** — `GET /Api/v1/PrefundedBalance` → `{ "balance": { "balance": 1234.5 } }`
(HTG). Check it before large payouts.

---

## 8. Error handling

- CreatePayment `4xx` with `{"message":"The amount is incorrect"}` → the amount is
  too high/invalid; surface a clean "try a smaller amount" to the user rather
  than a 500.
- Distinguish `4xx` (client error → clean message to user) from `5xx` (provider
  outage → retry / keep as pending, don't refund yet).
- Always log the raw provider body (truncated) so failures are diagnosable.

---

## 9. Reference Node.js adapter (abridged, battle-tested)

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

const BASE = process.env.MONCASH_BASE_URL!; // sandbox or prod
let cached: { token: string; exp: number } | null = null;

async function getToken(): Promise<string> {
  if (cached && cached.exp > Date.now() + 30_000) return cached.token;
  const basic = Buffer.from(
    `${process.env.MONCASH_CLIENT_ID}:${process.env.MONCASH_CLIENT_SECRET}`,
  ).toString("base64");
  const res = await fetch(`${BASE}/Api/oauth/token`, {
    method: "POST",
    headers: {
      Authorization: `Basic ${basic}`,
      "Content-Type": "application/x-www-form-urlencoded",
      Accept: "application/json",
    },
    body: "scope=read,write&grant_type=client_credentials",
  });
  if (!res.ok) throw new Error(`MonCash OAuth ${res.status}`);
  const j = (await res.json()) as { access_token: string; expires_in: number };
  cached = { token: j.access_token, exp: Date.now() + j.expires_in * 1000 };
  return j.access_token;
}

async function api(path: string, body: unknown) {
  const token = await getToken();
  const res = await fetch(`${BASE}${path}`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
      Accept: "application/json",
    },
    body: JSON.stringify(body),
  });
  return { res, json: await res.json().catch(() => ({})) };
}

// Cash-in
export async function createPayment(orderId: string, amountHtg: number) {
  const { res, json } = await api("/Api/v1/CreatePayment", { amount: amountHtg, orderId });
  if (!res.ok) throw new Error(`CreatePayment ${res.status}`);
  const token = (json as any).payment_token.token;
  return { redirectUrl: `${BASE}/Moncash-middleware/Payment/Redirect?token=${token}` };
}

export async function retrieveOrderPayment(orderId: string) {
  const { res, json } = await api("/Api/v1/RetrieveOrderPayment", { orderId });
  if (!res.ok) throw new Error(`RetrieveOrderPayment ${res.status}`);
  const p = (json as any).payment;
  return { paid: p?.message === "successful", amountHtg: p?.cost, transactionId: p?.transaction_id };
}

// Cash-out (prefunded)
export async function transfert(reference: string, amountHtg: number, receiver: string, desc: string) {
  const { res, json } = await api("/Api/v1/Transfert", { amount: amountHtg, receiver, desc, reference });
  if (!res.ok) throw new Error(`Transfert ${res.status}`);
  const t = (json as any).transfer;
  if (t?.message !== "successful") throw new Error("Transfert unconfirmed");
  return { transactionId: t.transaction_id };
}

// Verify a notification's HMAC signature (constant-time)
export function verifyWebhook(rawBody: string, signature: string, secret: string) {
  const expected = createHmac("sha256", secret).update(rawBody).digest("hex");
  const a = Buffer.from(expected), b = Buffer.from(signature);
  return a.length === b.length && timingSafeEqual(a, b);
}
```

> Notes distilled from a production integration: cache the OAuth token; treat
> `RetrieveOrderPayment` as the source of truth; reconcile payouts by reference
> before refunding; and make any dev/stub adapter **fail closed** in production so
> it can never mark money moved without moving it.
