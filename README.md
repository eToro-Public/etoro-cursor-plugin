# eToro Plugin for Cursor

Official eToro plugin for Cursor — **for developers building apps and integrations on top of the eToro Public API**. Provides AI-powered guidance in your IDE for trading, market data, portfolio management, OAuth/SSO, instrument resolution, and session handling.

> **Building a runtime LLM agent instead?** If you're not writing IDE code but rather building a runtime agent that acts on behalf of an end user (a chatbot that places trades when the user asks), see the companion skill package at <https://github.com/guyba-tr/etoro-agent-skills>. It carries runtime-specific patterns (intent confirmation, percentage-of-equity display rules, anchor-freeze, at-most-once trade execution, conversational onboarding for agent-portfolios) that intentionally aren't in this Cursor plugin.

## What's Included

### MCP Server

- **etoro-api-docs** — connects to eToro's official MCP server at `https://api-portal.etoro.com/mcp` for live API documentation search and reference.

### Rules

Always-on guardrails for working with the eToro Public API:

- **etoro-api-conventions** — hosts (Public API vs SSO/STS), required headers, the never-mix auth rule, casing inconsistency across endpoints, demo vs. real environments, environment probing (200 vs 403 to detect a key's environment), rate limits including the 20 req/min trade-execution sub-limit, the at-most-once / no-idempotency-key rule for trade-execution endpoints, response-shape gotchas, and trading defaults (Leverage, SL/TP, closing positions).
- **etoro-id-resolution** — instrument IDs and user CIDs are both stable; persist `symbol→ID` and `username→CID` maps for known assets/users and live-resolve only unknowns.
- **etoro-sso-identity** — never use the OIDC `sub` claim as a cross-app key; resolve `gcid` via `/api/v1/me`; pass `realCid` (not `gcid`) to CID-taking endpoints; persist refresh tokens atomically.
- **etoro-account-snapshot** — rules for any data about the user's eToro account (balance, equity, open positions, copy-trading mirrors, pending orders, P&L), all of which comes from a single endpoint that returns a full account snapshot. Includes the official aggregation formulas (Available Cash, Total Invested, Profit/Loss, Equity), the ~10-second response cache, and per-field correctness rules (native currency, leveraged margin, fees-vs-dividends blending, mirror-array dedup).

### Skills

Procedural playbooks triggered by specific tasks:

- **etoro-apps** — high-level guidance for building applications on top of the eToro API (OAuth, common patterns, trading bots, dashboards).
- **building-etoro-api-client** — robust HTTP client design: dual hosts, retry strategy by error class, typed error hierarchy, 401-then-refresh-once.
- **implementing-etoro-sso** — end-to-end OAuth auth-code grant with PKCE, token exchange, atomic refresh-token rotation, identity resolution.
- **resolving-etoro-instruments** — symbol→ID search, batched metadata lookups with 25–50-per-batch + adaptive 413/414 sizing, image-variant selection, background straggler retries.
- **handling-etoro-session-expiry** — detect refresh-token revocation, surface a "Reconnect to eToro" CTA instead of looping on dead tokens.

## Getting Started

1. Install this plugin in Cursor.
2. Get your eToro API credentials:
   - Log in to [eToro](https://www.etoro.com/).
   - Go to **Settings > Trading**.
   - Create a new API key (choose Virtual for demo or Real for live trading).
3. The MCP server provides live documentation — ask the agent about any eToro API endpoint.

## API Documentation

- **API Portal**: <https://api-portal.etoro.com/>
- **MCP Server**: `https://api-portal.etoro.com/mcp`
- **Documentation Index**: <https://api-portal.etoro.com/llms.txt>

## License

MIT
