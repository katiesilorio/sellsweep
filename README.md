# Sellsweep

Photograph an item, answer a few questions, and get complete listings published across eBay, Etsy, Square, and Facebook Marketplace - with variations intact and platform policy enforced before anything posts.

## The problem

Selling online means re-creating the same listing five times. Every marketplace wants the same underlying information - photos, title, description, price, category, condition - in a different shape, with a different taxonomy, different limits, and different required attributes.

The *thinking* work gets done once. The *transcription* work gets done five times. That transcription is the tax, and it's why most sellers list on one platform and stop.

Sellsweep does the thinking once and eliminates the transcription.

## What makes this non-trivial

"Post everywhere" isn't purchasable at any price. Research into the actual platform landscape found that the marketplaces with the most consumer demand have the least programmatic access - Facebook Marketplace, OfferUp, and Craigslist have no usable API, and every existing cross-listing tool reaches them through browser automation that violates those platforms' terms.

So Sellsweep tiers destinations by what's genuinely achievable, and is explicit about which tier each one is in:

- **Direct API** - eBay, Etsy, Square. Fully automated.
- **Distribution rider** - Facebook Marketplace and Poshmark via eBay's Meta partnership; Facebook Shops and Instagram via Square's Meta channel. Free reach, zero extra integration.
- **Assisted fill** - Facebook Marketplace direct. Chrome fills the entire form, the seller reviews and publishes.
- **Out of scope** - documented, with reasoning, rather than quietly omitted.

## The guardrail engine

The seller picks destinations. Sellsweep evaluates every (item, destination) pair before anything posts, separating two things that are often conflated:

- **Validation** - hard limits the API enforces. Not overridable; there's no point offering an override on a call that will fail.
- **Guardrails** - policy and legal risk. Block or warn, both overridable with explicit per-item acknowledgment, and every override logged.

The distinction the engine exists to get right: **reselling** authentic licensed goods is legal and permitted on eBay, while **manufacturing** new goods bearing licensed marks is neither. Conflating them would either cost the seller real revenue or walk them into a takedown. Etsy adds a third rule stricter than the law - no resale at all, regardless of legality.

## Documentation

- **[PRODUCT_DECISIONS.md](PRODUCT_DECISIONS.md)** - the full decision log: problem framing, platform research, every decision with its rationale and the alternatives rejected
- `SKILL.md` - the skill itself
- `references/` - per-platform mechanics, guardrail rules, classification logic, pricing cascade

## Status

Planning complete. Implementation in progress.
