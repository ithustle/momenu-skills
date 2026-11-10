---
name: mom-factura-testing
description: Test and debug Mom Factura Payment API integrations using QA environment and payment simulation. Use when setting up test environments, simulating payment outcomes (success, failure, timeout), debugging API errors, or validating integration before production.
license: MIT
metadata:
  author: mom-factura
  version: "1.1"
  language: pt
---

# Mom Factura Testing & QA

Test payment integrations without processing real transactions.

**Base URL:** `https://api.momenu.online`

## Enable QA Mode

Add header `x-env-qa: true` to use the test environment. Works from any origin.

```
Content-Type: application/json
x-api-key: YOUR_API_KEY
x-env-qa: true
```

## Simulate Payment Results

The `simulateResult` field in the MCX endpoint body triggers specific scenarios. Only active when `x-env-qa: true`.

| Value | Behavior |
|-------|----------|
| `success` | Payment succeeds, returns transactionId and invoiceUrl |
| `insufficient_balance` | Fails: client has no balance |
| `timeout` | Fails: no response from provider |
| `rejected` | Fails: payment explicitly rejected |
| `invalid_number` | Fails: phone not registered |

Internal test phone mapping (automatic, no action needed):
- success → 244900000000
- insufficient_balance → 244900000001
- timeout → 244900000002
- rejected → 244900000003
- invalid_number → 244999999999

## Test Examples

### Successful MCX Payment

```bash
curl -X POST https://api.momenu.online/api/payment/mcx \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-env-qa: true" \
  -d '{"paymentInfo":{"amount":1000,"phoneNumber":"244923456789"},"products":[{"id":"1","productName":"Artigo","productPrice":1000,"productQuantity":1}],"instantWithdraw":true,"simulateResult":"success"}'
```

### Insufficient Balance

```bash
curl -X POST https://api.momenu.online/api/payment/mcx \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-env-qa: true" \
  -d '{"paymentInfo":{"amount":5000,"phoneNumber":"244923456789"},"products":[{"id":"1","productName":"Artigo","productPrice":5000,"productQuantity":1}],"instantWithdraw":true,"simulateResult":"insufficient_balance"}'
```

### Amount Validation Error

```bash
curl -X POST https://api.momenu.online/api/payment/mcx \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-env-qa: true" \
  -d '{"paymentInfo":{"amount":2500,"phoneNumber":"244923456789"},"products":[{"id":"1","productName":"P","productPrice":3000,"productQuantity":1}],"simulateResult":"success"}'
```

Returns: `{ "success": false, "code": "AMOUNT_MISMATCH" }`

### E-kwanza in QA (no simulateResult support)

```bash
curl -X POST https://api.momenu.online/api/payment/ekwanza \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-env-qa: true" \
  -d '{"paymentInfo":{"amount":2000,"phoneNumber":"244923456789"},"products":[{"id":"1","productName":"Artigo","productPrice":2000,"productQuantity":1}]}'
```

### Stored payment state in QA

```bash
curl -X GET "https://api.momenu.online/api/payment/status/TRANSACTION_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-env-qa: true"
```

Returns the order already stored for that `transactionId`: `open`, `paid`, or `failed`. It does not confirm the payment.

### Reference in QA

```bash
curl -X POST https://api.momenu.online/api/payment/reference \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-env-qa: true" \
  -d '{"paymentInfo":{"amount":10000},"products":[{"id":"1","productName":"Artigo","productPrice":10000,"productQuantity":1}],"instantWithdraw":true}'
```

## Test Helpers

### JavaScript

```javascript
const API = "https://api.momenu.online";
const headers = {
  "Content-Type": "application/json",
  "x-api-key": "YOUR_API_KEY",
  "x-env-qa": "true"
};

async function testMCX(simulateResult = "success") {
  const res = await fetch(`${API}/api/payment/mcx`, {
    method: "POST", headers,
    body: JSON.stringify({
      paymentInfo: { amount: 1000, phoneNumber: "244923456789" },
      products: [{ id: "1", productName: "Artigo", productPrice: 1000, productQuantity: 1 }],
      instantWithdraw: true,
      simulateResult
    })
  });
  return res.json();
}

// Run all scenarios
for (const s of ["success","insufficient_balance","timeout","rejected","invalid_number"]) {
  testMCX(s).then(r => console.log(s, r.success ? "PASS" : "FAIL", r));
}
```

### Python

```python
import requests

API = "https://api.momenu.online"
headers = {"Content-Type": "application/json", "x-api-key": "YOUR_API_KEY", "x-env-qa": "true"}

def test_mcx(simulate="success"):
    r = requests.post(f"{API}/api/payment/mcx", json={
        "paymentInfo": {"amount": 1000, "phoneNumber": "244923456789"},
        "products": [{"id": "1", "productName": "Artigo", "productPrice": 1000, "productQuantity": 1}],
        "instantWithdraw": True,
        "simulateResult": simulate
    }, headers=headers)
    return r.json()

for s in ["success", "insufficient_balance", "timeout", "rejected", "invalid_number"]:
    result = test_mcx(s)
    print(f"{s}: {'PASS' if result.get('success') else 'FAIL'} - {result}")
```

### Marketplace split (MCX)

```bash
curl -X POST https://api.momenu.online/api/payment/mcx \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-env-qa: true" \
  -d '{
    "paymentInfo":{"amount":10000,"phoneNumber":"244923456789"},
    "products":[{"id":"1","productName":"Artigo","productPrice":10000,"productQuantity":1}],
    "instantWithdraw":true,
    "split":true,
    "iban":"AO06000000000000000000001",
    "isPercent":true,
    "value":10,
    "beneficiaryName":"Loja M1",
    "beneficiaryNif":"5417000000",
    "description":"Comissão de marketplace pela venda",
    "simulateResult":"success"
  }'
```

The usual MCX fields remain (`success`, `transactionId`, `invoiceUrl` when the PDF exists). Also expect `split.platformAmount = 980`, `split.beneficiaryAmount = 8820`. Invalid IBAN → `SPLIT_INVALID_IBAN`. 100 Kz at 10% → `SPLIT_LEG_BELOW_MINIMUM`. E-kwanza + split → `SPLIT_NOT_SUPPORTED`.

## Common Issues

| Error | Cause | Fix |
|-------|-------|-----|
| DOMAIN_NOT_ALLOWED | Origin not registered | Register the domain in the merchant panel |
| MISSING_API_KEY | Header missing | Check header is `x-api-key` (lowercase, hyphens) |
| AMOUNT_MISMATCH | amount != products total | Verify SUM(price*qty) == amount |
| MISSING_PRODUCTS | products omitted | Send at least one valid product |
| INSTANT_WITHDRAW_REQUIRED | instantWithdraw missing/false on MCX/Reference | Send `"instantWithdraw": true` |
| SPLIT_INVALID_IBAN | split:true with bad/missing IBAN | Angolan IBAN AO + 23 digits, valid checksum |
| SPLIT_LEG_BELOW_MINIMUM | a split share is < 50 Kz | Raise the amount or the commission so both legs are ≥ 50 |
| SPLIT_NOT_SUPPORTED | split on E-kwanza / POOL / PLAN | Use MCX or Reference |
| SPLIT_BENEFICIARY_REQUIRED | missing beneficiary name/NIF | Send beneficiaryName and a real beneficiaryNif |
| SPLIT_DESCRIPTION_REQUIRED | missing split description | Send description |
| FE_CONFIG_REQUIRED | electronic invoicing inactive | Activate AGT FE in the portal — no PDF fallback |
| MISSING_PHONE | No phone for MCX/E-kwanza | Add `paymentInfo.phoneNumber` (not needed for Reference) |
| simulateResult ignored | Not in QA mode | Add header `x-env-qa: true` |
| simulateResult ignored | Wrong endpoint | Only works on MCX, not E-kwanza or Reference |

## Pre-Production Checklist

1. Remove `x-env-qa: true` header
2. Remove `simulateResult` from request bodies
3. Register production domain in merchant panel
4. Store API key in environment variables (never hardcode)
5. Implement error handling for all error codes
6. Implement webhook endpoint to receive payment confirmations
7. Confirm Reference and MCX by webhook. Read the stored state with `GET /api/payment/status/:transactionId`. Confirm E-kwanza with `GET /api/payment/ekwanza/status/:code`
8. Verify amount validation with products
9. Test with real phone numbers
