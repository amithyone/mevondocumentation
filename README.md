# MevonPay Documentation

Official-style MevonPay API documentation — virtual accounts, transfers, FX, virtual dollar cards, VTU bill payments, and webhooks.

**Live site (after Fern setup):** [mevonpay.docs.buildwithfern.com](https://mevonpay.docs.buildwithfern.com)

## Repository layout

| Path | Purpose |
|------|---------|
| `fern/` | Fern docs site (OpenAPI + MDX guides) — **this is what Fern publishes** |
| `reference/mevonpay-api-reference.md` | Full Markdown reference (source of truth) |
| `testing/` | Postman collection + REST Client `.http` file |

## Connect to Fern (one-time)

1. Open [Fern Dashboard](https://dashboard.buildwithfern.com).
2. Create or select the **mevonpay** organization.
3. **Connect this GitHub repo:** `amithyone/mevondocumentation`
4. Set the docs URL in `fern/docs.yml` if needed (`mevonpay.docs.buildwithfern.com`).
5. Add GitHub repository secret **`FERN_TOKEN`** (API key from Fern Dashboard → Settings → API keys).

## Publish

**Automatic:** Push to `main` → GitHub Actions runs `fern generate --docs`.

**Manual (local):**

```bash
npm install -g fern-api
fern login          # or export FERN_TOKEN=...
cd fern && fern check
fern generate --docs --preview   # preview URL
fern generate --docs             # production
```

## Local preview

```bash
npx fern-api docs dev
```

Runs a local server with hot reload (default http://localhost:3000).

## API testing

Import `testing/MevonPay.postman_collection.json` + `testing/MevonPay.postman_environment.json` into Postman, Bruno, or Insomnia.

In Cursor/VS Code, open `testing/mevonpay-api.http` and use **Send Request** (REST Client extension).

## MCP (Cursor / AI tools)

After publishing, connect Cursor to the docs MCP server:

```json
"mevonpay.docs.buildwithfern.com": {
  "url": "https://mevonpay.docs.buildwithfern.com/_mcp/server",
  "headers": {}
}
```

## Changelog

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | May 2026 | Initial integrated reference (21 API endpoints + webhooks) from CheckoutPay production code |
