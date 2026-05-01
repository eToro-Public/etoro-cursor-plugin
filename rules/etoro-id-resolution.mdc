---
description: Resolve eToro instrument IDs from symbols (e.g. AAPL, MSFT, BTC) and user CIDs from usernames. Apply when looking up instruments by symbol or ticker, searching for users / popular investors / copy-trading targets, building watchlists, populating positions for display, fetching instrument metadata (display name, logos, exchange, instrument type), or any external-reference → eToro-numeric-ID lookup via `/market-data/search`, `/market-data/instruments`, or `/user-info/people`.
alwaysApply: false
---

# eToro ID Resolution

eToro identifies tradable and copyable entities via numeric IDs:

- **Instruments** (stocks, ETFs, crypto, commodities, indices, currencies) are identified by `instrumentID`.
- **Users** (eligible for copy-trading, social features, follower lookups) are identified by `CID` (Customer ID).

Both kinds of IDs are stable, both follow the same resolution pattern, and both share the same "what am I acting on?" mental slot — when a user types a query into the eToro search box, instruments and copyable users come back side by side. This rule covers them together.

## 1. IDs are stable — persist a mapping for known assets/users

eToro IDs aren't expected to change. The recommended pattern for any app:

- **Instruments:** maintain a hardcoded `symbol → instrumentID` map for instruments your app uses regularly (major equities, indices, popular cryptos). This is the primary use case — instruments are referenced constantly across trade execution, metadata fetch, and live rate lookup.
- **Users / CIDs:** maintain a `username ↔ CID` lookup **specifically for labelling copy-trading mirrors in portfolio views** — `clientPortfolio.mirrors[].userId` is a CID, and you need to reverse-resolve it to a username (or `fullName`) for human-readable display. A persisted bidirectional mapping for the popular traders your customers commonly copy avoids re-resolving on every portfolio render. (Most other Public-API user endpoints accept username directly — e.g. `/user-info/people?usernames=`, `/user-info/people/{username}/portfolio/live` — so a username → CID map outside the mirror-display use case is rarely worth maintaining.)
- Resolve once, commit/persist the mapping, and reuse on every request.

This avoids round-trips to search on every call and avoids hitting search rate limits when many users hit the same lookups concurrently.

```typescript
const KNOWN_INSTRUMENTS: Record<string, number> = {
  AAPL: 1001,
  MSFT: 1002,
  TSLA: 1003,
  // …
};

const KNOWN_USERS: Record<string, number> = {
  jeppekirkbonde: 12345,
  heloise: 67890,
  // …
};
```

## 2. Live-resolve only for unknowns

### Instruments (symbol → instrumentID)

```
GET /market-data/search?internalSymbolFull=<SYMBOL>
```

- The response array `items[]` may contain multiple matches (e.g. `MSFT`, `MSFT.RTH`, `MSFT.EUR`). Confirm `internalSymbolFull` matches exactly before using the result.
- The search endpoint returns `instrumentId` (lowercase `d`). Most other endpoints use `instrumentID` (capital `D`) — don't accidentally model the search response with the wrong casing. See `etoro-api-conventions` for the full casing table.
- Once resolved, **cache or commit the new mapping** so the next request hits your map.

### Users (CID ↔ username, for mirror-display)

You almost never need a username→CID map for runtime workflows — most user endpoints accept username directly. The exception is **reverse-resolving CIDs to usernames** so you can display copy-trading mirrors by name (the `userId` field in `clientPortfolio.mirrors[]` is a CID).

Two endpoints, depending on direction:

```
GET /user-info/people?cidList=<cid1>,<cid2>,...     # CID → username (mirror display)
GET /user-info/people?usernames=<username>          # username → CID (rare; e.g. user types a name)
```

Both return the public profile fields (display name, avatar, country, etc.) alongside the other identifier — a single call resolves AND enriches. Persist the resulting bidirectional mapping for traders that appear repeatedly across your users' mirrors.

## 3. The signed-in user is special

For the *currently signed-in* user, don't go through `/user-info/people`. Use:

```
GET /api/v1/me
```

Returns `{ gcid, realCid, demoCid }` — see the **`etoro-sso-identity`** rule for which CID to use where, the `cidList` vs `gcid` data-leak trap, and why `gcid` is the recommended primary key on your own user records.

## 4. Enrich instruments with metadata after ID resolution

Once you have an `instrumentID` (or batch of them), fetch display data:

```
GET /market-data/instruments?instrumentIds=<id1>,<id2>,...
```

- Send IDs in batches of **25–50**; this endpoint rejects large batches with HTTP 413/414.
- The response uses `instrumentID` (capital `D`) and includes display name, full symbol, exchange, instrument type, and image variants.

The full batching / retry / image-variant playbook lives in the **`resolving-etoro-instruments`** skill.

(Users don't need a separate enrichment step — `/user-info/people` already returns the public profile fields alongside the CID.)

## 5. Never guess an ID

If you don't have a verified mapping for a symbol or username, resolve it via search. **Never** make up an ID, transform a number, or hash a string — eToro's IDs are dense small integers and you will silently hit a real but wrong instrument or user.
