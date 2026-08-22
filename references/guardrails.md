# Guardrails

Two categories. They behave differently and must not be conflated.

## Validation - hard, not overridable

These are not risks. The API will reject them. Resolve before calling anything; an override here just produces a failed call.

| Check | Limit | Destination |
|---|---|---|
| Photo present | ≥1 | all |
| Own photography (not supplier/stock) | - | all |
| Variation axes | ≤2 | Etsy (3 in preview, not GA) |
| Variation axes | ≤3 | Shopify |
| Variation axes | ≤5 | eBay |
| Variation combinations | ≤250 | eBay, Square |
| Title length | ≤80 | eBay |
| Title length | ≤140 | Etsy |
| Images per listing | ≤24 | eBay (12 per variation) |
| Images per listing | ≤20 | Etsy |
| Image hosting | no mixing EPS + self-hosted in one listing | eBay |
| Required fields | category, condition | eBay |
| Required fields | `who_made`, `when_made`, `taxonomy_id`, shipping profile, `readiness_state_id` | Etsy |
| Item has ≥1 variation | required | Square |

If validation fails, say which rule and which item. Do not silently truncate a title or drop a variation.

## Guardrails - overridable with explicit acknowledgment

### BLOCK level

| Rule | Trigger | Consequence if overridden |
|---|---|---|
| Etsy resale ban | provenance = resale, destination = Etsy | Policy violation → shop strike. Applies **even to legally resold authentic goods** - Etsy's rule is stricter than the law. |
| Etsy originality rule | design not the seller's own (templated, purchased, or third-party), made with computerized tools, destination = Etsy | Policy violation → shop strike. In force since 2025-06-10. |
| Self-manufactured third-party IP | provenance = user-made AND ip_status ≠ clean, destination = eBay/Etsy/Amazon | Rights-holder takedown → account strike. Statutory copyright damages up to $150k/work. First-sale does **not** apply - manufacturing is reproduction, not distribution. |

**The distinction that matters most:** *reselling* authentic licensed goods is legal and permitted on eBay. *Manufacturing* new goods bearing licensed marks is not. Do not conflate them - blocking legal resale would cost the user real revenue for no reason.

### WARN level

| Rule | Trigger | Consequence if overridden |
|---|---|---|
| Third-party photo | image appears to be supplier, manufacturer, studio, or stock | Takedown **even on lawfully resold items**. Most common cause of strikes on genuine goods. |
| Marketplace volume | cumulative listings today >50, destination = FB Marketplace | Listing throttle, possible restriction |
| Business-pattern signature | many similar listings from one personal account, destination = FB Marketplace | Personal account ban - Meta explicitly bans C2C sellers who "sell as a business" |
| Low pricing confidence | <5 comparables | Mispricing |
| Price outside comp range | price >30% off the comp band | Mispricing |
| Vintage exception available | resale item, plausibly 20+ years old, destination = Etsy | Informational - Etsy's vintage category is a legitimate path for genuinely old licensed stock |

### Account separation advisory

If the user is posting both self-manufactured IP-bearing items and licensed-resale items to the same eBay account, raise it once: escalation ladders are account-level, not listing-level. A sweep on the IP items takes down the legal resale business alongside it.

Raise once. Don't repeat it every batch.

## Throttle - automatic and silent

Pace; don't prompt.

| Destination | Limit |
|---|---|
| FB Marketplace | ~50/day, space listings several minutes apart |
| Etsy | 10 QPS, 10,000 QPD |
| eBay Trading | generous; not a practical constraint |
| Square | no published hard cap |

## Override logging

Append to `override_log.md`:

```
| timestamp | item_id | destination | rule | level | reason given |
```

**Never post over a BLOCK without an explicit, in-conversation acknowledgment for that specific item.** Not a blanket "ignore all warnings," and not a preference inherited from a previous item, even an identical one. Per item, every time, so the decision is real.

Accepting risk on one item never carries to the next. The explanation can get terse after the first time a rule fires; the confirmation cannot be skipped. See the override policy in `routing.md`.
