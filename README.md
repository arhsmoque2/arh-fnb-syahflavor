# SyahFlavor QR Ledger

A one-page, stateless helper for a food truck operator drowning in manual Excel
work: today, closing out QR sales means exporting the Maybank QRPay report,
switching to the Downloads folder on a phone, and hand-summing columns per QR
terminal. This replaces that with one upload.

## What it does

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
adds or reorders columns, as long as `Date & Time` and a
`Cashier ID/Terminal ID` column are still present.

## Local dev

```bash
npm install
npm run dev      # wrangler dev, serves index.html locally
```

## Deploy

```bash
npm run deploy    # wrangler deploy — static assets only, no Worker code runs
```

## Scope

This is the first, most solvable piece of a larger reconciliation effort
(till flow + QR/cash payment tracking + bank-export matching) — deliberately
scoped down to just the Excel pain point, standalone, no dependency on the
rest of that system.
