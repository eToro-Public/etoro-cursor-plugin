---
name: implementing-etoro-sso
description: Developer playbook for implementing eToro OAuth SSO in a server-side app — auth-code grant with PKCE, token exchange, atomic refresh-token rotation, and identity resolution via /api/v1/me. Use when adding "Sign in with eToro" to an app or hardening an existing SSO integration.
---

# Implementing eToro SSO

Use this skill when adding "Sign in with eToro" to an app, or when migrating from manual API keys to SSO. It covers the OAuth flow, token storage, refresh handling, and the identity resolution that follows.

## When to use

- Building a new app where the user authenticates with their eToro account.
- Replacing manual API-key entry with SSO.
- Hardening an existing SSO integration that loses sessions or duplicates user records.

## Step 1 — Kick off the OAuth flow (auth-code grant + PKCE)

```typescript
// /api/auth/etoro/start
router.get('/api/auth/etoro/start', (req, res) => {
  const codeVerifier = crypto.randomBytes(32).toString('base64url');
  const codeChallenge = crypto
    .createHash('sha256')
    .update(codeVerifier)
    .digest('base64url');
  const state = crypto.randomBytes(16).toString('hex');

  req.session.etoroOauth = { codeVerifier, state };

  const params = new URLSearchParams({
    response_type: 'code',
    client_id: process.env.ETORO_CLIENT_ID!,
    redirect_uri: process.env.ETORO_REDIRECT_URI!,
    scope: 'openid offline_access',
    state,
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
  });
  res.redirect(`https://www.etoro.com/sso/oauth2/authorize?${params}`);
});
```

PKCE is recommended for all clients and required for public clients (SPA / mobile). For confidential server clients you can also send `client_secret` on the token endpoint.

## Step 2 — Handle the callback and exchange the code

```typescript
router.get('/api/auth/etoro/callback', async (req, res) => {
  const { code, state } = req.query as { code?: string; state?: string };
  const stored = req.session.etoroOauth;
  if (!code || !stored || stored.state !== state) {
    return res.status(400).json({ error: 'Invalid OAuth state' });
  }

  const body = new URLSearchParams({
    grant_type: 'authorization_code',
    code,
    redirect_uri: process.env.ETORO_REDIRECT_URI!,
    client_id: process.env.ETORO_CLIENT_ID!,
    code_verifier: stored.codeVerifier,
  });

  const tokenRes = await fetch('https://www.etoro.com/sso/oidc/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body,
  });

  if (!tokenRes.ok) {
    const errBody = await tokenRes.json().catch(() => ({}));
    return res.status(502).json({
      error: errBody.error_description || 'Token exchange failed',
    });
  }

  const tokens = (await tokenRes.json()) as {
    access_token: string;
    refresh_token: string;
    id_token: string;
    expires_in: number;
    token_type: 'Bearer';
  };

  // Resolve gcid via /api/v1/me — see Step 4
  const me = await fetchMe(tokens.access_token);

  // Persist atomically
  await persistTokens({
    gcid: me.gcid,
    accessToken: tokens.access_token,
    refreshToken: tokens.refresh_token,
    expiresAt: new Date(Date.now() + tokens.expires_in * 1000),
  });

  delete req.session.etoroOauth;
  res.redirect('/dashboard');
});
```

**Note the host and content type:** SSO endpoints use `https://www.etoro.com` and `application/x-www-form-urlencoded`. Sending JSON to this host returns 400 with effectively empty error bodies.

## Step 3 — Refresh atomically (rotation is mandatory)

Every successful refresh returns a NEW refresh token; the previous one is immediately invalidated. Persist both new tokens in the same DB transaction — if you only write `access_token`, the next refresh will fail with `400 invalid_grant`.

```typescript
async function refreshAndPersist(gcid: number): Promise<EtoroRequestContext> {
  const { refresh_token: oldRT } = await getStoredTokens(gcid);

  const body = new URLSearchParams({
    grant_type: 'refresh_token',
    refresh_token: oldRT,
    client_id: process.env.ETORO_CLIENT_ID!,
  });

  const r = await fetch('https://www.etoro.com/sso/oidc/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body,
  });

  if (!r.ok) {
    const err = await r.json().catch(() => ({}));
    if (err.error === 'invalid_grant') {
      // Session is dead — see the handling-etoro-session-expiry skill
      throw new SsoSessionExpiredError('Refresh rejected');
    }
    throw new SsoError(err.error_description || 'Refresh failed', r.status);
  }

  const t = await r.json();

  await db.transaction(async (tx) => {
    await tx.query(
      `UPDATE etoro_sso_tokens
          SET access_token = $1, refresh_token = $2, expires_at = $3
        WHERE gcid = $4`,
      [
        t.access_token,
        t.refresh_token,
        new Date(Date.now() + t.expires_in * 1000),
        gcid,
      ],
    );
  });

  return { mode: 'bearer', accessToken: t.access_token };
}
```

## Step 4 — Resolve identity via `/api/v1/me`, use `gcid` as the primary key

The OIDC `sub` claim is per-OAuth-client and unsafe as a cross-app key. The three IDs returned by `/api/v1/me` (`gcid`, `realCid`, `demoCid`) are all stable cross-app identifiers in a 1:1 relationship — `gcid` is the recommended primary key because it's environment-independent (same value in real and demo), while `realCid` / `demoCid` are what eToro endpoints accept when they want a "CID".

```typescript
async function fetchMe(accessToken: string) {
  const r = await fetch('https://public-api.etoro.com/api/v1/me', {
    headers: {
      Authorization: `Bearer ${accessToken}`,
      'x-request-id': crypto.randomUUID(),
    },
  });
  if (!r.ok) throw new Error(`/me failed: ${r.status}`);
  return r.json() as Promise<{
    gcid: number;
    realCid: number;
    demoCid: number;
  }>;
}
```

Persist all three on your user record if you'll be calling CID-taking endpoints; otherwise persisting `gcid` alone is fine since you can always re-derive the others from `/api/v1/me` on demand.

See the **`etoro-sso-identity`** rule for the full identity rules and the `realCid` vs `gcid` data-leak trap.

## Step 5 — Token storage schema

Minimum schema:

```sql
CREATE TABLE etoro_sso_tokens (
  gcid          BIGINT PRIMARY KEY,
  access_token  TEXT NOT NULL,
  refresh_token TEXT NOT NULL,
  expires_at    TIMESTAMPTZ NOT NULL,
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Key the eToro token row on `gcid` so a user signing in from different OAuth client registrations (dev / prod) lands on the same record — `gcid` is identical across registrations, the OIDC `sub` claim is not. Associate `gcid` with your own user table via a join column (`etoro_gcid`), not by overwriting your users table's PK. If you also persist `realCid` / `demoCid` you can save a `/me` round-trip on every CID-taking call; alternatively re-derive them from `/me` on demand since all three IDs share a 1:1 relationship.

## Sanity checks

- [ ] The `state` parameter is verified on every callback (CSRF defense).
- [ ] PKCE is used for all flows; `code_verifier` is single-use and stored in the session.
- [ ] The token-exchange and refresh requests both use `application/x-www-form-urlencoded`.
- [ ] Refresh writes both new tokens atomically — never just `access_token`.
- [ ] The user record is keyed on `gcid` resolved from `/api/v1/me`, never on the OIDC `sub` claim.
- [ ] `400 invalid_grant` from the SSO host triggers a "Reconnect to eToro" flow, not another refresh attempt (see `handling-etoro-session-expiry`).
