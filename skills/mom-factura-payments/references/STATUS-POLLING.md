# Webhook and stored payment state

## Webhook

Payment confirmation for Reference, and for a Multicaixa Express charge that stays pending, is delivered by webhook. E-kwanza is confirmed by its own status route.

**Your webhook receives sequential events.** `payment.confirmed` first:

```json
{
  "event": "payment.confirmed",
  "merchantTransactionId": "abc123...",
  "ekwanzaTransactionId": "EKZ456...",
  "operationStatus": "1",
  "operationData": { ... }
}
```

With `split: true` on the original payment, `payment.confirmed` also includes a `split` object. Then `invoice.created` with `invoiceUrl`.

**operationStatus values:** `"1"` Paid · `"3"` Cancelled/Expired · `"4"` Failed/Refused · `"5"` Error

**Configuration:** Login at [momenu.toquemedia.net](https://momenu.toquemedia.net), go to **Desenvolvedores** menu, and add your webhook URL.

**URL requirements:** `https://` only (valid TLS certificate), and it must be the final endpoint — redirects (3xx) are not followed and the delivery fails. Every address the host resolves to must be public (localhost, private, link-local and similar ranges are refused). A non-compliant URL receives no events; each refusal is recorded with its reason in **Desenvolvedores → Webhooks**. After saving, use **Enviar evento de teste** to confirm.

## Stored payment state

`GET /api/payment/status/:transactionId` reads the order. `:transactionId` is the id returned when the payment was created. One request when you want the stored state. It does not confirm the payment.

Open:

```json
{
  "success": true,
  "transactionId": "abc123...",
  "status": "open",
  "paymentMethod": "Referência Bancária",
  "amount": 2500
}
```

Paid:

```json
{
  "success": true,
  "transactionId": "abc123...",
  "status": "paid",
  "paymentMethod": "Referência Bancária",
  "amount": 2500,
  "paidAt": "2026-09-29T12:00:00.000Z",
  "invoiceUrl": "https://invoice-momenu.toquemedia.net/invoices/..."
}
```

Failed:

```json
{
  "success": true,
  "transactionId": "abc123...",
  "status": "failed",
  "paymentMethod": "Referência Bancária",
  "amount": 2500,
  "failureReason": "3002"
}
```

`failed` means the callback closed without a payment. `failureReason` is the short callback code when the gateway sent one. When the payment was created with `split: true`, the body includes `split` (`pending` → `processing` → `completed` | `partial` | `failed`). The updated split state is this read, or the `payment.confirmed` webhook.

## E-kwanza confirmation

**GET** `/api/payment/ekwanza/status/:code`

This call confirms the E-kwanza payment.

Pending:

```json
{ "success": true, "status": "pending", "operationCode": "OP123..." }
```

Paid:

```json
{
  "success": true,
  "status": "paid",
  "operationCode": "OP123...",
  "invoiceUrl": "https://invoice-momenu.toquemedia.net/invoices/..."
}
```

## Rate limiting

- General: 100 req/min per IP
- Payments: 20 req/min per IP (POST only)
- `GET /api/payment/status/:transactionId` shares the general limit
