# eToro Plugin for Cursor

Official eToro plugin for Cursor — **for developers building apps and integrations on top of the eToro Public API**. Provides AI-powered guidance in your IDE for trading, market data, portfolio management, OAuth/SSO, instrument resolution, and session handling.

## What's Included

- **MCP Server** — live access to eToro's official API documentation at `https://api-portal.etoro.com/mcp`, so the agent can look up any endpoint on demand.
- **Rules** — always-on conventions and guardrails the agent applies whenever you write code against the eToro API: request shape, authentication, identifier handling, account-data semantics, and identity model.
- **Skills** — procedural playbooks the agent loads when you tackle a specific task: building an API client, implementing SSO, resolving instruments, handling session expiry, or scaffolding a full app.

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
