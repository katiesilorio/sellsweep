# Auth & Credentials

All credentials live in `.env` in the project root - git-ignored, never committed, never pasted into chat. Everything there is scoped and revocable. **No account passwords, ever.**

Read `.env` at the start of a run. If a required value is missing for a requested destination, say which one and stop - don't attempt the call.

## Square - no OAuth needed

Developer Console → Credentials → copy the access token. Separate sandbox and production tokens; mixing them is an auth error.

```
SQUARE_ACCESS_TOKEN, SQUARE_LOCATION_ID
```

Get the location ID: `GET https://connect.squareupsandbox.com/v2/locations` (or `connect.squareup.com` for production).

Non-expiring. Nothing to refresh.

## eBay - OAuth authorization code grant

**One-time, requires the account owner at the login screen.**

1. Register an app at `developer.ebay.com`. Record **App ID** (client ID) and **Cert ID** (client secret).
2. **Enable the production keyset** - it stays disabled until the account subscribes to or opts out of marketplace account-deletion notifications. This is a separate step and the usual first blocker.
3. Configure a **RuName** - eBay's redirect identifier, used as the `redirect_uri` value. Requires a legal-address confirmation form.
4. Build the consent URL:
   ```
   https://auth.ebay.com/oauth2/authorize
     ?client_id={APP_ID}
     &response_type=code
     &redirect_uri={RUNAME}
     &scope=https://api.ebay.com/oauth/api_scope/sell.inventory
            https://api.ebay.com/oauth/api_scope/sell.account
   ```
5. **Account owner opens it, signs in, approves.** eBay redirects with `?code=...`
6. Exchange the code at `POST https://api.ebay.com/identity/v1/oauth2/token`, `grant_type=authorization_code`, Basic auth of `APP_ID:CERT_ID`.
7. Store the **refresh token** (`refresh_token_expires_in` ≈ 47,304,000s ≈ 18 months) in `.env`.

Access tokens last 2 hours and are minted from the refresh token per run - never stored.

**Trading API:** pass the OAuth access token via the **`X-EBAY-API-IAF-TOKEN`** header, not the `<eBayAuthToken>` XML element. Same token works for the Media API.

Sandbox uses `auth.sandbox.ebay.com` and a separate keyset; tokens are not cross-valid.

## Etsy - OAuth authorization code with PKCE

**One-time, requires the account owner. Repeat roughly quarterly** - the refresh token expires in ~90 days.

1. Register a **Seller App** at `developers.etsy.com`. Approves in minutes for a seller working on their own shop. Record the **keystring**.
2. Generate a PKCE pair: random `code_verifier`, then `code_challenge = BASE64URL(SHA256(code_verifier))`.
3. Consent URL:
   ```
   https://www.etsy.com/oauth/connect
     ?response_type=code
     &client_id={KEYSTRING}
     &redirect_uri={REDIRECT}
     &scope=listings_w%20listings_r
     &state={RANDOM}
     &code_challenge={CHALLENGE}
     &code_challenge_method=S256
   ```
4. **Account owner opens it, signs in, approves.**
5. Exchange at `POST https://api.etsy.com/v3/public/oauth/token`, `grant_type=authorization_code`, including the original `code_verifier`.
6. Store the **refresh token** in `.env`.

**Every Etsy call needs both** an `x-api-key` header and an `Authorization: Bearer {access_token}` header. Access token lasts 1 hour.

**⚠️ The `x-api-key` value is `{KEYSTRING}:{SHARED_SECRET}`, colon-separated, not the keystring alone.** Etsy's docs describe the keystring alone, which returns `403 {"error":"Shared secret is required in x-api-key header."}` on a personal app. Sending the secret alone returns `403 ... incorrect shared secret for API key`. Reading the two errors together is what gives it away: Etsy wants both values in the one header.

Verified 2026-08-21 against a live personal app. Host is `openapi.etsy.com`.

**The access token is prefixed with the user id**, as `{user_id}.{token}`. Useful for a sanity check without an extra call.

`shop_id` comes from `GET /v3/application/users/me`. A 200 with no `shop_id` means the account has no open shop, which is a different problem from an auth failure and should be reported as such.

Get `shop_id` from `GET /v3/application/users/me`.

**No sandbox.** Development is against the live shop at $0.20/listing - use draft state, never publish while testing.

## Facebook Marketplace - session, not credentials

No tokens, nothing in `.env`. The operator logs into Facebook in Chrome themselves; the skill drives the already-authenticated session and **stops before submit**.

**The skill never enters a password.** If a login screen appears, hand it to the operator.

## Refresh cadence

| Platform | Refresh token life | Action |
|---|---|---|
| Square | n/a | none |
| eBay | ~18 months | re-authorize before expiry |
| Etsy | ~90 days | re-authorize quarterly |
| Facebook | session | re-login when Chrome logs out |

Check refresh-token age at the start of a run. Warn when Etsy's is within 7 days of expiring - silently failing mid-batch is worse than an early warning.
