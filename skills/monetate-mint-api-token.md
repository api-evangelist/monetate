---
name: monetate-mint-api-token
description: Sign a JWT with your RSA private key, exchange it at the Monetate Auth API for a short-lived bearer token, and use that token against the Data API and Metadata API — including when to refresh and how to tell a revoked key from an expired token.
api: Monetate Auth API
generated: '2026-08-12'
method: generated
source: openapi/monetate-auth-api-openapi.yml, https://developer.monetate.com/auth-api
operations:
  - GET /refresh/   # overlay operationId: refreshaToken
base_url: https://api.monetate.net/api/auth/v0
---

# Mint a Monetate API token

Every authenticated Monetate API call needs a token, and the token is not the credential you were given
— it is minted from it. There are **two different `Authorization` header formats** in this flow and
swapping them is the single most common failure.

## Step 0 — Provision the key pair (one time, in the UI)

1. In the Monetate platform, go to **Settings → Sites → API Keys**.
2. Generate an API key. The platform gives you a **username** (e.g. `api-111-demo`).
3. Upload the **public** half of an RSA key pair you generate yourself. Keep the private half.

There is no API for this step and no self-service signup — you need an existing Monetate account.

## Step 1 — Sign a JWT

Payload must contain exactly two claims:

- `iat` — Unix timestamp of when the request is sent. Monetate allows **60 seconds** of clock skew, so
  sign immediately before sending. A cached JWT will start failing.
- `username` — the API username from the API Keys tab.

Sign with your private key using **RS256, RS384 or RS512**. No other algorithms are accepted.

```python
import time, jwt, requests

payload = {'iat': int(time.time()), 'username': 'api-111-demo'}
with open('private.pem') as f:
    jwt_token = jwt.encode(payload, f.read(), algorithm='RS256')
```

## Step 2 — Exchange it for a token

```
GET https://api.monetate.net/api/auth/v0/refresh/?ttl=3600
Authorization: JWT <signed_jwt>
Accept: application/json
```

Note the scheme is `JWT`, not `Bearer` and not `Token`.

`ttl` is optional and is the requested token lifetime in seconds. Default **3600** (1 hour); the
documented range is **600** (10 minutes) to **43200** (12 hours).

The response is the standard Monetate envelope:

```json
{"meta": {"code": 200, "errors": [], "warnings": []},
 "data": {"token": "...", "expires_in": 3600, "expires_at": 1786549121}}
```

- `data.token` — the opaque bearer token.
- `data.expires_in` — always equals the `ttl` you asked for.
- `data.expires_at` — seconds since epoch. **Use this.** Refresh a minute or two before it, rather than
  waiting for a 401 and retrying.

## Step 3 — Call the Data API and Metadata API

```
GET https://api.monetate.net/api/metadata/v1/{retailerShortname}/production/metadata/account
Authorization: Token <token_string>
```

Now the scheme is `Token`. Same header, different word from step 2.

The base path embeds both the tenant (`{retailerShortname}`) and the environment (`production`), so
switching environments means rewriting the URL — there is no environment header and no environment claim
in the token.

## Telling the failures apart

| Status | Meaning | What to do |
|---|---|---|
| `401` | No token sent, or the token is invalid/expired. | Re-mint (step 1–2) and retry once. If a fresh token also 401s, check the header scheme — `JWT` vs `Token`. |
| `403` | **The public key has been revoked.** | Do not retry. A human must re-enable or re-issue the key in Settings → Sites → API Keys. Retrying will never succeed. |
| `500` | Unknown error. | Retry with backoff; escalate to the Monetate account manager if persistent. |

A `403` is a permanent, human-in-the-loop stop. Treating it as retryable is the second most common
failure in this flow.

## Things the contract does not tell you

- The spec declares this scheme as `type: apiKey` rather than `type: http, scheme: bearer,
  bearerFormat: JWT`, so generated clients will not know a JWT is expected.
- The Auth API is on `/v0` while everything that depends on it is on `/v1`.
- There are **no scopes**. A token carries whatever the API user can do; there is no way to mint a
  read-only or narrowed token.
- The Engine API takes **no** token at all — do not send one there.
