# VSDham Sales-Purchase App — Database Schema (Stage 1)

This is the foundation of the app: the Supabase/Postgres schema, FEFO stock
logic, and role-based permissions. Everything below has been **actually run
and tested** against a real Postgres instance (not just written and assumed
correct) — see "What was tested" at the bottom.

## Files

1. **`001_schema.sql`** — all tables, the two core functions
   (`create_stock_batch`, `create_sale_with_fefo`), and two reporting views
   (`current_stock`, `expiry_alerts`).
2. **`002_rls_policies.sql`** — Row Level Security policies that enforce:
   - SPOC: full read/write on product master, vendors, shelf-life slabs.
     **Insert-only** (no edit/delete, ever) on purchases (stock_batches),
     sales, expenses, and account balances.
   - Admin: everything SPOC can do, **plus** edit/delete/correct any entry,
     and manage center settings.
   - A SPOC only sees and affects their **own assigned center's** data.
     Admin sees/affects all centers.
   - Every admin correction to a locked record is automatically written to
     `audit_log` (old value, new value, who, when) — so nothing gets
     silently changed.

## How to deploy

1. Open your Supabase project → SQL Editor.
2. Run `001_schema.sql` first, then `002_rls_policies.sql`.
3. That's it — Supabase already provides `auth.users` and `auth.uid()`
   natively, so no extra setup is needed (unlike my local test environment,
   where I had to stub those out — see note below).
4. Create your first Admin and SPOC users via **Supabase → Authentication →
   Users**, then add a matching row in the `profiles` table for each,
   setting their `role` (`admin` or `spoc`) and `center_id`.

## How the seasonal shelf-life works

Each product can have multiple rows in `product_shelf_life_slabs`, e.g.:

| Product    | Valid From | Valid To | Shelf Life |
|------------|-----------|----------|-----------|
| Wafer 100g | 03-01     | 06-15    | 15 days   |
| Wafer 100g | 06-16     | 10-31    | 20 days   |
| Wafer 100g | 11-01     | 02-28    | 30 days   |

Dates are stored as `MM-DD` (month-day only, no year), so the same slabs
apply every year without re-entry. Ranges that cross the calendar year-end
(like `11-01` to `02-28` above) are handled correctly — tested explicitly.

When a SPOC scans a product and enters its manufacture date, the function
`create_stock_batch()` automatically finds the matching slab for that date
and computes the expiry date — no manual calculation needed.

## How FEFO selling works

When a sale is recorded via `create_sale_with_fefo()`, the system:
1. Checks total available stock across all active batches of that product
   at that center. If not enough, it **blocks the sale** with a clear error.
2. Pulls stock from the batch with the **nearest expiry date first**,
   splitting across multiple batches if one batch alone isn't enough.
3. Records exactly which batch(es) supplied how much, in `sale_allocations`
   — so you always have a precise trail.

This matches how your SPOCs physically arrange stock (near-expiry items in
front), keeping the digital record in sync with the shelf.

## What was tested (not just written — actually run)

I built a real Postgres instance in my working environment and ran both
SQL files against it, then exercised the actual behavior:

- ✅ Seasonal shelf-life resolution for 4 different dates, including a
  year-end-wrapping slab (Dec 25 and Jan 15 both correctly matched the
  Nov–Feb slab).
- ✅ Three stock batches of the same product with different manufacture
  dates → correctly stored as separate lines with independently computed
  expiry dates.
- ✅ FEFO sale of 8 units correctly split 5 from the nearest-expiry batch
  + 3 from the next, leaving the furthest-expiry batch untouched.
- ✅ Selling more than available stock is blocked with a clear error
  message stating exactly how much is available.
- ✅ SPOC's attempt to UPDATE or DELETE a stock batch, an expense, after
  creation is silently blocked by RLS — confirmed the row is unchanged.
- ✅ Admin's correction to the same row succeeds **and** is automatically
  recorded in `audit_log` with old/new values.
- ✅ Routine FEFO-driven stock depletion during a sale does **not** get
  mistaken for an admin correction in the audit log (this was a real bug
  I caught and fixed during testing — see note below).
- ✅ A SPOC from one center cannot see another center's expenses, and is
  hard-blocked from writing stock into another center.
- ✅ A product with `expiry_alert_days = 0` never appears in the expiry
  alert view, while a product with a threshold correctly appears once
  within its window.

**One real bug caught during testing:** my first draft of the audit
trigger logged *every* update to `stock_batches`, including the system's
own routine stock depletion during a sale — which would have wrongly
filled your audit log with "admin correction" entries for ordinary sales.
Fixed by having `create_sale_with_fefo()` flag its own updates as routine,
so the audit trail only captures genuine admin corrections.

## Note on local testing vs. real Supabase

Plain Postgres doesn't have Supabase's `auth.users` table or `auth.uid()`
function — those are part of Supabase's hosted auth layer. To test locally,
I created a throwaway stub for them. **You don't need this stub file** —
it's not included in your deliverable, since real Supabase provides this
natively.

## How centers work (fully independent)

Each center has its own completely independent product master and vendor
master — not shared or global. The same real-world product (e.g. "Wafer
100g") sold at two different centers is represented as **two separate
rows** in `products`, each with its own:
- buying price and selling price (a center can pay/charge differently)
- QR code (scanning resolves directly to one specific center+product)
- shelf-life slabs (attached to that center's product row)
- vendor relationships (each center has its own vendor list — even if
  it happens to be the same real-world vendor)

A product's SKU and QR code only need to be unique **within one center**,
not globally — so "WAF100" can exist at every center simultaneously as
unrelated rows.

Because the QR encodes a specific center+product combination, scanning it
already tells the system exactly which center the transaction belongs to
— `create_stock_batch()` and `create_sale_with_fefo()` no longer take a
separate center parameter; they derive it from the scanned product itself,
which also makes it impossible to accidentally record a transaction
against the wrong center. If a vendor is specified for a purchase batch,
the system verifies it belongs to the same center as the product and
blocks the save with a clear error if not — tested explicitly.

Row Level Security enforces this same independence at the database level:
a SPOC at one center cannot see, list, or modify another center's
products, vendors, shelf-life slabs, stock, sales, expenses, or balances
— confirmed by test (a Thane SPOC querying Kalyan's data gets zero rows,
and an update attempt against Kalyan's product silently affects nothing).

## Note on this design's evolution

This schema went through one structural revision after initial review:
products and vendors were originally global tables shared across centers.
Once it was confirmed that different centers may charge different prices
for the "same" product and print separate QR codes, both tables were
changed to carry `center_id` directly, with their uniqueness constraints
(SKU, QR code) re-scoped to be unique per-center rather than globally.
All functions, views, and RLS policies were updated and re-tested against
a real two-center scenario to confirm correctness end-to-end.

## Not yet built (upcoming stages)

- Expense & account balance entry screens (UI)
- Downloadable reports (stock, expense, profit — daily/monthly/yearly)
- Dashboard with expiry alerts

We'll tackle these one at a time, same as this stage.

---

## Stage 3: Purchase & Sales Entry App (`vsdham_entry_app.html`)

A single self-contained HTML file with three screens: **Login**, **Sales**
(scan-to-cart), and **Purchase** (scan → confirm date → enter qty). It
connects directly to your Supabase project using the embedded Supabase JS
client — no separate backend server needed.

### Before you use it — one-time setup

Open `vsdham_entry_app.html` in a text editor and find these two lines
near the top of the script (search for `YOUR-PROJECT-REF`):

```js
const SUPABASE_URL = 'https://YOUR-PROJECT-REF.supabase.co';
const SUPABASE_ANON_KEY = 'YOUR-ANON-PUBLIC-KEY';
```

Replace both with your actual project's values, found in your Supabase
project under **Settings → API**. Use the **anon / public** key here —
never the service role key, since this file runs in the browser and
anyone using the app can see its source.

### How the sales screen works
- Scan a product/rack QR → looked up against `products`, scoped to the
  logged-in SPOC's own center (centers are independent, so a SPOC can
  only scan their own center's products).
- New product → added as a new cart row with quantity 1.
- **Same QR scanned again → that row's quantity goes up by 1** (not a
  new row) — tested explicitly.
- Quantity can also be edited directly by typing into the row, or with
  the +/- buttons.
- **Confirm Sale** calls the database's `create_sale_with_fefo` function
  once per cart row (so 3 different products in the cart = 3 separate
  sale records, each independently FEFO-allocated) — tested explicitly,
  confirmed exactly one RPC call per row with the correct product and
  quantity.
- If a row fails to save (e.g. insufficient stock), it stays in the cart
  so the SPOC can see what went wrong and retry; successfully saved rows
  are removed so they aren't submitted twice.

### How the purchase screen works
- Scan a product/rack QR → shows product name and buying price, with a
  manufacture-date field (defaulting to today, editable) and a quantity
  field.
- Save calls `create_stock_batch`, which auto-resolves the shelf-life
  slab and expiry date as already verified in Stage 1.
- A running session history shows each entry made and whether it
  succeeded, so the SPOC has a visible trail without leaving the screen.

### What was tested (and how, given sandbox limitations)

I cannot connect this sandboxed environment to a real Supabase project
(the network here is restricted to package registries only, not
`supabase.co`). So testing happened in two layers:

1. **Real, verified**: the Supabase JS client itself was bundled from the
   real `@supabase/supabase-js` npm package (not hand-written), and
   confirmed to initialize correctly and expose the expected `.from()`
   and `.auth` methods.
2. **Logic-tested against a realistic mock**: I built a simulated
   Supabase client that mimics real responses (product lookup, login,
   RPC calls) and ran the actual app code against it in a real DOM
   environment, confirming:
   - Login loads the correct profile and shows the right screen.
   - Scan → cart-add → rescan → qty increment → manual edit → cart total
     recalculation, all in sequence, produced the exact expected values
     at every step.
   - Confirm Sale issues exactly one `create_sale_with_fefo` RPC call per
     distinct cart row, with the correct `product_id`, `quantity`, and
     `scanned_by` values.
   - A SPOC scanning another center's QR code is correctly rejected
     ("No product found... at your center") rather than silently leaking
     data — consistent with the fully-independent-centers design.
   - Purchase save calls `create_stock_batch` with the right parameters,
     including a manufacture date that matches the real current date.
   - Failure handling: a simulated "insufficient stock" RPC error is
     shown clearly to the SPOC, and the affected cart row is kept (not
     silently dropped) so they can see and retry it.
   - Edge cases: empty scans and unknown/typo'd QR codes are handled
     without crashing, with a clear message either way.

**What still needs verification once you plug in your real project
credentials:** the actual network round-trip to Supabase, real
authentication behavior, and that your real RLS policies (already tested
independently in Stage 1) behave the same way when called through this
app rather than through a direct SQL session. I'd recommend testing
login + one scan of each type with a test SPOC account before handing
this to your actual volunteers.


---

## Stage 2: QR Label Sheet Generator (`qr_label_sheet_generator.html`)

A single, self-contained HTML file — open it directly in any browser, no
internet connection or installation needed (the QR code library is embedded
inline, not loaded from a CDN). Print directly to A4 or save as PDF.

It supports **two modes**, switchable via tabs at the top:

### Mode A — Sticker mode (1 product per sheet)
One A4 sheet per product, with that product's QR code + name repeated 48
times (6×8 grid, sized to match standard 30×30mm 48-label A4 sticker sheets
available in the market). Print one sheet per SKU, cut, and stick a label
on each incoming packet as stock arrives.

### Mode B — Rack/Catalog mode (all products together)
All your products packed together, **one QR per product**, across as few
A4 sheets as possible (48 different products per sheet). For example, 80
products fit on exactly 2 sheets (48 + 32) — verified by test.

This supports an alternative workflow: instead of labelling every incoming
packet, the SPOC sticks **one QR permanently on the product's rack/shelf**.
Stock-in and sales are then recorded by scanning the rack's QR (entering
mfg date + qty for purchases, or qty for sales) — no per-packet labelling
needed at all. Print once, cut out each product's QR, stick it on the
matching rack.

Both modes use the same QR/label size for reliable scanning — Mode B does
not shrink QR codes to fit more in, per your requirement.

### How to use it
1. Open `qr_label_sheet_generator.html` in any browser (Chrome, Edge, Firefox).
2. Pick a mode (Sticker or Rack/Catalog) using the tabs.
3. Paste your product list, one per line: `QR_CODE_VALUE | Product Name`
   (the QR_CODE_VALUE should match each product's `qr_code_value` in the
   `products` table from Stage 1).
4. Click **Generate sheets** to preview, then **Print / Save as PDF**.
5. Adjust the **label size (mm)** field if your sticker sheets use a
   slightly different size than 30mm — the layout adapts automatically.

### What was tested
- Both modes generate the correct number of sheets and exactly 48 cells
  per sheet, with QR codes actually rendering (not just placeholder boxes).
- The 80-products-in-2-sheets scenario was tested explicitly and confirmed
  exact: sheet 1 has products 1–48, sheet 2 has products 49–80, with the
  remaining 16 cells on sheet 2 correctly left blank.
- Switching back and forth between modes regenerates cleanly with no
  leftover/stale content from the previous mode.
- Edge cases: blank lines, lines with only a code and no name (falls back
  to using the code as the label), extra whitespace, and special characters
  in product names — all handled without errors.
- **One real bug caught and fixed during testing:** my first draft tried to
  find the QR's target DOM cells via `querySelector` inside the QR
  library's callback, but the library's callback can fire **synchronously**
  — before the cells were appended to the page. This caused 0 QR codes to
  render despite no visible error. Fixed by building all cells first, then
  generating and injecting the QR SVG into already-existing cells.
- No external CDN dependency: the QR generation library is bundled and
  embedded directly in the HTML file, so it keeps working even with no
  internet connection, on any device, indefinitely.

---

## Schema addition for Stage 4: auto-generated QR codes

`001_schema.sql` now includes a new table (`center_qr_sequences`) and
function (`next_product_qr_code`) that auto-generate a unique QR code
value per center in the form `{center_code}-{3-digit sequence}` — e.g.
`KAL01-001`, `KAL01-002`. **If you already ran the schema in Supabase
before this addition, just re-run `001_schema.sql` and `002_rls_policies.sql`
again** — re-running is safe and won't disturb your existing data, though
as always, test on a non-production project first if you've already got
real data in there.

The function uses row-level locking on the counter table so two SPOCs
creating products at the exact same moment can never be handed the same
code — verified by firing 20 truly concurrent calls and confirming all
20 results were unique and sequential with no gaps or duplicates.

---

## Stage 4: Product & Vendor Master (`vsdham_master_app.html`)

A single self-contained HTML file with two screens: **Vendors** and
**Products**, both scoped to the logged-in SPOC's own center (consistent
with the fully-independent-centers design from Stage 1).

### Before you use it

Same one-time setup as Stage 3 — open the file and replace
`YOUR-PROJECT-REF` and `YOUR-ANON-PUBLIC-KEY` near the top of the script
with your actual Supabase project values.

### Vendors screen
- Add a vendor (name required; contact person, phone, email, address,
  GST number, notes all optional).
- Click any vendor in the list to edit it, including marking it
  inactive (a soft-delete — the vendor disappears from active dropdowns
  but its history with past purchases is preserved).

### Products screen
- **Add product**: name, SKU, category, unit, buying price, selling
  price, expiry alert threshold (days before expiry, 0 = no alert
  needed), and an optional primary vendor (picked from the vendor list).
- **QR code is auto-generated** the moment you save a new product —
  you never type it in. It's shown read-only and stays fixed for that
  product's lifetime (editing other fields later does not regenerate
  it, since the QR may already be printed and stuck on a rack).
- **Shelf-life (seasonal slabs)**: add one or more date-range rows
  (`MM-DD` to `MM-DD` → number of days), exactly matching the seasonal
  shelf-life logic verified in Stage 1. At least one range is required.
  Validation rejects malformed dates (e.g. `13-99`) before saving.
- **After saving a new product**, a link appears to open the **QR Label
  Sheet Generator (Stage 2)** in a new tab, pre-filled with that
  product's code and name — ready to print immediately. (This works by
  passing `?code=...&name=...` in the URL; Stage 2 was updated to read
  these and pre-fill its input automatically when present, instead of
  showing its sample data.)
- Click any product in the list to edit it — editing replaces its
  shelf-life slabs with whatever is currently in the form (so removing
  a row and saving actually removes that slab, not just hides it).

### What was tested

Same approach as Stage 3 — I can't reach a live Supabase project from
this sandbox, so I built a realistic in-memory mock of the Supabase
client (handling insert/update/delete/select with filters) and ran the
actual app code against it in a real DOM environment:

- Login, vendor creation, and product creation all work correctly end
  to end, including the QR auto-generation handoff from the database
  function into the saved product record.
- Validation correctly blocks: missing product name, missing SKU,
  invalid buying/selling price, missing shelf-life slabs, and malformed
  `MM-DD` date values — confirmed nothing reaches the database when
  validation fails.
- Editing an existing product loads its current values and slabs
  correctly into the form, an edit-then-resave correctly updates the
  changed field, **the QR code stays exactly the same after editing**
  (confirmed explicitly), and re-saving the same single slab does not
  create a duplicate (the delete-then-reinsert replacement strategy
  works as intended).
- Vendor deactivation correctly flips `is_active` to false.
- The vendor dropdown on the product form populates correctly from
  saved vendors.
- QR codes generate sequentially and correctly across multiple new
  products in the same session (`KAL01-001`, then `KAL01-002`, ...).
- **One real bug caught and fixed during testing**: newly created
  vendors and products were briefly showing as "Inactive" in the list
  immediately after creation. The cause was that `is_active` was only
  being explicitly set when *editing* a record — for a *new* record it
  was left for the database's column default to fill in, which is
  correct for a real Postgres/Supabase database, but meant my local
  mock (which doesn't apply column defaults) exposed the gap. Fixed by
  explicitly setting `is_active: true` when creating new vendors and
  products, which is more robust and self-documenting either way.

**What still needs verification once you plug in your real project
credentials:** the real network round-trip, and that RLS behaves
correctly through this app for a real SPOC account (already verified
independently via direct SQL in Stage 1's testing, but worth a quick
real check here too).

---

## Update: Categories, persistent QR printing, and an RLS bugfix

Following real usage feedback after Stage 4, several changes were made:

### RLS bugfix (run `003_categories_migration.sql`)
Recording a sale was failing with an RLS error on `sale_allocations`.
Root cause: `create_sale_with_fefo()` was not `SECURITY DEFINER`, so its
internal insert into `sale_allocations` ran under the calling SPOC's own
permissions — and that table had no INSERT policy granted to anyone,
only SELECT. Fixed by making `create_sale_with_fefo()` and
`create_stock_batch()` both `SECURITY DEFINER`; both already validate
center-correctness internally (derived from the product, never trusted
from caller input), so this does not weaken access control. **Confirmed
fixed**: tested as a real (non-superuser) SPOC login — a sale that
previously failed now succeeds and correctly creates the allocation row.

### Shelf life is now per-category, not per-product
Every product must belong to a category, and **shelf-life slabs now
belong to the category**, not to individual products — so all "Snacks"
share one set of seasonal rules, all "Sweets" share their own, etc.,
exactly as requested. `003_categories_migration.sql` introduces a new
`categories` table and automatically migrates your existing free-text
`products.category` values and per-product slabs into this new
structure.

**Important post-migration step:** if two of your existing products
shared a category name but had *different* shelf-life slabs, the
migration preserves all of them under that one category rather than
guessing which is correct — this was tested explicitly and confirmed.
Open the new **Categories** tab in `vsdham_master_app.html` after
running the migration and review each category's slabs, removing any
duplicates/conflicts so each category has one clean, non-overlapping
set of seasonal rules.

### Product Master changes (`vsdham_master_app.html`)
- **New "Categories" tab**: add/edit categories, each with its own
  shelf-life slabs (the slab editor moved here from the product form).
- **Product form**: category is now a dropdown of existing categories,
  with a **"+ Add new category…"** option that switches to the
  Categories tab, pre-fills the name you typed, and prompts you to set
  up its shelf life before returning to finish the product — so a
  category always has shelf-life rules before any product can use it.
- **Persistent "Print QR" button** on every product row in the list —
  this fixes the earlier issue where the QR link only appeared once,
  right after creating a product, and disappeared on refresh.
- **Checkbox selection + "Print QR for selected"**: select any number of
  products and print all of their QR codes together, packed onto as few
  A4 sheets as possible (reuses Stage 2's Rack/Catalog mode). Works for
  a single product too (the per-row button) or all ~50-80 at once.

### QR Label Sheet Generator changes (`qr_label_sheet_generator.html`)
Now accepts a `?items=` URL parameter (a JSON array of `{code, name}`
pairs) in addition to the earlier single-product `?code=&name=`, and a
`?mode=A` or `?mode=B` parameter to land directly on the right tab. This
is what powers the new bulk-print handoff from the Product Master.
Tested: single product, multiple products (packed correctly into the
fewest sheets), and a deliberately malformed `?items=` value (falls back
gracefully to the sample data instead of crashing the page).

### What was tested
Same mock-based approach as before — added category creation with
slabs, the category dropdown correctly listing saved categories, product
save using the dropdown (confirmed no leftover shelf-life UI in the
product form itself), the persistent QR button opening the correct URL,
bulk selection building a correct multi-product URL, selection clearing
correctly, and the full "add new category from product form" round trip
(switches screen, pre-fills name, saves, then shows up in the dropdown
back on the product form).

---

## Opening & Closing Report (`004_reports.sql` + Reports tab in `vsdham_entry_app.html`)

### Schema: `stock_as_of()` and `daily_opening_closing_report()`
`current_stock` only ever showed the *live* stock position — it had no
way to answer "what was our stock at the start of last Tuesday." This
migration adds:

- **`stock_as_of(center_id, timestamp)`**: reconstructs per-product
  stock quantity and cost value exactly as it stood at any point in
  time, by working backward from each batch's original received
  quantity minus whatever had been sold out of it (via
  `sale_allocations`) strictly before that timestamp — rather than
  trusting `quantity_remaining`, which only reflects right now.
- **`daily_opening_closing_report(center_id, date)`**: a convenience
  function combining opening stock (start of that date), closing stock
  (end of that date, or right now if the date is today), and that day's
  cash account opening/closing balance from `account_balances` — all in
  one JSON result, which is what the Reports tab calls directly.

**Tested with a controlled, backdated 2-day scenario** (received 50
units, sold 20 on day one; received 10 more, sold 15 on day two):
opening stock for day two correctly came back as exactly 30 units (50
− 20), closing stock for day two correctly came back as exactly 25
units (30 + 10 − 15) — confirmed the historical reconstruction is exact,
not approximate. Also tested: a date with no account balance entry yet
(correctly distinguishes "not entered" from "entered as zero"), and
today's date (correctly uses the current moment as the closing cutoff
rather than treating an unfinished day as if it had already ended).

### Reports tab in the entry app
- A date picker (defaulting to today) drives the whole screen — change
  the date and the report reloads automatically.
- Two cards show the day's cash account opening and closing balance,
  clearly marked "Not entered" if no balance entry exists for that date
  yet (rather than showing a misleading zero).
- A table shows every product with stock activity that day, opening qty
  next to closing qty next to the net change (colored green for a net
  increase, red for a net decrease).
- A **Download as CSV** button exports the same data as a clean,
  Excel-readable file (verified the actual generated CSV content directly
  in testing — correct headers, correctly quoted fields, accurate figures
  matching what's shown on screen).

### What was tested
Switching to the Reports tab correctly auto-loads today's date and
calls the database function with the right parameters; the balance
cards and stock table render the returned figures accurately; changing
the date picker correctly reloads with the new date; a report with no
account entry and no stock activity at all renders gracefully instead of
breaking; and the CSV download function produces well-formed, accurate
output without crashing even when triggered with no real browser
download mechanism available (as in this sandboxed test environment).

**What still needs verification with your real project:** the real
network round-trip and that a SPOC's RLS-scoped view of their own
center's data flows through correctly into this report (the underlying
tables' RLS was independently verified in Stage 1; this report's
functions are read-only and rely on that same RLS, rather than
bypassing it).




