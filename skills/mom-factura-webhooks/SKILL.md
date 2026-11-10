---
name: mom-factura-webhooks
description: Implement webhook-based payment confirmation for Mom Factura API and a one-shot read of the stored payment state. Use when building payment confirmation flows, receiving webhook notifications, confirming E-kwanza, reading GET /api/payment/status/:transactionId, or handling order state transitions from OPEN to PAID.
license: MIT
metadata:
  author: mom-factura
  version: "2.2"
  language: pt
---

# Mom Factura Webhooks & Status

Receive payment confirmations by webhook. Read the stored state with one GET when you need it. E-kwanza is confirmed by its own status route.

**Base URL:** `https://api.momenu.online`
**Auth:** `x-api-key` header required on all requests.

## How It Works

### Bank Reference (Webhook)

1. Login at [momenu.toquemedia.net](https://momenu.toquemedia.net)
2. Go to the **Desenvolvedores** menu
3. Add your webhook URL — `https://` only, the final endpoint (redirects are not followed) and resolving to public addresses only (localhost and private/link-local ranges are refused)
4. Save the configuration and use **Enviar evento de teste** (Desenvolvedores → Webhooks) to confirm delivery

### E-kwanza

1. Create an E-kwanza payment — receive a `code` in the response
2. Confirm with `GET /api/payment/ekwanza/status/:code`
3. When `status` is `"paid"`, the payment is confirmed and `invoiceUrl` is returned

### Stored payment state

`GET /api/payment/status/:transactionId` reads the order. `:transactionId` is the id returned when the payment was created. It does not confirm the payment and it does not call the gateway. Call it once when you want the stored state.

`status` is `open` (waiting for the callback), `paid`, or `failed`. `paidAt` and `invoiceUrl` are present only when the order is paid and the invoice exists. `failureReason` is present only when `status` is `failed`. A payment created with `split: true` includes `split`. Another merchant's order, or a missing one, is `404 NOT_FOUND`. A database failure is `500`, not `404`.

## Webhook Payload (POST to your URL)

Confirmed payments send two events. First:

```json
{
  "event": "payment.confirmed",
  "merchantTransactionId": "abc123...",
  "ekwanzaTransactionId": "EKZ456...",
  "operationStatus": "1",
  "operationData": { ... }
}
```

If the payment was created with `split: true`, `payment.confirmed` also includes a `split` object (`iban`, `isPercent`, `value`, `grossAmount`, `feeAmount`, `netAmount`, `platformAmount`, `beneficiaryAmount`, `status`, `orderId`, optional `beneficiaryName` / `beneficiaryNif` / `platformIban` / `reason`). On confirmation `status` is `processing` (or later). `GET /api/payment/status/:transactionId` returns the same object with the current split status (`completed` / `partial` / `failed`).

Then `invoice.created` with `invoiceUrl`.

**operationStatus values:** `"1"` Paid · `"3"` Cancelled/Expired · `"4"` Failed/Refused · `"5"` Error

## Webhook Server Example - Node.js / Express

```javascript
const express = require("express");
const app = express();

app.use(express.json());

app.post("/webhook/meu-webhook", (req, res) => {
  const { merchantTransactionId, operationStatus } = req.body;

  if (operationStatus === "1") {
    // Payment confirmed — update your database, notify the customer, etc.
    console.log("Payment confirmed:", merchantTransactionId);
  } else if (["3", "4", "5"].includes(operationStatus)) {
    console.log("Payment failed:", operationStatus);
  }

  res.status(200).json({ received: true });
});

app.listen(3000);
```

## Webhook Server Example - Python / Flask

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/webhook/meu-webhook", methods=["POST"])
def momenu_webhook():
    data = request.json
    transaction_id = data.get("merchantTransactionId")
    status = data.get("operationStatus")

    if status == "1":
        # Payment confirmed
        print(f"Payment confirmed: {transaction_id}")
    elif status in ["3", "4", "5"]:
        print(f"Payment failed: {status}")

    return jsonify({"received": True}), 200
```

## E-kwanza confirmation example

```javascript
async function verificarStatusEkwanza(code) {
  const response = await fetch(
    `https://api.momenu.online/api/payment/ekwanza/status/${code}`,
    { headers: { "x-api-key": "SUA_API_KEY" } }
  );

  const data = await response.json();

  if (data.status === "paid") {
    console.log("Pago! Fatura:", data.invoiceUrl);
  }

  return data;
}
```

## Read the stored state

`GET /api/payment/status/:transactionId`

```json
{
  "success": true,
  "transactionId": "abc123...",
  "status": "open",
  "paymentMethod": "Referência Bancária",
  "amount": 2500
}
```

When `status` is `paid`, the body also includes `paidAt` and `invoiceUrl` if the invoice exists. When `status` is `failed`, it may include `failureReason` (a short callback code such as `3002`).

## Notes

- The webhook confirms the payment. `GET /api/payment/status/:transactionId` only shows what is already stored
- Your webhook endpoint must return HTTP 2xx to acknowledge receipt
- E-kwanza is confirmed by `GET /api/payment/ekwanza/status/:code`
- MCX is immediate when the charge confirms at once; a pending MCX settles by webhook. The create response does not include a status URL
- Marketplace `split` (when present) is on `payment.confirmed` and on `GET /api/payment/status/:transactionId` — not on `invoice.created`
- Rate limiting: 100 req/min general. The status read shares that limit
