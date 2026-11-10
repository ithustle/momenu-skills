---
name: mom-factura-pools
description: Manage a MoMenu Factura Pool — create the pool, register client companies (NIF, logo, FE key) and issue them API keys. Use when integrating the Pools API as a reseller, onboarding pool clients, uploading logos, activating AGT Facturação Electrónica, or the user runs /mom-factura-pools.
license: MIT
metadata:
  author: mom-factura
  version: "1.0"
  language: pt
---

# MoMenu Factura — Pool (revendedor)

Turn a MoMenu account into a **billing provider for other companies**. Each pool client has its own NIF, logo, AGT FE and invoice sequence. This skill is for the **reseller** (your MoMenu API key). Invoice emission by the client is `mom-factura-pool-invoices`.

**Base URL:** `https://api.momenu.online`

## Authentication (reseller)

All management calls use the **account** API key from Developers — never a pool-client key.

```
Content-Type: application/json
x-api-key: <MOMENU_ACCOUNT_API_KEY>
```

A client key on these routes returns **403**.

## Model

1. One pool per MoMenu user (`POST /api/pools/claim-included`, idempotent).
2. Clients live under the pool (`/api/pools/clients`). Required: `nif`, `company.name`, `company.address`.
3. Create API keys on the client (`POST /api/pools/clients/:clientId/keys`, max 5). Hand that key to the client.
4. Activate FE with the **client NIF** private key (`POST /api/pools/clients/:clientId/fe-key`). Fiscal FR/FT/NC are blocked until FE is on.
5. Optional logo: `POST /api/pools/clients/:clientId/logo` (`multipart/form-data`, field `file`, JPEG/PNG/WebP ≤ 5MB). The server compresses it. **Do not** send a logo on `POST /api/invoices`.

Trial (no active Pro): 1 client; emission quotas are enforced on the client key (see `mom-factura-pool-invoices`).

## Endpoints

| Method | Path | Notes |
|--------|------|--------|
| GET | `/api/pools` | Array with 0 or 1 pool |
| POST | `/api/pools/claim-included` | Create/get the pool |
| GET | `/api/pools/:poolId` | Owner only |
| GET | `/api/pools/clients` | Optional `?poolId=` |
| POST | `/api/pools/clients` | 201. Does **not** create keys |
| GET | `/api/pools/clients/:clientId` | |
| PATCH | `/api/pools/clients/:clientId` | Body `{ company }` only |
| PATCH | `/api/pools/clients/:clientId/remove` | Deactivates keys |
| PATCH | `/api/pools/clients/:clientId/recover` | Fails if NIF already active |
| POST | `/api/pools/clients/:clientId/logo` | `file` field |
| DELETE | `/api/pools/clients/:clientId/logo` | Invoice shows company name |
| POST | `/api/pools/clients/:clientId/keys` | Body `{ "label": "Produção" }` |
| PATCH | `/api/pools/clients/:clientId/keys/:keyId` | `{ label?, active? }` |
| DELETE | `/api/pools/clients/:clientId/keys/:keyId` | |
| POST | `/api/pools/clients/:clientId/fe-key` | `{ "privateKey": "-----BEGIN PRIVATE KEY-----..." }` |
| GET | `/api/pools/clients/:clientId/fe/status` | `{ feEnabled, hasKey, nif }` |
| DELETE | `/api/pools/clients/:clientId/fe-key` | Existing AGT docs stay |

## Examples

Create pool:

```bash
curl -X POST https://api.momenu.online/api/pools/claim-included \
  -H "Content-Type: application/json" \
  -H "x-api-key: ACCOUNT_KEY"
```

Add client:

```bash
curl -X POST https://api.momenu.online/api/pools/clients \
  -H "Content-Type: application/json" \
  -H "x-api-key: ACCOUNT_KEY" \
  -d '{
    "nif": "541723810",
    "company": {
      "name": "Padaria Central Lda",
      "address": "Rua 1, Luanda",
      "phone": "+244 923 000 000",
      "email": "geral@padaria.ao"
    }
  }'
```

Logo:

```bash
curl -X POST https://api.momenu.online/api/pools/clients/pc_xxx/logo \
  -H "x-api-key: ACCOUNT_KEY" \
  -F "file=@./logo.png"
```

Client key (give `data.key` to the client — shown once):

```bash
curl -X POST https://api.momenu.online/api/pools/clients/pc_xxx/keys \
  -H "Content-Type: application/json" \
  -H "x-api-key: ACCOUNT_KEY" \
  -d '{"label":"Produção"}'
```

Activate FE (client NIF, not the reseller):

```bash
curl -X POST https://api.momenu.online/api/pools/clients/pc_xxx/fe-key \
  -H "Content-Type: application/json" \
  -H "x-api-key: ACCOUNT_KEY" \
  -d '{"privateKey":"-----BEGIN PRIVATE KEY-----\nMIIE...\n-----END PRIVATE KEY-----"}'
```

## Rules

- Emitter identity (NIF, name, logo, IBAN) comes from the **client record**, never from the invoice body.
- Omitting `company.logo` on PATCH does not delete the logo.
- Soft-delete (`/remove`) keeps documents; keys stop working until `/recover`.
- Error shape: `{ "success": false, "error": "...", "code": "..." }`.
