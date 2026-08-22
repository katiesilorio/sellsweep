# Sellsweep - Product Decisions Log

**Status:** Planning
**Last updated:** 2026-08-17
**Purpose:** Living record of the problem, the decisions, and the reasoning behind them. Source material for the public README.

**Note on specifics:** the MVP user is a real working seller. Identifying details (business name, location, account handles) are generalized here since this repo is public. The operational detail lives in the private build repo.

---

## 1. The Problem

Selling an item online means re-creating the same listing five times.

Every marketplace wants the same underlying information - photos, a title, a description, a price, a category, a condition - but each one wants it in a different shape, with a different character limit, a different category taxonomy, and a different set of required attributes. eBay wants "item specifics." Etsy wants "who made it" and "when was it made." Shopify wants options and variants. Facebook Marketplace wants none of that but has its own form.

The result is that the *thinking* work - deciding what this thing is, what it's worth, and how to describe it so it sells - gets done once, and then the *transcription* work gets done five times. The transcription is the tax, and it's the reason most casual sellers list on one platform and stop.

**Sellsweep's job: do the thinking work once, then eliminate the transcription tax.**

### Why the variation case matters

A specific and under-served version of this problem: **one product, many variations.** A mug with 30 different sports team designs. A t-shirt in 5 colors and 4 sizes. The transcription tax doesn't scale linearly here - it multiplies. Thirty near-identical listings, each needing its own photo assignment and attribute set, is where manual listing stops being annoying and becomes genuinely prohibitive.

This is also where the platforms diverge most sharply in how they model the problem, which makes it the most interesting design constraint in the product.

---

## 2. Who This Is For

Two distinct sellers, sharing one core engine:

| | **Scenario A - Product Line** | **Scenario B - Declutter** |
|---|---|---|
| **What they sell** | A repeatable product, often with variations | One-off used household items |
| **Example** | Custom drinkware, 30 team designs | Old couch, kid's bike, a lamp |
| **Volume shape** | Deep (many variants of one thing) | Wide (many unrelated things) |
| **Where it sells** | eBay, Etsy, Shopify, Square | Facebook Marketplace, OfferUp, Craigslist |
| **Platform reality** | Every target platform has a real API with native variation support | Almost no target platform has any API at all |

**These two scenarios have inverted difficulty.** Scenario A sounds harder - variations, inventory, multiple SKUs - but every platform it touches offers a documented API that models variations as a first-class concept. Scenario B sounds trivially simple - one item, one listing - but the platforms it depends on are precisely the ones with no programmatic access.

**Decision: MVP builds Scenario A first.** See §5.

---

## 3. The Constraint That Shaped Everything

The original concept was "posts everywhere." Research into the actual platform landscape (Aug 2026) established that "everywhere" is not purchasable at any price, and that the platforms with the *most* consumer demand have the *least* programmatic access.

This is the central product constraint, and rather than either overpromising or abandoning the goal, Sellsweep's response is to **tier platforms by what is genuinely achievable and be explicit about which tier each one is in.**

### Platform access research - findings

| Platform | Listing API? | Notes |
|---|---|---|
| **eBay** | ✅ Full public API | Inventory API. Native multi-variation via `InventoryItemGroup`. Built for used goods. |
| **Shopify** | ✅ Full public API | GraphQL Admin API, `productSet`. 3 options / 2,048 variants max. |
| **Square** | ✅ Full public API | Catalog API, item options auto-generate variation combinations. Lowest auth friction of any platform. |
| **Etsy** | ✅ Full public API | Seller App approval in minutes. **Category-restricted** - handmade / 20+ yr vintage / craft supplies only. |
| **Depop** | ⚠️ Partner API | Real, actively granted (Vendoo went live Jul 2026). Requires application. |
| **Facebook Marketplace** | ❌ No API | See §3.1 - reachable indirectly. |
| **OfferUp** | ❌ No API | ToS bans automation *and* third-party apps without written consent. |
| **Craigslist** | ❌ No usable API | Bulk API exists but excludes for-sale-by-owner; dealer access is phone-gated, hundreds of posts/month, $5 each. |
| **Nextdoor** | ❌ Excluded by policy | FSF API is real, but access terms state it is "not available for promoting your own business." |
| **Mercari / Poshmark** | ❌ No API | Extension-automation only. Poshmark's sanctioned route (via Dsco) syncs price/quantity only. |

### 3.1 The Facebook Marketplace finding

Facebook Marketplace has no listing API and no sanctioned self-serve path. Specifically verified as **dead**:

- Shopify's, Square's, and BigCommerce's official Meta sales-channel apps sync to **Facebook Shops and Instagram Shopping only** - not Marketplace.
- Meta's Commerce Platform catalog API is partner-gated ("contact your representative"), and its documented prerequisite - Checkout on Facebook - was itself deprecated in August 2025.
- Meta discontinued automated inventory catalog listings on Marketplace in Sept 2021, and killed vehicle/real-estate Page listings in Jan 2023. The trend is contraction.
- In July 2026 Meta shipped a standalone **Seller app** for high-volume Marketplace sellers - with AI listing creation and in-app bulk listing. Building a manual tool for power sellers is what a platform does *instead of* opening an API.

**But there is a working indirect path.** Since January 2025, eBay listings surface natively inside Facebook Marketplace through an official Meta partner integration:

- US included, **opt-out rather than opt-in** - participation is the default
- No additional cost to the seller
- **Requires shipping to be enabled** on the listing (local-pickup-only is excluded)
- eBay and Meta select which listings surface, based on trends and listing quality
- Meta extended the same arrangement to Poshmark in Nov 2025; partner listings carry an icon in the Marketplace feed

**Decision: treat Facebook Marketplace reach as a property of the eBay integration, not as a separate integration.** Sellsweep defaults every eBay listing to shipping-enabled specifically so it qualifies for Marketplace distribution. This buys real Marketplace presence for zero additional integration work and zero platform risk.

The trade-off is honest and worth stating: no control over which items surface, no guarantee any given item appears, and no Marketplace-side analytics. It is distribution, not placement.

---

## 4. Platform Tiering - the core architectural decision

Every platform lands in exactly one tier, and the tier determines how Sellsweep treats it.

### Tier 1 - Direct API (fully automated)
**eBay · Shopify · Square · Etsy**

Sellsweep authenticates and creates the listing end to end. No human in the loop. Variations handled natively through each platform's own variation model.

*Rationale:* These are documented, supported, public APIs. Automation here is the intended use, not a workaround. Any tool not using them is leaving capability on the table.

### Tier 1b - Distribution rider (free reach, no integration)
**Facebook Marketplace · Poshmark**, via eBay

Not integrations. Consequences of Tier 1 done correctly. The only engineering requirement is defaulting eBay listings to shipping-enabled.

*Rationale:* Maximum reach for minimum surface area. This is the single highest-leverage decision in the plan.

### Tier 2 - Assisted fill (browser, human publishes)
**Facebook Marketplace (direct) · OfferUp**

Sellsweep drives Chrome, uploads the photos, and fills every field on the platform's own create form - then **stops before submit**. The seller reviews and clicks publish.

*Rationale, three reasons:*

1. **It removes the actual pain.** The transcription tax is retyping and re-uploading, not clicking a submit button. Assisted fill eliminates ~95% of the effort while leaving the last 5% deliberately manual.
2. **It's a genuine quality gate, not just a hedge.** A human seeing the fully-composed listing before it goes live catches wrong categories and mispriced items - exactly the errors that are expensive to fix after publishing.
3. **It materially reduces account risk.** Full auto-posting to these platforms violates their terms, and the consequences land on the seller's personal account, not on the tool. A human reviewing and publishing each listing is meaningfully different in both intent and behavioral signature.

*Explicitly out of bounds within Tier 2:* Sellsweep will not implement IP rotation, browser-fingerprint randomization, CAPTCHA circumvention, or timing patterns tuned to defeat bot detection. Automating a form in a session the user is already logged into is a different act from circumventing a platform's access controls - and in practice, evasion engineering is what converts soft enforcement into a terminated account.

### Tier 3 - Out of scope (documented, not forgotten)
**Craigslist · Nextdoor · Mercari · Poshmark (direct) · Depop (v1)**

| Platform | Why excluded |
|---|---|
| **Craigslist** | Terms name posting software explicitly and carve out only "general purpose web browsers." Liquidated damages of $1,000/violation, with an actual litigation record ($31M Instamotor settlement, $1M copyright settlement, CFAA claims sustained in *3Taps*). The sanctioned bulk API excludes for-sale-by-owner entirely; dealer access requires hundreds of posts/month at $5 each, approved by phone. Uniquely high risk, uniquely low reward. |
| **Nextdoor** | The FSF creation API is real and well-designed, but access terms state it is "not available for promoting your own business or your clients' businesses." Sellsweep's primary scenario is precisely that. Excluded on policy, not capability. |
| **Mercari** | No US API. Extension-only. |
| **Poshmark (direct)** | No listing API. Reached indirectly via Tier 1b instead. |
| **Depop** | Real partner API, genuinely obtainable - deferred to v2 purely on scope, not viability. Best candidate for the first post-MVP platform. |

---

## 5. Scope Decisions

### 5.1 Build Scenario A (product line + variations) first

**Decision:** MVP targets one product with many variations, before one-off used goods.

**Reasoning:**
- Every platform in Scenario A has a real API with first-class variation support. Scenario B depends on the platforms with no API at all - meaning it delivers less while costing more.
- Variations are where the pain is worst and the leverage is highest. Thirty listings collapsing into one operation is a visible, demonstrable win; one listing becoming slightly easier is not.
- It forces the hard modeling problem early. Getting the variation abstraction right up front is far cheaper than retrofitting it onto a single-item design.

**Rejected alternative - build Scenario B first:** simpler per listing, but leans entirely on Tier 2 browser work, which is the slowest, most fragile, and least demonstrable part of the system. Building the weakest surface first would have produced a worse MVP more slowly.

### 5.2 Build a skill, not an app

**Decision:** MVP ships as a Claude skill, not a web or mobile application.

**Reasoning:**
- The genuinely hard part is judgment - reading photos, inferring condition and attributes, pricing, and writing copy that converts. That is the model's work, not a UI's.
- A skill needs no auth system, no hosting, no image pipeline, and no frontend. The entire scaffolding budget goes into the actual problem.
- Chrome automation for Tier 2 is already available to a skill, running in the user's own logged-in browser session. An app would have to ship a browser extension to reach parity.
- It tests the core hypothesis - *does generated listing content actually hold up?* - without first building everything around it.

**Rejected alternative - full web app:** most of the build would have gone into account systems and image handling before the first useful listing existed, and the interesting question would have stayed untested longest.

### 5.3 Etsy is in scope specifically because of the variation case

Etsy prohibits general secondhand resale - only handmade, 20+ year vintage, and craft supplies / personalized goods. That rules Etsy out for Scenario B entirely.

But custom-designed drinkware is squarely "made, designed, handpicked or sourced by you," which is exactly what Scenario A produces. Etsy also has the fastest approval of any platform (Seller App, minutes) and charges per *listing* rather than per variation - so a 30-variant listing costs $0.20 total.

**Decision:** Etsy is in for Scenario A, and explicitly out for Scenario B. The skill enforces this rather than letting the user hit a policy violation.

---

## 6. Variation Modeling - the central technical problem

Every Tier 1 platform supports variations, and every one models them differently. Sellsweep needs a single internal representation that translates cleanly into all four.

| Platform | Model | Ceiling | Per-variation images |
|---|---|---|---|
| **eBay** | One inventory item + one offer per variation, joined by `InventoryItemGroup`; `variesBy.specifications` defines axes | 5 axes × 50 values, 250 total | ✅ via `aspectsImageVariesBy`, 12/variation |
| **Shopify** | `productOptions` + variants; `productSet` does it in one mutation | **3 options**, 2,048 variants | ✅ per variant |
| **Square** | `CATALOG_ITEM_OPTION` + values; Square auto-generates every combination | 250 variations | ✅ per variation |
| **Etsy** | `products[]` with `property_values` + `offerings` | **2 properties** (3 in preview) | ❌ listing-level only, 20 images |

**Binding constraints, set by the strictest platform in each dimension:**
- **Max 2 variation axes** in MVP (Etsy's limit; 3 in preview but not generally available)
- **Max 250 variations** (eBay and Square)
- **Per-variation images are not universal** - Etsy can't do it, so the internal model treats per-variation imagery as an enhancement, never a requirement

**Decision:** Sellsweep's internal model is `{axes: [{name, values[]}], combinations: [{axis_values, sku, price, qty, image_ref?}]}` - designed against the *intersection* of platform capabilities, then enriched per platform where the target supports more. Validation happens before any API call, so limit violations surface as a clear message rather than a rejected request four platforms in.

---

## 7. Non-Goals

Named explicitly so scope creep has to argue its way in:

- **Not an inventory management system.** Sellsweep creates listings. It does not reconcile stock across platforms or handle the sold-here-so-delist-there problem. That is a real problem and a different product.
- **Not order or fulfillment management.** No shipping labels, no buyer messaging, no returns.
- **Not a pricing engine.** Sellsweep suggests a price from comparables and item condition. It does not reprice dynamically or track competitors.
- **Not multi-user.** Single seller, own accounts, own credentials. No multi-tenancy, no seller onboarding.
- **Not a bot.** See Tier 2 boundaries in §4.

---

## 8. Open Questions - RESOLVED 2026-08-17 (see §11)

- [ ] Where does price research come from - eBay sold comparables via API, or model judgment with a confidence range?
- [ ] How much does the skill ask vs. infer? Each question improves the listing and costs patience. Where is the line?
- [ ] What is the artifact between generation and posting - a reviewable file the user edits, or a conversational confirm?
- [ ] Should photo quality be assessed and re-shoots suggested, given that image quality drives both conversion and eBay's Marketplace surfacing eligibility?
- [ ] eBay's Inventory API creates listings that **cannot be edited in Seller Hub** - all edits must return through the API. Is that one-way door acceptable, or is the Trading API's `AddFixedPriceItem` the safer choice despite being the older path?
- [ ] Does Square Online actually auto-publish API-created items via dashboard item defaults? Documented behavior is ambiguous and needs empirical testing.

---

## 9. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Tier 2 browser automation breaks when Meta or OfferUp change their DOM | High likelihood, medium impact | Treat Tier 2 as best-effort; never let a Tier 2 failure block Tier 1 posting |
| eBay's Marketplace surfacing is discretionary and could end without notice | Medium | Tier 1b is framed as a bonus, never as a promised feature |
| eBay new-seller monthly selling limits (item count + dollar value) throttle a 30-variant launch | Medium | Check limits before publishing; warn on projected overage |
| Etsy has no sandbox - development happens against the live shop, at $0.20 per listing | Low cost, high annoyance | Use draft state aggressively; never publish during development |
| Generated copy is fluent but generically wrong, and the seller doesn't notice | **Highest product risk** | Human review gate before publish; treat listing quality as the thing to test first |

---

## 10. MVP Success Criteria

Sellsweep v1 succeeds if, for a single product with up to 20 variations:

1. Photos plus a short Q&A produce a complete, accurate listing - title, description, category, attributes, price - with no manual rewriting
2. It publishes to eBay, Etsy, and one of Shopify/Square via API, with variations intact
3. eBay listings default to shipping-enabled and therefore qualify for Facebook Marketplace distribution
4. It fills the Facebook Marketplace and OfferUp forms and stops for review
5. **Total human time from photos to live-everywhere is under 10 minutes** - versus roughly an hour doing it by hand

The honest test of #1 is not whether the copy reads well. It is whether the seller publishes it unedited.

---

## Decision Log

| Date | Decision | Rationale |
|---|---|---|
| 2026-08-17 | Named "Sellsweep" | "Sweep" = covering everything in one pass; "sell" keeps the outcome legible. Chosen over Sellreach for sound, accepting it needs a tagline. |
| 2026-08-17 | Ship as a skill, not an app | The hard part is judgment, not UI. Tests the core hypothesis fastest. |
| 2026-08-17 | Scenario A (variations) before Scenario B (declutter) | Every Scenario A platform has a real API with native variation support; Scenario B depends on platforms with none. |
| 2026-08-17 | Four-tier platform framework | "Post everywhere" is not achievable. Tiering makes the constraint explicit instead of overpromising. |
| 2026-08-17 | Facebook Marketplace via eBay, not directly | Sanctioned, free, zero additional integration, zero platform risk. Trade-off: no control over which items surface. |
| 2026-08-17 | Tier 2 stops before submit | Removes ~95% of the effort, adds a real quality gate, keeps account risk low. |
| 2026-08-17 | No evasion engineering | Automating your own logged-in session ≠ circumventing access controls. Also the practical line between soft enforcement and a dead account. |
| 2026-08-17 | Craigslist and Nextdoor out of scope | Craigslist: litigation record, owner listings ineligible. Nextdoor: access terms exclude promoting your own business. |
| 2026-08-17 | Variation model built to platform intersection | 2 axes / 250 combinations, enriched per platform. Validate before calling any API. |

---

## 11. Resolved Design Decisions (2026-08-17)

### 11.1 Pricing - a cascade, because the good source is closed

**Finding that forced the design:** eBay's **Marketplace Insights API** is the only official surface for sold-comparable data, and it is not obtainable. eBay Partner Network's developer questionnaire states access *"cannot be granted upon request"* - it is invitation-only for major partners, with no application path and no published criteria. The legacy `findCompletedItems` was decommissioned February 2025. Terapeak has no public API.

**Decision: a four-tier cascade with an explicit confidence floor.**

| Tier | Source | Access | What it gives |
|---|---|---|---|
| 1 | **PriceCharting API** | Self-serve, ~$49/mo | Genuine licensed sold comps - vertical-limited (video games, TCG, comics, Funko, LEGO, coins) |
| 2 | **Terapeak**, via the user's own Seller Hub session | Delegated | Avg sold price + sell-through, 2–3 years of history |
| 3 | **eBay Browse API** | Free, 5,000/day, no approval | *Active asking* prices - an upper bound, not realized value |
| 4 | **Model judgment + explicit range** | Always | The floor. Never silently absent. |

**Threshold rule:** below **5 sold comparables**, Sellsweep does not report a point price. It reports a range, labels confidence as low, and names which tier the number came from. Provenance is always surfaced - the user should never see a number without knowing where it came from.

*Rationale for Tier 2 - delegated access:* this is the pattern the market leader uses. Vendoo's price checker requires the user to connect their own eBay account with Seller Hub enabled, then reads Terapeak - free to every eBay seller since 2021. Making the *user's* account the data credential scales per-user and doesn't depend on a platform approval that cannot be obtained.

*Boundary:* eBay's User Agreement prohibits *"any robot, spider, scraper... or other automated means (including, without limitation buy-for-me agents, LLM-driven bots...)"* - a clause that names this category explicitly. Reading Terapeak within the user's own authenticated Seller Hub session is the industry-standard interpretation; scraping eBay search results is not, and is out of scope.

*Rejected:* Amazon Creators API (requires ~10 qualifying affiliate sales per 30 days), Google Custom Search (closed to new signups, shuts down Jan 2027), scraper APIs such as SoldComps/SerpApi (the only programmatic route to eBay sold data, and all of them violate eBay's User Agreement).

### 11.2 eBay - Trading API, not Inventory API

**Decision: build on Trading API `AddFixedPriceItem`**, with OAuth 2.0 and the Media API for images.

| | Trading `AddFixedPriceItem` | Inventory API |
|---|---|---|
| **Editable in Seller Hub** | ✅ Yes | ❌ **No** - revisions must return through the API |
| **Calls for a 30-variation listing** | **1** | ~6 bulk calls + one-time setup |
| **Prerequisites** | None | Merchant location + all 3 business policies |
| **Business policies** | Optional (inline shipping/return/payment) | **Mandatory** |
| **Deprecation posture** | Not deprecated (v1475, Jun 2026), retired call-by-call | Where eBay is investing |

*Rationale:* the editability constraint is documented and one-directional. eBay's own guidance for a locked Inventory listing is to end and relist - losing watchers and item history. For a personal tool where hand-editing a price or typo is routine, that cost dominates the periodic small migrations Trading requires.

*Honest counter-position:* eBay explicitly recommends the Inventory API for new integrations, and decommissions Trading calls piecemeal on a roughly quarterly cadence with ~10–12 months notice. This decision buys hand-editability at the price of recurring small maintenance. **If Sellsweep ever became a multi-seller commercial product where nobody hand-edits, this decision should flip.**

**Two operational constraints this creates:**

1. **Never call the Inventory API on the production seller account.** Doing so opts the account into Business Policies *account-wide*, which changes `AddFixedPriceItem` behavior everywhere - inline `ShippingDetails` / `ReturnPolicy` / `PaymentMethods` stop working and `SellerProfiles` IDs become required. Check `SellerProfileOptedIn` via `GetUserPreferences` before starting. Experiment in sandbox only.
2. **`UploadSiteHostedPictures` is decommissioned 2026-09-30.** Use Media API `createImageFromFile` / `createImageFromUrl` from day one; the returned EPS URL drops directly into `Item.PictureDetails.PictureURL`. Also note: a single listing cannot mix EPS-hosted and self-hosted image URLs.

**Mitigation:** all eBay XML lives behind an internal `createListing(product)` interface, so a forced migration is a contained rewrite rather than a rebuild.

*Trading variation limits:* 250 variations, 5 variation detail sets, 12–24 images per variation. Per-variation images vary along **one** axis only (e.g. images change by Team, not by Team × Size).

### 11.3 Input model - required vs. optional vs. profile

**Decision: exactly one required input.**

| Input | Status |
|---|---|
| **Photo(s)** | **Required** - the only hard requirement |
| Price, or lowest acceptable price | Optional, encouraged |
| Keywords / key features | Optional, encouraged |
| Disclaimers | Optional, encouraged |

*Rationale:* every question improves the listing and spends patience. Making anything beyond a photo mandatory raises the activation cost of the core promise - point a camera, get a listing. Optional-but-encouraged inputs are prompted once, skippable, and never blocking.

**Seller profile (captured once at setup):** standard boilerplate that applies to every listing - shipping/handling language, return terms, care instructions, shop policies. Many eBay sellers maintain exactly this and paste it into every listing manually.

Composition rule: `[generated item-specific description] + [profile boilerplate]`, with the boilerplate **overridable per platform** (Etsy and eBay have different conventions and different audiences). Editable at any time, not just at setup.

### 11.4 Review artifact - CSV, with a photo folder

**Decision: the artifact between generation and posting is a CSV the user reviews and edits, paired with a folder of photos referenced by filename.**

- One row per listing (or per variation, for variation listings)
- A column referencing photo filenames in the accompanying folder
- Supports **bulk import**: hand Sellsweep a folder of photos plus a CSV and it processes the batch
- Tier 1 platforms post the whole batch via API; Tier 2 platforms still fill forms one at a time after review, which is expected and acceptable

*Rationale:* a CSV is reviewable, diffable, and editable in tools the user already knows. It creates a natural checkpoint between generation and posting - which is the mitigation for the highest-rated product risk (fluent-but-wrong generated copy). It also makes bulk a first-class path rather than a later bolt-on.

*Open sub-questions:* photo-filename convention for variation rows; whether variations are one row each or one row with a variation column.

### 11.5 Photo quality assessment - in scope

**Decision: Sellsweep assesses photo quality and suggests re-shoots before generating a listing.**

*Rationale:* image quality drives conversion directly, and eBay states that listings surfaced on Facebook Marketplace are selected partly on "listing quality" - so photos affect Tier 1b reach, not just click-through. Flagging a bad photo before listing costs seconds; discovering it after costs a relist. Checks include resolution, blur, lighting, background clutter, and whether the item is fully in frame.

### 11.6 Setup capture - build the setup log as we go

**Decision: record the account-connection and credential setup steps *as they are performed*, then fold that log into the skill as an onboarding flow.**

*Rationale:* setup is a one-time, high-friction, easy-to-forget process - the exact thing that is expensive to reconstruct from memory later and painful to repeat on a new machine. Captured live, it becomes both the skill's onboarding path and honest portfolio documentation of real integration friction. Deliverable: `SETUP_LOG.md`.

---

## 12. Open Questions (current)

- [ ] Does Square Online auto-publish API-created items via dashboard item defaults? **Test protocol defined - see §13.** Outcome determines whether Square stays in MVP.
- [ ] Photo-filename convention for variation rows in the bulk CSV
- [ ] One CSV row per variation, or one row with a variation column?
- [ ] Does PriceCharting's $49/mo tier justify itself before the product has a proven vertical, or should Tiers 2–4 ship first and Tier 1 be added only if a covered vertical becomes the actual use case?
- [ ] Etsy has no sandbox - what is the safe development protocol against a live shop? (Draft state only, never publish during dev.)

---

## 13. Square Auto-Publish Test Protocol

**Question:** does an item created via the Catalog API appear on a Square Online site automatically, or does it require a manual dashboard step?

**Why it's uncertain:** `CatalogItem.channels` is documented as read-only, and `ecom_visibility` is undocumented - Square staff have confirmed online visibility is not programmatically settable. A dashboard "item defaults" setting is the only known workaround, and its reliability has unresolved complaints dating to 2025.

**Protocol:**

1. Square Dashboard → Items & Orders → Settings → **item defaults**. Set new-item site visibility to **Listed**; assign new items to the Square Online site. Screenshot the configuration.
2. Developer Console → Credentials → copy the **Sandbox** access token.
3. Run `batchUpsertCatalogObjects` creating one `ITEM` with two `ITEM_VARIATION`s.
4. Check the Square Online site: does the item appear, and is it purchasable, with **no** dashboard interaction?
5. Repeat once in **production** with a disposable item - sandbox and Square Online do not always behave identically. Delete afterward.

**Decision rule:**
- **Passes** → Square stays a full Tier 1 integration.
- **Fails** → Square degrades to "creates the catalog item; user flips visibility manually," which is a materially weaker integration and is likely grounds to cut Square from MVP in favor of Shopify.

---

## Decision Log (continued)

| Date | Decision | Rationale |
|---|---|---|
| 2026-08-17 | eBay Trading API over Inventory API | Inventory listings can't be edited in Seller Hub; 1 call vs. 6 for variations. Accepts recurring small migrations as the cost. |
| 2026-08-17 | Never touch Inventory API on the production account | It opts the account into business policies account-wide and changes `AddFixedPriceItem` behavior. |
| 2026-08-17 | Media API for images from day one | `UploadSiteHostedPictures` decommissions 2026-09-30. |
| 2026-08-17 | Four-tier pricing cascade | Marketplace Insights is invitation-only and unobtainable - "cannot be granted upon request." |
| 2026-08-17 | Delegated access for pricing (user's own Terapeak) | The pattern Vendoo uses. Free to every eBay seller, 2–3 yrs of data, no platform approval needed. |
| 2026-08-17 | Confidence floor at 5 comps | A confident wrong price is worse than an honest range. Always surface provenance. |
| 2026-08-17 | Photo is the only required input | Every question spends patience. Protects the core promise: point a camera, get a listing. |
| 2026-08-17 | Seller profile for boilerplate, overridable per platform | Mirrors what eBay sellers already do by hand; platforms differ in convention and audience. |
| 2026-08-17 | CSV + photo folder as the review artifact | Reviewable, editable, diffable. Makes bulk a first-class path and creates the quality checkpoint. |
| 2026-08-17 | Photo quality assessment in scope | Drives conversion and eBay's Marketplace surfacing eligibility. |
| 2026-08-17 | Capture setup steps live into SETUP_LOG.md | One-time, high-friction, expensive to reconstruct. Doubles as portfolio documentation. |

---

## 14. MVP User & Platform Set (2026-08-17)

### 14.1 The MVP user is a real seller, not a hypothetical

**Decision: the MVP user is a working custom-goods seller (the shop's Instagram) - a family member.** He makes most items himself and buys some in bulk to resell. He does not outsource fulfillment.

*Why this matters to the design:* setup friction is now a first-class concern, not an afterthought. The person doing setup is not the person who built the tool. Onboarding has to be a documented, followable path - which is why §11.6 (`SETUP_LOG.md`) exists.

**Existing accounts:** Facebook Marketplace, eBay, Etsy, Square. Every platform in the MVP set is one he already has, plus Amazon Custom as the single new addition.

### 14.2 Two product classes with different legal routing - a core skill behavior

His catalog splits into two classes that are **not interchangeable across platforms**:

| Platform | Class A: made / personalized by him | Class B: bought in bulk, resold as-is |
|---|---|---|
| eBay | ✅ | ✅ |
| Square (+ FB Shops / Instagram) | ✅ | ✅ |
| Facebook Marketplace | ✅ | ✅ |
| **Etsy** | ✅ | ❌ **Policy violation** |
| **Amazon Handmade** | ✅ (hand-altered permitted) | ❌ **Policy violation** |
| Amazon Custom | ✅ | ✅ if personalized |

**Decision: Sellsweep classifies every item as Class A or Class B and refuses to route Class B to Etsy.** Etsy prohibits resale of mass-produced non-vintage goods; a violation is a strike against the shop.

*Rationale:* encoding platform policy into the tool is more valuable than maximizing reach. A tool that lets the user unknowingly violate a marketplace policy has actively harmed them. This turns compliance from a caveat in the docs into a product feature.

### 14.3 Square → Meta - added to setup

**Decision: add Square's official Meta channel to the setup sequence.**

Square Dashboard → **Channels → Meta for Business**. Official, self-serve, $0 marginal cost. Syncs the Square catalog to **Facebook Shops and Instagram Shopping**; catalog updates sync automatically. Requires a Meta Business Manager account, a Facebook business page, a Meta catalog, and domain verification.

*Important scope limit:* Meta phased out native Shops checkout (rollout June–August 2025). This is now a **discovery and catalog surface that drives traffic to Square Online checkout** - not a marketplace with its own transactions. Real value, but not organic sales on its own.

*Note the distinction:* Facebook **Shops/Instagram** (via Square, sanctioned and automatic) is a different surface from Facebook **Marketplace** (via the eBay rider in §3.1, or Tier 2 assisted fill). Sellsweep reaches both, by two unrelated mechanisms.

### 14.4 Amazon Custom - in scope. Amazon Handmade - out.

**Decision: use Amazon Custom. Skip Amazon Handmade entirely.**

*Why Custom over Handmade:*
- **Custom has no maker requirement.** Nothing in the program requires manufacturing the base product - buying blanks wholesale and personalizing them is the mainstream use. It therefore covers *both* product classes.
- **Handmade prohibits reselling third-party products**, so Class B items are ineligible regardless.
- Handmade's only advantage is a waiver of the $39.99/mo Professional fee - same 15% referral either way - in exchange for a ~2-week vetted application requiring production-process photos, and a **6-month reapplication penalty if rejected**.
- **Handmade cannot create listings via API at all** (offer sync only), which is disqualifying for a variation-heavy catalog.

**Setup path:**
1. Professional plan, $39.99/mo (Individual plan ineligible). Requires government ID, chargeable credit card, bank account, business license/registration, proof of address within 180 days, tax info. Identity verification by photo check or live video call, ~3 business days.
2. Register at `sellercentral.amazon.com/customization/manageCustomRegistration`. Short access request - no portfolio. Approval typically within an hour, up to 24 hours.
3. **Brand name + GS1 UPCs or a GTIN exemption.** Custom listings must be unique to the seller, so he creates his own ASINs. This is the real friction point - start early.
4. Handling time is settable per SKU up to 30 days. Standard FBM metrics still bind (ODR <1%, Late Shipment <4%).
5. **FBM only** - FBA is prohibited for customized products.
6. 15% referral (Home & Kitchen), $0.30 minimum, no Custom-specific fee. Customized products are not returnable (Feb 2023), though the flag is reported to apply inconsistently.

### 14.5 Amazon Custom is only half-automatable - and that's a design input, not a bug

**Finding:** the customization layer of an Amazon Custom listing **cannot be created through any API.** No feed type exists (`JSON_LISTINGS_FEED` is the only listings feed), nothing in the Product Type Definitions API exposes customization config, and SP-API release notes through August 2026 show no movement. Sellers have raised this at Amazon's own "Ask Amazon: Customized Products" events without response.

| Piece | Automatable |
|---|---|
| Base SKU/ASIN - title, price, images, quantity, handling time | ✅ SP-API Listings Items / `JSON_LISTINGS_FEED` |
| Customization schema - surfaces, text/image fields, option lists, fonts | ❌ Seller Central UI, or an in-console Excel → Unicode `.txt` upload |
| Buyer customization data on orders | ✅ Orders API + Restricted Data Token |

**Decision: Sellsweep treats Amazon as a Tier 1-partial platform.** It creates and maintains base SKUs via SP-API, and *generates the customization upload file* for the seller to load through the console. It does not pretend to fully automate Amazon.

*Mitigation:* customization templates are reusable per product family (11oz mug, 20oz tumbler, engraved slate), so marginal per-SKU cost drops sharply. Tolerable for dozens-to-low-hundreds of SKUs; a genuine bottleneck at thousands, with no compliant workaround.

**The higher-value Amazon automation is order-side, not listing-side.** Orders API + Restricted Data Token returns `BuyerCustomizedInfo.CustomizedURL` - a ZIP of the buyer's actual custom text and uploaded images - which can feed the production queue directly. **Apply for restricted-data access on day one;** small sellers consistently report this as the hardest gate, and without it fulfillment stays manual too.

### 14.6 No browser automation of Amazon - a hard line, for a specific reason

**Decision: Sellsweep will not browser-automate Amazon Seller Central, including for Handmade or Custom listing creation.**

*Rationale:* Amazon added **BSA Section 19, an "Agent Policy," effective 2026-03-04**, requiring any automated software or AI agent accessing Amazon Services to (1) **clearly identify itself as an automated system at all times**, (2) comply with the Agent Policy, and (3) cease access immediately on request with no grace period.

Requirement (1) is disqualifying by construction: browser automation's premise is presenting as an ordinary human session. Self-identifying defeats the purpose; not self-identifying is a contractual breach. Amazon also sued Perplexity in November 2025 over its Comet agentic browser, alleging CFAA violations, with the central claim that it was "disguising automated activity as if it were human shopping."

*And the consequences are categorically worse than elsewhere.* The BSA permits reserves, offsets, and **permanent withholding** of seller funds. For a seller whose income runs through Amazon, that is not a shadowban - it is cash-flow failure. This is the clearest risk-asymmetry case in the whole platform set: highest downside, and the alternative (SP-API for base listings + a generated upload file) is genuinely adequate.

### 14.7 Pricing comps - Etsy yes via API, Facebook Marketplace no

**Etsy - no sold data exists anywhere; use active prices via API.**
- Etsy does not expose per-item sold prices at any auth level, in the browser or the API. `sold_out` state returns the *asking* price with zero quantity. Third-party tools (eRank et al.) estimate by polling the public shop sales counter and modeling attribution.
- **`findAllListingsActive`** (`api_key` only, no shop ownership) supports keywords, taxonomy, min/max price, and sort-by-price - a genuine marketplace-wide competitive price research endpoint.
- **Etsy's API terms explicitly prohibit browser extensions** for accessing Etsy data. Here the API is both the easier *and* the compliant path, and browsing is the violating one - inverted from the usual pattern.
- *Caveat to track:* Etsy's API terms prohibit using API data for "analytics" without written authorization. Acceptable for the seller's own shop tooling; would require a conversation with Etsy if Sellsweep became a product.

**Facebook Marketplace - treated as unavailable for price data, on two independent grounds.**
1. **The data doesn't exist.** No price-history surface. Even a seller's own "Sold" record shows the price they *entered*, not what was negotiated - Marketplace is meet-locally, and Meta never observes most transactions. Unreliable in principle, not just inaccessible.
2. **Worst legal posture in the platform set.** *Meta v. BrandTotal* (N.D. Cal., 2022) held automated collection violated both Meta's terms **and** the CFAA, expressly rejecting the *hiQ v. LinkedIn* defense because *hiQ* concerned public, non-authenticated data. Logged-in collection is categorically worse than scraping a public page.

*Corroborating signal:* purpose-built Facebook Marketplace pricing tools (Underpriced, SellItRight) source **eBay** sold data instead. The specialists don't use Facebook's own data, because it isn't there.

### 14.8 Reframe - active asking prices are the *correct* signal here

For a **new, made-to-order** product, active competitor asking prices are the appropriate pricing input; sold comps are not. Etsy's Seller Handbook, Printify's pricing guide, and Craftybase all recommend competitive positioning against current listings, and none reference sold data. Sold comps are appraisal logic for **unrepeatable** goods - a used couch, a vintage watch - where supply is fixed and the object can't be remade. This seller's goods are reproducible on demand and he controls his own cost floor, so the live distribution of asking prices *is* the choice set his buyers face.

*Known weakness and its standard fix:* asking-price distributions are inflated by stale listings that never sell. The correction is not to hunt for sold prices, but to **weight competitors showing evidence of actually selling** - public shop sales count, review volume and recency, bestseller badges, listing age vs. sales. Etsy exposes all of that publicly, which makes the active-price method defensible without any sold data.

### 14.9 Platforms explicitly declined

| Platform | Reason |
|---|---|
| **Google Merchant Center** | Declined by the user. (Note if revisited: free listings remain free and default-on, but Content API for Shopping sunsets 2026-08-18 - build only on Merchant API v1.) |
| **TikTok Shop** | Declined by the user. Obtainable (2–3 day API approval, expedited "seller in-house developer" path) but it is a *content* channel - sales require video and affiliate creators - with tight ship-by SLAs that conflict with made-to-order production. |
| **Walmart Marketplace** | Requires business tax ID (SSN not accepted), documented "history of ecommerce success," and GS1 barcodes per variant. Wrong audience for personalized giftware. |
| **Amazon Merch on Demand** | Invite-gated, 2–8 week approval, no public upload API, and a royalty model where Amazon manufactures - incompatible with in-house production. |
| **Shopify** | Redundant with Square Online. Revisit only if the bottleneck becomes design personalization at scale, where Shopify's configurator app ecosystem (Customily, Teeinblue, Zakeke) has no Square equivalent. |
| **Print-on-demand (Printful / Printify / Gelato)** | Not applicable - the seller does not outsource fulfillment. (If that changed: Printful has Square support but its API *explicitly will never* create products on external platforms; Printify has API publishing but no Square.) |

---

## 15. MVP Platform Set - final

| Platform | Tier | Mechanism |
|---|---|---|
| **eBay** | 1 | Trading API `AddFixedPriceItem` + Media API |
| **Facebook Marketplace** | 1b | Free rider on shipping-enabled eBay listings |
| **Etsy** | 1 | Open API v3 - **Class A items only** |
| **Square** | 1 | Catalog API *(pending the §13 auto-publish test)* |
| **Facebook Shops + Instagram** | 1b | Square's official Meta channel - sanctioned, automatic |
| **Amazon Custom** | 1-partial | SP-API for base SKUs; generated upload file for the customization layer |
| **Facebook Marketplace (direct)** | 2 | Chrome fills the form, seller publishes |
| **OfferUp** | 2 | Chrome fills the form, seller publishes |

Six destinations reachable, from four accounts the seller already has plus one new one.

---

## Decision Log (continued)

| Date | Decision | Rationale |
|---|---|---|
| 2026-08-17 | MVP user is a real seller (a family member), not the builder | Makes setup friction and onboarding documentation first-class concerns. |
| 2026-08-17 | Classify items as Class A (made/personalized) vs. Class B (bulk resale) | Etsy and Amazon Handmade prohibit resale. The skill must block, not warn. |
| 2026-08-17 | Add Square → Meta channel to setup | Official, self-serve, $0. Reaches FB Shops + Instagram. Best value-to-effort on the list. |
| 2026-08-17 | Amazon Custom in; Amazon Handmade out | Custom has no maker requirement and covers both product classes. Handmade blocks Class B, can't create listings via API, and risks a 6-month penalty box. |
| 2026-08-17 | Amazon is Tier 1-*partial* | The customization layer has no API. Sellsweep generates the upload file rather than pretending to automate it. |
| 2026-08-17 | Apply for Amazon restricted-data access on day one | Gates `CustomizedURL`, which is the highest-value Amazon automation (feeding the production queue). Hardest gate for small sellers. |
| 2026-08-17 | No browser automation of Amazon Seller Central, ever | BSA §19 Agent Policy (2026-03-04) requires agents to self-identify - disqualifying by construction. Downside includes permanent funds withholding. |
| 2026-08-17 | Etsy comps via `findAllListingsActive`, not the browser | Etsy has no sold data at all, and its API terms explicitly ban browser extensions. The API is both easier and compliant. |
| 2026-08-17 | Facebook Marketplace unavailable as a comps source | Data doesn't exist (even seller-side "sold" is the entered price), and *Meta v. BrandTotal* makes logged-in collection the worst legal posture available. |
| 2026-08-17 | Active asking prices are the primary pricing signal | Correct method for new made-to-order goods per Etsy/Printify/Craftybase. Weight by seller-velocity signals to correct stale-listing bias. |
| 2026-08-17 | Declined: Google Merchant Center, TikTok Shop, Walmart, Merch on Demand, Shopify, POD | User preference (first two); poor fit or redundant (rest). Documented so the reasoning survives. |

---

## 16. The Real User, The Real Catalog (2026-08-17)

### 16.1 What the seller actually sells - verified by direct observation

**the seller's shop** (the shop's Instagram) - physical booth at a local mall booth, a live Square Online store at `the shop's Square Online site`, and an Instagram presence (~190 followers, 47 posts).

**Observed catalog:**

| Category | Price | Notes |
|---|---|---|
| 3'×5' NFL team flags | $20 | ~32 SKUs - one per team. Bought-in-bulk resale. |
| Key holder plaques | $25 | Religious themes (Joshua 24:15; "Holy Spirit You Are Welcome Here"). His original work. |
| Metal signs | $15 | Dozens. BBQ, floral, farmhouse - plus Batman & Robin, Deadpool, Yoda, Betty Boop. Resale. |
| Custom tiles | - | Printed imagery: 49ers, Dodgers, Steelers, Fast & Furious, The Goonies. **He makes these.** |
| Wood signs, rustic plaques, tumblers, keychains | - | His work. Mixed IP status. |
| Hot Wheels / die-cast | - | Resale. |

### 16.2 Two facts that set the priority

**1. The listed catalog is not the real catalog.** The Square store shows only what was easy to list. He has a large backlog of custom items that never got online *because listing is slow* - which is precisely the problem this project exists to solve. Initial inference from the Square store ("catalog looks resale-heavy, so Etsy may be useless") was **wrong** and is corrected here.

**2. He is closing the physical shop in ~2 months.** The booth is currently his primary channel. This shifts online presence from an opportunity to a **business-continuity requirement with a hard deadline**, and makes listing *throughput* the dominant success metric.

*Implication for the build:* bulk CSV import (§11.4) moves from convenience feature to core requirement. Time-to-list-100-items matters more than polish on any single listing.

### 16.3 The IP finding - and a correction to an earlier overstatement

An earlier draft of this analysis treated "licensed IP" as one category. **That was wrong.** There are two legally distinct acts with opposite platform outcomes.

**(A) Reselling authentic licensed goods - legal, and permitted on eBay.**
First-sale doctrine (17 U.S.C. § 109) exhausts the copyright *distribution* right; a parallel trademark exhaustion rule exists (*Bluetooth SIG v. FCA US*, 30 F.4th 870, 9th Cir. 2022). eBay's own VeRO policy states rights owners may **not** use VeRO to "control where or how a product's resold" or to "prevent sale of items below a price point." Genuine licensed NFL flags bought from a legitimate wholesaler are fine on eBay.

*Caveat:* first-sale requires the *original* sale to have been authorized, and the burden of establishing that falls on the reseller (*FURminator v. Kirk Weaver*, 545 F. Supp. 2d 685; *RFA Brands v. Beauvais*). Keep supplier invoices and chain-of-custody documentation - it is also what an appeal attaches.

**(B) Manufacturing new goods bearing third-party marks or images - infringement, no available defense.**
First-sale exhausts distribution only; it does not reach the reproduction right (§ 106(1)) or derivative works (§ 106(2)). Printing a team logo or a movie still onto a tile is both trademark infringement (likelihood of confusion as to sponsorship - consumers expect team merch to be licensed) and copyright infringement.

The controlling case is ***Warner Bros. Entertainment v. X One X Productions*, No. 15-3728 (8th Cir. 2016)**: the defendant extracted character images from **public-domain** movie posters and applied them to shirts, lunch boxes, and figures. Result: **$2.57M in statutory damages ($10,000 × 257 items)**, affirmed, plus permanent injunction. The injunction permitted only *exact duplication* of public-domain publicity material - extraction and re-application to products was infringing.

No exception applies: fair use fails on commerciality, non-transformativeness, and market harm; de minimis fails because the image *is* the product's appeal; and the artistic-works defense (*Univ. of Alabama v. New Life Art*, 683 F.3d 1266, 11th Cir. 2012) expressly excludes **"mundane products"** like mugs, where "the artistic work is much less likely to have been considered significant by the purchaser." A printed tile is a mundane product.

**(C) A third, separate exposure: listing photographs.**
eBay's IP policy bans "using third-party photos, images, or videos without permission," "images or text copied from other websites or internet searches," and unauthorized stock photos. This means a **100% genuine, first-sale-protected item can generate a VeRO strike purely because the listing used the supplier's catalog photo.** Most VeRO hits on authentic goods are image and title violations, not authenticity violations.

*Sellsweep already solves this by construction:* photo is the only required input (§11.3), so every listing carries the seller's own photography. **This is now a stated feature, not a side effect.**

### 16.4 Etsy is narrower than expected - three independent grounds for rejection

1. **Reselling authentic commercially-available goods is prohibited *regardless of IP status*.** Etsy: "Reselling is not allowed on Etsy… selling a commercially available item that the seller did not make or design." Legal ≠ permitted. This removes the flags, the metal signs, and the die-cast.
2. **IP policy** removes anything bearing third-party marks or images.
3. **Original-design requirement (effective 2025-06-10):** items produced with computerized tools (laser, CNC, printer, Cricut) must be based on "a seller's original design." Templated or third-party designs now fail on originality grounds *in addition* to IP.

**What remains for Etsy:** his genuinely original work - the Joshua 24:15 key holder, the "Holy Spirit You Are Welcome Here" plaque, plain wood signs, unbranded tumblers. Real, but a narrow slice of the catalog.

**One escape hatch:** authentic licensed merchandise **over 20 years old**, handpicked, qualifies under Etsy's vintage category. Worth auditing his old stock for.

*Also note:* Etsy accepts DMCA counter-notices for **US copyright claims only** - there is no counter-notice mechanism for trademark claims, which is what a team-logo complaint will be.

### 16.5 Corrected classification - two independent axes, not a ladder

Provenance and IP status vary **independently**. A hand-made 49ers tile is Class A *and* IP-flagged. A generic floral metal sign is Class B *and* IP-clean. The earlier three-way ladder (§14.2) is superseded by this matrix:

| Item type | eBay | Etsy | Own Square store | FB Marketplace |
|---|---|---|---|---|
| **Original design, IP-clean** - religious plaques, plain wood signs, unbranded tumblers | ✅ | ✅ | ✅ | ✅ |
| **Self-manufactured, third-party IP** - 49ers tile, Goonies plaque, F&F tile | ❌ **highest exposure** | ❌ | ⚠️ seller's own risk | ⚠️ |
| **Authentic licensed resale** - NFL flags, character metal signs | ✅ legal + permitted | ❌ resale banned | ✅ | ✅ |
| **Generic resale, IP-clean** - BBQ / floral signs, die-cast | ✅ | ❌ resale banned | ✅ | ✅ |

**Decision: Sellsweep classifies every item on both axes and blocks routing accordingly.** Warn-and-block for eBay/Etsy/Amazon on self-manufactured third-party-IP items; block Etsy for all resale; allow everything on his own Square store, where IP is ordinary business risk rather than account-suspension risk.

**Universal rule enforced by the skill:** original photography only. Never a supplier, manufacturer, or studio image - this is the most common VeRO trigger on otherwise-legal listings.

### 16.6 Two structural recommendations

**1. Separate the accounts.** Do not run self-manufactured IP items and licensed-resale items from the same eBay account. Every platform's escalation ladder is **account-level**, not listing-level - eBay escalates warning → restriction → suspension, and publishes no strike threshold. If the logo tiles draw a rights-holder sweep, they take down the legal, profitable flag-and-sign resale business with them, at exactly the moment it becomes his only channel.

**2. The capability is the asset; the borrowed imagery is the liability.** The same tile press and laser produce **original** designs - city and state silhouettes, team colorways with no marks, generic sports typography, public-domain imagery used *as-is* (note *X One X*: exact duplication of public-domain material was permitted; extraction and re-application was not). That version of the product line is compliant on every platform and carries none of the statutory exposure. A transition to online selling is the natural moment to shift the mix.

*Reported as source content, not legal advice.* Self-manufactured goods bearing licensed imagery carry statutory copyright damages up to $150,000 per work for willful infringement. That warrants an hour with an IP attorney before scaling listings.

---

## Decision Log (continued)

| Date | Decision | Rationale |
|---|---|---|
| 2026-08-17 | Corrected: resale of authentic licensed goods is legal and eBay-permitted | First-sale doctrine; VeRO explicitly may not police resale channels. Earlier blanket "licensed IP = risk" was an overstatement. |
| 2026-08-17 | Self-manufactured third-party-IP goods blocked from eBay/Etsy/Amazon | Reproduction is not exhausted by first sale. *X One X* = $2.57M on public-domain source material. No defense applies. |
| 2026-08-17 | Original photography enforced on every listing | Supplier/studio images are a *separate* violation that strikes even legal resale listings. Sellsweep's photo-required input already guarantees this. |
| 2026-08-17 | Etsy limited to original-design, IP-clean items only | Three independent grounds: no-resale rule (regardless of legality), IP policy, and the 2025-06-10 original-design requirement. |
| 2026-08-17 | Classification is two independent axes, superseding the §14.2 three-way ladder | Provenance and IP status vary independently - a hand-made 49ers tile is Class A *and* IP-flagged. |
| 2026-08-17 | Recommend separate eBay accounts for IP-manufactured vs. licensed-resale | Escalation is account-level. A sweep on the tiles would kill the resale business alongside it. |
| 2026-08-17 | Bulk CSV import promoted from convenience to core requirement | Shop closes in ~2 months; listing throughput is now the dominant success metric. |

---

## 17. Architecture Shift: Guardrail Engine over Build Sequencing (2026-08-17)

### 17.1 The decision

**Earlier framing** ranked platforms into a build order (Square first, then eBay, then Etsy) driven by risk. **Superseded.**

**New decision: the user selects destinations per item or per batch, and a guardrail engine evaluates every (item, destination) pair - warning, blocking, or throttling as appropriate, with override available.**

*Rationale:* sequencing bakes a risk judgment into the architecture, where it becomes expensive to revisit. A guardrail layer externalizes that judgment to the person who owns the risk. The seller's appetite may differ per item, per platform, and over time - especially with a closing shop and inventory to move. Encoding "eBay is risky for IP items" as a *rule the user can see and override* is more useful, and more honest, than encoding it as a build order that silently withholds the option.

It also collapses several planned features into one mechanism, which matters against a deadline.

### 17.2 Two categories that must not be conflated

**Validation - hard, non-overridable.** Not risks; the API will simply reject these. Evaluated and resolved *before* any call:

| Check | Source of limit |
|---|---|
| Photo present | Sellsweep requirement (§11.3) |
| Variation axes ≤ platform max | Etsy 2, Shopify 3, eBay 5 |
| Variation count ≤ 250 | eBay, Square |
| Title within platform length | eBay 80, Etsy 140 |
| Required fields present | category, condition, `who_made`, `when_made`, shipping profile |
| Image count / format / hosting | eBay: no mixing EPS and self-hosted in one listing |

Offering an override on these would only produce a failed API call.

**Guardrails - overridable, with explicit acknowledgment.**

| Level | Rule | Consequence if overridden |
|---|---|---|
| **BLOCK** | Etsy + resale item | Policy violation → shop strike |
| **BLOCK** | Etsy + non-original design (2025-06-10 rule) | Policy violation → shop strike |
| **BLOCK** | eBay / Etsy / Amazon + self-manufactured third-party IP | VeRO strike; statutory copyright damages up to $150k/work |
| **WARN** | Supplier / studio / stock photo detected | VeRO strike **even on lawfully resold items** - most common trigger on genuine goods |
| **WARN** | FB Marketplace cumulative volume > 50/day | Listing throttle, possible restriction |
| **WARN** | FB Marketplace business-pattern signature | Personal-account ban risk (Meta bans C2C sellers who "sell as a business") |
| **WARN** | Price outside comp range, or < 5 comps | Mispricing (ties to §11.1 confidence floor) |

**Throttle - automatic and silent.** Pacing, not warnings: FB Marketplace daily cap with inter-listing spacing; Etsy 10 QPS / 10,000 QPD; eBay per-API limits. The skill self-paces rather than prompting.

### 17.3 Override logging

**Decision: every override is logged - timestamp, item, destination, rule triggered, and the reason given.**

*Rationale:* if a listing is later removed or an account is actioned, the seller needs a record of what was posted where and what was knowingly accepted. It supports appeals, it supports reconstructing a pattern, and it keeps the tool honest about what it did on his behalf. Also good portfolio evidence: the tool records its own risk decisions rather than silently absorbing them.

---

## 18. Custom Orders - Square Invoices, on the Free Tier (2026-08-17)

### 18.1 The finding

**A 50% deposit custom-order invoice is fully API-automatable on Square's $0 plan.**

`payment_requests: [{request_type: "DEPOSIT", percentage_requested: "50", due_date}, {request_type: "BALANCE", due_date}]`

`DEPOSIT` requires **no subscription**. Only `INSTALLMENT` (up to 12 milestones) requires Invoices Plus. Using a gated feature without the subscription returns `403 MERCHANT_SUBSCRIPTION_NOT_FOUND`.

### 18.2 The automatable chain

1. **`SearchCustomers` / `CreateCustomer`** - find or create the buyer
2. **`POST /v2/orders`** - **ad-hoc line items** (no `catalog_object_id` needed): `name`, `quantity: "50"`, `base_price_money`, **`note`** for personalization details / name lists, `metadata` (≤10 keys) for internal job tracking
3. **`POST /v2/invoices`** - with the deposit/balance split, `delivery_method: EMAIL` or `SMS`
4. **`POST /v2/invoices/{id}/publish`** - this is what actually sends it
5. **`POST /v2/invoices/{id}/attachments`** - mockups and proofs (≤10 files, ≤25 MB prod / 1 KB sandbox)
6. **Webhooks** - `invoice.payment_made`, `invoice.published`, `order.updated` → trigger "deposit received, start production" and "balance due"

**Gotchas:** ad-hoc line items cannot receive catalog rule-based discounts (type the quoted unit price instead); once an order is attached to an invoice its `line_items` / `taxes` / `discounts` are frozen; payment must flow through the invoice, not `PayOrder` / `CreatePayment`; `primary_recipient` is a snapshot; deposits do **not** adjust inventory - only full payment does.

### 18.3 Skip Square Estimates

**Decision: do not use Square Estimates.**

*Rationale:* there is **no Estimates API** - Square staff have confirmed this repeatedly, most recently 2025-12-29 ("it's a bit more complicated than just a feature flag being enabled"). It is Dashboard/POS-only and requires **Invoices Plus at $49/mo per location**. A published invoice with a 50% deposit accomplishes the same commercial outcome, on the free tier, and is the only version a skill can drive. Estimates also don't appear in transaction reports.

### 18.4 Intake - manual, but nearly free

Square Online includes a **native form builder** on the **Free** tier, with prebuilt templates including **"Custom quote"** and **"Wholesale inquiry"** - exactly the two needed. Submissions route to email, **Square Messages**, the Square POS app, and a Dashboard **Form Submissions** page with **CSV export**.

**No API** exists to create forms or read submissions (the only Square Online APIs are Snippets and Sites). There is also **no Square Messages API**.

**Decision: manual intake, automated fulfillment.** The seam is the CSV export (or the seller pasting a request). Everything downstream - pricing, customer creation, order, invoice, deposit, send - is automated.

*Note:* Square Online cannot do "contact for pricing" items - a price is mandatory. Workaround is a content page plus a contact form.

### 18.5 Bulk / wholesale pricing

Square has **no native tiered price table and no per-customer wholesale price lists** (a feature request open since ~2016).

**What works:** `CatalogDiscount` + `CatalogProductSet` (`quantity_min` / `quantity_exact`) + `CatalogPricingRule`, stacked to create quantity breaks (`quantity_min: 11` → 20%, `quantity_min: 51` → 35%; highest applicable applies). **Critical caveat:** automatic discounts on Square Online must enumerate **individual items** - category-based rules do not apply to websites or online ordering.

**Decision: do both.** Quantity-break pricing rules via API for self-serve bulk, **plus** explicit bulk-pack variations ("25-pack", "50-pack", "100-pack") so simple bulk orders never need a quote at all. Quotes are reserved for genuinely custom work.

### 18.6 Personalization capture - use Payment Links, not text modifiers

**Confirmed still true as of 2026-08:** Square `CatalogModifierList` with `modifier_type: TEXT` captures buyer text at checkout, but **the Orders API cannot return it.** `OrderLineItemModifier` has no text field. Square staff, 2025-01-03: "the Orders API doesn't currently support text modifiers." No 2026 changelog entry alters this. The seller sees the value on the order ticket and receipt - *but Square's docs never explicitly confirm where the text lands, and a community thread asking exactly this went unanswered.* **Place one real test order and verify visually before relying on it.**

**The API-readable alternative:** Payment Links `checkout_options.custom_fields` (`CustomField` object, **max 2**) *are* retrievable - values surface at **`fulfillments[0].delivery_details.note`**, confirmed by Square staff 2026-01-22. Undocumented and not guaranteed, but it is the only programmatic path.

**Decision: where Sellsweep needs to read personalization programmatically, use Payment Links + custom fields. Text modifiers are for cases where the seller reads the value himself off the order.**

Also note: text modifiers work on Square Online / Square Websites and Square Kiosk, but **not on Payment Links** (Square staff, 2025-05-12) - so the two mechanisms are mutually exclusive per channel, not layerable.

### 18.7 Why not Etsy for custom orders

Etsy has a nicer *buyer-facing* flow - a "Request Custom Order" button and a dedicated Custom Requests folder in Messages. But:

- **No messaging API.** The entire quote conversation is unautomatable.
- **No deposit mechanism.** Etsy requires full payment; sellers hack 50/50 with two separate listings.
- **No volume/quantity discounts.** Discount types are promo codes, sales, discounted bundles, and targeted offers only. Bundles require buying *every* item in the bundle.
- **Etsy Wholesale is dead** - closed 2018-07-31.

Etsy's personalization system (rebuilt April–May 2026: up to 5 typed questions, text/dropdown/upload, buyer **image uploads** up to 10 files, `add_on_price` $0.20–$500) is genuinely the best of any platform for *per-item* personalization - and it is API-settable, though via a **separate two-step call** (`POST .../listings/{id}/personalization?supports_multiple_personalization_questions=true`), not on `createDraftListing`. Personalization answers arrive in the transaction's `variations` array under **`property_id: 54`**.

**Decision: Etsy handles per-item buyer personalization on eligible (original-design, IP-clean) items. Square handles bulk and quoted custom orders.** Different tools for genuinely different jobs.

### 18.8 The insight linking both use cases

**The catalog he has already made *is* the custom-order catalog.** Every keychain, coaster, and sign he has built is simultaneously a ready-to-ship item and a template - "same style, your name on it," "same design, 50 for an event."

**Decision: one intake produces both.** Each item Sellsweep processes yields a stock listing *and* a customizable/bulk variant, from the same photo and the same questions. This is why the two priorities are one build, not two.

---

## Decision Log (continued)

| Date | Decision | Rationale |
|---|---|---|
| 2026-08-17 | Guardrail engine replaces platform build-sequencing | Externalizes risk judgment to the person who owns it, instead of baking it into architecture. Collapses several features into one mechanism. |
| 2026-08-17 | Validation (hard) separated from guardrails (overridable) | An override on a limit the API enforces just produces a failed call. Different mechanisms for different failures. |
| 2026-08-17 | All overrides logged with timestamp, item, destination, rule, reason | Supports appeals and pattern reconstruction; keeps the tool accountable for what it did on the seller's behalf. |
| 2026-08-17 | Custom orders via Square Invoices API, 50% `DEPOSIT` + `BALANCE` | Fully automatable on the $0 plan. Deposit type needs no subscription. |
| 2026-08-17 | Skip Square Estimates | No API exists (confirmed by Square staff through 2025-12-29) and it costs $49/mo. A published deposit invoice does the same job. |
| 2026-08-17 | Manual intake via native Square Online forms; automate everything downstream | No form or Messages API. Free-tier native forms + CSV export is the pragmatic seam. |
| 2026-08-17 | Both quantity-break pricing rules and explicit bulk-pack variations | Self-serve bulk shouldn't require a quote; quotes reserved for genuinely custom work. |
| 2026-08-17 | Payment Links custom fields for machine-readable personalization | Text modifiers still unreadable via Orders API as of 2026-08. Custom fields surface at `fulfillments[0].delivery_details.note`. |
| 2026-08-17 | Etsy for per-item personalization; Square for bulk/quoted custom orders | Etsy's rebuilt personalization is the best available and API-settable; Etsy has no deposits, no volume pricing, no messaging API. |
| 2026-08-17 | One intake produces both a stock listing and a custom/bulk variant | The existing catalog *is* the custom-order catalog. Not two builds. |

---

## 19. Scope Correction: Order Management Is Out (2026-08-17)

**§18 drifted into order management.** Square Invoices, deposit splits, `PublishInvoice`, payment webhooks, and production-queue triggers are **out of scope for v1** and the relevant parts of §18 are superseded by this section.

**Why it was wrong:** the seller already manages incoming orders himself and has no stated problem doing so. The validated problem is a listing backlog he cannot get online fast enough before his shop closes. Building an invoicing pipeline would have spent the deadline on a problem nobody has.

**The scope line: Sellsweep creates listings. It does not process what happens after someone buys.**

That line still includes making a listing *capable of accepting* custom orders, because configuring a listing is listing work:

| In scope (listing configuration) | Out of scope (order handling) |
|---|---|
| Etsy `personalization_questions` on a listing | Reading personalization off a transaction |
| Square TEXT modifiers on a catalog item | Retrieving buyer-entered text from an order |
| Bulk-pack variations ("25-pack", "50-pack") | Quoting, invoicing, deposits |
| Quantity-break pricing rules on catalog items | Payment webhooks, production triggers |
| - | Square Estimates, Invoices API, Customers API |

**Retained from §18 as still-relevant listing knowledge:**
- Etsy personalization is set by a **separate two-step call** (`POST .../listings/{id}/personalization?supports_multiple_personalization_questions=true`), not on `createDraftListing`. Up to 5 typed questions; text/dropdown/upload; buyer image uploads; `add_on_price` on optional text questions only.
- Square TEXT modifiers work on Square Online but **not** on Payment Links, and their values are **not readable via the Orders API**. Since Sellsweep no longer reads orders, this is now the seller's concern, not the tool's - but it's worth telling him at setup so he knows to check order tickets manually.
- Quantity-break discounts on Square Online must enumerate **individual items**; category-based rules don't apply to websites.

**Deferred to v2, explicitly:** order retrieval, invoicing and deposits, quote generation, inventory sync across platforms, and the sold-here-so-delist-there problem. All real, none urgent, and each is a different product.

---

## Decision Log (continued)

| Date | Decision | Rationale |
|---|---|---|
| 2026-08-17 | Order management cut from v1; §18 partially superseded | The seller has no problem handling orders. The validated problem is listing throughput against a closing-shop deadline. |
| 2026-08-17 | Scope line: configure listings, don't process orders | Listings that *accept* custom orders are still listing work; everything post-purchase is not. |
| 2026-08-17 | eBay pacing constraint dropped | Seller has an established account with sales history - no new-seller selling limits to design around. |

---

## 20. The Inventory Ledger: One Source of Truth (2026-08-21)

### 20.1 Why a ledger, and why now

Requested by the operator while the catalog held exactly one item. That timing looks premature and is not: a ledger backfilled after a bulk load is a reconstruction, and reconstructions are wrong. Started before the first real load, it is just a record.

Two jobs, and they are not equally ready.

**Job 1, know where everything is.** One physical item can be live on four marketplaces at once. Nothing else in the system knows all four placements simultaneously. Each platform knows only its own listing. When a quantity of one sells, something has to say *"it is also live on eBay and Etsy, end those."* That job works from the first row.

**Job 2, price new items from real history.** Handmade goods have no usable external comp. A hand-poured rooster coaster from one seller is not in PriceCharting, and eBay's nearest match is somebody else's mass-produced coaster. What the seller does have, eventually, is their own realized prices. That job does not work yet.

### 20.2 The cold-start problem, stated rather than hidden

Below five matching sales the ledger has no pricing power. The temptation is to use it anyway and dress up one data point as evidence. The rule instead: **under five matching sales, tier 0 is skipped entirely and the seller is told why.** It becomes genuinely useful somewhere north of a dozen sales per category, which is months out.

A tool that says "I do not have enough data yet" is worth more than one that produces a confident number from a sample of one. This is the same discipline as the existing five-comparable floor, applied to the tool's own data.

### 20.3 What tier 0 gives that external comps cannot

`days_listed`. A sold price alone cannot tell you whether the number was right. Sold in three days means underpriced. Sold in two hundred means overpriced, and it got there by attrition. External sold comps carry the price and drop the clock, which is half the signal.

### 20.4 Denormalized, deliberately

Two candidate shapes:

| Shape | For | Against |
|---|---|---|
| Normalized: `items` + `placements` tables | Clean, handles N platforms, per-platform price falls out naturally | Two files to keep in sync, and the "where is this live" question needs a join every time |
| **Denormalized: one row per unit, platform columns inline** | The whole answer is one row. Readable in Excel by a non-engineer. No join. | 31 columns. Adding a fifth platform means a schema change. |

**Chose denormalized.** The platform set is four and fixed, and the operator has to be able to open this in Excel and see the truth without running a query. Normalization earns its cost when the platform count is open-ended. Here it is not. If a fifth destination ever lands, three columns get added; that is cheaper than a join on every read for the next two years.

Per-platform price is stored rather than derived, even though the derivation is known (`etsy_price = base_price + shipping`). Storing what was actually listed survives a change to the derivation rule. Deriving it does not.

### 20.5 Grain: one row per sellable unit

A variation listing is one listing with N variants on every platform that supports it. The ledger mirrors that: the parent row carries the platform listing IDs, each variant row carries its own SKU and its own quantity, linked by `parent_sku`.

32 NFL flags is one parent plus 32 variant rows, not 32 listings. Quantity lives on the variants because that is where stock actually is. Parent quantity stays blank on purpose; a rolled-up number there would be a second place for stock to be wrong.

### 20.6 The constraint that shaped the schema most

**No buyer fields. None. Ever.**

An early draft carried `buyer_state`, on the reasoning that regional demand would be useful for pricing. It was cut, and the reason is not privacy sentiment.

The same day, the eBay Marketplace Account Deletion exemption was filed under *"I do not persist eBay data."* A `buyer_state` column would have made a filed attestation false. The connection is not obvious in the moment: a schema decision in one file and a compliance checkbox in a browser tab look like unrelated work, and they are the same decision.

This is also why the OAuth scope set was cut from eBay's default 25 down to 5. `sell.fulfillment` reads orders including buyer names and addresses. Holding a token that *can* read every buyer is a materially weaker claim than holding one that cannot, even if those endpoints are never called.

**Generalized:** when a system files an attestation about what it does not store, the attestation becomes a schema constraint and a permissions constraint. It is not a one-time form.

### 20.7 Relationship to the §19 scope cut

§19 deferred inventory sync and the sold-here-so-delist-there problem to v2. The ledger does not reverse that. It does not sync anything and it does not end any listing on its own.

What it does is make the problem **visible**: when a sale is logged, the skill can name every other live placement and offer to end them. The seller still runs the command. Full automatic sync remains deferred.

The line: **v1 tells you what is out of sync. v2 fixes it for you.** Knowing is most of the value and none of the risk.

### 20.8 Where the files live

| File | Repo | Tracked |
|---|---|---|
| `inventory.csv`, `sales_log.csv` | private only | Yes. Git history is the audit trail: every price change and delisting is a timestamped diff. |
| `assets/inventory_template.csv`, `assets/sales_log_template.csv` | both | Headers only, no seller data, so the skill is self-contained for anyone. |
| `references/inventory.md` | both | Schema documentation, generic. |

The live ledgers are git-ignored in the public repo. They hold a real business's SKUs, prices, and platform IDs.

### 20.9 Verified, not assumed

The schema was round-tripped against the two cases most likely to break it, before anything was wired into the skill:

- **32-variant flag listing:** parent quantity blank, stock summing correctly across variants, `parent_sku` linkage intact.
- **One-of-a-kind live on three platforms, sold on Square:** correctly returns `[ebay, etsy]` as the placements that must come down, decrements quantity to zero, flips status to `sold`, and produces a sales row containing no buyer field.
- **Pricing lookup with one matching sale:** correctly refuses tier 0 and falls through.

---

## Decision Log (continued)

| Date | Decision | Rationale |
|---|---|---|
| 2026-08-21 | Inventory ledger as source of truth, two CSVs | Nothing else knows all of an item's placements at once. Started before the first bulk load so it is a record, not a reconstruction. |
| 2026-08-21 | Denormalized over normalized | Four fixed platforms and a non-engineer operator. Normalization earns its cost only when the platform count is open-ended. |
| 2026-08-21 | Own sales history as pricing tier 0, gated at 5 matching sales | Realized prices beat asking prices, and `days_listed` is signal no external comp carries. Below 5 it produces confident nonsense, so it is skipped and said out loud. |
| 2026-08-21 | No buyer fields in any ledger file | The eBay account-deletion exemption is filed as "I do not persist eBay data." A schema decision and a compliance checkbox are the same decision. |
| 2026-08-21 | eBay OAuth scopes cut from 25 to 5 | Same reasoning. A token that cannot read buyers is a stronger claim than one that can but promises not to. |
| 2026-08-21 | Ledger surfaces sync problems, does not fix them | Keeps the §19 deferral intact. Knowing what is out of sync is most of the value and none of the risk. |
