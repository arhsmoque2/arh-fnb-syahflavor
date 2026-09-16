# AGENTS.md — arh-fnb-syahflavor

Read this before changing anything in `index.html` or the deploy setup.
`README.md` says what this is and where it's headed; this file is the
operational memory — the mistakes already made here, so you don't remake
them, and where to go for anything not obvious from a cold start.

## Cold-start map

| Need to... | Do this |
| --- | --- |
| Run it locally | `npm install && npm run dev` (wrangler dev, serves `index.html`) |
| Deploy | See "Deploying on this machine" below — **not** plain `npm run deploy` |
| Understand the real file this parses | See "The actual Maybank export" below |
| Touch column-detection logic | Read "Column detection: two failure modes already found" first — both were real bugs caught by testing, not hypotheticals |
| Verify a parsing change without a browser | See "Verifying without the browser tool" below |
| Understand where this fits in the bigger picture | `README.md` → "What this is planned to become". This repo has no code dependency on any of it and shouldn't grow one casually |
| Find the sibling repos this borrowed patterns from | `D:\ARH-GITHUB\arhsmoque2\ARH-FNB-Beelal-Coffee` (QR-proof + cashier-handshake pattern, ADR-0012 in `ARH-MAKAN/docs/decisions/`) — check there before building something new that looks like ordering/payment UI |

## Deploying on this machine

`npm run deploy` / plain `wrangler deploy` will fail with "not
authenticated" — there's no `wrangler login` session here. Credentials live
in the SOPS vault and must be injected:

```bash
arh secrets inject --id CLOUDFLARE_API_TOKEN --id CLOUDFLARE_ACCOUNT_ID -- wrangler deploy
```

Two traps in that one line:

- **`arh secrets inject` cannot spawn a `.cmd` shim on Windows.** If the
  command after `--` is `npx ...`, it fails with
  `Error running injected command: [WinError 2] The system cannot find the
  file specified` — every time, regardless of `--id` vs the older
  `KEY=ID,KEY=ID` syntax, regardless of Bash tool vs PowerShell tool. It's
  not an arg-parsing issue; the subprocess spawn itself can't resolve a
  `.cmd`. Workaround: call a real `.exe` directly. `wrangler.exe` is
  already on PATH via bun's global bin
  (`C:\Users\<user>\.bun\bin\wrangler.exe`) — use `wrangler deploy`, not
  `npx wrangler deploy`.
- **The syntax is `--id <ID> --id <ID2> -- <command>`**, one flag per
  secret. A comma-joined `"KEY=ID,KEY=ID"` positional form will also fail
  silently the same way — it's not a valid syntax, it just happens to
  produce the identical unhelpful WinError. Confirmed the right form by
  reading `arh-secrets-vault/AGENTS.md`, not by guessing.

## Cloudflare Workers static-assets deploy uploads the whole directory

`assets.directory: "."` in `wrangler.jsonc` means wrangler ships **every
file it finds**, including `.git/*`. First deploy of this repo actually
served `.git/config` and `.git/objects/*` publicly at the live URL. Fixed
with `.assetsignore` (same idea as `.gitignore`, Cloudflare-specific,
already in this repo — don't remove entries from it without checking what
they're hiding first).

Don't trust wrangler's "No updated asset files to upload" message on a
redeploy as confirmation that `.assetsignore` took effect — that message
just means content hashes matched a previous upload, and doesn't
distinguish "correctly excluded" from "already had this queued." The only
real confirmation is an actual HTTP check:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://arh-fnb-syahflavor.arh-homelab.workers.dev/.git/config
# must be 404
```

## git default branch here is `master`, GitHub convention here is `main`

`git init` on this machine defaults to `master`. Every sibling repo and
`gh repo create` expect `main`. Rename before the first push:
`git branch -m master main`.

`gh repo create --push` also defaults to an SSH remote
(`git@github.com:...`), which fails here with "Permission denied
(publickey)" — no key registered for plain `github.com` (some sibling repos
use a custom SSH host alias like `github.com-arhsmoque2` instead). Simplest
fix, matching what `ARH-FNB-Beelal-Coffee` already does: switch the remote
to plain HTTPS before pushing —
`git remote set-url origin https://github.com/arhsmoque2/<repo>.git`.

## Column detection: two failure modes already found by testing, not theory

The detection logic (in `index.html`) tries an exact/substring header-name
match first, and only on a miss — after the user explicitly opts in — falls
back to searching the data by shape. Both fallback heuristics had a real bug
caught by simulating a renamed column against the actual export, not by
inspection:

1. **QR ID / terminal column**: "alphanumeric, length ≥ 4" is *not*
   selective enough on this schema — Merchant ID, Corporate ID, and
   Transaction ID are all alphanumeric strings of that length too. The
   heuristic needs a second signal: a QR ID **repeats** across many rows (a
   handful of distinct terminals, not one-per-row like Transaction ID and
   not constant like Merchant ID). The current bound is "more than 1
   distinct value, but no more than `max(10, rows*0.05)` distinct values" —
   if you change this, re-test against the real export, not a synthetic
   one; the real file is what defines what "selective enough" means here.
2. **Amount column**: picking "the numeric column with the largest average"
   fails when a before-discount and after-discount column are numerically
   identical — which they currently **are** in this merchant's real data,
   because every discount-related column (`Discount Rate`, `Discount
   Amount`, `MBB Promo (RM)`, `GST`) is 0 throughout. A first attempt at
   flagging this ambiguity was itself buggy: it only checked for ties
   *within* the header-hint-filtered candidate pool (columns whose name
   contains "amount"/"paid"/"total"), so a renamed column that happened to
   lose its hint keyword dropped out of the pool entirely and was never
   compared — the tie went undetected. Fixed by computing ties against the
   full numeric candidate set, always, regardless of hint filtering. If you
   touch this again: the tie-check has to run over every candidate, not
   just whichever subset got ranked first.

Both are one hard requirement away from silent wrongness: if you "simplify"
either heuristic back to a single signal, you will reproduce a bug that was
already found and fixed once.

## Verifying without the browser tool

`claude-in-chrome` was unavailable this session ("Browser extension is not
connected"). Rather than skip verification, the parsing/detection logic was
tested by running the *same* logic in Node against the real downloaded
export, using the same `xlsx` (SheetJS) package the page loads from cdnjs —
close enough fidelity to trust, since it's the identical library:

```bash
npm install xlsx --no-save   # in a scratch dir, not this repo
node -e "..."                 # port the relevant function(s) from index.html, run against the real file
```

This is the fallback pattern whenever the browser tool isn't connected —
don't ship an unverified change to detection logic just because the browser
tool wasn't available; port the logic to Node and run it against the real
file instead.

## The actual Maybank export

Real file used for all verification in this repo: a Maybank QRPay merchant
report (`QRPay_transactions_report_*.xlsx`), one sheet, these columns in
this order:

```
Date & Time, Transaction ID, Corporate ID, Outlet Name, Merchant ID,
Cashier ID/Terminal ID, Reference Number, Promo Code, Promo Type,
Promotion, Actual Amount Before Promo (RM), Discount Rate, Discount Amount,
MBB Promo (RM), Amount Paid to Merchant (RM), GST, Merchants Reference No,
Client Reference (API)
```

Facts worth knowing before changing anything that touches these columns:

- **`Cashier ID/Terminal ID` is the "QR ID."** One Merchant ID can have
  several named Terminal IDs — Maybank QRPayBiz's multi-terminal feature,
  one settlement account serving several physical QR standees. In the real
  387-row export: two named terminals
  (`...SYAHFLAVOR02` = food till, `...MCXAIR7954` = drinks — "AIR" is Malay
  for water, corroborated by a lower/more-uniform amount cluster, RM6/RM8
  dominant) plus 86 rows with a **blank** terminal ID — a third,
  unregistered QR standee. That third one needs the business owner to name
  it properly in QRPayBiz before it'll show up labeled in future exports;
  nothing this repo does can retroactively label it.
- **`Client Reference (API)` is present but always empty** in the real
  data — it's Maybank's merchant-supplied-reference slot for API-driven
  dynamic QR generation, unused because this merchant only displays a
  static QR, no API integration. If that ever changes, this column becomes
  the exact-match join key and the amount/time heuristics in this repo
  become unnecessary for anything using that flow.
- **`Reference Number` has two visibly different formats** in the same
  file — e.g. `999999133Q` (digits + `Q` suffix) vs `QR68929072` (`QR` +
  digits). Never confirmed with the merchant what distinguishes them
  (possibly two different QR-generation paths, static vs dynamic) — don't
  build logic that assumes one format without checking first.
- **Every discount-related column is 0 throughout this merchant's real
  data.** This isn't a schema fact that's guaranteed to stay true — if it
  ever stops being true, `Actual Amount Before Promo (RM)` and `Amount Paid
  to Merchant (RM)` will finally diverge, which is a good thing for
  disambiguation but means totals that matched before may start looking
  "different" even though nothing broke — check discount columns first if
  a total ever looks off after this changes.

## Deliberately stateless — don't add a backend without being asked

`wrangler.jsonc` has no `main` Worker script, only `assets.directory`. All
parsing happens client-side. This is a privacy property the tool was built
around (a food truck's real transaction data never leaves the phone it's
processed on), not an oversight — don't add an API route or server-side
processing to "improve" this without the user explicitly asking for it.
