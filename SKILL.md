---
name: sellsweep
description: Turn photos of items into complete, ready-to-post listings and publish them across eBay, Etsy, Square, and Facebook Marketplace. Use when the user wants to list items for sale, post products to marketplaces, bulk-list inventory from a folder of photos, write listing descriptions or titles, price an item against comparables, or set up listings that accept custom/personalized orders. Triggers include "list this", "post these items", "create listings", "put this on eBay/Etsy/Square", "bulk list my inventory", "write a listing for this", "what should I price this at", "make this listing customizable".
---

# Sellsweep

Do the thinking work once, then eliminate the transcription tax.

Every marketplace wants the same underlying information - photos, title, description, price, category, condition - in a different shape, with different limits and a different taxonomy. Sellsweep generates that once and adapts it per destination.

## The one hard rule

**Never post anything the user has not reviewed.** Generated copy is fluent by default and wrong often enough to matter. A confident, well-written listing with the wrong category or a bad price is worse than no listing. Every path in this skill routes through a review step.

## Flow

```
intake → classify → research price → generate → review (CSV) → recommend destinations → guardrail check → post → log
```

Three intake modes. Detect which applies; don't ask the user to declare it.

- **Chat photos** - the user drops images directly into the conversation. This is the most common real path: someone texts them a batch of product photos and they forward them along. Never make the user build a spreadsheet from photos they were just handed.
- **Folder + CSV** - a folder of photos plus a filled CSV (see `assets/listing_template.csv`). For a large backlog where details are already recorded.
- **Folder only** - a folder of photos, no CSV. Treat like chat photos, at higher volume.

All three converge on the same review CSV. Chat intake *produces* the CSV rather than requiring one. See `references/intake.md`.

### Step 1 - Setup

**Read `references/setup.md` and run the guided flow.** It is an interactive, resumable, gate-by-gate walkthrough - not a checklist to hand over.

Enter setup when any of these is true:
- `.env` is missing or lacks credentials for a requested destination
- `seller_profile.md` doesn't exist
- a refresh token has expired (Etsy's lasts ~90 days)
- the user asks to set up, connect, or add a platform

The flow builds OAuth consent URLs, exchanges authorization codes, writes `.env`, opens the right browser pages, and verifies every credential with a real API call. The user only does what requires them: signing in, approving consent, clicking through dashboards.

**Never enter a password.** If a login screen appears during browser work, hand it to the user.

Setup is resumable across sessions - read `SETUP_LOG.md`, resume at the first unchecked gate, and never re-ask something already answered. Record each gate's outcome as it completes.

If setup is already complete, just load `.env` and `seller_profile.md` and continue. Don't re-run gates.

### Step 2 - Intake

**Check the ledger first.** Read `inventory.csv` and look for a matching SKU or near-identical title before creating anything. Re-listing something already live is how a quantity of one gets sold twice.

**Group photos into items before asking anything.** A batch of 8 photos is rarely 8 products - usually 2 or 3 items shot from several angles plus a back shot. Group by visual similarity, confirm in one message, and get that confirmed before generating. Wrong grouping corrupts every downstream step and is the cheapest error to catch.

**Batch questions by item type, never per item.** Someone listing 40 items does not want 40 conversations. Ask once per type, apply across the group.

**Required: photos.** That is the only mandatory input. Everything else is optional-but-encouraged, asked once, skippable, never blocking:

- Price, or the lowest price they'd accept
- Keywords / key features
- Disclaimers (condition notes, made-to-order timing, etc.)

**Assess photo quality before generating anything.** Check resolution, blur, lighting, background clutter, and whether the item is fully in frame. Flag problems and suggest a re-shoot. Image quality drives conversion directly, and eBay selects listings for Facebook Marketplace surfacing partly on "listing quality" - so a bad photo costs reach, not just clicks.

**Photos must be the user's own.** Never use a supplier's catalog image, a manufacturer photo, or a studio still. This is a separate infringement from anything about the product itself, and it is the most common cause of takedowns on items that are otherwise perfectly legal to sell. If the user supplies an image that looks like stock or catalog photography, flag it.

### Step 3 - Classify

Read `references/classification.md`. Every item gets two independent labels:

- **Provenance** - did the user make it, or buy it to resell?
- **IP status** - does it bear third-party trademarks or copyrighted imagery?

These vary independently. A hand-made team-logo tile is user-made *and* IP-bearing. A generic floral sign bought wholesale is resale *and* IP-clean. Getting this wrong routes an item somewhere it will be removed, so **when uncertain, ask rather than guess.**

### Step 4 - Research price

Read `references/pricing.md`. Cascade, in order, stopping when you have enough:

0. **The seller's own sales history** (`sales_log.csv`), where 5 or more sales match on item type, material, and theme
1. Licensed comps API, where the vertical is covered
2. The user's own marketplace research tools, via their authenticated session
3. Active asking prices via sanctioned APIs
4. Model judgment with an explicit range

**Below 5 comparables, do not report a point price.** Report a range, label confidence as low, and name which tier produced it. Always surface provenance - the user should never see a number without knowing where it came from.

Tier 0 outranks everything below it because it is what this seller actually realized, not what someone else is asking. It also carries `days_listed`, the only signal that separates "priced right" from "priced too low and it moved fast." Below 5 matching sales it has no power at all; skip it and say so rather than leaning on one data point.

For new, made-to-order goods, active competitor asking prices are the correct signal, not sold comps. Sold comps are appraisal logic for unrepeatable items. Correct for stale-listing bias by weighting competitors that show evidence of actually selling.

### Step 5 - Generate

Per destination, produce: title (within that platform's length limit), description, category, condition, item specifics/attributes, tags/keywords, and variation structure if applicable.

Compose descriptions as `[item-specific generated content] + [profile boilerplate]`.

**Variations:** build to the intersection of destination capabilities, then enrich per platform. Internal model:

```
{
  axes: [{name, values[]}],           // max 2 in v1 (Etsy's limit)
  combinations: [{axis_values, sku, price, qty, image_ref?}]   // max 250
}
```

Per-variation images are an enhancement, never a requirement - not every platform supports them.

**Custom-order capable listings:** where the user wants an item to accept personalization, configure it at listing time - Etsy personalization questions, Square text modifiers, bulk-pack variations. Sellsweep configures the listing. It does not handle what happens after someone orders.

### Step 6 - Review

Emit a CSV (schema in `assets/listing_template.csv`). One row per listing, or per variation for variation listings, with a column referencing photo filenames.

The user edits the CSV. This is the checkpoint that catches fluent-but-wrong output, and it is not optional.

### Step 6.5 - Recommend destinations

Read `references/routing.md`.

**Check `inventory.csv` for existing placements first.** Never recommend a platform where the item is already live.

**Propose where each item should go. Do not wait to be asked.** The seller should not have to know which marketplace fits which item; working that out is the skill's job.

Weigh six factors, not just IP: provenance, IP status, price point, audience match, shipping economics, and repeatability. They trade off against each other, and price or shipping can veto an otherwise good audience fit.

Present a recommendation with **where, why, why not the others, and what risk is being accepted.** Never present a bare menu of platforms.

The seller overrides freely. That is the design.

### Step 7 - Guardrail check

Read `references/guardrails.md`. For every (item, destination) pair, evaluate:

- **Validation** - hard limits the API enforces. Not overridable; fix or report.
- **Guardrails** - policy and risk rules. Block or warn, both overridable with explicit acknowledgment.
- **Throttle** - pacing. Automatic and silent.

Present blocks and warnings together, per item, with the specific consequence of overriding. Let the user decide. **Log every override** - timestamp, item, destination, rule, reason - to `override_log.md`. If a listing is later removed, that record is what supports an appeal.

### Step 8 - Post

Read `references/platforms.md` for per-destination mechanics.

- **API destinations** - post directly, variations intact.
- **Browser destinations** - drive Chrome to fill the platform's own form completely, including photo upload, then **stop before submit**. The user reviews and publishes.

Never submit a form on a browser destination. The stopping point is the feature: it removes the retyping while keeping a human gate and keeping the behavioral signature far from bot territory.

A failure on a browser destination must never block API destinations. Post what can be posted; report what could not.

### Step 9 - Log

Read `references/inventory.md`. Update `inventory.csv`: one row per item touched, carrying platform listing IDs, per-platform price, per-platform status, and `last_updated`. This is the ledger, and it is the only thing that knows all of an item's placements at once.

Also note what was skipped and why, and what was overridden.

**When something sells:** append to `sales_log.csv`, decrement `quantity`, and **name every other platform where that item is still live.** A quantity of one that sold on Square is still for sale on eBay until someone ends it. Offer to end them; never end them silently.

## Working style

- Check in after each batch rather than running hundreds of items silently.
- When a platform rejects something, report the actual error - don't retry blindly.
- Prefer telling the user something can't be done over finding a clever way around a platform's rules.
- The user owns the risk decisions. Surface them clearly, then respect the answer.

## Reference files

| File | Read when |
|---|---|
| `references/intake.md` | Every run - mode detection, photo grouping, question batching |
| `references/routing.md` | Every run - recommending destinations before the guardrail check |
| `references/setup.md` | First run, or any missing credential - the guided onboarding flow |
| `references/auth.md` | Start of every run - reading `.env`, token refresh, OAuth mechanics |
| `references/classification.md` | Every run - classifying items |
| `references/guardrails.md` | Every run - before posting |
| `references/platforms.md` | Posting; per-destination field mapping and limits |
| `references/pricing.md` | Researching price |
| `references/inventory.md` | Every run - the ledger schema, when to read it, when to write it |
| `assets/listing_template.csv` | Bulk intake, and generating the review CSV |
