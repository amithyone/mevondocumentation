# MevonPay Documentation

Official MevonPay API reference and **ready-to-run API collections** — no doc platform required.

**Repo:** [github.com/amithyone/mevondocumentation](https://github.com/amithyone/mevondocumentation)

## Quick start — test the API in 2 minutes

### Postman (easiest full experience)

1. Download [Postman](https://www.postman.com/downloads/)
2. Import from `testing/`:
   - `MevonPay.postman_collection.json`
   - `MevonPay.postman_environment.json`
3. Set `secret_key` and `base_url` in the **MevonPay Sandbox** environment
4. Send **Account → Balance**

**Step-by-step:** [`testing/GET_STARTED.md`](testing/GET_STARTED.md)

### Other tools (same collection)

| Tool | Why use it |
|------|------------|
| [Bruno](https://www.usebruno.com) | Free, lightweight — import Postman JSON |
| [Hoppscotch](https://hoppscotch.io) | Browser-only — import `testing/mevonpay-openapi.json` |
| Cursor REST Client | Open `testing/mevonpay-api.http` → **Send Request** |

## Documentation

| Path | What it is |
|------|------------|
| [`reference/mevonpay-api-reference.md`](reference/mevonpay-api-reference.md) | Complete Markdown API reference (969 lines) |
| [`testing/`](testing/) | Postman collection, OpenAPI, `.http` file |
| [`fern/`](fern/) | Optional Fern site config (ignore if not using Fern) |

## What’s covered

21 outbound endpoints + webhooks: virtual accounts, transfers, payouts, TSQ, FX, virtual dollar cards, VTU (airtime/data/electricity/cable/betting).

## Changelog

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | May 2026 | Initial integrated reference from CheckoutPay production code |
