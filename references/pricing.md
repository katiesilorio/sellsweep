# Pricing

## The cascade

Work down until you have enough. Always record which tier produced the number.

| Tier | Source | Access | Gives |
|---|---|---|---|
| **0** | **The seller's own `sales_log.csv`** | Local | **Realized** sold prices plus `days_listed`, for this seller's own goods. Needs 5+ matching sales. |
| 1 | Licensed comps API (e.g. PriceCharting) | Paid, self-serve | True sold comps - vertical-limited (games, TCG, comics, Funko, LEGO, coins) |
| 2 | The user's own marketplace research tool, via their authenticated session (eBay Terapeak) | Delegated | Avg sold price + sell-through, 2–3 yrs |
| 3 | eBay Browse API | Free, 5,000/day, no approval | **Active asking** prices - an upper bound |
| 4 | Etsy `findAllListingsActive` | `api_key` only | Active asking prices, keyword/taxonomy/price-range filtered |
| 5 | Model judgment with an explicit range | Always | The floor. Never silently absent. |

## Tier 0 - the seller's own history

Handmade goods have no external comp worth trusting. A hand-poured coaster from a single seller is not in PriceCharting, and eBay's closest match is somebody else's mass-produced coaster. What the seller does have, eventually, is their own record.

Match on `item_type` + `material` + `design_theme`, and treat `personalizable` as a hard split. Personalized work carries a premium, and mixing it with stock work corrupts both sides of the comparison.

**`days_listed` is the part external comps cannot give you.** A sold price alone cannot tell you whether the number was right. Sold in 3 days means it was probably underpriced. Sold in 200 days means it was overpriced and got there by attrition. Weight recent sales over old ones, and say when a comp is stale.

**The cold-start problem is real, so state it.** Under 5 matching sales this tier produces nothing usable. Skip it, fall to tier 1, and tell the seller the ledger is not deep enough yet rather than dressing up a single sale as evidence. It becomes genuinely useful somewhere north of a dozen sales per category, which is months away, not weeks.

## Always recommend, never just report

**The skill recommends a specific price and shows its reasoning.** Handing back a range and asking the seller to decide pushes the work back onto them, which is the opposite of the job.

Format:

> **$18.** Comparable 4x4 scripture tiles with easels run $16 to $26. At $18 he nets about $10 after shipping and fees instead of $6 at his current $8. The Etsy version lands at ~$24 with free shipping baked in, which stays inside the comp band rather than topping it.
>
> Confidence: low. Tier 5, my judgment rather than pulled comps.

Every recommendation carries three things: **the number**, **why that number**, and **the confidence with its tier**. A seller can overrule a number. They cannot overrule a number they were never given.

**Anchor against what they currently charge when it's known.** A booth price is set for walk-by impulse buying and usually does not survive the fee and shipping structure of online selling. Name the gap explicitly and show the arithmetic.

## The confidence floor

**Below 5 comparables, do not report a point price.** Report a range, label confidence low, and name the tier. A confident wrong number is worse than an honest range - it is the highest-rated product risk in this build.

Always show provenance: *"$18–24, based on 7 active Etsy listings for similar handmade slate coasters. No sold data available."*

## New goods vs. used goods

Different problems; don't use one method for the other.

**New, made-to-order** (most of what this skill handles): active competitor asking prices are the **correct** signal. Supply is elastic, the item is reproducible, the seller controls their own cost floor - so the live distribution of asking prices *is* the choice set buyers face. This is what Etsy's own seller guidance, print-on-demand pricing guides, and competitor-based pricing theory all recommend. Sold comps barely feature.

**Used, unrepeatable** (a secondhand couch, a vintage watch): sold comps are right, because supply is fixed and the only meaningful question is what someone actually paid for that specific thing. This is appraisal logic.

## Correcting for stale listings

Asking-price distributions are inflated by listings that never sell. The fix is *not* to hunt for sold data - it is to **weight competitors that show evidence of actually selling**:

- Public shop sales count (Etsy exposes this and it can't be hidden)
- Review volume and recency
- Bestseller badges
- Listing age relative to sales

Weight those competitors more heavily and the active-price method holds up without any sold data.

## What is not available

Don't build toward these; they're closed:

- **eBay Marketplace Insights** (sold comps API) - invitation-only. eBay Partner Network states access "cannot be granted upon request." No application path exists.
- **eBay `findCompletedItems`** - decommissioned February 2025.
- **Terapeak** - no public API. Reachable only through the user's own Seller Hub session.
- **Etsy sold prices** - do not exist at any auth level, in the browser or the API. `sold_out` state returns the *asking* price with zero quantity.
- **Facebook Marketplace price data** - no read API, and no reliable data even for a seller's own sales, since Marketplace has no payment rail on local pickup. The listed price is not the transacted price.

## Boundaries

Scraping marketplace search results is out of scope. eBay's User Agreement now explicitly names "LLM-driven bots" among prohibited automated access. Reading the user's own research tools inside their own authenticated session is a different act and is the industry-standard approach; scraping public search results is not.

Etsy's API terms additionally prohibit using API data for "analytics" without written authorization - fine for a seller's own shop tooling, but flag it if this ever becomes a multi-seller product.
