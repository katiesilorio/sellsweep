# Routing

**Recommend destinations. Do not wait to be told.**

The seller should not have to know which marketplace fits a given item. That is the skill's job. Propose a destination set with reasoning, surface the risks, and let the seller override.

Order: **classify → recommend → guardrail check → seller confirms or overrides → post.**

Recommendation runs *before* guardrails. Guardrails then evaluate whatever set the seller lands on.

## The factors

Routing is not just IP and provenance. Six things matter, and they trade off.

| Factor | Why it routes |
|---|---|
| **Provenance** | Etsy bans resale outright, regardless of legality |
| **IP status** | Self-manufactured third-party IP draws takedowns on eBay/Etsy |
| **Price point** | Each venue has a band where buyers actually convert |
| **Audience** | Collectors, gift buyers, and local deal-seekers are different people |
| **Shipping economics** | Weight-to-price ratio decides whether shipping eats the margin |
| **Repeatability** | One-offs should not multi-list (see the mark-sold problem) |

## Venue profiles

| | eBay | Etsy | Own store (Square) | FB Marketplace |
|---|---|---|---|---|
| **Price band** | $15 to $500+ | $20 to $150 | any | **$5 to $60** |
| **Audience** | national; collectors, hobbyists, deal-seekers | national; gift buyers, handmade shoppers | existing followers, referrals | local, price-anchored |
| **Resale** | ✅ legal and permitted | ❌ banned regardless of IP | ✅ | ✅ |
| **Self-made third-party IP** | ⚠️ VeRO exposure | ❌ blocked | ⚠️ seller's own risk | ⚠️ low enforcement |
| **Best for** | collectibles, resale, searchable niches | original handmade, gifts, personalization | everything, full control | heavy/bulky, local pickup, low price |
| **Weak for** | low-value items after fees | anything not original and handmade | discovery (no traffic of its own) | **anything over ~$60**, national niches |
| **Variations** | good (250, 5 axes) | best (2 axes, but personalization is native) | good (250) | poor (one listing per item) |

## Rules that decide most cases

**Etsy is the narrowest gate.** Original design, handmade, no third-party IP, no resale. If an item fails any of those, Etsy is out and there is nothing to weigh.

**Price sets the floor and ceiling.** A $130 collectible on Facebook Marketplace is showing a specialty item to bargain hunters. An $8 item on eBay loses most of its value to fees. Match the band.

**Audience beats reach.** A national niche (die-cast collectors, scripture decor) belongs where that niche searches, not where the most people are. Local reach is only an advantage for items that are heavy, bulky, or cheap enough that shipping kills them.

**Shipping economics can veto everything else.** Compute weight-to-price before recommending. A $15 item costing $12 to ship is a local-only item no matter how good the audience fit is elsewhere.

**One-offs get one destination.** Quantity 1 across four platforms means overselling the first time it sells. See the mark-sold scope revision.

## The trademark asymmetry (eBay vs Etsy)

Guidance that holds on Etsy **inverts** on eBay, so do not apply one platform's rule to the other.

**On Etsy**, keep brand names out of titles and tags. Sweeps are keyword searches, and Etsy has no resale exception to fall back on.

**On eBay**, naming a brand the item genuinely contains is defensible. eBay's keyword policy bans brand names when the item is not associated with them, but expressly permits them when "the product depicts or is genuinely associated with them." A set that physically contains a genuine branded item can say so. That is nominative use.

The residual eBay risk is usually not the resold branded object. It is any **manufactured** element bearing a third-party mark or image.

## Output format

Present a recommendation, not a menu. Name the destination, give the reason, and flag anything the seller is accepting.

```
Recommended: eBay

Why: die-cast collectors are eBay's core audience, and reselling the
authentic branded car is legal and permitted there. Custom display
pieces are an established searchable category.

Not Facebook Marketplace: $129.99 is well above where Marketplace
buyers convert, and collectors are a national niche, not a local one.

Not Etsy: contains a resold item, which Etsy bans regardless of IP.

⚠️ Accepting: the plaque reproduces an identifiable licensed vehicle.
That is manufactured IP, not resale, so first sale does not cover it.
Enforcement by this rights holder against small custom sellers is
lighter than average, but the exposure is real.

Post here? [confirm / change destinations]
```

Every recommendation carries **where, why, why not the others, and what is being accepted.** The seller overrides freely; that is the design, not a failure of it.

## Override policy

Overrides are expected and supported. Log per §17.3 (timestamp, item, destination, rule, reason).

**Confirmation is per item. Always. Never inherited.**

A decision to accept risk on one item does not carry to the next one, even when the rule is identical. Two reasons this matters:

1. **"IP-flagged" is not one risk level.** The NFL, Disney, and a small automotive rights holder enforce at wildly different intensities. Collapsing them into a standing preference flattens a distinction the seller needs in order to decide well.

2. **Inherited approval is how accounts die.** If the skill stops asking, it eventually posts something the seller would have declined, and the seller finds out when the sweep lands.

**What can shorten is the explanation, not the ask.**

- **First time a rule fires:** full reasoning. What the rule is, why it exists, what the specific consequence is.
- **Every time after:** one line naming the item, the rule, and the specific rights holder. Still requires a yes.

```
⚠️ Deadpool Metal Sign - self-manufactured third-party IP (Marvel/Disney).
   Disney enforces aggressively. Post to eBay anyway? [y/n]
```

That is not nagging. It is the difference between the seller choosing 30 times and choosing once and being surprised 29 times.
