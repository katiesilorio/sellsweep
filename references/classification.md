# Classification

Every item gets two **independent** labels. They do not correlate - resolve each separately.

## Axis 1 - Provenance

| Value | Meaning |
|---|---|
| `made_original` | The user made it, from their own design |
| `made_thirdparty_design` | The user made it, but the design is someone else's (a logo, a movie still, a purchased/templated design) |
| `resale_authentic` | Bought to resell, genuine branded/licensed product |
| `resale_generic` | Bought to resell, no third-party branding |

How to determine: ask. Vision can suggest - a hand-finished wood edge, an engraving, a print registration mark - but it cannot reliably distinguish "he printed this tile" from "he bought this tile." Ask once per item type, not per item: if the user says all their coasters are made in-house, apply that to the batch.

## Axis 2 - IP status

| Value | Meaning |
|---|---|
| `clean` | No third-party marks or copyrighted imagery |
| `trademark` | Team names, team logos, brand marks |
| `copyright` | Movie stills, poster art, character art, photographs |
| `both` | Common - a team logo *and* official photography |

Vision handles this well. Flag anything showing:

- Sports team names, logos, colors-plus-wordmark combinations (NFL, MLB, NBA, NCAA)
- Film and TV imagery, posters, character art
- Comic and animation characters
- Brand logos and product marks
- Celebrity likenesses
- Song lyrics, book passages, recognizable quotes

**Bias toward flagging.** A false positive costs one question. A false negative can cost an account.

## The matrix

| Provenance × IP | eBay | Etsy | Own store | FB Marketplace |
|---|---|---|---|---|
| `made_original` + `clean` | ✅ | ✅ | ✅ | ✅ |
| `made_original` + IP | ❌ block | ❌ block | ⚠️ user's risk | ⚠️ |
| `made_thirdparty_design` + clean | ✅ | ❌ originality rule | ✅ | ✅ |
| `made_thirdparty_design` + IP | ❌ block | ❌ block | ⚠️ | ⚠️ |
| `resale_authentic` (any IP) | ✅ **legal + permitted** | ❌ resale ban | ✅ | ✅ |
| `resale_generic` | ✅ | ❌ resale ban | ✅ | ✅ |

### Why resale of licensed goods is fine on eBay but not Etsy

Two different rules doing two different things:

- **The law** (first-sale doctrine, 17 U.S.C. § 109 plus trademark exhaustion): reselling a genuine, lawfully acquired licensed product is legal. eBay's own VeRO policy states rights owners may not use it to "control where or how a product's resold."
- **Etsy's policy** is stricter than the law: no reselling of commercially available goods at all, regardless of IP status or legality. The only exceptions are craft/party supplies and genuine **vintage - 20+ years old, handpicked.**

So an authentic licensed team flag from a wholesaler is fine on eBay and prohibited on Etsy. Don't apply one platform's rule to the other.

### Why manufacturing is different from reselling

First-sale exhausts the **distribution** right only. It does not touch **reproduction** or **derivative works**. Printing a logo or a film still onto a product you make is a new copy - first-sale is not a weak defense there, it is not a defense at all. Fair use, de minimis, and the artistic-works defense all fail for decorative reproduction on ordinary merchandise.

## When uncertain

Ask. Specifically:

- "Did you make this yourself, or buy it to resell?"
- "Is this your own design, or one you bought / found?"
- "Is that a [team/character] logo on it?"
- For plausibly old resale stock: "Roughly how old is this? Over 20 years opens up Etsy's vintage category."

Batch the questions by item type. Do not ask 300 times.

## Recording

Store both labels in the CSV (`provenance`, `ip_status`, `ip_detail`) so classification is visible, auditable, and correctable by the user before anything posts.
