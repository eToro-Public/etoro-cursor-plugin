---
name: building-etoro-api-client
description: Developer playbook for building a robust HTTP client for the eToro Public API and SSO/STS — request shape, retry strategy by HTTP error class, error-class hierarchy, and 401-then-refresh-once recovery. Use when a developer is writing or refactoring server-side code that calls eToro endpoints (TypeScript / Python / Go / etc.).
---

# Building an eToro API Client

Use this skill when starting a new eToro integration or when refactoring an existing one to be production-ready. It covers the wrapper shape, retry strategy, and error handling that any client of the eToro Public API needs.

## When to use

- Building a Node / Python / Go / etc. service that talks to the eToro Public API.
- Adding the first `etoroFetch`-style helper to a project that's been making raw `fetch` calls.
- Hardening an existing client with retry + error classification.

## Step 1 — Two distinct clients (Public API + SSO/STS)

The Public API and SSO/STS hosts have different content types and error envelopes; **don't try to share one wrapper.** See the `etoro-api-conventions` rule for the host table.

```typescript
// Public API client — JSON in/out
export async function etoroFetch<T>(
  path: string,
  ctx: EtoroRequestContext,
): Promise<T> {
  const url = `https://public-api.etoro.com/api/v1${path}`;
  const headers = buildEtoroHeaders(ctx);
  const res = await fetchWithRetry(url, { headers });
  return parseEtoroResponse<T>(res);
}
```

List-valued query params (e.g. `instrumentIds=1,2,3`) must be joined with a literal `,` — see `etoro-api-conventions`.

```typescript
// SSO/STS client — form-encoded in, OAuth-style errors out
export async function etoroSsoPost<T>(
  path: string,
  body: Record<string, string>,
): Promise<T> {
  const url = `https://www.etoro.com${path}`;
  const res = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams(body),
  });
  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new SsoError(
      err.error_description || err.error || `SSO ${res.status}`,
      res.status,
      err.error,
    );
  }
  return res.json();
}
```

## Step 2 — Discriminated-union credentials

Model the two valid auth modes as a tagged union; the never-mix rule becomes structurally enforced — you literally can't accidentally set both header families.

```typescript
export type EtoroRequestContext =
  | { mode: 'bearer'; accessToken: string }
  | { mode: 'apiKey'; userKey: string };

export function buildEtoroHeaders(ctx: EtoroRequestContext): Record<string, string> {
  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
    'x-request-id': crypto.randomUUID(),
  };
  if (ctx.mode === 'bearer') {
    headers['Authorization'] = `Bearer ${ctx.accessToken}`;
  } else {
    const apiKey = process.env.ETORO_API_KEY!;
    headers['x-api-key'] = apiKey;
    headers['x-user-key'] = ctx.userKey;
  }
  return headers;
}
```

Adding a new endpoint then becomes one typed line:

```typescript
export async function fetchPnl(env: 'demo' | 'real', ctx: EtoroRequestContext) {
  return etoroFetch<EtoroPnlResponse>(`/trading/info/${env}/pnl`, ctx);
}
```

## Step 3 — Retry strategy by HTTP error class

Different failure classes look superficially similar but require completely different recovery. Mixing them produces thundering-herd loops or unnecessary user-visible errors.

| Status | Likely cause | Same-request retry? | Strategy |
|---|---|---|---|
| **401** | Access token expired or session revoked | Once, after refresh | Refresh access token via SSO; if refresh itself fails with `invalid_grant`, surface "Reconnect to eToro" (see the `handling-etoro-session-expiry` skill). |
| **413 / 414** | Payload / URI too large (typically `/market-data/instruments` with too many IDs) | Yes — with a smaller payload | Halve the batch and retry. Same-size retry will fail again. |
| **429** | Rate limit | No (no payload change helps) | Backoff (e.g. 1s → 5s → 30s) with the same payload. |
| **5xx** | Transient eToro-side error | Yes, with backoff | 2–3 attempts with short exponential delays (e.g. 200ms → 600ms → 1500ms). |
| **Other 4xx** | Bad request / not found / forbidden | No | Surface to caller — the request shape itself is wrong. |

**Two derived rules worth applying:**

- **Adaptive batching, not adaptive backoff.** 413/414 means "this payload was too big" — halve and retry. 429/5xx mean "try again later" — wait, don't shrink. Conflating the two hides real rate-limit problems behind shrinking batch sizes that succeed by accident.
- **Background straggler retry.** For non-blocking enrichment (logos, instrument metadata, public-profile lookups), schedule a delayed retry (~10–30s after the user request resolves) for any IDs that didn't resolve, write results into a local cache, and let the next polling cycle pick them up. Don't make the user wait synchronously for stragglers.

## Step 4 — Typed error class hierarchy

Set `this.name` on each subclass — `instanceof` works inside one bundle but `name` survives serialization, source maps, and bundler renames, so callers can branch on it across module boundaries.

```typescript
export class EtoroApiError extends Error {
  constructor(message: string, public statusCode: number) {
    super(message);
    this.name = 'EtoroApiError';
  }
}

export class EtoroRateLimitError extends EtoroApiError {
  constructor(msg = 'Rate limit') {
    super(msg, 429);
    this.name = 'EtoroRateLimitError';
  }
}

export class SsoSessionExpiredError extends EtoroApiError {
  constructor(msg = 'eToro session expired') {
    super(msg, 401);
    this.name = 'SsoSessionExpiredError';
  }
}

export class SsoError extends Error {
  constructor(message: string, public statusCode: number, public oauthCode?: string) {
    super(message);
    this.name = 'SsoError';
  }
}
```

## Step 5 — 401-then-refresh-once

Wrap your client so a 401 triggers exactly one refresh attempt before propagating. More than one retry burns through rotated refresh tokens and locks the user out faster.

```typescript
export async function etoroFetchWithRefresh<T>(
  path: string,
  userId: string,
): Promise<T> {
  const ctx = await getEtoroAuthCtx(userId);
  try {
    return await etoroFetch<T>(path, ctx);
  } catch (err) {
    if (err instanceof SsoSessionExpiredError) {
      const refreshedCtx = await refreshAndPersist(userId);
      return etoroFetch<T>(path, refreshedCtx);
    }
    throw err;
  }
}
```

If the second call also throws `SsoSessionExpiredError`, treat the session as dead — see the `handling-etoro-session-expiry` skill.

## Sanity checks before merging

- [ ] Every Public API call sends `x-request-id` (UUID v4 per request).
- [ ] No call site sends both `Authorization: Bearer` and `x-api-key` / `x-user-key`.
- [ ] Errors propagate as typed subclasses with stable `name`.
- [ ] 401, 429, 413/414, and 5xx each have distinct recovery paths.
- [ ] Background work uses `void (async () => { try { ... } catch { ... } })()` — never throws into a user request.
- [ ] The SSO and Public API clients are kept separate (different host, content-type, error parser).
