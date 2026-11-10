---
name: mom-factura-pool-invoices
description: Issue fiscal documents as a MoMenu Factura pool client (sub-tenant with its own NIF) — FR, FT, receipts, credit notes, proformas, PDFs and AGT status with the client's API key. Use when integrating pool invoicing or the user runs /mom-factura-pool-invoices.
license: MIT
metadata:
  author: mom-factura
  version: "1.1"
  language: pt
---

# MoMenu Factura — invoicing as a pool client

A pool client is a separate fiscal entity inside a reseller's pool. Its **client API key** (issued by the reseller) emits in the client's name: NIF, company name, logo and IBAN come from the client record — never send an issuer in the body. Reseller setup (pool, clients, keys, logo, FE key) is `mom-factura-pools`.

**The invoicing API is the same as for merchants — follow `mom-factura-invoicing`** (contract: `GET https://api.momenu.online/api/invoicing.openapi.json`). Differences for a pool client key:

| | Pool client key |
|---|---|
| Documents seen | only this client's documents (`404 INVOICE_NOT_FOUND` for anything else) |
| Numbering | the client's own series (`FR-MM2026-001`, `NC-…`, `RC-…`, `PP-…`) |
| Facturação Electrónica | the **client's** FE key must be active (`403 FE_CONFIG_REQUIRED`) — set by the reseller with `POST /api/pools/clients/{clientId}/fe-key` |
| `customerId`, `items[].productId`, `/api/customers`, `/api/products` | not available (`403 POOL_CLIENT_CATALOG_FORBIDDEN`): customers and catalog belong to the reseller's account. Send `customer` and line `description`/`unitPrice` |
| Templates | the reseller's invoice template is never used |

```bash
curl -X POST https://api.momenu.online/api/invoices \
  -H "x-api-key: $POOL_CLIENT_KEY" -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "invoiceType": "FR",
    "customer": { "name": "Cliente Exemplo, Lda", "nif": "5001328930" },
    "items": [{ "description": "Consultoria", "quantity": 1, "unitPrice": 57000, "iva": 14 }],
    "paymentMethod": "Dinheiro"
  }'
```

- `unitPrice` includes VAT by default; `iva` is 14, 7, 5 or 0 (default 0 — then `exemptionScheme`/`scheme` M00–M94 is required).
- FT paid later → `POST /api/invoices/{n}/convert-to-receipt` (issues a receipt, **RC**).
- Credit notes → `POST /api/invoices/{n}/cancel` or `/rectify` (not `invoiceType: "NC"`).
- The `Idempotency-Key` is scoped to this client key: other clients of the same reseller never share responses.
- Errors: `{ "success": false, "error": "...", "code": "..." }` — branch on `code`.
