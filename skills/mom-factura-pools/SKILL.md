---
name: mom-factura-pools
description: Manage Mom Factura invoice pools for resellers (multi-company sub-tenant invoicing). Use when onboarding sub-tenants with their own NIF, issuing standalone pool API keys, uploading client logos, or emitting invoices with pool client keys.
license: MIT
metadata:
  author: mom-factura
  version: "1.0"
  language: pt
---

# Mom Factura Pools (Reseller / Sub-tenants)

Manage invoice pools: a PRO reseller onboard fiscal sub-tenants (each with its own NIF) and hands each one a standalone API key to emit invoices via the same `/api/invoices` endpoints.

**Base URL:** `https://api.momenu.online`
**Auth:** `x-api-key` header on all requests.

## Concepts

| Entity | What it is |
|--------|------------|
| **Pool** | Container owned by the reseller. One pool is included free with the PRO plan (`claim-included`). |
| **Pool Client** | Sub-tenant fiscal record (NIF, company, logo). The emitter identity on invoices. |
| **Pool Client Key** | Standalone API key bound to one client. Used for **emission only** — management endpoints reject it (403). |
| **Reseller Key** | The account API key. Required for all `/api/pools/*` management endpoints. |

Key rules:

- Clients must be cadastrados **before** emission. The invoice body does not register the emitter — NIF, company and logo come from the client's record via its API key.
- Invoices emitted with a pool client key are stored in `poolInvoices` (not `invoices`), scoped by `clientId`.
- A pool client key can only: emit invoices, list/get its own invoices, refresh FE status. Everything else returns 403 `POOL_MGMT_FORBIDDEN_FOR_SUBTENANT` (or `POOL_CLIENT_MGMT_FORBIDDEN_FOR_SUBTENANT` / `POOL_KEY_MGMT_FORBIDDEN_FOR_SUBTENANT`).
- FE (Facturação Electrónica, AGT) is per-client: FR/FT/NC emission requires the client's own AGT private key (`fe-key` endpoint). PROFORMA is not fiscal and works without FE.

## Pools

### List pools

**GET** `/api/pools`

```bash
curl https://api.momenu.online/api/pools -H "x-api-key: RESELLER_KEY"
```

### Get pool

**GET** `/api/pools/:poolId`

### Claim the PRO-included pool

**POST** `/api/pools/claim-included`

Idempotent — returns 400 if already claimed. Returns the pool with its `poolId`.

```bash
curl -X POST https://api.momenu.online/api/pools/claim-included \
  -H "x-api-key: RESELLER_KEY"
```

## Pool Clients

### List clients

**GET** `/api/pools/clients` — all clients across the reseller's pools
**GET** `/api/pools/clients?poolId=POOL_ID` — clients of one pool

### Create client

**POST** `/api/pools/clients`

Required body:
- `poolId` (string)
- `nif` (string) — sub-tenant tax ID
- `company` (object) — `name` (required), `address`, `phone`, `email`, `iban`

```bash
curl -X POST https://api.momenu.online/api/pools/clients \
  -H "Content-Type: application/json" \
  -H "x-api-key: RESELLER_KEY" \
  -d '{
    "poolId": "POOL_ID",
    "nif": "5001234567",
    "company": {
      "name": "Cliente Lda",
      "address": "Rua Exemplo 12, Luanda",
      "phone": "+244923456789",
      "email": "cliente@example.com",
      "iban": "AO06004000012345678910123"
    }
  }'
```

Response (201): `{ "success": true, "client": { "clientId", "nif", "company", ... } }`

### Update client

**PATCH** `/api/pools/clients/:clientId` — body: `{ "company": { ... } }` (partial merge of the company record)

### Remove / recover client

- **PATCH** `/api/pools/clients/:clientId/remove` — soft-delete: marks inactive and frees the slot. Documents already emitted remain intact.
- **PATCH** `/api/pools/clients/:clientId/recover` — reactivates a removed client (occupies a slot again).

### Client logo

- **POST** `/api/pools/clients/:clientId/logo` — `multipart/form-data`, field `file` (JPEG/PNG/WebP ≤ 5MB). The logo is printed on the client's invoices.
- **DELETE** `/api/pools/clients/:clientId/logo`

```bash
curl -X POST https://api.momenu.online/api/pools/clients/CLIENT_ID/logo \
  -H "x-api-key: RESELLER_KEY" \
  -F "file=@logo.png"
```

### Client API keys

- **POST** `/api/pools/clients/:clientId/keys` — body: `{ "label": "Produção" }` → returns the generated key (shown once).
- **PATCH** `/api/pools/clients/:clientId/keys/:keyId` — body: `{ "label"?, "active"? }`. Deactivating blocks new emissions with that key.
- **DELETE** `/api/pools/clients/:clientId/keys/:keyId`

### Client FE key (AGT)

Facturação Electrónica is configured per client with the AGT private key:

- **POST** `/api/pools/clients/:clientId/fe-key` — body: `{ "privateKey": "<PEM or base64>" }` (validated and stored encrypted by NIF)
- **GET** `/api/pools/clients/:clientId/fe/status` — FE state for the client
- **DELETE** `/api/pools/clients/:clientId/fe-key`

Without an active FE key, FR/FT/NC emission via this client's keys returns 403 `FE_NOT_CONFIGURED` (PROFORMA still works).

## Pool Keys (standalone, unassigned)

Keys created before the client exists. A key cannot emit until it is associated with a cadastrado pool client.

- **POST** `/api/pools/keys` — body: `{ "poolId", "label" }` (201)
- **GET** `/api/pools/keys?poolId=POOL_ID` — all keys (claimed + unclaimed)
- **PATCH** `/api/pools/keys/:keyId` — body: `{ "active"?: boolean, "label"?: string }`
- **DELETE** `/api/pools/keys/:keyId` — removes an **unclaimed** key only; claimed keys live inside the client's `apiKeys[]` and must be removed via `/api/pools/clients/:clientId/keys/:keyId`

## Emission with a Pool Client Key

Use the standard invoice endpoints with the client's key in `x-api-key`. The emitter (NIF, company, logo) is the pool client; `customer` in the body is the invoice recipient.

```bash
curl -X POST https://api.momenu.online/api/invoices \
  -H "Content-Type: application/json" \
  -H "x-api-key: CLIENT_KEY" \
  -d '{
    "invoiceType": "FR",
    "customer": { "name": "Consumidor Final", "nif": "999999999" },
    "items": [
      { "productName": "Serviço X", "productPrice": 5000, "productQuantity": 1, "iva": 14 }
    ],
    "paymentMethod": ["Multicaixa Express"]
  }'
```

- `invoiceType`: `FR` (Factura-Recibo), `FT` (Factura), `PROFORMA`, `NC` (Nota de Crédito, against an existing invoice)
- `scheme` (M00–M94) is **required** when all items have `iva=0` or omitted — otherwise 400 `INVALID_SCHEME`
- Available endpoints with a client key:
  - `POST /api/invoices` — emit (FR/FT/PROFORMA/NC)
  - `GET /api/invoices` — list **this client's** invoices (from `poolInvoices`, filtered by `clientId`)
  - `GET /api/invoices/:invoiceNumber` — get by number (e.g. `FR-MOM2026-001`)
  - `POST /api/invoices/:invoiceNumber/fe-status` — check and, if recoverable (E09/E34/E36/E42), resubmit the AGT registration
  - `POST /api/invoices/:invoiceNumber/convert-to-receipt` — FT → FR (idempotent)
- The response includes `invoiceUrl` (signed PDF link)

## Trial (reseller without active PRO plan)

Pool trial quotas enforced on emission with client keys (403 `POOL_TRIAL_LIMIT`):

| Quota | Limit |
|-------|-------|
| Pool clients | 1 |
| Proformas | 1 |
| Fiscal invoices (FT/FR) | 1, at exactly 100 Kz |
| Credit notes | 1, up to 100 Kz |
| FT → FR conversion | Not allowed |

Upgrade to the Profissional plan to remove quotas.

## Error Codes

| Code | Description |
|------|-------------|
| MISSING_API_KEY / INVALID_API_KEY | Auth header missing or invalid |
| POOL_MGMT_FORBIDDEN_FOR_SUBTENANT | Pool endpoints require the reseller's account key |
| POOL_CLIENT_MGMT_FORBIDDEN_FOR_SUBTENANT | Client management requires the reseller's account key |
| POOL_KEY_MGMT_FORBIDDEN_FOR_SUBTENANT | Key management requires the reseller's account key |
| MISSING_POOL_ID / MISSING_NIF / MISSING_COMPANY / MISSING_LABEL | Required body field missing |
| MISSING_PRIVATE_KEY | FE key missing on `fe-key` |
| INVALID_CONTENT_TYPE / MISSING_FILE | Logo upload must be multipart with field `file` |
| FE_NOT_CONFIGURED | FR/FT/NC emission without an active AGT key for the client |
| INVALID_SCHEME | `scheme` required when iva is 0/omitted |
| POOL_TRIAL_LIMIT | Trial quota exceeded (see above) |
| RATE_LIMIT_EXCEEDED | 100 req/min exceeded |

Error format: `{ "success": false, "error": "message", "code": "ERROR_CODE" }`

## Typical Onboarding Flow

```javascript
const API = "https://api.momenu.online";

// 1. Reseller claims the pool included with PRO
await fetch(`${API}/api/pools/claim-included`, {
  method: "POST",
  headers: { "x-api-key": RESELLER_KEY }
});

// 2. Register the sub-tenant (fiscal record)
const clientRes = await fetch(`${API}/api/pools/clients`, {
  method: "POST",
  headers: { "Content-Type": "application/json", "x-api-key": RESELLER_KEY },
  body: JSON.stringify({
    poolId: POOL_ID,
    nif: "5001234567",
    company: { name: "Cliente Lda", address: "Luanda" }
  })
});
const { client } = await clientRes.json();

// 3. Upload the client's logo (printed on invoices)
await fetch(`${API}/api/pools/clients/${client.clientId}/logo`, {
  method: "POST",
  headers: { "x-api-key": RESELLER_KEY },
  body: logoFormData // multipart field "file"
});

// 4. Configure the client's AGT key (required for FR/FT/NC)
await fetch(`${API}/api/pools/clients/${client.clientId}/fe-key`, {
  method: "POST",
  headers: { "Content-Type": "application/json", "x-api-key": RESELLER_KEY },
  body: JSON.stringify({ privateKey: AGT_PRIVATE_KEY })
});

// 5. Generate the client's API key (shown once — store it)
const keyRes = await fetch(`${API}/api/pools/clients/${client.clientId}/keys`, {
  method: "POST",
  headers: { "Content-Type": "application/json", "x-api-key": RESELLER_KEY },
  body: JSON.stringify({ label: "Produção" })
});
const { apiKey } = await keyRes.json();

// 6. Hand CLIENT_KEY to the sub-tenant — they emit on their own
await fetch(`${API}/api/invoices`, {
  method: "POST",
  headers: { "Content-Type": "application/json", "x-api-key": apiKey },
  body: JSON.stringify({
    invoiceType: "FR",
    customer: { name: "Consumidor Final", nif: "999999999" },
    items: [{ productName: "Serviço X", productPrice: 5000, productQuantity: 1, iva: 14 }]
  })
});
```

## Notes

- Management endpoints are reseller-only; the pool client key is emission-only.
- `remove` is a soft-delete — emitted documents remain for SAFT/legal purposes; `recover` reactivates the client.
- QA testing: add header `x-env-qa: true` (see the mom-factura-testing skill).
- Payment endpoints (`/api/payment/*`) always use the reseller's account key — pool client keys are for invoicing only.
