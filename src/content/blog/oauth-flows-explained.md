---
title: "OAuth 2.0 Flows Explained Without the Hand-Waving"
description: "A clear breakdown of Authorization Code, Client Credentials, and PKCE flows — when to use which, what the tokens actually mean, and where implementations go wrong."
category: "security"
tags: ["oauth", "jwt", "authentication", "oidc"]
publishedAt: 2025-03-22
---

OAuth 2.0 is one of those topics where a lot of tutorials stop at the happy path. This is a ground-up explanation of the flows that matter in production.

## The Three Flows You Actually Use

### Authorization Code + PKCE (Public Clients)

For SPAs and mobile apps — anywhere you can't store a secret safely.

PKCE (Proof Key for Code Exchange) prevents authorization code interception attacks. The client generates a random `code_verifier`, hashes it into a `code_challenge`, and sends the challenge with the authorization request. The verifier is sent on the token exchange — only the original client can complete the flow.

```
1. Client generates: code_verifier (random 64-byte string)
2. Client computes:  code_challenge = BASE64URL(SHA256(code_verifier))
3. Auth request:     GET /authorize?code_challenge=...&code_challenge_method=S256
4. Token exchange:   POST /token { code, code_verifier }
5. Server verifies:  SHA256(code_verifier) == stored code_challenge
```

### Authorization Code (Confidential Clients)

Same flow, but without PKCE — the client secret substitutes as proof. Only for server-side apps where the secret can be stored safely.

### Client Credentials (Machine-to-Machine)

No user involved. A service exchanges its `client_id` + `client_secret` for an access token directly:

```
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=...&client_secret=...&scope=api:read
```

Used for API-to-API calls, scheduled jobs, background services.

## What the Access Token Actually Is

In most modern implementations, it is a JWT — a signed JSON payload. The server validates the signature without a database lookup, which is why access tokens should be short-lived (15 minutes is common).

```json
{
  "sub": "user-uuid",
  "iss": "https://auth.yourapp.com",
  "aud": "https://api.yourapp.com",
  "exp": 1716979200,
  "scope": "read:cases write:cases",
  "tenant_id": "abc-tenant"
}
```

## Where Implementations Go Wrong

**Long-lived access tokens** — a compromised token can't be revoked (JWTs are stateless). Keep them short; use refresh tokens for session continuity.

**Missing `aud` validation** — your API must verify that the token was intended for it, not just that it's a valid token.

**Implicit flow** — deprecated. Do not use it. Access tokens in URL fragments are exposed in browser history and to third-party scripts.

**Storing tokens in localStorage** — accessible to any JavaScript on the page. Use `httpOnly` cookies for refresh tokens.
