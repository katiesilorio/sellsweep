# Intake

Three front doors, one destination. All modes converge on the review CSV.

| Mode | Trigger | Best for |
|---|---|---|
| **Chat photos** | User drops images into the conversation | 1-15 items, ad hoc - "someone just texted me these" |
| **Folder + CSV** | Folder path plus a filled CSV | Large backlog, details already recorded |
| **Folder only** | Folder path, no CSV | Large backlog, no details yet |

Detect the mode from what the user provides. Don't ask them to pick.

## Chat photos - the default path

Photos dropped into the conversation land on disk under the uploads directory and are readable there. Confirm the files exist before promising anything downstream - posting needs the actual file, not just a view of it.

### Step 1 - Group before asking anything

A batch is almost never one-photo-per-item. Expect angles, back shots, detail crops, and staged-versus-plain-background pairs of the same object.

Group by visual similarity, then confirm in a single message:

```
I see 9 photos - looks like 3 items:
  • Scripture tile, sunflower / Matthew 21:22 - 2 photos
  • Team logo tile, red and gold train - 5 photos
  • Wood key holder, baseball - 2 photos

Right, or did I split something wrong?
```

**Do not skip this confirmation.** Wrong grouping corrupts every downstream step, and it's the cheapest possible error to catch.

Same item: identical object at different angles, matching background and lighting, consecutive filenames or timestamps, one shot clearly the back or a detail of another.

Different items: different object shape, different subject, different staging.

When genuinely unsure, ask about that specific pair rather than guessing.

### Step 2 - Pick the primary photo

Per item, choose the best as primary - sharpest, best-lit, fully in frame, cleanest background. Keep the rest as additional images in sensible order: front, angle, detail, back.

Flag quality problems now, before generating copy: blur, harsh shadows, cluttered background, item cropped, personal items visible in frame.

**Reject supplier, manufacturer, studio, and stock photography.** If an image looks like catalog work rather than something the user shot, say so. Using it is a separate infringement from anything about the product itself, and it's the most common cause of takedowns on items that are otherwise perfectly legal to sell.

### Step 3 - Ask in one consolidated pass

**Batch by item type. Never interrogate per item.**

```
Three tiles and one key holder. To price and list these:

Tiles - all three the same size (looks like 4x4 ceramic)?
  What do you charge, and what's your floor?
  Made by you, or bought finished?

Key holder - same questions, plus how many you have.

Skip anything you don't know; I'll leave it blank in the CSV.
```

Ask once per item *type*, then apply across the group. Everything except photos is optional - blanks show up in the CSV for the user to fill or ignore.

### Step 4 - Emit the CSV

Chat intake **produces** the review CSV rather than consuming one. Write it, show a compact summary, and let the user edit before anything posts.

This is what makes the modes converge: whichever door they came in, review and posting are identical.

## Folder + CSV

Validate before processing:

- Every `photo_files` entry resolves to a real file
- Photos in the folder with no CSV row → report, don't silently skip
- CSV rows referencing missing photos → report and skip that row
- Required columns present

Then go straight to classification - the questions are already answered.

## Folder only

Identical to chat photos, higher volume. Group, confirm, batch-ask, emit CSV.

For large folders, work in batches of 10–20 and check in between. Never run silently through 300 items - a flaw in the classification logic should surface on item 5, not item 300.

## Photo naming, for users doing volume

Once someone is regularly working in batches, suggest a convention so grouping becomes trivial:

```
{item-slug}_{sequence}.jpg
scripture-tile-sunflower_01.jpg
scripture-tile-sunflower_02.jpg
keyholder-baseball_01.jpg
```

Mention it once, when they first hit a batch worth naming. Never require it - the grouping logic exists precisely because real photo batches are messy.
