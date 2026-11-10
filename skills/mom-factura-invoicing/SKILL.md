---
name: mom-factura-invoicing
description: Issue and manage Angolan fiscal documents through the MoMenu Factura API — FR, FT, receipts (RC), credit notes (cancel or rectify by line), proformas, PDFs (A4/C7), email, AGT e-invoicing status, customers and products. Use when integrating invoicing (not payments) with api.momenu.online, or the user runs /mom-factura-invoicing.
license: MIT
metadata:
  author: mom-factura
  version: "1.1"
  language: pt
---

# MoMenu Factura — invoicing API

Everything the MoMenu portal does with fiscal documents, from the merchant's backend. Documents are registered with the AGT (Facturação Electrónica) by MoMenu.

- **Base URL:** `https://api.momenu.online`
- **Contract (source of truth):** `GET /api/invoicing.openapi.json` — fetch it when you need a field or error code not listed here. Human docs: `/docs#invoicing-overview`; plain text: `/llms.txt`.
- **Plan:** Profissional. **Currency:** Kwanza (AOA).

## Authentication

```
x-api-key: <API_KEY>
Content-Type: application/json
```

Call from the backend only (an `Origin` header outside the account's domains → 403 `DOMAIN_NOT_ALLOWED`). Limits: 120 requests/min per API key and 300/min per client IP on these routes (429 `RATE_LIMIT_EXCEEDED` + `Retry-After`), body up to 512 KB. Accounts with a team must use a generated key. A pool client key issues as that client (see `mom-factura-pool-invoices`).

**Scope:** every operation sees only the documents of the key's scope — account key: the account's documents; generated key: documents without a key plus its own; pool key: that client's documents. Out of scope → `404 INVOICE_NOT_FOUND`.

**Prerequisite:** FR, FT, NC and RC need Facturação Electrónica active on the account (`403 FE_CONFIG_REQUIRED`). Proformas don't.

## Which document

| Situation | Call |
|---|---|
| Sale paid now | `POST /api/invoices` `invoiceType: "FR"` |
| Sale to be paid later | `POST /api/invoices` `invoiceType: "FT"`, later `POST /api/invoices/{n}/convert-to-receipt` (issues an **RC**) |
| Quote | `POST /api/proformas/a4`, then `POST /api/proformas/{n}/convert` (`FR`/`FT`) or `/cancel` |
| Undo a whole invoice | `POST /api/invoices/{n}/cancel` `{ reason }` |
| Return / correct some units | `POST /api/invoices/{n}/rectify` `{ reason, lines: [{ line, quantity }] }` |

Never send `invoiceType: "NC"` (→ 400 `CREDIT_NOTE_ENDPOINT_REQUIRED`) and never send `invoiceNumber` on creation.

## Issue a document — `POST /api/invoices`

```bash
curl -X POST https://api.momenu.online/api/invoices \
  -H "x-api-key: $MOMENU_API_KEY" -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "invoiceType": "FR",
    "customerId": "3f2a9c10-…",
    "items": [
      { "productId": "b7f6c1d2-…", "quantity": 2, "discountPercent": 5 },
      { "description": "Livro técnico", "quantity": 1, "unitPrice": 25000, "iva": 0, "exemptionScheme": "M04" }
    ],
    "withholding": { "type": "IRT", "percent": 6.5 },
    "paymentMethod": "Transferência Bancária"
  }'
```

- Customer: `customerId` (registered — its NIF is used as stored) or `customer: { name, nif, phone?, address?, email? }`. An inline NIF that isn't a real Angolan NIF → issued to Consumidor Final (`999999999`) and the other customer fields are dropped.
- Lines (1–100): `productId` (catalog: description, net price, VAT and exemption come from the product) or `description` + `unitPrice`; `quantity`; `iva` 14/7/5/0; `exemptionScheme` (M00–M94) when `iva` is 0 (or a document-level `scheme`; required when every line is exempt — in a mixed document an exempt line without a reason is issued as M00); `discount` (Kz) or `discountPercent`.
- Prices include VAT by default; `pricesIncludeVat: false` for net prices. Catalog prices are always net.
- `discount` (document) is spread over the lines. `withholding` `{ type: IRT|II|IS|IVA|IP|IAC|OU, percent | amount }`, max 30% of the total; the customer pays `amountDue`.
- `paymentMethod` (required except proformas): `Multicaixa Express`, `E-Kwanza`, `Referência Bancária`, `Dinheiro`, `Transferência Bancária`, `Cartão de Crédito/Débito`, `TPA`, `Afrimoney`.
- `format`: `A4` (default) or `C7` (thermal; FR only — FT and proformas are A4). `notes` printed on the document.
- **Totals:** the net unit price is rounded to the cent so the AGT registers exactly the document total. With VAT-inclusive prices the total may differ by cents from Σ quantity × price — **always use `total`/`amountDue` from the response**.

201 → `{ success, invoiceNumber, invoiceType, documentId, invoiceUrl, total, totalIncidence, totalTaxes, totalDiscount, withholdingAmount, amountDue, format, createdAt }`. Store `invoiceNumber`: every other operation uses it.

## After issuing

- `GET /api/invoices/{n}` → `lines[]` (`line`, net/VAT/gross, `creditedQuantity`, `remainingQuantity`), `status` (`ACTIVE`, `PARTIALLY_CREDITED`, `CREDITED`, `CANCELLED`; proformas `ACTIVE`/`CONVERTED`/`CANCELLED`/`EXPIRED`), `totals`, `agt`, `creditNotes`, `receipt`.
- `GET /api/invoices` → filters `type=FR,FT,NC,RC,PROFORMA`, `status=active|cancelled|rectified`, `paid=true|false` (FT), `agtStatus`, `customerNif`, `from`/`to` (YYYY-MM-DD, Luanda), `limit` (≤100, default 30), `offset`.
- `GET /api/invoices/{n}/pdf?format=A4|C7&redirect=true` → current PDF (with the AGT QR once registered). A signed `invoiceUrl` (30 days) opens the PDF in a browser; fetched by a program it returns `{ url }`.
- `POST /api/invoices/{n}/email` `{ to?, cc?, bcc?, message? }` → PDF attached; default recipient = the customer's email. Limit 10/min per key, 200/day per account.
- `POST /api/invoices/{n}/fe-status` → AGT status (`PENDING`, `VALID`, `INVALID`, `SUBMIT_FAILED`); resubmits recoverable failures.

## Credit notes

- **Cancel** (`/cancel`, `{ reason }` 3–200 chars): reverses every line. Repeating returns the same NC (`alreadyCancelled: true`). An invoice with rectifications can't be cancelled (`409 ORIGIN_PARTIALLY_CREDITED`) — rectify the rest.
- **Rectify** (`/rectify`): **`Idempotency-Key` required**. Lines are the `line` numbers of `GET /api/invoices/{n}`; quantities up to `remainingQuantity` (`409 QUANTITY_EXCEEDS_REMAINING` with `details`). Credits the price actually charged (after discount) at the line's VAT rate. A line with a fractional quantity (e.g. 2.5 kg) can only be credited whole.
- The origin must already be registered and not rejected by the AGT (`409 ORIGIN_NOT_REGISTERED` → retry shortly, or fix it via `/fe-status`).
- `POST /api/credit-notes/a4|c7` is deprecated.

## Customers and products

`/api/customers` (`GET ?q=&nif=`, `POST`, `GET|PATCH|DELETE /{id}`): NIF unique per account (`409 CUSTOMER_NIF_EXISTS` with `details.customerId`).
`/api/products` (`GET ?type=&active=&q=`, `POST`, `POST /bulk` 1–50 all-or-nothing, `GET|PATCH|DELETE /{id}`): `FISICO`/`SERVICO`, net `price`, `iva` + `exemptionScheme`; PATCH can't change `stock` or `type`. Not available to pool client keys (`403 POOL_CLIENT_CATALOG_FORBIDDEN`).

## Reliability rules

1. Send `Idempotency-Key` (UUID, ≤ 255 chars) when issuing (`POST /api/invoices`, `/api/proformas/a4`), cancelling and rectifying. On a network error, `503` or `504 UPSTREAM_TIMEOUT` retry **with the same key** — you get the same document (`Idempotency-Replayed: true`), never a duplicate. Same key + different body → `409 IDEMPOTENCY_KEY_REUSED`.
2. Receipts, proforma conversion/cancel and cancellation also repeat safely without a key. Customers don't duplicate (unique NIF → 409 with the existing id). **Products and emails are not idempotent**: retrying creates another product / sends another email.
3. Branch on `code`, never on the message. Errors: `{ success: false, error, code, details? }`.
4. `504 UPSTREAM_TIMEOUT` means the operation may have happened — retry with the same key or `GET` the document before doing anything else. `503`, `REQUEST_IN_PROGRESS` and `429` → retry with backoff.
5. Documents are fiscal and registered with the AGT: test with a staging account first.
