# ASTON OS Omni-Matrix Apex 8.1.2 — Cloudflare-ready package

This package fixes the missing Worker entry point and includes an authenticated live diagnostic route for D1, KV, and R2.

## Repository layout
- `src/index.js` — Worker entry point and route handlers.
- `wrangler.toml` — Wrangler configuration.
- `schema.sql` — D1 schema.
- `test/run.mjs` — local smoke tests.

## Deploy from GitHub / Cloudflare Workers Builds
In the Cloudflare build configuration:
- **Root directory:** repository root (the folder containing `wrangler.toml`)
- **Build command:** leave empty; this Worker does not need a build step
- **Deploy command:** `npm run deploy` or `npx wrangler deploy`
- **Dependencies:** `npm install`

The file `src/index.js` is present at the exact path declared in `wrangler.toml`.

## Required pre-deploy review
Open `wrangler.toml` and verify the D1 database IDs, KV namespace ID, and R2 bucket name match the resources in the Cloudflare account that will receive this deployment. Resource names and IDs cannot be verified from this ZIP alone.

Configure secrets in Cloudflare (never commit secret values to Git):
```bash
npx wrangler secret put ASTON_API_TOKEN
npx wrangler secret put ASTON_JWT_SECRET
```

The API token is required for authenticated routes. For storage diagnostics, authenticate using the API token as Bearer and request admin access through that token:
```bash
curl -X POST "https://YOUR-WORKER-URL/v1/diagnostics/storage" \
  -H "Authorization: Bearer YOUR_ASTON_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}'
```
The diagnostic route writes uniquely named temporary values to D1 (schema/query only), KV, and R2, reads KV/R2 back, then deletes its temporary KV key and R2 object. It requires a valid API token and the configured bindings. Do not paste tokens into chat or commit them to GitHub.

## Validation
```bash
npm install
npm test
npm run check
npm run deploy
```
After deployment:
1. `GET /health?deep=1` checks Worker and D1 query.
2. `POST /v1/diagnostics/storage` checks D1, KV, and R2 end-to-end.
3. Check the returned `diagnostics` object. Each service should return `"ok": true`.

## Included endpoints
- `GET /`, `GET /health`, `GET /health?deep=1`
- `POST /v1/diagnostics/storage` (authenticated live D1/KV/R2 check)
- `POST /v1/ai/chat`
- `GET/POST /v1/records/:sector`
- `POST /v1/ingest/:sector`
- `POST /v1/search`
- `GET/POST/DELETE /v1/session`
- `POST /v1/ledger`
- `GET /v1/stats`
- `GET /v1/audit`

## Important implementation limitations
- `ASTON_DB_BACKUP` is configured but automatic D1 replication/failover is not implemented here.
- R2 diagnostics verify temporary object writes and reads; they are not a backup/replication pipeline.
- Vectorize and Queues are optional and disabled until the resources exist.
- `/v1/auth/token` does not mint JWTs in this build; use the API token as a Bearer token. JWT verification is supported for compatible HS256 JWTs, but issuance is not provided.
- The Durable Object rate limiter is present but is not yet applied as middleware to every route.
- A successful deployment does not by itself certify production security, payment handling, or regulatory compliance.

## Troubleshooting
- Missing `src/index.js`: ensure `src/index.js` is committed and Cloudflare's root directory is the repository root.
- `D1_ERROR: no such table`: run `npm run db:init` or call the relevant route; the Worker creates the schema for supported operations.
- Binding unavailable: compare the binding names in `wrangler.toml` with the Cloudflare dashboard.
- Unauthorized: set `ASTON_API_TOKEN` as a Worker secret and send it in the Authorization header.

Private — ASTON OS
