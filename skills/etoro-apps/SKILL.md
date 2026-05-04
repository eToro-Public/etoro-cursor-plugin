---
name: etoro-apps
description: For developers building apps and integrations on top of the eToro API — covers OAuth SSO authentication flows, common patterns for trading bots, portfolio dashboards, and social trading tools. Use when starting a new eToro-backed app or extending an existing one with new endpoint coverage.
---

https://api-portal.etoro.com/
## About

This skill guides you through building applications that integrate with the eToro Public API, covering authentication setup, common app patterns, and best practices.

## OAuth SSO Integration

### Overview

eToro supports **"Login with eToro"** via OpenID Connect with PKCE. This allows third-party apps to authenticate users and access the eToro API on their behalf.

### OAuth Flow

1. **Redirect user** to `https://www.etoro.com/sso/` with PKCE challenge parameters.
2. **User authenticates** on eToro's side.
3. **eToro redirects back** to your `redirect_uri` with an authorization `code`.
4. **Exchange code for tokens** via `POST https://www.etoro.com/sso/oidc/token`:
   - `grant_type=authorization_code`
   - `code=<auth_code>`
   - `redirect_uri=<callback_url>`
   - `code_verifier=<pkce_verifier>`
   - `Authorization: Basic <base64(client_id:client_secret)>` header
5. **Response** contains:
   - `access_token` — Bearer token for API calls (~2130 chars, JWT)
   - `id_token` — JWT with user identity (`sub` claim = 128-char encoded user ID)
   - `token_type`: `"Bearer"`
   - `expires_in`: (varies)

### Using the Token

All API requests use the same base URL and endpoints regardless of auth method:

```bash
curl -X GET "https://public-api.etoro.com/api/v1/watchlists" \
  -H "x-request-id: <UUID>" \
  -H "Authorization: Bearer <access_token>"
```

### Auth Method Selection in Code

```typescript
if (ctx.accessToken) {
  // SSO auth — Bearer token from OAuth token exchange
  headers['Authorization'] = `Bearer ${ctx.accessToken}`;
} else {
  // Manual API key auth
  headers['x-api-key'] = ctx.apiKey;
  headers['x-user-key'] = ctx.userKey;
}
```

## Common App Patterns

### Trading Bot

1. Authenticate via OAuth SSO or API keys.
2. Resolve instrument IDs: `GET /market-data/search?internalSymbolFull=<SYMBOL>`.
3. Enrich with metadata: `GET /market-data/instruments?instrumentIds=<id>` to get `instrumentDisplayName`, `symbolFull`, `images[]`.
4. Monitor prices via `GET /market-data/instruments/rates?instrumentIds=<ids>`.
5. Execute trades via `POST /trading/execution/demo/market-open-orders/by-amount`. 
6. Monitor portfolio via `GET /trading/info/demo/pnl` — response is `{ clientPortfolio: { credit, positions[], mirrors[], orders[], ordersForOpen[] } }`.
7. Close positions via `POST /trading/execution/demo/market-close-orders/positions/{positionID}`.

### Portfolio Dashboard

1. Authenticate the user.
2. Fetch portfolio and PnL: `GET /trading/info/demo/pnl` (or `/real/pnl`).
   - Response: `{ clientPortfolio }` containing `credit`, `positions[]`, `mirrors[]`, `orders[]`, `ordersForOpen[]`.
   - **Identifier fields inside these arrays are capital-suffix** (`positionID`, `instrumentID`, `mirrorID`, `parentPositionID`, `orderID`, `CID`) — model them exactly as returned.
   - Collect ALL positions: `[...clientPortfolio.positions, ...clientPortfolio.mirrors.flatMap(m => m.positions)]`.
3. Enrich with instrument metadata: `GET /market-data/instruments?instrumentIds=<ids>`.
   - Response: `{ instrumentDisplayDatas: [{ instrumentID, instrumentDisplayName, symbolFull, images[] }] }`.
4. Get live rates: `GET /market-data/instruments/rates?instrumentIds=<ids>`.
   - Response: `{ rates: [{ instrumentID, ask, bid, lastExecution }] }`.
5. Calculate account summary using the formulas in the **`etoro-account-snapshot`** rule (Available Cash, Total Invested, Profit/Loss, Equity). Don't shortcut them — the official formulas include per-mirror-position amounts, a `mirrors[].availableAmount − closedPositionsNetProfit` adjustment, and a `totalExternalCosts` term that simpler approximations miss.
6. Per-position PnL comes from `position.unrealizedPnL.pnL` (nested object, not flat field).

### Social Trading Analytics

1. Search for popular investors: `GET /user-info/people/search`.
2. View investor performance: `GET /user-info/people/{username}/gain`.
3. View investor portfolio: `GET /user-info/people/{username}/portfolio/live`.
4. Read social feeds: `GET /feeds/user/{userId}` or `GET /feeds/instrument/{marketId}`.
   - Response: `{ discussions: [...], paging, metadata }`. Each entry is `{ id, post: { owner: { username }, message: { text }, created } }` — the post body, author, and timestamp are nested under `post`, not on the discussion itself.

## API Base URL

`https://public-api.etoro.com/api/v1`

## Required Headers

Every request must include:
- `x-request-id`: unique UUID per request
- Authentication header (Bearer token or API key pair)

## Demo vs Real

- **Demo endpoints** contain `/demo/` in the path — use for testing and paper trading.
- **Real endpoints** omit `/demo/` — use for live trading.
- Ensure your API key environment (Virtual vs Real) matches the endpoint you're calling.

## Resources

- **API Portal**: https://api-portal.etoro.com/
- **MCP Server**: `https://api-portal.etoro.com/mcp`
- **API Documentation Index**: https://api-portal.etoro.com/llms.txt
