# MevonPay API — Get started (no Fern needed)

Use **Postman**, **Bruno**, or **Hoppscotch** to explore and call every MevonPay endpoint. Full field reference: [`../reference/mevonpay-api-reference.md`](../reference/mevonpay-api-reference.md).

## Files in this folder

| File | Use with |
|------|----------|
| `MevonPay.postman_collection.json` | Postman, Bruno, Insomnia |
| `MevonPay.postman_environment.json` | Postman (variables template) |
| `mevonpay-openapi.json` | Hoppscotch, Insomnia, Swagger UI |
| `mevonpay-api.http` | Cursor / VS Code REST Client |

---

## Option 1 — Postman (recommended)

1. Install [Postman](https://www.postman.com/downloads/) (free tier is fine).
2. **Import → File** → select both:
   - `MevonPay.postman_collection.json`
   - `MevonPay.postman_environment.json`
3. Top-right: choose environment **MevonPay Sandbox**.
4. Edit environment variables:
   - `base_url` — your MevonPay API base (sandbox or production)
   - `secret_key` — your MevonPay secret key
   - `current_password` — transaction password (for transfers/payouts)
   - `debit_account_name` / `debit_account_number` — merchant debit account
5. Run **Account → Balance** first (read-only, safe test).
6. Use folders: Virtual Accounts, Transfers, Virtual Cards, VTU.

**Publish as docs (optional):** In Postman, open the collection → **View documentation** → **Publish** — gives you a shareable doc URL without Fern.

**Auth reminder:** The collection sets the correct header per request:
- Raw secret — most endpoints
- Bearer — VTU + TSQ
- Token — `/V1/payout`

---

## Option 2 — Bruno (simpler than Postman)

Free, open-source, no account required: [usebruno.com](https://www.usebruno.com)

1. Install Bruno.
2. **Import Collection → Postman Collection** → `MevonPay.postman_collection.json`
3. Create an environment in Bruno with the same variables as the Postman environment file.
4. Send requests from the sidebar.

---

## Option 3 — Hoppscotch (browser, zero install)

1. Open [hoppscotch.io](https://hoppscotch.io)
2. **Import → OpenAPI** → upload `mevonpay-openapi.json`
3. Set base URL and Authorization header manually per request (OpenAPI lists all paths).

---

## Option 4 — Cursor / VS Code (fastest if you’re already here)

1. Install extension **REST Client** (Huachao Mao).
2. Open `mevonpay-api.http`.
3. Edit variables at the top (`@secretKey`, `@baseUrl`, …).
4. Click **Send Request** above any `###` block.

---

## Safe testing order

1. **Balance** — read-only  
2. **Bank service → Get Bank List** — read-only  
3. **VTU → getInfo** (airtime, data, etc.) — read-only catalog  
4. **Name enquiry / verify** — validation only  
5. **Transfer / payout / VTU buy** — moves real money; use sandbox only  

## Need help?

See the full reference: [`reference/mevonpay-api-reference.md`](../reference/mevonpay-api-reference.md)
