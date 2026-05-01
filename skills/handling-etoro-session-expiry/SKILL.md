---
name: handling-etoro-session-expiry
description: Developer playbook for detecting and recovering from eToro SSO session expiry — distinguish refresh-token revocation from transient 401s, and surface a "Reconnect to eToro" flow instead of looping on dead tokens. Use when wiring the error-handling layer of an app or designing the frontend's "session expired" UI.
---

# Handling eToro Session Expiry

Use this skill when designing the failure modes of an eToro-SSO-backed app. The wrong error UX (a generic "Retry" button on a dead session) traps users in a 401 loop forever; this playbook makes the recovery path correct from day one.

## When to use

- Implementing the error-handling layer for an eToro client.
- Hooking up the frontend's "session expired" UI.
- Diagnosing a user who is stuck in an unrecoverable state after a long-idle session.

## Background — two failure modes that look identical

1. **Access token expired (recoverable).** The access token has hit its `expires_in`; refresh it via the SSO token endpoint and retry. This is routine and invisible to the user.
2. **Refresh token revoked (NOT recoverable by refresh).** eToro can revoke a perfectly valid refresh token server-side independently of any rotation event — typically returning `400 invalid_grant` on the next refresh attempt. The user did nothing wrong; eToro just decided to revoke the session (session policy, OAuth client re-registration, suspected abuse, internal STS rotation, etc.). **No amount of retrying the refresh will fix this** — the user has to re-authorize.

## Step 1 — Detect refresh-token revocation

Two signals indicate a dead session:

- **Direct:** `400 invalid_grant` from the SSO token endpoint with `grant_type=refresh_token`.
- **Indirect:** A `401` from the Public API where the subsequent refresh attempt also returns `400 invalid_grant`.

```typescript
function isSessionDead(err: unknown): boolean {
  if (err instanceof SsoSessionExpiredError) return true;
  if (
    err instanceof SsoError &&
    err.statusCode === 400 &&
    err.oauthCode === 'invalid_grant'
  ) {
    return true;
  }
  return false;
}
```

## Step 2 — Stop retrying immediately

The biggest UX bug in this area is a generic "Retry" button that calls the refresh endpoint again and again, racing the user's clicks. Don't do this:

```typescript
// BAD — re-issues the same dead refresh token until the user gives up
async function fetchPortfolioBad(userId: string) {
  try {
    return await etoroFetchWithRefresh('/trading/info/real/pnl', userId);
  } catch (err) {
    if (err.statusCode === 401) return fetchPortfolioBad(userId); // <- loop
    throw err;
  }
}
```

Replace with a single refresh attempt, then surface the dead-session error specifically:

```typescript
async function fetchPortfolio(userId: string) {
  try {
    return await etoroFetchWithRefresh('/trading/info/real/pnl', userId);
  } catch (err) {
    if (isSessionDead(err)) {
      throw new EtoroReconnectRequiredError();
    }
    throw err;
  }
}
```

## Step 3 — Surface a dedicated "Reconnect to eToro" flow

`EtoroReconnectRequiredError` should map to a distinct frontend UI: a banner or modal that explains the session expired and offers a single button that re-runs the OAuth `/api/auth/etoro/start` endpoint. **Do not** offer "Retry" or "Refresh" — neither will help.

When the user goes through SSO again, the callback handler (see the `implementing-etoro-sso` skill) writes a fresh access + refresh token pair against the same `gcid`-keyed user record, and the session continues seamlessly.

```tsx
function PortfolioPage() {
  const { data, error } = useQuery({
    queryKey: ['portfolio'],
    queryFn: fetchPortfolio,
  });
  if (error instanceof EtoroReconnectRequiredError) {
    return (
      <ReconnectBanner
        onReconnect={() => location.assign('/api/auth/etoro/start')}
      />
    );
  }
  // …
}
```

## Step 4 — Don't auto-revoke on close

It's tempting to call `/sso/v1/revoke` on logout to "be tidy." That's fine, but make sure it can't run on a dead session — calling revoke with an already-revoked refresh token returns 400 and pollutes your error logs. Treat revoke as best-effort:

```typescript
revokeRefreshToken(rt).catch(() => undefined);
```

## Step 5 — What NOT to do

- **Don't** call the SSO token endpoint repeatedly with the same refresh token expecting different results.
- **Don't** clear the user's tokens on the first 401 — that strands them, and the next page load looks identical to a fresh login (no message, just a redirect to start).
- **Don't** unify "session expired" and "rate limited" UX. They have different recovery paths (re-auth vs. wait + retry).

## Sanity checks

- [ ] A user whose refresh token was revoked sees a "Reconnect to eToro" CTA, not a spinner.
- [ ] Reconnecting writes a fresh token pair against the existing user record (keyed on `gcid`).
- [ ] Logs distinguish "refresh failed (revoked)" from "refresh failed (network)" — only the former is a session-expired event.
- [ ] The retry/refresh layer makes at most one refresh attempt per failed request.
