# Inventory ledger

Two CSVs in the project root. They are the source of truth for what exists, where it is listed, and what it sold for.

| File | Grain | Written |
|---|---|---|
| `inventory.csv` | one row per sellable unit | updated in place |
| `sales_log.csv` | one row per sale | append only, never edited |

Both are git-tracked in the **private** repo. Git history is the audit trail: every price change and every delisting is a diff with a timestamp. Neither file ever goes in the public repo.

## The two jobs

**1. Know where everything is.** One physical item can be live on four marketplaces at once. When it sells on one, it has to come down from the other three, fast. Nothing else in the system knows all four placements at the same time.

**2. Price new items from real history.** A hand-poured rooster coaster made by one seller in El Paso is not in PriceCharting and has no meaningful eBay comp. What it *does* have, after enough sales, is the seller's own record: this material, this size, this theme, sold for this much, in this many days, on this platform. That is a better comp than anything external.

**Job 2 does not work yet and will not for months.** With a handful of sales the ledger has no pricing power, and pretending otherwise produces confident nonsense. Below **5 matching sales**, do not use it as a comp tier at all. Say so plainly rather than dressing up a single data point.

## SKU convention

```
{PREFIX}-{TYPE}-{NNN}          parent or standalone   SHOP-FLAG-004
{PREFIX}-{TYPE}-{NNN}-{VAR}    variant                SHOP-FLAG-004-DAL
```

`PREFIX` is a short seller code, set once in `seller_profile.md`. `TYPE` is short and stable: `TILE` `COAST` `SIGN` `TUMB` `FLAG`. Numbers never get reused, including after an item is sold or deleted. A SKU is permanent.

## Variations

A variation listing is **one listing with N variants** on every platform that supports it. The ledger mirrors that:

- The **parent row** holds the platform listing IDs and `parent_sku` empty.
- Each **variant row** carries its own `sku`, its own `quantity`, and `parent_sku` pointing at the parent.

32 NFL flags is 1 parent row plus 32 variant rows, not 32 listings. Quantity lives on the variants because that is where stock actually is.

## Fields that carry weight

| Field | Why it exists |
|---|---|
| `design_theme` | The main matching key for pricing. `scripture` `sports` `floral` `seasonal` `humor` `rooster` `custom` |
| `personalizable` | Personalized work commands a premium. Comparing it against stock work skews both. |
| `days_listed` (sales log) | The real signal. Sold in 3 days means underpriced. Sold in 200 means overpriced. Price alone cannot tell you which. |
| `cost_basis` | Optional and usually blank. When present it turns the ledger from a price log into a margin tool. |
| `{platform}_price` | Prices genuinely differ per platform. Etsy bakes shipping into the price, eBay does not. Store what was actually listed, not what was derived. |
| `status` | `draft` `active` `sold` `delisted` `archived` |

**There are no buyer fields, and there must never be.** No name, address, email, or order ID. The eBay Marketplace Account Deletion exemption is filed under *"I do not persist eBay data"* and that claim is only true while this holds. Adding a buyer column silently makes a filed attestation false.

## When the skill reads

**Step 4, research price.** Before any external lookup, query `sales_log.csv` for sales matching `item_type` + `material` + `design_theme`. With 5 or more, that becomes tier 0, ahead of every external source, because it is this seller's actual realized price rather than someone else's asking price. Report it as such. Under 5, skip it and say why.

**Step 6.5, recommend destinations.** Check whether the item, or its siblings, are already live somewhere. Do not recommend a platform where it is already listed.

**Intake.** Before creating anything, check whether the SKU or a near-identical title already exists. Re-listing something already live is how you end up double-selling a quantity of one.

## When the skill writes

**Step 9, log.** After posting, write or update the row: platform IDs, per-platform price, per-platform status, `last_updated`. One row per item touched. This replaces the old prose `listing_log.md`.

**On a sale.** Append to `sales_log.csv`, decrement `quantity` in `inventory.csv`, and if quantity hits zero set `status` to `sold` and flag every other platform where it is still live.

## The delist rule

**A quantity of one that sells on one platform is still for sale on three others until someone takes it down.** Overselling costs more on eBay than anywhere else: a cancellation there creates an account defect and suppresses search visibility. Etsy penalizes less. Square and Facebook Marketplace are the seller's own channels where a cancellation is merely awkward.

So:

1. One-of-a-kind items should be listed on **fewer** platforms, and should avoid eBay unless eBay is clearly the best venue for that item.
2. When a sale is logged, the skill names every other live placement and offers to end them. It does not end them silently.
3. Repeatable items, where the seller can just make another, escape this problem entirely. Quantity stops being 1, restocking replaces delisting, and multi-listing becomes safe. This is the same conclusion the Etsy originality rule reaches from a different direction.
