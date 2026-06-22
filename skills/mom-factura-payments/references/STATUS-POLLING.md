# Webhook & Status Reference

## Webhook (Primary)

Payment confirmation for Reference is delivered via webhook.

**Your webhook receives a POST with:**
```json
{
  "merchantTransactionId": "abc123...",
  "ekwanzaTransactionId": "EKZ456...",
  "operationStatus": "1",
  "operationData": { ... }
}
```

**operationStatus values:** `"1"` Paid · `"3"` Cancelled/Expired · `"4"` Failed/Refused · `"5"` Error

**Configuration:** Login at [momenu.toquemedia.net](https://momenu.toquemedia.net), go to **Desenvolvedores** menu, and add your webhook URL.

## Status Endpoints (Fallback)

Use these if webhook delivery fails or as a manual check.

### Bank Reference Status

**GET** `/api/payment/reference/status/:operationId`

Pending:
```json
{ "success": true, "payment": { "status": "pending", "message": "Aguardando pagamento" } }
```

Paid:
```json
{
  "success": true,
  "payment": { "status": "paid", "message": "Pagamento confirmado" },
  "invoiceUrl": "https://invoice-momenu.toquemedia.net/invoices/..."
}
```

## Rate Limiting

- General: 100 req/min per IP
- Payments: 20 req/min per IP (POST only)
- Status endpoints (GET) count toward general limit
- Minimum polling interval: 30 seconds (Reference)
