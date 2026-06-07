# Linq B2B Offramp API

Convert USDSUI stablecoins to Nigerian Naira (NGN) with direct bank account payouts — programmatically, via API.

---

## Overview

The Linq offramp API lets your application:

1. Generate a unique deposit wallet address for a user
2. Accept a USDSUI transfer from that user
3. Automatically pay out NGN to any Nigerian bank account
4. Receive real-time status updates via signed webhooks

**Coin supported:** USDSUI on Sui  
**Contract:** `0x44f838219cf67b058f3b37907b655f226153c18e33dfcd0da559a844fea9b1c1::usdsui::USDSUI`  
**Decimals:** 6 (same as USDC)  
**Payout currencies:** NGN

---

## Base URL

```
https://confidential-brianna-uselinq-52e2b233.koyeb.app
```

---

## Getting access

Access to the API is invite-only. To get started:

1. Contact the Linq team to request an invite code
2. You'll receive a 7-character code (e.g. `A7X2K9M`) via the team
3. Use that code in `POST /b2b/signup` to create your account — see [Onboarding via curl](#onboarding-via-curl) below

---

## Authentication

All offramp endpoints (except `GET /b2b/rate`) require an API key.

Pass your API key in the request header:

```
X-API-Key: your_api_key_here
```

You receive your API key when you register via `POST /b2b/signup`. Keep it secret — treat it like a password.

---

## Webhooks

When an order changes state, Linq sends a `POST` request to the `webhookUrl` you registered on your account.

### Verifying webhook authenticity

Every webhook request includes an `X-Linq-Signature` header. This is an HMAC-SHA256 signature of the raw request body, signed with your **webhook secret** (separate from your API key).

**Verify in Node.js:**
```js
const crypto = require('crypto');

function verifyWebhook(rawBody, signatureHeader, webhookSecret) {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', webhookSecret)
    .update(rawBody)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(signatureHeader),
    Buffer.from(expected)
  );
}
```

**Verify in Python:**
```python
import hmac, hashlib

def verify_webhook(raw_body: bytes, signature_header: str, webhook_secret: str) -> bool:
    expected = 'sha256=' + hmac.new(
        webhook_secret.encode(),
        raw_body,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(signature_header, expected)
```

> Always verify the signature before processing a webhook. Reject any request where verification fails.

### Complete webhook endpoint examples

**Node.js (Express):**
```js
const express = require('express');
const crypto = require('crypto');

const app = express();
const WEBHOOK_SECRET = process.env.LINQ_WEBHOOK_SECRET;

// IMPORTANT: use express.raw() on this route, not express.json()
// You must receive the raw bytes to verify the signature correctly.
app.post('/webhooks/linq', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-linq-signature'];

  // 1. Verify signature
  const expected = 'sha256=' + crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(req.body)
    .digest('hex');

  if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    return res.status(401).send('Invalid signature');
  }

  // 2. Parse body
  const event = JSON.parse(req.body);

  // 3. Handle idempotently — check orderId before processing
  switch (event.event) {
    case 'order.processing':
      console.log(`Order ${event.orderId} — deposit confirmed, bank payout in progress`);
      // update your DB: status = 'processing'
      break;

    case 'order.completed':
      console.log(`Order ${event.orderId} — ₦${event.amountNGN} sent to bank account`);
      // update your DB: status = 'completed'
      break;

    case 'order.failed':
      console.log(`Order ${event.orderId} — failed: ${event.status}`);
      // update your DB: status = 'failed', notify user
      break;
  }

  // 4. Acknowledge — must return 2xx within 10 seconds
  res.status(200).send('ok');
});

app.listen(3000);
```

**Python (FastAPI):**
```python
import hmac, hashlib, os
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()
WEBHOOK_SECRET = os.environ["LINQ_WEBHOOK_SECRET"]

@app.post("/webhooks/linq")
async def linq_webhook(request: Request):
    raw_body = await request.body()
    signature = request.headers.get("x-linq-signature", "")

    # 1. Verify signature
    expected = "sha256=" + hmac.new(
        WEBHOOK_SECRET.encode(), raw_body, hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(signature, expected):
        raise HTTPException(status_code=401, detail="Invalid signature")

    # 2. Parse body
    event = await request.json()

    # 3. Handle idempotently
    match event["event"]:
        case "order.processing":
            print(f"Order {event['orderId']} — deposit confirmed")
            # update your DB: status = 'processing'

        case "order.completed":
            print(f"Order {event['orderId']} — ₦{event['amountNGN']} paid out")
            # update your DB: status = 'completed'

        case "order.failed":
            print(f"Order {event['orderId']} — failed: {event['status']}")
            # update your DB: status = 'failed', notify user

    # 4. Acknowledge
    return {"ok": True}
```

> **Node.js gotcha:** if you have `express.json()` applied globally, you must register the Linq webhook route *before* that middleware, or use a separate router. `express.json()` consumes the raw body, making signature verification impossible.

### Webhook events

| Event | When it fires |
|---|---|
| `order.processing` | USDSUI deposit detected — bank payout is now in progress |
| `order.completed` | Bank payout succeeded and USDSUI has been settled to treasury |
| `order.failed` | Order timed out (no deposit within 10 minutes) or bank payout failed |

### Webhook payload

```json
{
  "event": "order.completed",
  "orderId": "3f7c1b2a-...",
  "amountStableCoin": 50.0,
  "amountNGN": 82750.00,
  "amountFormatted": "50.000000",
  "currency": "NGN",
  "chain": "usdsui",
  "txHash": "",
  "status": "Settled in treasury",
  "timestamp": "2026-06-07T14:23:01Z"
}
```

### Responding to webhooks

Return any `2xx` status code to acknowledge receipt. If your endpoint returns a non-2xx response or times out (10 second timeout), the delivery is marked as failed. Implement idempotency on your side using `orderId` — we recommend processing each event once and ignoring duplicates.

---

## Onboarding via curl

Everything you need to go live can be done with curl. No dashboard required.

### Step 1 — Create your account

Use the invite code the Linq team sent you:

```bash
curl -X POST https://confidential-brianna-uselinq-52e2b233.koyeb.app/b2b/signup \
  -H "Content-Type: application/json" \
  -d '{
    "registrationCode": "A7X2K9M",
    "businessName": "Acme Corp",
    "email": "you@acme.com",
    "password": "your-strong-password",
    "webhookUrl": "https://acme.com/webhooks/linq"
  }'
```

**Response — save this immediately, these values are shown only once:**

```json
{
  "businessId": 1,
  "apiKey": "biz_live_...",
  "apiSecret": "sec_...",
  "webhookSecret": "whsec_...",
  "message": "Save your API secret and webhook secret — they will not be shown again."
}
```

Store `apiKey` and `webhookSecret` securely. The `apiSecret` is for future authentication extensions — store it but you don't need it for offramp orders today.

---

### Step 2 — Check the current rate

```bash
curl https://confidential-brianna-uselinq-52e2b233.koyeb.app/b2b/rate
```

```json
{ "rate": 1655.00, "currency": "NGN", "coin": "USDSUI" }
```

---

### Step 3 — Verify a bank account before creating an order

```bash
curl -X POST https://confidential-brianna-uselinq-52e2b233.koyeb.app/b2b/verifybank \
  -H "X-API-Key: biz_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "bankCode": "033",
    "accountNumber": "1234567890"
  }'
```

```json
{ "accountName": "John Doe", "bankName": "United Bank for Africa", "accountNumber": "1234567890", "bankCode": "033" }
```

---

### Step 4 — Create an offramp order

```bash
curl -X POST https://confidential-brianna-uselinq-52e2b233.koyeb.app/b2b/offramp \
  -H "X-API-Key: biz_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "amountStableCoin": 50.0,
    "bankAccount": "1234567890",
    "bankCode": "033",
    "bankName": "United Bank for Africa",
    "accountName": "John Doe",
    "currency": "NGN",
    "customerRef": "user_789",
    "idempotencyKey": "order_abc_20260607"
  }'
```

```json
{
  "id": "3f7c1b2a-...",
  "walletAddress": "0xabc123...",
  "coinType": "0x44f838219cf67b058f3b37907b655f226153c18e33dfcd0da559a844fea9b1c1::usdsui::USDSUI",
  "amountStableCoin": 50.0,
  "amountNGN": 82750.00,
  "rate": 1655.00,
  "currency": "NGN",
  "status": "initiated"
}
```

Send exactly `amountStableCoin` USDSUI to `walletAddress` within 10 minutes.

---

### Step 5 — Poll order status

```bash
curl "https://confidential-brianna-uselinq-52e2b233.koyeb.app/b2b/status?id=3f7c1b2a-..." \
  -H "X-API-Key: biz_live_..."
```

---

### Key rotation

Rotate your API key (old key stops working immediately):

```bash
curl -X POST https://confidential-brianna-uselinq-52e2b233.koyeb.app/b2b/regenerate-api-key \
  -H "X-API-Key: biz_live_..."
```

Rotate your webhook signing secret:

```bash
curl -X POST https://confidential-brianna-uselinq-52e2b233.koyeb.app/b2b/regenerate-webhook-secret \
  -H "X-API-Key: biz_live_..."
```

---

## Endpoints

---

### Get current rate

```
GET /b2b/rate
```

Returns the current NGN exchange rate for USDSUI. No authentication required.

Use this before creating an order to show users how much NGN they will receive.

**Response**

```json
{
  "rate": 1655.00,
  "currency": "NGN",
  "coin": "USDSUI"
}
```

**Rate interpretation:** `1 USDSUI = 1655 NGN`

> Rates fluctuate. The rate is locked at order creation time — the value returned by this endpoint is for display purposes only.

**Example**

```bash
curl https://confidential-brianna-uselinq-52e2b233.koyeb.app/b2b/rate
```

---

### Create an offramp order

```
POST /b2b/offramp
```

Creates a new offramp order. Returns a temporary Sui wallet address where the user must send their USDSUI. Once the deposit is detected, Linq automatically initiates the bank payout.

**Headers**

| Header | Value |
|---|---|
| `X-API-Key` | Your API key |
| `Content-Type` | `application/json` |

**Request body**

```json
{
  "amountStableCoin": 50.0,
  "bankAccount": "1234567890",
  "bankCode": "033",
  "bankName": "United Bank for Africa",
  "accountName": "John Doe",
  "currency": "NGN",
  "customerRef": "your-internal-user-id-or-reference",
  "idempotencyKey": "unique-key-per-order"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `amountStableCoin` | number | Yes | Amount of USDSUI the user will send (e.g. `50.0` = 50 USDSUI) |
| `bankAccount` | string | Yes | 10-digit Nigerian bank account number |
| `bankCode` | string | Yes | Nigerian bank code (e.g. `"033"` for UBA) |
| `bankName` | string | Yes | Full bank name |
| `accountName` | string | Yes | Account holder name as it appears at the bank |
| `currency` | string | Yes | Payout currency — currently `"NGN"` |
| `customerRef` | string | No | Your own reference for this order (echoed in webhooks) |
| `idempotencyKey` | string | Yes | A unique string per order. Sending the same key twice returns the original order, not a duplicate |

**Response `201 Created`**

```json
{
  "id": "3f7c1b2a-84e9-4c11-b3d2-0a9f7e123456",
  "walletAddress": "0xabc123...",
  "coinType": "0x44f838219cf67b058f3b37907b655f226153c18e33dfcd0da559a844fea9b1c1::usdsui::USDSUI",
  "amountStableCoin": 50.0,
  "amountNGN": 82750.00,
  "rate": 1655.00,
  "currency": "NGN",
  "status": "initiated"
}
```

| Field | Description |
|---|---|
| `id` | Unique order ID — use this to poll for status or match with webhooks |
| `walletAddress` | Sui wallet address the user must send USDSUI to |
| `coinType` | The exact coin type to send (USDSUI contract address) |
| `amountStableCoin` | Expected USDSUI amount |
| `amountNGN` | NGN amount the recipient will receive |
| `rate` | Locked exchange rate for this order |

**Example**

```bash
curl -X POST https://confidential-brianna-uselinq-52e2b233.koyeb.app/b2b/offramp \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "amountStableCoin": 50.0,
    "bankAccount": "1234567890",
    "bankCode": "033",
    "bankName": "United Bank for Africa",
    "accountName": "John Doe",
    "currency": "NGN",
    "customerRef": "user_789",
    "idempotencyKey": "order_abc_20260607"
  }'
```

---

### Get order status

```
GET /b2b/status?id={orderId}
```

Poll this endpoint to check the current state of an order.

**Headers**

| Header | Value |
|---|---|
| `X-API-Key` | Your API key |

**Response `200 OK`**

```json
{
  "id": "3f7c1b2a-...",
  "status": "disbursed",
  "amountStableCoin": 50.0,
  "amountNGN": 82750.00,
  "currency": "NGN",
  "created": "2026-06-07T14:00:00Z",
  "updated": "2026-06-07T14:04:33Z"
}
```

**Order statuses**

| Status | Meaning |
|---|---|
| `initiated` | Order created — waiting for USDSUI deposit |
| `processing: wallet worker on it..` | Actively watching the deposit wallet |
| `processing: in bank queue` | Deposit confirmed — bank payout queued |
| `disbursed` | NGN sent to bank account successfully |
| `Settled in treasury` | Fully complete — USDSUI swept to treasury |
| `timeout: no deposit received` | No deposit was detected within 10 minutes |
| `failed` | Bank payout failed |

---

### Verify bank account

```
POST /b2b/verifybank
```

Resolve an account number to a name before creating an order. Always verify before submitting — submitting an incorrect account name results in a failed payout.

**Headers**

| Header | Value |
|---|---|
| `X-API-Key` | Your API key |
| `Content-Type` | `application/json` |

**Request body**

```json
{
  "bankCode": "033",
  "accountNumber": "1234567890"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `bankCode` | string | Yes | Nigerian bank code (e.g. `"033"` for UBA) |
| `accountNumber` | string | Yes | 10-digit account number |

**Response `200 OK`**

```json
{
  "accountName": "John Doe",
  "bankName": "United Bank for Africa",
  "accountNumber": "1234567890",
  "bankCode": "033"
}
```

**Example**

```bash
curl -X POST https://confidential-brianna-uselinq-52e2b233.koyeb.app/b2b/verifybank \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"bankCode": "033", "accountNumber": "1234567890"}'
```

---

### Rotate API key

```
POST /b2b/regenerate-api-key
```

Issues a new API key. The old key stops working immediately — update your integration before rotating.

**Headers**

| Header | Value |
|---|---|
| `X-API-Key` | Your current API key |

**Response `200 OK`**

```json
{
  "apiKey": "biz_live_...",
  "message": "API key rotated successfully. Update your integration immediately — the old key is now invalid."
}
```

---

### Rotate webhook secret

```
POST /b2b/regenerate-webhook-secret
```

Issues a new webhook signing secret. All future webhooks will be signed with the new secret. Update your verification logic before rotating.

**Headers**

| Header | Value |
|---|---|
| `X-API-Key` | Your API key |

**Response `200 OK`**

```json
{
  "webhookSecret": "...",
  "message": "Webhook secret rotated successfully. Update your verification logic immediately."
}
```

> Your webhook secret is also visible in the [B2B merchant dashboard](https://dashboard.uselinq.com). It is never returned after signup unless you explicitly rotate it here.

---

## Integration flow

```
1. Call GET /b2b/rate
   → Show user: "You will receive ₦82,750 for 50 USDSUI"

2. Call POST /b2b/offramp
   → Get back walletAddress

3. Show walletAddress to your user
   → "Send exactly 50 USDSUI to: 0xabc123..."

4. User sends USDSUI from their Sui wallet

5. Linq detects deposit → fires "order.processing" webhook
   → Update your UI: "Payment received, sending to bank..."

6. Bank payout completes → fires "order.completed" webhook
   → Update your UI: "₦82,750 sent to your bank account"
```

---

## Error responses

All error responses follow this shape:

```json
{
  "message": "human-readable error description"
}
```

| HTTP status | Meaning |
|---|---|
| `400` | Invalid request — check the `message` field |
| `401` | Missing or invalid API key |
| `403` | Account inactive |
| `429` | Rate limit exceeded — slow down requests |
| `500` | Server error — retry with exponential backoff |
| `503` | Rate fetch failed — try again in a few seconds |

---

## Idempotency

The `idempotencyKey` field prevents duplicate orders. If you send two requests with the same key (same business account), the second request returns the original order without creating a new one.

**You generate this key yourself** before making the request. It just needs to be unique per order — the value itself doesn't matter as long as you never reuse it.

**The simplest approach — use a UUID:**

```js
// Node.js
const { randomUUID } = require('crypto');
const idempotencyKey = randomUUID();
// → "a3f1c2d4-7e8b-4a9f-b1c2-d3e4f5a6b7c8"
```

```python
# Python
import uuid
idempotency_key = str(uuid.uuid4())
# → "a3f1c2d4-7e8b-4a9f-b1c2-d3e4f5a6b7c8"
```

```bash
# curl (generate inline)
curl -X POST .../b2b/offramp \
  -d "{\"idempotencyKey\": \"$(uuidgen)\", ...}"
```

**Or combine your own internal reference with a timestamp:**

```js
const idempotencyKey = `order_${yourUserId}_${Date.now()}`;
// → "order_user_789_1749254400000"
```

Store the key alongside the order in your own database so you can reuse it safely if you need to retry the same request (e.g. after a network timeout). Never generate a new key on a retry — that would create a duplicate order.

---

## Nigerian bank codes

The full list of supported bank codes is in [`nigerian-banks.json`](./nigerian-banks.json) (same directory as this file). Load it in your integration to populate bank selection dropdowns.

**Common banks:**

| Bank | Code |
|---|---|
| Access Bank | 044 |
| GTBank | 058 |
| First Bank | 011 |
| Zenith Bank | 057 |
| UBA | 033 |
| Kuda Microfinance Bank | 090267 |
| OPay | 100004 |
| PalmPay | 100033 |
| Moniepoint MFB | 090405 |
| Wema Bank | 035 |
| Sterling Bank | 232 |
| Fidelity Bank | 070 |
| FCMB | 214 |
| Stanbic IBTC | 039 |
| Polaris Bank | 076 |

See `nigerian-banks.json` for all 32 supported banks.

---

## Limits

| Parameter | Limit |
|---|---|
| Minimum order | No minimum (subject to bank payout minimum) |
| Maximum order | Contact support for high-volume limits |
| Rate limit | 10 orders per minute per API key |
| Deposit window | 10 minutes — user must send USDSUI within 10 minutes of order creation |

---

## FAQ

**What happens if the user sends a different amount than `amountStableCoin`?**  
Linq detects whatever USDSUI arrives in the wallet and pays out the equivalent NGN at the locked rate. The `amountStableCoin` in the order request is informational — the actual payout is based on what arrives.

**What happens if the user sends nothing within 10 minutes?**  
The order expires. An `order.failed` webhook fires with status `timeout: no deposit received`. Create a new order with a new `idempotencyKey` if the user wants to try again.

**Is there a fee?**  
No fee is deducted from the USDSUI. The full deposited amount converts to NGN at the market rate.

**What if my webhook endpoint is down when an event fires?**  
The webhook attempt is made once. Implement polling via `GET /b2b/status` as a fallback to reconcile any missed events.

**Can I verify a bank account before creating an order?**  
Yes — `POST /b2b/verifybank` with `{"bankCode": "033", "accountNumber": "1234567890"}` returns the account name. Always verify before submitting an offramp order.

---

## Support

For integration support, API key issues, or high-volume access — reach out to the Linq team.
