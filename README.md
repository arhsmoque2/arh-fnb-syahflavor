# SyahFlavor QR Ledger

A one-page, stateless helper for a food truck operator drowning in manual Excel
work: today, closing out QR sales means exporting the Maybank QRPay report,
switching to the Downloads folder on a phone, and hand-summing columns per QR
terminal. This replaces that with one upload.

## What this is

The first, deliberately standalone piece of a larger reconciliation effort
(till flow with cash-default/QR-tap ordering, an online-vs-bank-export ledger
merge, eventually inventory tie-in). It has **no dependency** on the rest of
that system and never will need one for its own job — it only exists to kill
the specific "switch apps, hand-sum columns on a phone" pain point, standalone.

## What's built now

1. Upload the Maybank QRPay `.xlsx` export.
2. Everything is parsed and totalled **in the browser** — no upload to a
   server, no storage, nothing persisted. Close the tab and it's gone;
   reload to process another file.
3. You get two things back:
   - An on-screen summary: grand total, one subtotal card per QR ID
     (terminal), each tap-to-copy — built for one-handed use on a phone.
   - A **Download processed file** button that regenerates the spreadsheet
     sorted by QR ID, with a subtotal row per QR ID group and a grand total
     row at the bottom, saved as `processed-<original filename>.xlsx`.

"Summable" columns are auto-detected (any column where every value in the
sheet is numeric) — no hardcoded column list, so it keeps working if Maybank
adds or reorders columns.

Column detection is resilient to the export format itself changing, not just
column order: `Date & Time`, the QR ID / terminal column, and the amount
column are matched by header name first, and if that fails, the tool asks
before searching the data by shape, then asks again before trusting what it
found. See `AGENTS.md` for exactly how that works and where it can still be
fooled — worth reading before touching the detection logic.

## What this is planned to become

Not committed to here yet, but the direction this repo's sibling work has
been heading:

- A till flow (order confirm → cash default → QR tap-to-reveal → close
  falls back to cash if QR was never confirmed) — a separate tool, not this
  one, but expected to produce an "online ledger" this repo's output could
  eventually reconcile against.
- A merge step between that online ledger and this tool's parsed bank
  export — matched by amount + nearest time, unmatched items surfaced for
  manual review rather than silently absorbed.
- Snapshot-before-merge with instant restore (an append-only event log with
  a movable pointer, not literal file copies), and a mandatory preview
  before any merge is confirmed.
- Inventory tie-in (cups for drinks, packaging counts for food) — blocked on
  finding out what the business already tracks; not started.

None of the above is built. This repo should keep working exactly as it does
now regardless of whether any of it ever happens.

## Local dev

```bash
npm install
npm run dev      # wrangler dev, serves index.html locally
```

## Deploy

```bash
npm run deploy    # wrangler deploy — static assets only, no Worker code runs
```

Needs Cloudflare credentials in the environment — see `AGENTS.md` for the
working invocation on this machine, it is not a plain `npm run deploy`.

## Live

<https://arh-fnb-syahflavor.arh-homelab.workers.dev>
