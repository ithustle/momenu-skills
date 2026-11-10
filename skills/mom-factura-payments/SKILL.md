---
name: mom-factura-payments
description: Integrate Mom Factura Payment API for Angolan payment methods (Multicaixa Express, E-kwanza, Bank Reference). Use when implementing checkout flows, processing payments, generating AGT electronic invoices (Facturação Electrónica com QR), or handling product-based billing with IVA tax.
license: MIT
metadata:
  author: mom-factura
  version: "1.3"
  language: pt
---

# Mom Factura Payments Integration

Process payments for Angolan payment methods with automatic AGT electronic invoice generation (Facturação Electrónica com QR).

**Base URL:** `https://api.momenu.online`

## Authentication

All requests require the `x-api-key` header.

```
Content-Type: application/json
x-api-key: <MERCHANT_API_KEY>
```

## Payment Methods

### 1. Multicaixa Express (MCX) - Immediate

**POST** `/api/payment/mcx`

Immediate payment. Creates order as PAID and generates invoice on success.

Required body:
- `paymentInfo.amount` (number) - Kwanzas (min 100)
- `paymentInfo.phoneNumber` (string) - Format: 244XXXXXXXXX
- `products` (array, min 1) - Invoice line items; source of truth for the amount
- `instantWithdraw` (boolean) - **must be `true`**. Auto-payout of the net (minus 2%) to the merchant's verified bank account. See [Instant Withdrawal](#instant-withdrawal).

Optional body:
- `products[].id` (string), `products[].productName` (string), `products[].productPrice` (number), `products[].productQuantity` (number)
- `products[].iva` (number) - IVA rate 0-14, default **0** (exempt, reason M00)
- `customer` (object) - `name` (string), `nif` (string), `phone` (string). A name requires a real NIF.
- `split` / `iban` / `isPercent` / `value` / `beneficiaryName` / `beneficiaryNif` / `description` — marketplace split. See [Split payments](#split-payments-marketplace).
- `simulateResult` (string) - QA only: success, insufficient_balance, timeout, rejected, invalid_number

Example body:
```json
{
  "paymentInfo": { "amount": 5000, "phoneNumber": "244923456789" },
  "products": [{ "id": "1", "productName": "Artigo", "productPrice": 5000, "productQuantity": 1 }],
  "instantWithdraw": true
}
```

Success (200):
```json
{
  "success": true,
  "transactionId": "abc123...",
  "invoiceUrl": "https://invoice-momenu.toquemedia.net/invoices/..."
}
```
With `split: true`, the response also includes the `split` object (`status: "processing"` when MCX confirms immediately).

### 2. E-kwanza - Deferred

**POST** `/api/payment/ekwanza`

Deferred payment. Creates order as OPEN, returns QR code. Payment confirmation is delivered via webhook.

Required: `paymentInfo.amount`, `paymentInfo.phoneNumber`, `products` (min 1)
Optional: `customer` (same as MCX). `split` is **not** supported (`SPLIT_NOT_SUPPORTED`).

Success (200):
```json
{
  "success": true,
  "code": "EKZ123456",
  "qrCode": "data:image/png;base64,...",
  "expirationDate": "2024-01-15T12:00:00Z",
  "paymentTimeout": 180
}
```

Payment confirmation arrives via webhook. `GET /api/payment/ekwanza/status/:code` confirms an E-kwanza payment.

### 3. Bank Reference - Deferred

**POST** `/api/payment/reference`

Generates bank reference. Client pays via ATM or Internet Banking.

Required: `paymentInfo.amount`, `products` (min 1), `instantWithdraw: true`
Optional: `customer` (same as MCX), `split` / `iban` / `isPercent` / `value` (see [Split payments](#split-payments-marketplace))

Success (200):
```json
{
  "success": true,
  "operationId": "op-123...",
  "referenceNumber": "123456789",
  "entity": "12345",
  "dueDate": "2024-01-20"
}
```

## Amount Validation

`products` is **required** (min 1 valid item) and defines the amount.
- `total = SUM(productPrice * productQuantity)` for all products
- If `paymentInfo.amount` is also sent, it must match that total (`AMOUNT_MISMATCH` otherwise)
- Missing/empty products → `MISSING_PRODUCTS`; invalid item → `INVALID_PRODUCT`
- Total below 100 Kz → `BELOW_MINIMUM`

## Webhook (Payment Confirmation)

All deferred payments (Bank Reference and E-kwanza) are confirmed via webhook. Configure the webhook URL in the portal: login at [momenu.toquemedia.net](https://momenu.toquemedia.net) → **Desenvolvedores** (the account webhook, or the API key's own webhook).

**URL requirements:** `https://` only (valid TLS certificate), and it must be the final endpoint — redirects (3xx) are not followed and the delivery fails. Every address the host resolves to must be public (localhost, private, link-local and similar ranges are refused). A non-compliant URL receives no events; each refusal is recorded with its reason in **Desenvolvedores → Webhooks**. After saving, use **Enviar evento de teste** to confirm.

When a payment is confirmed, the API sends **two sequential events**:

**Event 1: `payment.confirmed`** — Sent immediately after order status is updated to PAID:
```json
{
  "event": "payment.confirmed",
  "merchantTransactionId": "abc123...",
  "ekwanzaTransactionId": "EKZ456...",
  "operationStatus": "1",
  "operationData": { ... }
}
```
When the payment was created with `split: true`, the payload also includes `split` (`status` is `processing` or later). `GET /api/payment/status/:transactionId` returns the same object.

**Event 2: `invoice.created`** — Sent after invoice PDF is generated and uploaded:
```json
{
  "event": "invoice.created",
  "merchantTransactionId": "abc123...",
  "ekwanzaTransactionId": "EKZ456...",
  "operationStatus": "1",
  "operationData": { ... },
  "invoiceUrl": "https://invoice-momenu.toquemedia.net/invoices/..."
}
```

Non-paid events (operationStatus 3, 4, 5) are sent without the `event` field for backward compatibility.

**operationStatus values:** `"1"` Paid · `"3"` Cancelled/Expired · `"4"` Failed/Refused · `"5"` Error

**Stored state:** GET `/api/payment/status/:transactionId` reads the order (`open`, `paid`, or `failed`). It does not confirm the payment. When paid, the body includes `invoiceUrl` if the invoice exists, and `split` when the payment was created with `split: true`.

**E-kwanza confirmation:** GET `/api/payment/ekwanza/status/:code` — returns `status: "paid"` or `"pending"`, and `invoiceUrl` when paid.

## Split payments (marketplace)

Merchant M hosts other sellers (M1) on their platform. On MCX/Reference, send:

| Field | Required | Meaning |
|-------|----------|---------|
| `split` | yes (`true`) | Enable split. Omit/`false` = current behaviour (full net to the integrator). |
| `iban` | if split | Beneficiary Angolan IBAN (`AO` + 23 digits, valid checksum) |
| `isPercent` | no | `true` = `value` is % of the **net**; omit/`false` = `value` in Kz |
| `value` | if split | Integrator commission on the net after the 2% MoMenu fee |
| `beneficiaryName` / `beneficiaryNif` | if split | Beneficiary identity — electronic commission FR |
| `description` | if split | Text on KWiK rows and on the commission FR item |

Example: 10.000 Kz, `isPercent: true`, `value: 10` → fee 200, commission 980 (verified IBAN), remainder 8.820 (payload IBAN).

```json
{
  "paymentInfo": { "amount": 10000, "phoneNumber": "244923456789" },
  "products": [{ "id": "1", "productName": "Artigo", "productPrice": 10000, "productQuantity": 1 }],
  "instantWithdraw": true,
  "split": true,
  "iban": "AO06000000000000000000001",
  "isPercent": true,
  "value": 10,
  "beneficiaryName": "Loja M1",
  "beneficiaryNif": "5417000000",
  "description": "Comissão de marketplace pela venda"
}
```

`split` is **additive** — the usual fields stay in the response:

```json
{
  "success": true,
  "transactionId": "abc123...",
  "invoiceUrl": "https://invoice-momenu.toquemedia.net/invoices/...",
  "status": "paid",
  "split": {
    "iban": "AO06000000000000000000001",
    "isPercent": true,
    "value": 10,
    "date": "2026-08-31T12:00:00.000Z",
    "grossAmount": 10000,
    "feeAmount": 200,
    "netAmount": 9800,
    "platformAmount": 980,
    "beneficiaryAmount": 8820,
    "status": "processing",
    "orderId": "abc123...",
    "beneficiaryName": "Loja M1",
    "beneficiaryNif": "5417000000",
    "description": "Comissão de marketplace pela venda",
    "platformInvoiceUrl": "https://invoice-momenu.toquemedia.net/invoices/..."
  }
}
```

Reference initiation is the same idea: `success`, `operationId`, `transactionId`, `referenceNumber`, `entity`, `dueDate` **plus** `split` (`status: "pending"` until the reference is paid). `payment.confirmed` and `GET /api/payment/status/:transactionId` include `split` when it applies. `invoiceUrl` is present when the invoice PDF already exists (MCX confirmed immediately); otherwise it arrives on `invoice.created`, and on the status read once the invoice exists.

`status`: `pending` (not yet paid) → `processing` (confirmed, KWiK in flight) → `completed` | `partial` | `failed`. Failed/partial may include `reason`.

Rules:
- The 2% MoMenu fee applies **only to `grossAmount`**. It is never applied to `platformAmount` or `beneficiaryAmount`.
- That 2% is the only Toque Media (MoMenu) fee line (`feeTransactions`). Split legs are recorded as KWiK rows (`fee: 0` + payload `description`).
- `platformAmount` is invoiced as an **electronic FR with AGT QR**. Number/URL: `split.platformInvoiceNumber` / `split.platformInvoiceUrl`. `invoiceUrl` is the sale FR (also FE).
- Sale FR remains integrator → end customer (the gross). The commission FR is a separate document.
- MCX and Reference only. E-kwanza, POOL/PLAN, payment links, exempt merchants → `SPLIT_NOT_SUPPORTED`
- Each leg ≥ 50 Kz → otherwise `SPLIT_LEG_BELOW_MINIMUM`
- Missing beneficiary name/NIF → `SPLIT_BENEFICIARY_REQUIRED`
- Missing `description` → `SPLIT_DESCRIPTION_REQUIRED`
- FE inactive → `FE_CONFIG_REQUIRED` (no PDF fallback)
- If daily KWiK remaining cannot cover **both** legs, neither is sent; the net stays in M's balance
- If M1's IBAN equals M's verified IBAN, one KWiK of the full net is sent; the `split` object still shows the commission breakdown

## Instant Withdrawal

With `instantWithdraw: true`, the payment value — **minus the 2% fee** — is transferred automatically (via E-kwanza KWiK) to the merchant's verified bank account as soon as the payment is confirmed.

- **Applies to:** Multicaixa Express (`/api/payment/mcx`) and Bank Reference (`/api/payment/reference`).
- **Not eligible:** direct E-kwanza payments (`/api/payment/ekwanza`) — the field is ignored.
- **Requires** a verified (approved) bank account for the merchant. Without it, the payment request is rejected.
- The flag is read **only** from the order created at payment time.
- Amounts above the per-withdrawal ceiling stay in the merchant balance for a manual withdrawal.

> `instantWithdraw: true` is **required** on `/api/payment/mcx` and `/api/payment/reference`. Missing or `false` → `INSTANT_WITHDRAW_REQUIRED` (400). A verified bank account is required (`BANK_ACCOUNT_NOT_VERIFIED`).

## Error Codes

| Code | Description |
|------|-------------|
| MISSING_API_KEY | x-api-key header missing |
| INVALID_API_KEY | Key invalid or inactive |
| DOMAIN_NOT_ALLOWED | Origin not registered |
| INVALID_AMOUNT | Invalid amount |
| AMOUNT_MISMATCH | amount != SUM(products) |
| MISSING_PRODUCTS | products is required (min 1 item) |
| INVALID_PRODUCT | invalid product item |
| BELOW_MINIMUM | amount below 100 Kz |
| BANK_ACCOUNT_NOT_VERIFIED | verified bank account required (403) |
| INSTANT_WITHDRAW_REQUIRED | instantWithdraw must be true on MCX/Reference (400) |
| MISSING_PHONE | Phone required (MCX/E-kwanza) |
| MISSING_RESTAURANT_ID | Merchant not identified |
| RATE_LIMIT_EXCEEDED | 100 req/min exceeded |
| PAYMENT_RATE_LIMIT_EXCEEDED | 20 payment req/min exceeded |
| PAYMENT_FAILED | Payment refused by the payment processor — `error` carries the processor's business message (e.g. rejected by the customer) |
| GATEWAY_TIMEOUT | The charge was not confirmed in time. **MCX:** returned as `status: "pending"` (HTTP 200, not an error) — the customer may still authorize and the order settles by webhook; do NOT retry. **Reference:** plain failure (no charge issued). |
| GATEWAY_UNREACHABLE | Could not reach the payment processor — no charge was created, safe to retry |
| INTERNAL_ERROR | Server error |
| SPLIT_NOT_SUPPORTED | Split not available (E-kwanza, POOL/PLAN, payment links, or no instant payout) |
| SPLIT_INVALID_IBAN | Beneficiary IBAN missing or invalid |
| SPLIT_INVALID_VALUE | Split `value` missing or invalid |
| SPLIT_LEG_BELOW_MINIMUM | A split leg is below 50 Kz |
| SPLIT_BENEFICIARY_REQUIRED | split:true requires beneficiaryName and a real beneficiaryNif |
| SPLIT_DESCRIPTION_REQUIRED | split:true requires description |
| FE_CONFIG_REQUIRED | Electronic invoicing (AGT) required; without it a PDF invoice is not issued |

Error format: `{ "success": false, "error": "message", "code": "ERROR_CODE" }`

Errors carry no payment-provider branding. `isTimeout: true` marks a timeout/network failure on our side (`GATEWAY_TIMEOUT`); processor business failures come back as `PAYMENT_FAILED` with the processor's own message in `error`.

**MCX still pending:** if the charge is not confirmed in the create response, the API creates an OPEN order and responds `status: "pending"` (HTTP 200). The customer may still authorize. The webhook settles the order and sends `payment.confirmed` then `invoice.created`. **Do not retry** — that can charge twice. There is no status URL on this response. `GET /api/payment/status/:transactionId` shows the stored state when you ask.

## Fees

2% processing fee on all payments: `feeAmount = totalAmount * 0.02`

## Notes

- Phone format: 244XXXXXXXXX (12 digits)
- IVA defaults to 0 (exempt, reason M00). Send `"iva": 14` to apply IVA.
- Invoice PDFs hosted on CDN, returned as `invoiceUrl`
- MCX is immediate when the charge confirms at once. Reference and a pending MCX are confirmed by webhook. `GET /api/payment/status/:transactionId` reads the stored state. E-kwanza is confirmed by `GET /api/payment/ekwanza/status/:code`
