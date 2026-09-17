# Pay'm API: Integration Reference (MonCash · NatCash · Kashpaw)

Pay'm is an aggregator: one API for **MonCash + NatCash + Kashpaw**, plus a
signed payout flow. Everything here was verified **live**: there is no sandbox.
Where Pay'm's official docs are wrong, this file gives the value that works.

## Table of contents
1. Base URL & credentials
2. The 6 things that will bite you
3. Payment methods & fees
4. Cash-in: create → redirect → verify (poll)
5. Cash-out: 3-step HMAC payout
6. Error codes
7. Reference Node.js helpers

---

## 1. Base URL & credentials

| Item | Value |
|---|---|
| Base URL | `https://plopplop.solutionip.app` |
| Sandbox | ❌ none: test with 20 HTG and a real phone |
| `PAYM_CLIENT_ID` | `pp_...` |
| `PAYM_CLIENT_SECRET` | 64-char secret, **server-side only** (withdrawal HMAC) |

---

## 2. The 6 things that will bite you

1. **No sandbox.** Every call is real money.
2. **Auth header is `x-access-token`, NOT `Authorization: Bearer`.** Bearer → 401.
   The official docs are wrong here.
3. **NatCash rejects decimal amounts.** `montant: 150.15` → `503
   ERR_PARAMETERS_INVALID`. Always send **whole gourdes** (`Math.ceil`). MonCash
   accepts decimals, but round for both.
4. **Webhooks arrived in 2026**, and polling is still the fallback. Use the
   webhook as a trigger, then call `/api/paiement-verify` to learn a payment succeeded,
   from the client AND a server cron.
5. **Cash-in reference field is `refference_id`** (double `f`). It's `reference`
   on the cash-out endpoints. Match exactly.
6. **Credit exactly once**: unique `reference` + atomic pending→paid claim so
   client polling and the cron can't double-deliver.

---

## 3. Payment methods & fees

| Method | Cash-in | Cash-out |
|---|---|---|
| `moncash` | ✅ | ✅ |
| `natcash` | ✅ | ✅ |
| `kashpaw` | ✅ | ❌ |
| `all` | ✅ (customer picks on Pay'm's page) | — |

- Cash-in fee ≈ **3 %** (customer pays 1000 HTG → merchant balance gets 970).
- Minimum cash-in: **20 HTG**.
- **Prepaid balance model:** collections credit a merchant balance; payouts debit
  a prepaid pool. The payout pool must be separately **activated + funded** (§5.4).

---

## 4. Cash-in (collecte)

No auth token needed; send `client_id` in the body.

### 4.1 Create: `POST /api/paiement-marchand`
```json
{ "client_id": "pp_xxx", "refference_id": "ORDER-000123",
  "montant": 200, "payment_method": "moncash" }
```
- `refference_id`: your unique id (double `f`).
- `montant`: HTG, **whole gourdes**, `>= 20`.
- `payment_method`: `moncash` | `natcash` | `kashpaw` | `all`.

→ `{ "status": true, "url": "https://.../pay/...", "transaction_id": "PM_..." }`
Redirect the customer to `url`.

> ⚠️ NatCash shows an **ad page after payment**; this is Pay'm/NatCash-side. There
> is **no `return_url`** parameter: extra body fields are silently ignored. The
> payment still completes; you detect it via polling.

### 4.2 Verify (poll): `POST /api/paiement-verify`
```json
{ "client_id": "pp_xxx", "refference_id": "ORDER-000123" }
```
→ observed live, 2026-09: `{ "status", "message", "montant", "trans_status",
  "transaction_id", "refference_id", "date", "heure", "method", "id_client",
  "toPhone" }`

**Their documentation calls the id `id_transaction`; the live response calls it
`transaction_id`.** Read the documented name and you get `undefined`, silently.
There is **no fee field** on a collection, so the provider's cut of a cash-in is
not knowable per transaction. Payouts do report theirs.
- `trans_status`: `"no"` = unpaid, `"ok"` = paid.
- `montant` is a **string**: coerce with `Number()`. Check `>= expectedHtg`.
- Poll ~every 2.5–3 s from the client after redirect, **and** run a server cron
  (~every 2 min) that re-verifies still-pending orders.
- Deliver via an **atomic claim** (pending→paid) so client + cron don't double-deliver.

---

## 5. Cash-out (payout): signed 3-step flow

Only implement if you need to send HTG out. `moncash` / `natcash` only.

### 5.1 Auth: `POST /api/auth/marchand`
```json
{ "client_id": "pp_xxx", "client_secret": "64charsecret" }
```
→ `{ "success": true, "token": "<marchand_token>", "expires_in": 300 }` (cache ~5 min).

### 5.2 Signed token: `POST /api/auth/marchand/withdrawal-token`
Header **`x-access-token: <marchand_token>`**.
```json
{ "amount": 500, "method": "natcash", "recipient": "50912345678",
  "reference": "PAYOUT-0001", "timestamp": 1715691234,
  "withdrawal_signature": "<hmac hex>" }
```
Signature (server-side only):
```js
const payload = [amount, method, recipient, reference, timestamp].join("|");
const signature = crypto.createHmac("sha256", CLIENT_SECRET).update(payload).digest("hex");
```
- Order exact: `amount|method|recipient|reference|timestamp`.
- `timestamp` = Unix seconds, rejected if off by > ±5 min.
- `recipient` = `509XXXXXXXX` (no `+`); `amount` = whole gourdes.

→ `{ "success": true, "withdrawal_token": "<token>" }`

### 5.3 Execute: `POST /api/withdraw/marchand`
Header **`x-access-token: <withdrawal_token>`** (the step-2 token). Body must be
identical to the signed values (minus timestamp/signature):
```json
{ "amount": 500, "method": "natcash", "recipient": "50912345678", "reference": "PAYOUT-0001" }
```
→ `{ "success": true, "data": { "transaction_id": "...", "fee": 12.5, "total": 512.5,
     "status": "success", "balance_before": 5000, "balance_after": 4487.5 } }`

Status (reconcile before refunding): `POST /api/withdraw/marchand/verify`
`{ "reference": "PAYOUT-0001" }` with header `x-access-token: <marchand_token>`.

### 5.4 Payout prerequisites (learned the hard way)
The payout side uses a **separate "compte prépayé"** that must be:
1. **Activated** in the Pay'm dashboard: else every payout is `400 NO_PREPAID_ACCOUNT`.
2. **Funded**: an activated-but-empty account returns a vague
   `400 {"error_code":"empty","message":"Réessayer dans quelques instants"}`.
   Cash-in collections do not necessarily auto-fund it; confirm with Pay'm.

---

## 6. Error codes

**Cash-in**
| HTTP / code | Meaning | Fix |
|---|---|---|
| `503 ERR_PARAMETERS_INVALID` | decimal `montant` on NatCash | send whole gourdes |
| `status: false` | rejected | read `message` |

**Cash-out (step 3)**
| Code | Meaning | Fix |
|---|---|---|
| `NO_PREPAID_ACCOUNT` | payout account not activated | activate in dashboard |
| `INSUFFICIENT_BALANCE` / `empty` | prepaid balance low / unfunded | fund the account |
| `METHOD_NOT_CONFIGURED` | method not enabled for payout | enable in dashboard |
| `WITHDRAWAL_COOLDOWN` (429) | < 120 s since last payout from this IP | wait ~120 s |
| `DUPLICATE_REFERENCE` (409) | reference already used | fresh reference |
| `INVALID_SIGNATURE` (403) | HMAC mismatch | recompute exactly (§5.2) |
| `TIMESTAMP_EXPIRED` | clock off > ±5 min | current Unix time |
| `INVALID_TOKEN_TYPE` / `TOKEN_ALREADY_USED` (403) | wrong/used token | regenerate steps 1+2 |
| `PARAMETER_MISMATCH` (403) | step-3 body ≠ signed values | send identical values |
| `API_TRANSFER_FAILED` | MonCash/NatCash rejected the transfer | check recipient number |

---

## 7. Reference Node.js helpers

```js
const BASE = process.env.PAYM_BASE_URL || "https://plopplop.solutionip.app";
const CLIENT_ID = process.env.PAYM_CLIENT_ID;
const CLIENT_SECRET = process.env.PAYM_CLIENT_SECRET;
const crypto = require("node:crypto");

async function post(path, body, token) {
  const res = await fetch(`${BASE}${path}`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      ...(token ? { "x-access-token": token } : {}), // NOT Authorization: Bearer
    },
    body: JSON.stringify(body),
  });
  const text = await res.text();
  let json; try { json = text ? JSON.parse(text) : {}; } catch { json = { raw: text }; }
  if (!res.ok) throw new Error(`Pay'm ${path} ${res.status}: ${text.slice(0, 200)}`);
  return json;
}

// --- cash-in ---
async function createPayment({ reference, montantHtg, method }) {
  const montant = Math.ceil(montantHtg);          // whole gourdes (NatCash!)
  if (montant < 20) throw new Error("Minimum 20 HTG");
  const j = await post("/api/paiement-marchand", {
    client_id: CLIENT_ID, refference_id: reference, montant, payment_method: method,
  });
  if (!j.url) throw new Error("Pay'm: no redirect url");
  return { redirectUrl: j.url, transactionId: j.transaction_id };
}

async function verifyPayment(reference) {
  const j = await post("/api/paiement-verify", { client_id: CLIENT_ID, refference_id: reference });
  return { paid: j.trans_status === "ok", amountHtg: Number(j.montant) };
}

// --- cash-out (3-step) ---
async function authMarchand() {
  const j = await post("/api/auth/marchand", { client_id: CLIENT_ID, client_secret: CLIENT_SECRET });
  if (!j.token) throw new Error("Pay'm auth: no token");
  return j.token;
}

async function payout({ reference, amountHtg, recipient, method }) {
  const amount = Math.floor(amountHtg);           // whole gourdes, never overpay
  const marchandToken = await authMarchand();
  const timestamp = Math.floor(Date.now() / 1000);
  const signature = crypto
    .createHmac("sha256", CLIENT_SECRET)
    .update([amount, method, recipient, reference, timestamp].join("|"))
    .digest("hex");
  const tok = await post("/api/auth/marchand/withdrawal-token",
    { amount, method, recipient, reference, timestamp, withdrawal_signature: signature },
    marchandToken);
  if (!tok.withdrawal_token) throw new Error("Pay'm: no withdrawal_token");
  const exec = await post("/api/withdraw/marchand",
    { amount, method, recipient, reference }, tok.withdrawal_token);
  return { status: exec.data?.status ?? (exec.success ? "success" : "failed"),
           transactionId: exec.data?.transaction_id, raw: exec };
}

module.exports = { createPayment, verifyPayment, payout };
```

---

*Compiled from live testing of the Pay'm production API (July 2026). If a response
contradicts this file, trust the live response and update this doc.*
