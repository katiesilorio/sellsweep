# Guided Setup

Run this when `.env` is missing values, when `seller_profile.md` doesn't exist, or when the user asks to set up / connect / add a platform.

## Principles

**Resumable.** Setup spans sessions. Read `SETUP_LOG.md` first, find the first unchecked gate, and resume there. Never restart from the top or re-ask something already answered.

**One gate at a time.** Present a single gate, wait, verify, write state, then move on. Do not dump the whole sequence - it reads as homework and people stall.

**Verify, don't trust.** After every credential, make a real API call. "I pasted it" and "it works" are different claims. Each verification below also *collects* something needed, so it's not wasted work.

**Do the mechanical part.** The skill builds consent URLs, exchanges authorization codes, writes `.env`, runs verification calls, and opens the right browser pages. The human only does what genuinely requires them: signing in, approving consent, clicking things in dashboards.

**Never handle a password.** If a login screen appears during browser work, hand it over: *"Sign in here and tell me when you're through - I won't touch the password field."*

## State

`SETUP_LOG.md` is the source of truth. After each gate, check the box and record what was learned. If it doesn't exist, create it from the template in the repo.

Report progress as a count, not a lecture: *"Square's connected and verified. Three gates left - eBay, Etsy, and your seller profile. Want to keep going or pick this up later?"*

---

## Gate 0 - Which destinations?

Ask once. Don't set up things they don't want.

> Which of these do you want to list on? eBay, Etsy, Square (plus Facebook Shops and Instagram through Square), Facebook Marketplace.

Record the answer. Skip every gate for a destination they didn't pick.

If they're unsure, the short version: Square is the least work and reaches Facebook Shops + Instagram for free. eBay has the widest buyer reach and gets Facebook Marketplace exposure as a bonus. Etsy only works for original-design handmade items. Facebook Marketplace is local pickup, one listing at a time.

---

## Gate 1 - Square (start here)

Do Square first. It's the only destination with no OAuth, so it's a fast win, and it unblocks the one test that could still change scope.

**Skill does:** open `https://developer.squareup.com/apps` in a browser tab.

**Human does:**
1. Sign in with the Square account (their own credentials - skill doesn't touch this)
2. Create an application if there isn't one
3. Credentials → copy the **Sandbox** access token

**Ask for:** the sandbox access token.

**Verify - this call also gets the location ID, so nothing is wasted:**
```
GET https://connect.squareupsandbox.com/v2/locations
Authorization: Bearer {token}
Square-Version: 2025-01-23
```

- 200 → extract `locations[0].id`, write both to `.env` as `SQUARE_ACCESS_TOKEN` and `SQUARE_LOCATION_ID`. Confirm: *"Connected - I can see the location, [name]."*
- 401 → wrong token or wrong environment. Sandbox and production tokens are not interchangeable.

**Write:** Gate 1 checked, location name and ID recorded.

---

## Gate 2 - Square auto-publish test

**Do this before writing any Square listings.** It answers whether Square is a real API destination or a half-manual one, and the answer changes scope.

**Why:** `CatalogItem.channels` is read-only and `ecom_visibility` is undocumented - Square staff have confirmed online visibility isn't settable via API. Dashboard item defaults are the only known workaround, and its reliability has open complaints.

**Human does:** Square Dashboard → Items & Orders → Settings → **item defaults**. Set new-item site visibility to **Listed** and assign new items to the Square Online site. Screenshot it.

**Skill does:** `batchUpsertCatalogObjects` creating one `ITEM` with two `ITEM_VARIATION`s, clearly named as a test.

**Human does:** check the Square Online site. Does the item appear, and is it purchasable, with **no** dashboard interaction?

**Then repeat once in production** with a disposable item - sandbox and Square Online don't always behave alike. Delete it after.

**Record the result and say plainly what it means:**
- Passes → Square is a full API destination.
- Fails → Square degrades to "creates the item, you flip visibility by hand." At hundreds of items that's a real cost, and worth deciding whether Square stays in scope.

---

## Gate 3 - eBay

Three sub-steps. Do the policy check first - it's one page and it determines field mapping for every eBay listing.

### 3a - Business Policies state

**Skill does:** open `https://www.ebay.com/sh/ovw` and explain where to look.

**Human does:** My eBay → Account → **Business Policies** (or Seller Hub → Account).

- A "Manage business policies" page listing shipping/payment/return policies → **opted in**
- An opt-in invitation → **not opted in**

**Write** `EBAY_BUSINESS_POLICIES_OPTED_IN` to `.env`. If opted in, also collect the three policy names (IDs come from the API later).

Reassure them this isn't a problem either way: *"Both states work fine - it just changes which fields I send."*

### 3b - Developer app

**Skill does:** open `https://developer.ebay.com/my/keys`.

**Human does:**
1. Register a developer account if needed (free)
2. Create an application → **App ID** (client ID) and **Cert ID** (client secret)
3. **Enable the production keyset** - it stays disabled until they subscribe to or opt out of marketplace account-deletion notifications. **Warn about this proactively.** It's a separate step, it isn't obvious, and people assume their keys are broken.
4. Configure a **RuName** (redirect identifier) - needs a legal-address confirmation form

**Ask for:** App ID, Cert ID, RuName.

### 3c - OAuth

**Skill does:** build the consent URL:
```
https://auth.ebay.com/oauth2/authorize
  ?client_id={APP_ID}
  &response_type=code
  &redirect_uri={RUNAME}
  &scope=https://api.ebay.com/oauth/api_scope/sell.inventory%20https://api.ebay.com/oauth/api_scope/sell.account
```

**Human does:** open it, sign in as the seller, approve. Copy the `code` parameter from the redirect URL. Note the code is single-use and short-lived - if it expires, generate a fresh consent URL rather than retrying the old code.

**Skill does:** exchange it:
```
POST https://api.ebay.com/identity/v1/oauth2/token
Authorization: Basic base64({APP_ID}:{CERT_ID})
grant_type=authorization_code&code={CODE}&redirect_uri={RUNAME}
```

Store `refresh_token` (~18 months) in `.env`. Never store the access token - mint it per run.

**Verify - this call also answers the policy question independently:**
```
GetUserPreferences with ShowSellerProfilePreferences=true
```
Confirms the token works *and* returns `SellerProfilePreferences.SellerProfileOptedIn`. If it disagrees with what they saw in 3a, trust the API.

**Also collect:** their monthly selling limits (Seller Hub → Overview → Monthly limits) - item count and dollar cap. Needed before pushing a large batch.

---

## Gate 4 - Etsy

**Only if they have an Etsy shop.** Ask before starting. Also set expectations up front: Etsy accepts original-design handmade items only - no resale, no third-party designs, no team or character imagery. If their catalog is mostly resale, say so now rather than after the setup work.

### 4a - Seller App

**Skill does:** open `https://www.etsy.com/developers/register`.

**Human does:** register a **Seller App**. Approves in minutes with no review queue for a seller working on their own shop. Copy the **keystring**.

### 4b - OAuth with PKCE

**Skill does:** generate a random `code_verifier`, compute `code_challenge = BASE64URL(SHA256(verifier))`, build:
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

**Human does:** open, sign in, approve, copy the code.

**Skill does:** exchange at `POST https://api.etsy.com/v3/public/oauth/token` including the original `code_verifier`. Store the refresh token.

**Verify - also collects the shop ID:**
```
GET https://api.etsy.com/v3/application/users/me
x-api-key: {KEYSTRING}
Authorization: Bearer {access_token}
```

Then `getShopShippingProfiles` - **confirm at least one shipping profile exists.** A missing one causes the `readiness_state_id` failure that blocks every physical listing. Catch it here, not mid-batch.

**Tell them two things:**
- Refresh token expires in **~90 days**, so this recurs quarterly. Not a mistake, just the cadence.
- **No sandbox exists.** Development runs against the live shop at $0.20/listing, so we work in draft state and never publish while testing.

---

## Gate 5 - Square → Meta (Facebook Shops + Instagram)

Free, official, no browser automation, and it reaches an existing Instagram audience. Worth doing.

**Human does:**
1. Confirm a Facebook **business Page** exists (not just a personal profile)
2. Meta Business Manager account
3. **Domain verification** for the Square Online site domain
4. Square Dashboard → **Channels → Meta for Business** → connect

**Verify:** catalog items appear in Meta Commerce Manager.

**Set expectations:** Meta phased out native Shops checkout in 2025. This is a discovery and catalog surface that drives traffic to Square Online checkout - not in-app transactions. Real value, but not organic sales on its own.

---

## Gate 6 - Facebook Marketplace session

No credentials. Nothing in `.env`.

**Human does:** log into Facebook in Chrome themselves.

**Skill does:** confirm the Marketplace create form is reachable in that session.

**State the boundaries once:**
- The skill fills the entire form, then **stops before submit**. They review and publish.
- ~50 listings/day, spaced. The skill paces itself.
- The skill never enters a password. If a login screen appears, they take the keyboard.

---

## Gate 7 - Seller profile

An interview, not a form. Ask conversationally, accept "skip" on anything, and note that everything is editable later.

Write to `seller_profile.md`:

- Business/brand name as it should appear on listings
- Ship-from ZIP (needed for shipping calculation)
- Shipping approach - flat, calculated, or free-with-price-built-in; who pays
- Handling time for stock items
- Return policy - accepted? window? who pays return shipping?
- Care instructions boilerplate (by material, if it varies)
- Local pickup - offering it? where?
- Custom order lead time
- **Anything they already paste into every listing by hand** - ask this one directly; most sellers have it and it's the highest-value answer in the interview
- Per-platform overrides where eBay and Etsy conventions differ

---

## Done

Summarize concretely: which destinations are live and verified, what the Square test showed, when the Etsy token expires, and anything still outstanding.

Then propose the real next step: *"Ready for a test batch. Pick 3–5 items - ideally one handmade original with no logos, one with team or character imagery, one you bought to resell, and one that comes in multiple colors. That exercises every path on real data before we point this at the whole backlog."*
