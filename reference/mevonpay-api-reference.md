# MevonPay API Reference

**Version 1.0** · May 2026  
**Publisher:** Metravon Innovation Limited (MevonPay)

---

## Documentation feedback note

This reference was produced from live production integrations across virtual accounts, bank transfers, currency exchange, virtual dollar cards, bill payments (VTU), and webhooks. MevonPay recommends standardizing the following across all public documentation:

| Topic | Current integration reality |
|-------|----------------------------|
| Authentication | Three styles in use: raw secret, Bearer, and Token — see [Authentication](#authentication) |
| Success flags | `success`, `status`, or `code` at top level or nested under `data` |
| Transfer responses | Alternate envelope using `responseCode` / `responseMessage` instead of `success` |
| Field naming | Both camelCase and snake_case accepted on several endpoints (documented per endpoint) |
| Event field | Webhooks may send `event`, `eventType`, `type`, or `action` |

---

## Introduction

MevonPay provides REST-style JSON APIs for Nigerian fintech integrations:

- **Virtual accounts** — temporary, dynamic, and permanent Rubies accounts
- **Bank transfers & payouts** — NIP transfers from merchant float or customer virtual accounts
- **Currency exchange** — NGN ↔ USD wallet conversion
- **Virtual dollar cards** — issue, fund, freeze, withdraw, and transaction history
- **Bill payments (VTU)** — airtime, data, electricity, cable TV, and betting
- **Webhooks** — real-time funding and card lifecycle notifications

### Base URL

All endpoints use your assigned environment base URL:

| Environment | Example |
|-------------|---------|
| Production | `https://api.mevonpay.com` |
| Sandbox | `https://sandbox-api.mevonpay.com` |

Append the path (e.g. `/V1/balance`) to your base URL.

### Conventions

- **Method:** `POST` for all API calls unless stated otherwise
- **Content-Type:** `application/json`
- **Accept:** `application/json`
- **Bank codes:** 6-digit NIP format (e.g. `000058` for GTBank)
- **Amounts:** NGN amounts are numeric; VTU buy amounts are typically integer naira
- **References:** Must be unique per transaction; merchants generate their own reference strings

---

## Authentication

MevonPay supports multiple authorization header formats. Use the style documented for each endpoint.

| Style | Header format | Used on |
|-------|---------------|---------|
| **Raw secret** | `Authorization: {SECRET_KEY}` | Balance, exchange, createtransfer, bank_service, virtual accounts, most card endpoints |
| **Bearer** | `Authorization: Bearer {SECRET_KEY}` | VTU (electricity, cable, airtime, data, betting), TSQ default, card fallback |
| **Token** | `Authorization: Token {SECRET_KEY}` | `/V1/payout` |

### Transfer and payout password

Endpoints that move money (`/V1/createtransfer`, `/V1/payout`) require an additional field:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `currentPassword` | string | Yes | Account transaction password configured on your MevonPay merchant profile |

### Webhook authentication (inbound to merchant)

When MevonPay POSTs events to your server, merchants may validate:

| Method | Description |
|--------|-------------|
| Bearer secret | `Authorization: Bearer {WEBHOOK_SECRET}` |
| IP allowlist | Optional restriction to MevonPay egress IPs |
| Domain allowlist | Optional restriction on `Origin` / `Referer` headers |

---

## Common response envelope

### Standard JSON API response

Most endpoints return:

```json
{
  "success": true,
  "message": "Operation completed successfully",
  "data": { }
}
```

MevonPay also accepts these as success indicators (integrations check all):

| Field | Success values |
|-------|----------------|
| `success` | `true`, `1`, `"success"` |
| `status` | `true`, `1`, `"success"` |
| `code` | `"success"` |
| `data.success` | `true`, `1` (nested) |

Message text may appear as `message` or `msg`.

### Transfer / payout / TSQ response envelope

Bank transfer endpoints often return:

```json
{
  "responseCode": "00",
  "responseMessage": "Transfer successful",
  "status": true,
  "reference": "MERCHANT-REF-001"
}
```

Nested detail may appear under `details`:

```json
{
  "status": "success",
  "details": {
    "responseCode": "00",
    "responseMessage": "Verification complete",
    "transactionStatus": "successful"
  }
}
```

---

## Response codes (transfers, payouts, TSQ)

| Code | Meaning |
|------|---------|
| `00` | Successful — transfer completed |
| `09` | Pending — awaiting final status |
| `90` | Pending — processing |
| `99` | Pending — queued |
| Other | Failed or requires investigation |

Additional success signals integrations honor:

- `transactionStatus`: `success`, `successful`, `completed`
- `responseMessage` containing `transfer successful` or `verification complete`
- HTTP `2xx` with empty body (treated as success on some transfer paths)

---

## 1. Account balance

Retrieve live NGN and USD wallet balances.

| | |
|---|---|
| **Path** | `POST /V1/balance` |
| **Auth** | Raw secret |
| **Body** | `{}` (empty JSON object) |

### Success response (`data`)

| Field | Type | Aliases | Description |
|-------|------|---------|-------------|
| `bal` | number | `naira_balance` | Available NGN balance |
| `usd_balance` | number | — | Available USD balance |
| `ledger_bal` | number | `naira_ledger` | NGN ledger balance |
| `usd_ledger_bal` | number | `usd_ledger` | USD ledger balance |

### Example

```json
{
  "success": true,
  "message": "Balance retrieved",
  "data": {
    "bal": 1250000.50,
    "usd_balance": 842.30,
    "ledger_bal": 1248000.00,
    "usd_ledger_bal": 840.00
  }
}
```

---

## 2. Temporary virtual account

Create a reusable temporary Rubies virtual account tied to a customer identity.

| | |
|---|---|
| **Path** | `POST /V1/createtempva.php` |
| **Auth** | Raw secret |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Account type — use `"rubies"` |
| `fname` | string | Yes | Customer first name |
| `lname` | string | Yes | Customer last name |
| `registration_number` | string | Conditional | Preferred identity number |
| `bvn` | string | Conditional | Legacy fallback if `registration_number` omitted |

### Success response (`data`)

| Field | Aliases |
|-------|---------|
| `account_number` | `accountNumber` |
| `account_name` | `accountName` |
| `bank_name` | — |
| `bank_code` | `bankCode` |

### Example

```json
{
  "status": true,
  "message": "Account created successfully",
  "data": {
    "account_number": "8880324727",
    "account_name": "JOHN DOE",
    "bank_name": "Rubies MFB",
    "bank_code": "000023"
  }
}
```

---

## 3. Dynamic virtual account

Create an amount-specific virtual account that expires after payment or timeout.

| | |
|---|---|
| **Path** | `POST /V1/createdynamic` |
| **Auth** | Raw secret |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | number | Yes | Expected payment amount |
| `currency` | string | Yes | Currency code — `"NGN"` |

### Success response (`data`)

| Field | Aliases |
|-------|---------|
| `accountNumber` | `account_number` |
| `accountName` | `account_name` |
| `bankName` | `bank_name` |
| `bankCode` | `bank_code` |
| `expiresOn` | — |

---

## 4. Rubies permanent virtual account

Create a permanent Rubies MFB account (personal or business).

| | |
|---|---|
| **Path** | `POST /V1/createrubies` |
| **Auth** | Raw secret |

### 4.1 Personal account

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | Yes | `"create"` |
| `account_type` | string | Yes | `"personal"` |
| `fname` | string | Yes | First name |
| `lname` | string | Yes | Last name |
| `phone` | string | Yes | 11-digit local phone |
| `dob` | string | Yes | `YYYY-MM-DD` |
| `email` | string | Yes | Valid email |
| `bvn` | string | Conditional | 11-digit BVN |
| `nin` | string | Conditional | 11-digit NIN (if BVN not supplied) |

### 4.2 Business account

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | Yes | `"create"` |
| `account_type` | string | Yes | `"business"` |
| `cac` | string | Yes | CAC / RC registration number |
| `phone` | string | Yes | 11-digit local phone |
| `dob` | string | Yes | `YYYY-MM-DD` |
| `email` | string | Yes | Business contact email |

### Success response (`data`)

| Field | Description |
|-------|-------------|
| `reference` | Provider reference for the account |
| `account_number` | Permanent VA number |
| `account_name` | Account display name |
| `bank_name` | `"Rubies MFB"` |
| `bank_code` | Bank NIP code |

---

## 5. Bank service

Unified endpoint for bank list and account name enquiry.

| | |
|---|---|
| **Path** | `POST /V1/bank_service` |
| **Auth** | Raw secret |

### 5.1 Get bank list

```json
{ "action": "getBankList" }
```

**Response `data`:** Array of bank objects with `code`, `name`, and related fields.

### 5.2 Name enquiry

| Field | Type | Required |
|-------|------|----------|
| `action` | string | Yes — `"nameEnquiry"` |
| `bankCode` | string | Yes |
| `accountNumber` | string | Yes |

**Response `data`:**

| Field | Aliases |
|-------|---------|
| `account_name` | `accountName`, `AccountName` |
| `account_number` | `accountNumber` |
| `bank_code` | `bankCode` |

---

## 6. Create transfer

Initiate an NGN bank transfer from your MevonPay merchant debit account.

| | |
|---|---|
| **Path** | `POST /V1/createtransfer` |
| **Auth** | Raw secret |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | number | Yes | Transfer amount in NGN |
| `bankCode` | string | Yes | 6-digit destination bank code |
| `bankName` | string | Yes | Destination bank name |
| `creditAccountName` | string | Yes | Beneficiary account name |
| `creditAccountNumber` | string | Yes | Beneficiary account number |
| `debitAccountName` | string | Yes | Your MevonPay debit account name |
| `debitAccountNumber` | string | Yes | Your MevonPay debit account number |
| `narration` | string | No | Transfer description |
| `reference` | string | Yes | Unique merchant reference |
| `sessionId` | string | No | Optional session identifier |
| `currentPassword` | string | Yes | Transaction password |

### Example response

```json
{
  "responseCode": "00",
  "responseMessage": "Transfer successful",
  "status": true,
  "reference": "WD-20260529-001"
}
```

---

## 7. Payout

Debit a specific virtual account and pay out to an external bank account. Used when funds originate from a customer's permanent Rubies VA.

| | |
|---|---|
| **Path** | `POST /V1/payout` |
| **Auth** | Token |

### Request

Same fields as [Create transfer](#6-create-transfer), with explicit debit account per payout:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `debitAccountNumber` | string | Yes | Source VA to debit |
| `debitAccountName` | string | Yes | Source VA account name |

All other fields match createtransfer. Response envelope and codes are identical.

---

## 8. Transaction status query (TSQ)

Query the final status of a transfer or payout by reference.

| | |
|---|---|
| **Path** | `POST /V1/tsk` |
| **Auth** | Bearer (default), Token, or Raw — configurable |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reference` | string | Yes | Original transfer reference |
| `payoutApi` | string | No | Optional API discriminator |

### Example response

```json
{
  "status": "success",
  "message": "Status retrieved",
  "reference": "WD-20260529-001",
  "details": {
    "responseCode": "00",
    "responseMessage": "Verification complete",
    "transactionStatus": "successful"
  }
}
```

---

## 9. Currency exchange

Convert between NGN and USD within your MevonPay wallet.

| | |
|---|---|
| **Path** | `POST /V1/exchange` |
| **Auth** | Raw secret |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | number | Yes | Amount to convert (> 0) |
| `from_currency` | string | Yes | Source currency — `"NGN"` or `"USD"` |
| `to_currency` | string | Yes | Target currency |

### Success response (`data`)

| Field | Aliases | Description |
|-------|---------|-------------|
| `rate` | `exchange_rate`, `usd_ngn_rate` | Exchange rate applied |
| `converted_amount` | `usd_amount` | Result amount in target currency |

### Example (NGN → USD rate lookup)

```json
{
  "success": true,
  "message": "Exchange rate retrieved",
  "data": {
    "rate": 1378.08,
    "converted_amount": 1.00
  }
}
```

Request: `{ "amount": 1, "from_currency": "NGN", "to_currency": "USD" }`

---

## 10. Virtual dollar card — create

Issue a new USD virtual card.

| | |
|---|---|
| **Path** | `POST /V1/card_request` |
| **Auth** | Raw secret (Bearer accepted as fallback) |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | Yes | `"create"` |
| `amount` | number | Yes | Initial USD load amount |
| `firstName` | string | Yes | Cardholder first name |
| `lastName` | string | Yes | Cardholder last name |
| `email` | string | Yes | Cardholder email |
| `phoneNumber` | string | Yes | Phone number |
| `dob` | string | Yes | Date of birth |
| `homeNumber` | string | Yes | Home / house number |
| `homeAddress` | string | Yes | Full home address |
| `cardName` | string | Yes | Name printed on card |

Card creation is asynchronous — MevonPay sends a [card.created webhook](#card-lifecycle-webhooks) when the card is ready.

### Success response (`data`)

| Field | Aliases | Description |
|-------|---------|-------------|
| `reference` | `request_id` | Track this reference for webhooks |
| `card_id` | `card_code` | Provider card identifier |

---

## 11. Virtual dollar card — top up

Fund an existing virtual card with USD.

| | |
|---|---|
| **Path** | `POST /V1/card_topup` |
| **Auth** | Raw secret (Bearer fallback) |

### Request

| Field | Type | Required |
|-------|------|----------|
| `amount` | number | Yes — USD amount |
| `card_code` | string | Yes — Provider card code |

---

## 12. Virtual dollar card — withdraw

Withdraw USD from a virtual card back to merchant float.

| | |
|---|---|
| **Path** | `POST /V1/card_withdraw` |
| **Auth** | Raw secret |

### Request

| Field | Type | Required |
|-------|------|----------|
| `amount` | number | Yes |
| `card_code` | string | Yes |
| `reason` | string | No — default `"Withdrawal to Wallet"` |

---

## 13. Virtual dollar card — freeze / unfreeze

| | |
|---|---|
| **Path** | `POST /V1/card_status` |
| **Auth** | Raw secret |

### Request

| Field | Type | Required |
|-------|------|----------|
| `action` | string | Yes — `"freeze"` or `"unfreeze"` |
| `card_code` | string | Yes |

---

## 14. Virtual dollar card — balance

| | |
|---|---|
| **Path** | `POST /V1/card_balance` |
| **Auth** | Raw secret (Bearer fallback) |

### Request

| Field | Type | Required |
|-------|------|----------|
| `request_id` | string | Yes — Card request reference from creation |

---

## 15. Virtual dollar card — transactions

Retrieve merchant spend and decline history for a card.

| | |
|---|---|
| **Path** | `POST /V1/card_transactions` |
| **Auth** | Raw secret (Bearer fallback) |

### Request

| Field | Type | Required |
|-------|------|----------|
| `card_code` | string | Yes |

---

## 16. Virtual dollar card — details

Retrieve full card credentials (PAN, expiry, CVV) after activation.

| | |
|---|---|
| **Path** | `POST /V1/card_details` |
| **Auth** | Raw secret (Bearer fallback) |

### Request

| Field | Type | Required |
|-------|------|----------|
| `card_id` | string | Yes |
| `card_code` | string | Yes |

---

## 17. Electricity (VTU)

| | |
|---|---|
| **Path** | `POST /V1/electricity` |
| **Auth** | Bearer |

### Actions

| Action | Description |
|--------|-------------|
| `getInfo` | List providers and plans |
| `verify` | Validate meter before purchase |
| `buy` | Purchase electricity token |

### Verify request

| Field | Required |
|-------|----------|
| `action` | `"verify"` |
| `meter` | Yes |
| `providerCode` | Yes |
| `planCode` | Yes |

### Buy request

| Field | Required |
|-------|----------|
| `action` | `"buy"` |
| `meter` | Yes |
| `providerCode` | Yes |
| `planCode` | Yes |
| `amount` | Yes — integer NGN |
| `customerName` | Yes |
| `phone` | Yes — 11-digit |

### Verify response (`data`)

| Field | Aliases |
|-------|---------|
| `customerName` | `customer_name` |
| `customerId` | `customer_id` |
| `dueDate` | `due_date` |

---

## 18. Cable TV (VTU)

| | |
|---|---|
| **Path** | `POST /V1/cabletv` |
| **Auth** | Bearer |

### Verify request

| Field | Required |
|-------|----------|
| `action` | `"verify"` |
| `smartcard` | Yes |
| `providerCode` | Yes |
| `planCode` | Yes |

### Buy request

| Field | Required |
|-------|----------|
| `action` | `"buy"` |
| `smartcard` | Yes |
| `providerCode` | Yes |
| `planCode` | Yes |
| `amount` | Yes |
| `customerName` | Yes |
| `phone` | Yes |

---

## 19. Airtime (VTU)

| | |
|---|---|
| **Path** | `POST /V1/airtime` |
| **Auth** | Bearer |

### Buy request

| Field | Required | Notes |
|-------|----------|-------|
| `action` | Yes | `"buy"` |
| `providerCode` | Yes | e.g. `mtn`, `glo`, `airtel`, `9mobile` |
| `phone` | Yes | 11-digit |
| `amount` | Yes | Integer NGN (min ₦50, max ₦50,000) |
| `reference` | Yes | Unique merchant reference |

---

## 20. Data (VTU)

| | |
|---|---|
| **Path** | `POST /V1/data` |
| **Auth** | Bearer |

### Buy request

| Field | Required |
|-------|----------|
| `action` | `"buy"` |
| `providerCode` | Yes |
| `phone` | Yes |
| `planCode` | Yes |
| `amount` | Yes |
| `reference` | Yes — unique |

---

## 21. Betting (VTU)

| | |
|---|---|
| **Path** | `POST /V1/betting` |
| **Auth** | Bearer |

### Verify request

| Field | Required |
|-------|----------|
| `action` | `"verify"` |
| `providerCode` | Yes |
| `customerId` | Yes |

### Buy request

| Field | Required |
|-------|----------|
| `action` | `"buy"` |
| `providerCode` | Yes |
| `customerId` | Yes |
| `amount` | Yes |
| `phone` | Yes |
| `reference` | Yes — unique |

---

## 22. Webhooks (MevonPay → merchant)

MevonPay POSTs JSON events to the webhook URL you configure on your merchant dashboard.

**Recommended URL pattern:** `https://{your-domain}/api/v1/webhook/mevonpay`

### Webhook request headers

| Header | Value |
|--------|-------|
| `Content-Type` | `application/json` |
| `Authorization` | `Bearer {WEBHOOK_SECRET}` (if configured) |

### Event envelope

```json
{
  "event": "event.name",
  "data": { }
}
```

**Event field aliases MevonPay may use:** `event`, `eventType`, `type`, `action`, or nested `data.event` / `data.type`.

---

### Funding webhook — `funding.success`

Sent when a virtual account receives an inbound bank transfer.

```json
{
  "event": "funding.success",
  "data": {
    "account_number": "8880324727",
    "amount": 2575.00,
    "reference": "100004260518170536160223787208",
    "sender": "DANIEL DAVID JOSEPH",
    "bank_name": "OPAY",
    "narration": "Payment for order #1234",
    "timestamp": "2026-05-18 18:05:41.247"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `account_number` | string | Yes | Credited VA number |
| `amount` | number | Yes | Credited amount (> 0) |
| `reference` | string | No | Bank / provider reference |
| `sender` | string | No | Payer name |
| `bank_name` | string | No | Source bank |
| `narration` | string | No | Payment narration |
| `timestamp` | string | No | Event timestamp |

**Merchant response:** Return HTTP `200` with `{ "success": true }`.

---

### Card lifecycle webhooks

#### Card created — `card.created`

Sent when a virtual card is issued and ready for use.

```json
{
  "event": "card.created",
  "data": {
    "reference": "766f5cdb-9956-4cec-af77-b520f624acc3",
    "card_id": "bab449bb-15e9-404a-aa73-657519df4794",
    "email": "customer@example.com",
    "card_number": "4288520141503096",
    "last4": "3096",
    "expiry": "06/2029",
    "cvv": "486",
    "balance": 5.00,
    "card_name": "JOHN DOE"
  }
}
```

**Accepted event names:**

`card.created`, `card.created.success`, `card_created`, `card.create`, `virtual_card.created`, `virtual_card_created`, `card.success`, `card.ready`, `card.active`, `card_creation.success`

**Matching fields:** `data.reference`, `data.card_id`, `data.card_code`, `data.email`, `data.phone_number`

#### Card top-up — `card.topup.success`

```json
{
  "event": "card.topup.success",
  "data": {
    "card_code": "VCARD2026060611150700359",
    "last4": "3096",
    "new_balance": 138.00,
    "reference": "766f5cdb-9956-4cec-af77-b520f624acc3",
    "timestamp": "2026-06-10T10:39:29+01:00"
  }
}
```

**Accepted event names:** `card.topup.success`, `card.topup`, `card_topup`, `virtual_card.topup`

#### Card spend — `card.spend.success`

```json
{
  "event": "card.spend.success",
  "data": {
    "card_code": "VCARD2026060611150700359",
    "code": "TXN-SPEND-001",
    "reference": "ref-spend-001",
    "amount": 37.00,
    "drcr": "DR",
    "category": "Card Purchase",
    "description": "Netflix.com",
    "balance": 138.00,
    "createdOn": "2026-06-10T10:39:29+01:00"
  }
}
```

**Accepted event names:** `card.spend`, `card.spend.success`, `card.transaction`, `card.debit`, `card_debit`, `virtual_card.spend`

---

## Error handling

| HTTP status | Meaning |
|-------------|---------|
| `2xx` | Request accepted — check body for business-level success |
| `4xx` | Validation or authentication error |
| `5xx` | Provider error — retry with backoff |

### Error response example

```json
{
  "success": false,
  "status": false,
  "message": "Insufficient balance",
  "code": "INSUFFICIENT_FUNDS"
}
```

---

## Idempotency and reliability

1. **Unique references** — Every transfer, VTU purchase, and payout must use a unique `reference`. Reusing a reference may return the original result or fail validation.

2. **Webhook retries** — MevonPay retries failed webhook deliveries. Merchants should:
   - Return `200` for successfully processed events
   - Return `200` for duplicate events (idempotent handling)
   - Deduplicate by `reference` or event ID

3. **Pending transfers** — Poll `/V1/tsk` when `responseCode` is `09`, `90`, or `99`.

4. **TSQ timing** — Query status after 30–60 seconds for pending transfers; do not poll more frequently than once every 10 seconds.

---

## Quick reference

| # | Method | Path | Auth | Purpose |
|---|--------|------|------|---------|
| 1 | POST | `/V1/balance` | Raw | Wallet balances |
| 2 | POST | `/V1/createtempva.php` | Raw | Temporary VA |
| 3 | POST | `/V1/createdynamic` | Raw | Dynamic VA |
| 4 | POST | `/V1/createrubies` | Raw | Permanent Rubies VA |
| 5 | POST | `/V1/bank_service` | Raw | Bank list / name enquiry |
| 6 | POST | `/V1/createtransfer` | Raw | Merchant transfer |
| 7 | POST | `/V1/payout` | Token | VA debit payout |
| 8 | POST | `/V1/tsk` | Bearer | Transfer status |
| 9 | POST | `/V1/exchange` | Raw | NGN ↔ USD |
| 10 | POST | `/V1/card_request` | Raw | Create virtual card |
| 11 | POST | `/V1/card_topup` | Raw | Fund card |
| 12 | POST | `/V1/card_withdraw` | Raw | Withdraw from card |
| 13 | POST | `/V1/card_status` | Raw | Freeze / unfreeze |
| 14 | POST | `/V1/card_balance` | Raw | Card balance |
| 15 | POST | `/V1/card_transactions` | Raw | Card spend history |
| 16 | POST | `/V1/card_details` | Raw | Card PAN / CVV |
| 17 | POST | `/V1/electricity` | Bearer | Electricity VTU |
| 18 | POST | `/V1/cabletv` | Bearer | Cable TV VTU |
| 19 | POST | `/V1/airtime` | Bearer | Airtime VTU |
| 20 | POST | `/V1/data` | Bearer | Data VTU |
| 21 | POST | `/V1/betting` | Bearer | Betting VTU |
| 22 | POST | *(merchant webhook URL)* | Bearer | Inbound events |

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | May 2026 | Initial comprehensive reference from production integrations |

---

## Contact

**MevonPay** · Metravon Innovation Limited  
Website: [https://www.mevonpay.com.ng](https://www.mevonpay.com.ng)  
Support: integration-support@mevonpay.com.ng

For sandbox credentials and webhook URL configuration, contact your MevonPay account manager.
