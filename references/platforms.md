# Platforms

## eBay - API

**Use the Trading API `AddFixedPriceItem`, not the Inventory API.**

Why: listings created through the Inventory API **cannot be edited in Seller Hub** - every revision must go back through the API, and eBay's own advice for a stuck listing is to end and relist, losing watchers and item history. Trading listings stay ordinary eBay listings the user can fix from their phone. A 30-variation listing is also **one** Trading call versus ~6 Inventory calls plus one-time setup of a merchant location and three business policies.

**Two operational constraints:**

1. **Never call the Inventory API on the production account.** Doing so opts the account into Business Policies *account-wide*, after which `AddFixedPriceItem` stops accepting inline `ShippingDetails` / `ReturnPolicy` / `PaymentMethods` and requires `SellerProfiles` IDs instead. Check `SellerProfileOptedIn` via `GetUserPreferences` before starting. Experiment in sandbox only.
2. **Images: use the Media API.** `UploadSiteHostedPictures` was decommissioned 2026-09-30. Use `createImageFromFile` / `createImageFromUrl`; the returned EPS URL drops into `Item.PictureDetails.PictureURL`. A single listing **cannot mix** EPS-hosted and self-hosted URLs.

**Business Policies - check the account state first.** This changes field mapping, not API choice:

| Account state | `AddFixedPriceItem` uses |
|---|---|
| Opted in | `Item.SellerProfiles` → `SellerShippingProfile`, `SellerReturnProfile`, `SellerPaymentProfile` IDs |
| Not opted in | Inline `ShippingDetails`, `ReturnPolicy`, `PaymentMethods` |

Check via `GetUserPreferences` with `ShowSellerProfilePreferences=true` → `SellerProfilePreferences.SellerProfileOptedIn`, or in the UI at My eBay → Account → Business Policies. Both states work fine; just branch the mapping.

**Auth:** OAuth 2.0 user token (recommended over legacy Auth'n'Auth). Pass to Trading via the `X-EBAY-API-IAF-TOKEN` header rather than the `<eBayAuthToken>` element. The Media API requires OAuth regardless.

**Variations:** `Item.Variations` / `Variations.Variation`. Max 250 variations, 5 variation detail sets, 30–50 values each (docs conflict; test near the boundary). `VariationSpecificPictureSet` holds up to 12 images. **Per-variation images vary along one axis only** - images can change by Team *or* by Size, not both.

**Shipping:** default listings to **shipping-enabled**. Only shipping-enabled eBay listings are eligible to surface on Facebook Marketplace through eBay's Meta partner integration - that's free distribution for zero extra work. Local-pickup-only listings are excluded.

**Isolate the XML.** Keep all Trading calls behind an internal `createListing(item)` interface. eBay decommissions Trading calls piecemeal, roughly quarterly, with ~10–12 months notice. Contained module = contained migration.

**Facebook Marketplace via eBay:** opt-out by default, no cost, US included. eBay and Meta select which listings surface based on trends and listing quality. Frame this to the user as a bonus, never a guarantee - there's no control over which items appear and no Marketplace-side analytics.

## Etsy - API

**Eligibility first.** Etsy accepts only handmade, 20+ year vintage, and craft supplies. Check classification before generating anything - resale items and third-party designs are blocked (see `guardrails.md`).

**Auth:** `x-api-key` header **plus** an OAuth 2.0 bearer token. OAuth uses authorization code with **mandatory PKCE** (`S256`). Access token 1 hour, refresh ~90 days. Scope `listings_w` to create.

**App tier:** the **Seller App** track approves in minutes for a seller working on their own shop. Personal and Commercial tracks involve real review - not needed here.

**No sandbox.** Development happens against the live shop at **$0.20 per listing**. Use draft state aggressively; never publish during development.

**Create flow:** `createDraftListing` → `uploadListingImage` → `updateListingInventory` → `updateListing` with `state: "active"`.

Required on create: `quantity`, `title`, `description`, `price`, `who_made`, `when_made`, `taxonomy_id`. Physical items also need `shipping_profile_id` and **`readiness_state_id`** - the latter is a common breakage source since Processing Profiles launched; a listing without a properly assigned shipping profile returns a null readiness state and then fails on update.

**Variations:** `updateListingInventory` with a `products[]` array; each product is one purchasable combination of `property_values` plus `offerings`. Usable properties are driven by `taxonomy_id` - discover with `getPropertiesByTaxonomyId`. Prefer predefined `value_ids` over freeform `values` so variations appear in buyer-facing search filters. **Max 2 variation properties** (3 in preview, not generally available).

**Personalization** is a **separate two-step call**, not part of `createDraftListing`:
`POST /v3/application/shops/{shop_id}/listings/{listing_id}/personalization?supports_multiple_personalization_questions=true`

Up to 5 questions. Types: `text_input`, `dropdown`, `unlabeled_upload`, `labeled_upload` - max one upload-type question per listing. `question_text` 1–45 chars, `instructions` ≤120 chars, `max_allowed_characters` 1–1024, dropdown options ≤30. `add_on_price` ($0.20–$500) is allowed **only** on optional `text_input` questions.

**Cost:** $0.20 per listing, valid 4 months, charged **per listing not per variation** - so a 30-variant listing costs $0.20 total. Plus 6.5% transaction fee. A bug that mass-creates listings costs real money; validate before publishing.

**Rate limits:** 5 QPS, 5,000 QPD (confirmed on a live personal app, 2026-08-19). Etsy's docs cite higher figures as examples - trust the number shown on the app's own dashboard.

## Square - API

**Auth: personal access token** from the Developer Console. No OAuth server, no app review, no approval gate for a seller's own account. Lowest friction of any destination. Separate sandbox and production tokens - mixing them is an auth error.

**Create:** `upsertCatalogObject` / `batchUpsertCatalogObjects` (≤1,000 objects/batch). An `ITEM` **must** have at least one `ITEM_VARIATION`. Use `#`-prefixed temporary IDs to reference new objects within one request; include an `idempotency_key`.

**Variations:** prefer **item options** - create `CATALOG_ITEM_OPTION` + `ITEM_OPTION_VAL` objects, then reference `item_option_id` + `item_option_value_id` per variation. Square auto-generates a variation for every combination and composes display names. Freeform variation names lead to the inconsistency Square itself warns about. Max **250 variations** per item.

**Images:** `CreateCatalogImage` (multipart), attached by `object_id`. Use `image_ids` - `ecom_uri` and `ecom_image_uris` are deprecated.

**Online visibility: not settable, and not something you need to set.** The earlier version of this section was wrong and produced bad instructions to the seller. Corrected 2026-08-22 after creating a 29-variation item through `batchUpsertCatalogObjects` and checking the live storefront.

What was verified:

- **The item went live on the Square Online site with no dashboard step at all.** No "set to Listed", no Publish in the site editor. It was searchable and purchasable within seconds of the API call, with the variation dropdown, per-variation photos, and stock counts all working.
- The item's Channels panel already had the Square Online site checked and "Hide item from browsing & search on websites" off, by default, from the API create.
- **Square removed the Listed / Unavailable model.** Its own note in that panel reads: "We've changed how to remove items from your website. Instead of marking items as Unavailable, deselect the website above." Any instruction written against the old UI will not match what the seller sees, which is exactly how this section went stale.

What is still true: `CatalogItem` exposes no writable visibility field. `channels` is documented read only, and Square staff confirmed on the developer forum that the `ecom` fields "are read only and aren't documented even though they are returned in the API response." So visibility cannot be changed through the API. It also does not need to be, because a new item defaults to visible.

**Never tell the seller to go flip a switch before checking whether one is needed.** Hit the storefront, or `GET /v2/catalog/object/{id}` and read the returned `channels`. The item is probably already live.

**`batchUpsertCatalogObjects` returns variations nested, not flat.** When variations are sent inside `item_data.variations`, they come back inside the returned ITEM, and `objects[]` contains no top-level `ITEM_VARIATION` entries. Counting top-level variations returns zero, which looks like a failed create and is not. Read `object.item_data.variations`. Getting this wrong also hands the follow-up inventory call an empty `changes` array, which fails with `VALUE_EMPTY`.

**Per-variation images work.** `CatalogItemVariation` accepts `image_ids`, verified with 29 variations each carrying its own photo; the storefront swaps the image when the buyer picks a variation. Upload with `CreateCatalogImage` first (no `object_id`), collect the returned ids, then reference them per variation in the batch upsert. Item-level `image_ids` and per-variation `image_ids` can both be set on the same item.

Still outside the Catalog API and still required for an item to transact: every variation priced, stock tracked, and fulfillment methods configured for the site.

**Custom-order capable listings:** `CatalogModifierList` with `modifier_type: "TEXT"` (fields `max_length`, `text_required`) captures buyer-entered text at checkout. Works on Square Online; **not** on Payment Links. Note for the user at setup: the entered text is **not retrievable via the Orders API** - they'll read it off the order ticket or receipt. Verify with one real test order.

**Bulk-pack variations** ("25-pack", "50-pack") are the simplest path to self-serve bulk orders and need no special API support - they're just variations with their own prices.

**Facebook Shops + Instagram via Square:** Dashboard → **Channels → Meta for Business**. Official, self-serve, $0. Requires a Meta Business Manager account, a Facebook business page, a Meta catalog, and domain verification. Reaches **Shops and Instagram Shopping, not Marketplace**, which is a separate browser-driven surface. Meta phased out native Shops checkout in 2025, so this drives traffic to Square Online checkout rather than transacting in-app.

**⛔ Treat this channel as unavailable until proven otherwise, and prove it from the Manage page.** Two failure modes, both seen live on 2026-08-22:

1. **The connection silently breaks.** Square's Channels *list* showed a green **Connected** badge while the channel's own **Manage Channel** page read `Status: Disconnected` with "There was an error authenticating with Meta." The list badge is not trustworthy. Always open Manage and read `Status:` there. Re-authenticating requires the seller to sign in to Meta and click Resolve; never do this for them.

2. **Domain verification is structurally impossible on a Square subdomain.** Meta verifies three ways, a DNS TXT record, an HTML file at the site root, or a meta tag in the page head. A `*.square.site` store gives the seller none of them, so the Verify button can never succeed and product visibility stays capped. Only a custom domain clears it.

**Verify from the public side, not the dashboard.** An Instagram profile with Shopping live shows a fourth tab, Shop, alongside Posts, Reels and Tagged. If it shows three, nothing is syncing regardless of what any badge says.

## Facebook Marketplace - browser

No API. Drive Chrome to fill the create form completely - photos uploaded, title, description, price, category, condition, location - then **stop before submit**.

Never click submit. The user reviews and publishes.

**Constraints:**
- ~50 listings/day practical ceiling; space listings several minutes apart
- Meta bans C2C sellers who "sell as a business" - a high volume of similar listings from one personal account is that signature. For catalog-scale posting, Square → Meta (Shops/Instagram) is the right channel instead; Marketplace is best for individual higher-interest local-pickup items.
- The form changes; expect breakage. A Marketplace failure must never block API destinations.

**Out of bounds:** no IP rotation, no fingerprint randomization, no CAPTCHA circumvention, no timing patterns tuned to defeat bot detection. Filling a form in the user's own logged-in session is a different act from circumventing access controls - and in practice, evasion engineering is what turns soft enforcement into a terminated account.

## Not supported

| Platform | Why |
|---|---|
| **Amazon** | BSA §19 Agent Policy (2026-03-04) requires automated agents to self-identify, which is disqualifying for browser automation. Amazon Custom's customization layer has no API at all. Consequences include permanent withholding of funds. |
| **OfferUp** | ToS bans automation *and* third-party applications without written consent. |
| **Craigslist** | ToS names posting software and carves out only "general purpose web browsers." $1,000/violation liquidated damages with an actual litigation record. Bulk API excludes for-sale-by-owner. |
| **Mercari / Poshmark** | No listing API. Extension-only. |
| **Nextdoor** | FSF API exists but access terms exclude "promoting your own business." |
