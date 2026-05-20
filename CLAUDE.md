## Self-Update Protocol

At the end of every session Claude Code must:
1. Check off completed items in Build Status
2. Add new business rules or conventions established
3. Add new routes, tables, or integrations created
4. Add a session log entry with date and summary
5. Update Known Gaps if anything was resolved or added
6. Push to both repos:
   a. Commit and push CLAUDE.md in the consignment-store repo (private)
   b. Copy CLAUDE.md to `C:\claude-context\CLAUDE.md` and push to the claude-context repo (public — this is what claude.ai fetches)
   Note: the file is named CLAUDE.md (not consignment-store-CLAUDE.md) in the claude-context repo.
7. After pushing to claude-context, purge the CDN:
   `curl "https://purge.jsdelivr.net/gh/greghussey62/claude-context@main/CLAUDE.md"`

   Escalating retry schedule:
   - If it returns `{"status":"finished"}` → done.
   - If it returns `"throttled"` → wait **2 minutes** and retry once.
   - If still throttled → wait **5 more minutes** and retry once more.
   - If still throttled after the second retry → add to Pending:
     `CDN purge pending — run manually before next session`
     and stop. The jsDelivr throttle resets overnight, so the updated
     CLAUDE.md will be visible to claude.ai tomorrow.

   **Always log the result** in today's session log entry so we can tell
   later whether the purge succeeded. Use one of these exact tokens at
   the end of the entry:
   - `CDN purge: finished` (success)
   - `CDN purge: throttled — retried after 2m, finished`
   - `CDN purge: throttled — retried after 2m + 5m, finished`
   - `CDN purge: throttled — deferred to Pending`

Do this automatically without being asked.

## Pending From Last Session
- Move Anthropic API key to server-side before any public deployment
- Tech debt open: #19 (QZ Tray cert), #20–#21 (accessibility), #22 (archival)
- Feature backlog open: #23, #32–#36 (#29 done; email issues #23 skipped; #24–#31, #38 done)

## How to Share With claude.ai

The jsDelivr CDN fetch is unreliable for same-day updates (the CDN caches aggressively and purge requests are throttled). The most reliable way to give claude.ai today's context is to paste the file contents directly into the chat:

- In VS Code: open `CLAUDE.md`, `Ctrl+A`, `Ctrl+C`, then paste into the claude.ai chat.

The fetch URL remains as a backup for next-day reads, once the CDN has expired its old copy:
`https://cdn.jsdelivr.net/gh/greghussey62/claude-context@main/CLAUDE.md`

(The consignment-store repo is private, which is why claude.ai fetches from the public claude-context mirror.)

---

# Consignment Store — Claude Code Master Reference

---

## 1. What This App Is & Who Uses It

A consignment store management system for a small fashion/clothing retail shop (**Samira's**). The store takes in items from individual consignors, sells them, then pays out the consignor's share later.

**Primary user:** A non-technical store owner who does every task herself — intake, selling, printing, payouts. She runs the app on a Windows machine connected to thermal printers. She is not a developer.

**Design imperative:** Every UI decision must favor simplicity and plain language over technical sophistication. Short workflows, minimal clicks, no jargon.

**Secondary user:** Consignors, who have no login — they view their account via a public portal lookup (no auth required).

---

## 2. Complete Tech Stack

| Layer | Technology |
|---|---|
| UI Framework | React 19 + Vite 8 |
| Routing | React Router v7 (client-side SPA) |
| Backend / Database | Supabase (Postgres + Auth + Realtime + Storage) |
| AI Parsing | Anthropic Claude API (Haiku model) |
| Printing | QZ Tray (local desktop app, WebSocket) |
| QR Codes | qrcode.react |
| Build / Dev | Vite 5173, HMR, ESM |
| Testing | Vitest + React Testing Library + jsdom |
| Linting | ESLint |
| Styles | Plain CSS with CSS variables (no Tailwind, no CSS-in-JS) |
| Deployment | Vercel — production at `consignment-store-psi.vercel.app` |

**Key package versions (from package.json):**
- `react` 19.2.5
- `react-router-dom` 7.14.2
- `@supabase/supabase-js` 2.104.1
- `@anthropic-ai/sdk` 0.91.0
- `qz-tray` 2.2.6
- `vite` 8.0.10
- `vitest` 4.1.5

---

## 3. Hardware Context

This app runs on a physical retail machine. Hardware awareness matters:

| Device | Purpose | Protocol |
|---|---|---|
| Barcode scanner | POS item lookup, markdown scanning, reconciliation | USB HID (keyboard wedge — emits text + Enter) |
| Thermal tag printer | 3"×1.5" barcode labels | ZPL at 203 DPI via QZ Tray |
| Thermal receipt printer | ESC/POS receipts, 42-char wide | ESC/POS via QZ Tray |
| Phone camera | QR-based photo uploads for items | Mobile browser (no app) |
| Windows PC | Main workstation | — |

**QZ Tray** is a local desktop app that bridges the browser to raw printer access. It must be running on the machine. Printer names are configured in Admin settings and stored in `localStorage`. Dry-run mode is available for preview without hardware.

### Store Web Presence
| Property | Value |
|---|---|
| Store website | `www.samirasupsacleboutique.com` |
| App URL (current) | `consignment-store-psi.vercel.app` |
| App URL (future) | `app.samirasupsacleboutique.com` → point to Vercel deployment |
| Email receipts (current sender) | Resend default (`onboarding@resend.dev` or similar) |
| Email receipts (future sender) | `receipts@samirasupsacleboutique.com` — requires domain verification in Resend |

**Future domain work:** (1) Add `samirasupsacleboutique.com` as a verified domain in Resend to enable branded receipt sender. (2) Add a CNAME/A record in DNS to point `app.samirasupsacleboutique.com` to the Vercel deployment for a branded app URL.

---

## 4. Business Rules

### Contract Lifecycle
1. Create contract → assigns sequential `contract_num` (starts ~3000+)
2. Add items → each gets `item_id` format: `{contract_num}-{seq:02d}` (e.g., `3042-01`)
3. Items are on the floor until sold or returned
4. At payout time → calculate consignor share, issue check, return unsold items

### Split / Payout Calculation
- Default split: **50/50** (configurable per contract, `split_pct` = consignor's %)
- Consignor gets: `split_pct / 100 × sale_price`
- Store gets: `(100 - split_pct) / 100 × sale_price`
- Per-item override: `base_amount` field = guaranteed consignor payout (ignores split%)
- No-tax cash toggle can suppress tax on certain transactions at POS

### Item Status Flow
```
active → sold       (purchased at POS)
active → layaway    (customer hold)
active → returned   (unsold, consignor taking back)
active → pulled     (store claimed after pickup deadline)
sold   → active     (return/refund at POS)
```

### Markdown Workflow
- Eligible: items on floor **60+ days**
- Auto-markdown: 20% applied; stores `original_price`, updates `price`, sets `markdown_applied = true`
- Run Markdown button in Reports does a batch run
- Items need reprinted tags after markdown (`needs_new_tag = true`)
- Floor Sheet prints list of marked-down items for staff to re-price physically

### Payout Batches
- Each contract has one or more **payout batches** (different pickup date windows)
- First batch auto-created when contract is first viewed in detail
- Items can be assigned to different batches via dropdown
- Payout includes all batches for the contract
- After payout: store can confirm pickup, or claim unclaimed items as store property

### Layaway Workflow
1. Created from POS cart → items get status `layaway`, `layaway_ref` assigned
2. Customer makes partial payments (tracked in `payments` jsonb array)
3. Fully paid → Complete → creates sales records for all items
4. Not picked up → Forfeit → store keeps deposit, items return to `active`

### POS Command Cards
These barcodes/scan codes drive the POS:
- `CMD:CASH` → open cash payment screen
- `CMD:CARD` → card pending
- `CMD:APPROVED` → finalize transaction
- `CMD:DECLINED` → cancel
- `CMD:VOID` → remove last item
- `CMD:CLEAR` → empty cart
- `CMD:HOLD` → save cart, start new
- `CMD:RESUME` → restore held cart
- `CMD:10OFF / 20OFF / 50OFF` → discount last item

### Tax
- Default tax rate: **6%** (configurable per Admin → Receipt Printer settings)
- No-tax cash option available on POS for zero-tax transactions

### Receipt Numbers
- Format: `R-{seq}` starting at `R-1001`
- Backed by Postgres sequence `receipt_seq`
- Generated via RPC `next_receipt_number()`

### POS Price Negotiation
- Staff can override the sale price of any cart item before completing a transaction
- Each cart row has a **✎ Override** button; clicking opens a modal to enter the new price
- Modal shows the tag price, calculates the discount %, and accepts Enter to confirm
- The overridden price is recorded as `sales.sale_price` and `items.sold_price` — it is the price the item sold for
- The item's original `price` field is unchanged; it appears as a strikethrough in the cart after override
- Visual state after override: amber "Negotiated" badge, warm yellow row background, strikethrough original price
- The `listed_price` column in `sales` always stores the original tag price for reporting purposes

### Split Tender
- POS supports cash + card in one transaction
- Cashier enters cash portion; card amount = total − cash
- Stored as `payment_method='split'` + `payment_method_2='card'` + `amount_tendered_2` on transactions
- ESC/POS receipt shows both lines; browser receipt fallback also shows split

### Store Credit
- Accounts identified by phone number; looked up at POS during checkout
- Balance deducted from `store_credit` after `complete_pos_sale` commits
- Debit logged in `store_credit_transactions`; reason field tracks 'sale', 'return', 'adjustment'
- If partial credit: toast shown with remaining amount; cashier collects rest by cash or card
- Returns can issue store credit via the Admin → Store Credit page (adjust balance + log)

### Shift Reconciliation
- "Close Shift" on Dashboard; computes expected cash = opening float + cash sales − cash returns
- Over/short saved to `shifts` table for history
- Cash sales filtered to `payment_method = 'cash'` on `sale_date = today`

### Markdown Rules
- Multi-tier rules stored in `localStorage` key `markdown.rules` as JSON: `[{days, pct}, ...]`
- Configured via Admin → Markdown Rules tab; default: 60 days → 20%
- Batch applies highest applicable tier per item; sets `needs_new_tag = true` after markdown

### Appointment Scheduling
- Slots configured in `appointment_settings` DB table (not localStorage — must be DB-accessible from any device)
- Anon INSERT allowed on `intake_appointments`; `count_slot_bookings()` RPC checks capacity without exposing PII
- Booking URL: `{origin}/book` — share with consignors; settings and appointment list in Admin → Appointments

### Photo Management
- Max **4 photos** per item
- Storage path: `items/{contractNum}/{itemId}/photo_{slot}.jpg`
- Client-side compression before upload: max 1200px wide, JPEG 82% quality
- QR-based phone upload: store generates token → QR code → scan with phone → upload from camera
- Token expires in **15 minutes**
- Realtime subscription in EditItemModal picks up new photos immediately
- **Purge old photos:** Admin → Data Management → Photo Storage card. Deletes Storage files + sets `items.photos = []` for items with status sold / returned / pulled. Estimate: 0.3 MB/photo. Record itself is preserved.

### Markdown Scanner Mode
- Inventory toolbar → 📉 Markdown button opens full-screen scanner overlay (replaces page, like Payout Pull)
- Tier selector (20% Off / 50% Off) **inside** the overlay — scan input is disabled until a tier is chosen
- Calculations always from `items.original_price` (treated as the never-marked-down baseline). If `original_price` is null on first scan, save the current `price` as `original_price` then mark down. New price is rounded to the nearest whole dollar (no cents).
- Skip if `items.price` already equals the computed new price → mid-frequency tone + "Skip" speech + yellow flash, no DB write, no session entry.
- On success: `price = newPrice, markdown_applied = true, needs_new_tag = false` (staff hand-updates the physical tag with a red sharpie). High beep + speaks the new price as words ("One sixty", "Seventy five", "One twenty five") via SpeechSynthesis at rate 0.9.
- Every successful scan logs to `audit_log` with action `'markdown'`, `record_id = item_id`, and `details = { from_price, to_price, original_price, tier, markdown_applied_by }`.
- Audio respects the same `intake_audio_enabled` localStorage key as AI intake (no beeps, no speech when muted).
- Session list persists in localStorage key `markdown.scanner.session` (date-scoped; cleared when date changes). "New Session" wipes the list but keeps the tier. "Print Report" produces a floor sheet listing every marked-down item.

### Database Health (Admin → Data Management)
- Three live cards: **Photo Storage** (counts + estimated MB + purge button), **Database Summary** (rows per table, items grouped by status), **Items Missing Photos** ("View Items" → `/inventory?filter=nophotos`).
- Purge old photos parses public URLs (`/item-photos/...`), batches deletes (80/call) to `supabase.storage.from('item-photos').remove(...)`, then sets `items.photos = []` for affected items.
- Manager+ only (DataTab is already gated behind `isManager` in Admin.jsx).

### Bulk Tag Reprint (Admin → Data Management)
- Shows all items where `needs_new_tag = true`. Filters: markdown-applied vs manually flagged, plus consignor and category dropdowns.
- Uses existing `printTags()` (ZPL via QZ Tray, max 40 per spool batch). Dry-run is honored.
- After print, prompts "Mark as printed?" — if yes, sets `tag_printed = true` and `needs_new_tag = false` for the printed items.

---

## 5. Build Status — What Exists vs What Doesn't

### Fully Built & Working
| Area | Status |
|---|---|
| GitHub Issues backlog (39 items — tech debt + feature queue) | ✅ |
| Authentication (Supabase email/password) | ✅ |
| Contracts — create, view, edit, delete | ✅ |
| Items — add (AI + manual), edit, status changes | ✅ |
| AI Intake (voice + text → Claude Haiku parsing) | ✅ |
| Inventory list with filtering/sorting/bulk actions | ✅ |
| POS — scan, cart, payment, receipts, price negotiation | ✅ |
| Layaway — create, payments, complete, forfeit | ✅ |
| Payouts — calculate, process, pickup tracking | ✅ |
| Payout batches (multiple windows per contract) | ✅ |
| Thermal tag printing (ZPL via QZ Tray) | ✅ |
| Receipt printing (ESC/POS via QZ Tray) | ✅ |
| Photo uploads (direct + QR-based mobile) | ✅ |
| Markdown workflow (auto + manual, floor sheet) | ✅ |
| Reports & analytics | ✅ |
| Sales history + transaction reprint | ✅ |
| Consignor statements (printable) | ✅ |
| CSV export (contracts, items, sales, payouts) | ✅ |
| Admin — item types, designers, printer config | ✅ |
| Help chat (Claude-powered in-app assistant) | ✅ |
| Dashboard (stats, Z-report, alerts) | ✅ |
| Camera Mode (active item tracking via Realtime) | ✅ |
| Inventory + Contracts pagination (50/page, 25/page) | ✅ |
| Customer display `/display` (Realtime, idle/cart/payment) | ✅ |
| Payout sale price adjustment (inline ✎ editing, audit_log) | ✅ |
| Cash drawer kick on cash sales (ESC/POS) | ✅ |
| Store credit (accounts, POS payment, balance deduction) | ✅ |
| Role-based access (staff vs. manager, nav filtering) | ✅ |
| 1099 report (Reports page, year filter, CSV export) | ✅ |
| Markdown rules config (multi-tier, Admin tab) | ✅ |
| Shift reconciliation (Close Shift modal, over/short, shifts table) | ✅ |
| Split tender POS (cash + card, stored in transactions) | ✅ |
| Appointment scheduling (public /book form, Admin management) | ✅ |
| Automated test suite (171 tests — payout, tax, markdown, layaway, status, IDs) | ✅ |
| In-app Help Guide (/help route, sidebar nav, search, 14 sections incl. Pull Items) | ✅ |
| Printable manual test plan (public/test-plan.html, 97 tests, 12 sections) | ✅ |
| Printable staff guide (public/staff-guide.html, quick ref, workflows, troubleshooting) | ✅ |
| Three-tier user management (admin/manager/staff, full CRUD, lockout, suspend, welcome email) | ✅ |
| User profile page (/profile — display name, change password, accessible from topbar menu) | ✅ |
| Forgot password / reset password flow (/reset-password, Supabase email recovery) | ✅ |
| Forced first-login password change (/change-password, force_password_change flag) | ✅ |
| Account lockout (5 failed attempts → 30-min lock, api/auth-events.js, Admin can unlock) | ✅ |
| Appointment scheduling from Contracts page + Dashboard 7-day widget | ✅ |
| Appointment → Contract flow (pre-filled New Contract from appointment row) | ✅ |
| Phone number formatting (src/lib/phone.js — all phone inputs app-wide) | ✅ |
| Email receipts to customers (#29 — Resend via Vercel serverless function) | ✅ |
| Markdown Scanner Mode (Inventory overlay, tier selector, audio+speech, audit_log, same-day session) | ✅ |
| Bulk Tag Reprint (Admin → Data Management, filters, QZ Tray batch print, mark-as-printed) | ✅ |
| Database Health cards (Admin → Data Management — photo storage + purge, record counts, missing photos) | ✅ |

### Known Gaps / Placeholders
| Area | Status |
|---|---|
| Data Management tab in Admin | Built — Database Health + Bulk Tag Reprint + Danger Zone (contract delete) |
| Consignor portal (public view) | `getPortalData()` exists in db.js but no public route confirmed |
| Declined items (DB table exists, no UI) | Schema only |
| Deployment / CI pipeline | Auto-deploys from GitHub via Vercel; no manual deploy needed |
| Consignor sale email notifications | Skipped (#23) — needs Resend API key wired to sale events |
| Email receipts to customers | Done and active — RESEND_API_KEY configured in Vercel; receipts sending live |
| Appointment confirmation email | Resend infrastructure in place — needs wiring to appointment insert |
| User management in-app | Done and active — full three-tier system; SUPABASE_SERVICE_ROLE_KEY active in Vercel |
| Shopify product sync | Not built — push items to Shopify, webhook for online sales (#33) |
| AI Shopify descriptions | Not built — Claude generates product descriptions for Shopify (#34) |
| Offline POS mode | Deferred — requires IndexedDB + background sync queue (issue #16 closed) |
| Accessibility improvements | Open GitHub issues #20, #21 (aria-live, keyboard nav) |
| Data archival strategy | Open GitHub issue #22 |
| QZ Tray cert verification | Open GitHub issue #19 (currently bypassed) |

---

## 6. Active Integrations

### Supabase
- **Auth:** Email + password, sessions via browser storage
- **Database:** Postgres, all queries via `@supabase/supabase-js` in `src/lib/db.js`
- **RLS:** Every table has RLS enabled. Authenticated users get full access via `auth_all` policy. `photo_upload_sessions` has an anonymous read policy for unexpired tokens.
- **Storage:** Bucket `item-photos` (public bucket, items path)
- **Realtime:** Used in `EditItemModal` and `ItemDetail` to sync photo changes live
- **RPC functions:**
  - `next_receipt_number()` — returns next `R-{n}` receipt ID
  - `upload_item_photos(token, photos)` — security definer, allows anon uploads via token
  - `complete_pos_sale(...)` — security definer, atomic POS sale
  - `count_slot_bookings(date, slot)` — security definer, anon-accessible capacity check

### Anthropic Claude API
- **Model:** `claude-haiku-4-5-20251001` (fast, cheap, sufficient for JSON extraction)
- **Usage in `src/lib/aiParse.js`:**
  - Parses spoken/typed item descriptions into structured JSON: `{category, brand, color, size, price, notes}`
  - System prompt built dynamically from current item_types + designers (fetched from DB, cached 4 min in memory)
  - **Prompt caching** (ephemeral) on system prompt — cache TTL 5 min, in-memory refresh at 4 min
  - Returns `max_tokens: 300`
- **Usage in `src/components/HelpChat.jsx`:**
  - Full chat assistant with streaming
  - System prompt contains complete app documentation
  - Direct API calls from the browser using `VITE_ANTHROPIC_API_KEY`

**Warning:** The Claude API key is exposed client-side via `VITE_` prefix. Acceptable for a private single-user intranet app, but not suitable for public deployment.

### QZ Tray
- Local desktop WebSocket app (`localhost:8181`)
- Provides raw printer access from the browser
- **Tag printer:** ZPL, 3"×1.5" labels, 203 DPI — printer name stored in `localStorage`
- **Receipt printer:** ESC/POS, 42-char wide — settings stored in `localStorage`
- Config persisted in `localStorage` (not DB), set via Admin page
- Dry-run mode available: returns ZPL/ESC string without printing
- Max batch: 40 tags per spool to avoid printer buffer overflow

---

## 7. Database Schema Summary

All tables live in the `public` schema. `supabase/schema.sql` is the source of truth — re-run in the Supabase SQL editor after any schema change.

### Tables

#### `contracts`
Consignor agreements. One per consignor visit (a consignor can have multiple contracts over time).
```
id uuid PK
contract_num serial UNIQUE     -- human-facing ID (~3000+)
name text NOT NULL
phone text
email text
address text
payout_date date               -- agreed pickup date
split_pct int DEFAULT 50       -- consignor's % of sale price
status text DEFAULT 'active'   -- active | paid | cancelled
created_date date
notes text
```

#### `items`
Consigned inventory items.
```
item_id text PK                -- format: {contract_num}-{seq:02d}
contract_id uuid FK contracts
contract_num int               -- denormalized
consignor_name text            -- denormalized
seq int                        -- sequence within contract
description text               -- item type/category
brand text
color text
size text
price numeric(10,2)            -- current listed price
original_price numeric(10,2)   -- pre-markdown price
sold_price numeric(10,2)
sold_date date
added_date date
status text DEFAULT 'active'   -- active|sold|returned|pulled|layaway
markdown_applied bool
tag_printed bool
needs_photos bool
needs_new_tag bool
notes text
payout_batch_id uuid FK payout_batches
photos text[]                  -- array of Supabase Storage public URLs
base_amount numeric(10,2)      -- per-item guaranteed payout (overrides split%)
```

#### `sales`
Transaction line items. One row per item sold (or returned).
```
id uuid PK
item_id text
contract_id uuid FK contracts
contract_num int               -- denormalized
consignor_name text            -- denormalized
description text
brand text
sale_price numeric(10,2)       -- negative for returns
listed_price numeric(10,2)
sale_date date
notes text
payment_method text            -- cash | card | check
is_return bool DEFAULT false
transaction_id uuid FK transactions
```

#### `transactions`
Receipt groupings (one per POS checkout).
```
id uuid PK
receipt_number text UNIQUE     -- format: R-{seq}
sale_date date
sale_time timestamptz
payment_method text            -- cash | card | split | store_credit
subtotal numeric(10,2)
tax_rate numeric(6,4)          -- default 0.06
tax_amount numeric(10,2)
total_amount numeric(10,2)
amount_tendered numeric(10,2)
change_due numeric(10,2)
customer_name text
payment_method_2 text          -- second method for split tender (e.g. 'card')
amount_tendered_2 numeric(10,2) -- amount on second payment method
```

#### `payouts`
Consignor payout records (check runs).
```
id uuid PK
contract_id uuid FK contracts
amount numeric(10,2)
paid_date date
check_number text
pickup_deadline date
mailed_date date
items_retrieved bool DEFAULT false
items_retrieved_date date
void_check_number text
reissued_check_number text
reissued_date date
```

#### `payout_batches`
Multiple pickup windows per contract.
```
id uuid PK
contract_id uuid FK contracts
batch_num int                  -- sequence within contract
payout_date date
```

#### `layaways`
Customer layaway hold agreements.
```
id uuid PK
layaway_ref text               -- human-readable ID
customer_name text
customer_phone text
customer_email text
pickup_by date
total numeric(10,2)
paid numeric(10,2) DEFAULT 0
status text DEFAULT 'active'   -- active | completed | forfeited
notes text
items jsonb                    -- [{itemId, price}]
payments jsonb                 -- [{amount, date, note}]
created_date date
closed_date date
```

#### `item_types`
Controlled vocabulary for item categories (~150+ fashion types).
```
id uuid PK
name text UNIQUE               -- e.g., "Dress", "Coat"
category_group text            -- e.g., "Dresses & Skirts"
```

#### `designers`
Brand/designer canonical names with phonetic aliases for AI matching.
```
id uuid PK
name text UNIQUE               -- e.g., "Louis Vuitton"
aliases text                   -- comma-separated phonetic variants
```

#### `declined_items`
Items rejected during intake (schema exists, no UI yet).
```
id uuid PK
contract_id uuid FK contracts
contract_num int
description text
reason text
declined_date date
```

#### `audit_log`
Immutable log of significant data changes (price changes, status changes, payouts, contract deletions).
```
id uuid PK
user_id uuid FK auth.users ON DELETE SET NULL
action text                -- 'price_change' | 'status_change' | 'payout' | 'delete_contract'
table_name text
record_id text             -- item_id, contract uuid, etc.
details jsonb              -- e.g. {from: 45.00, to: 35.00}
created_at timestamptz
```

#### `photo_upload_sessions`
Short-lived tokens for QR-based mobile photo uploads.
```
token text PK
item_id text
item_label text
created_at timestamptz
expires_at timestamptz         -- 15-minute window
```
RLS: Anonymous SELECT allowed if `expires_at > now()`.

#### `user_active_item`
Tracks the active item per user for Camera Mode.
```
user_id uuid PK FK auth.users
item_id text
item_label text
updated_at timestamptz
```
RLS: Users can only access their own row.

#### `pos_sessions`
Single-row Realtime sync for customer display tablet.
```
session_key text PK            -- always 'main'
cart jsonb                     -- array of cart items for display
subtotal / tax_amount / total  numeric(10,2)
status text                    -- idle | active | card_pending | cash_pending
updated_at timestamptz
```
RLS: anon SELECT, auth ALL. POS publishes on every cart/payment state change.

#### `user_roles`
Role per authenticated user (staff vs. manager).
```
user_id uuid PK FK auth.users
role text DEFAULT 'manager'    -- 'staff' | 'manager'
```
Existing owner account defaults to 'manager' (no row needed).

#### `shifts`
End-of-day cash reconciliation records.
```
id uuid PK
opened_at / closed_at timestamptz
opening_float numeric(10,2)
cash_sales / cash_returns numeric(10,2)
expected_cash / counted_cash numeric(10,2)
over_short numeric(10,2)       -- positive = over, negative = short
notes text
closed_by uuid FK auth.users
```

#### `store_credit`
Customer store credit balances.
```
id uuid PK
customer_name text NOT NULL
customer_phone text            -- used for POS lookup
balance numeric(10,2)
created_at / updated_at timestamptz
```

#### `store_credit_transactions`
Individual credit/debit ledger entries.
```
id uuid PK
credit_id uuid FK store_credit
amount numeric(10,2)           -- positive = credit, negative = debit
reason text                    -- 'return' | 'sale' | 'adjustment' | 'promotion'
reference_id text              -- transaction id or other reference
created_at timestamptz
```

#### `appointment_settings`
Single-row config for the public booking form (anon-readable).
```
id int PK DEFAULT 1            -- always 1
available_days text[]          -- e.g. {Mon,Tue,Wed,Thu,Fri}
time_slots text[]              -- e.g. {10:00 AM,11:00 AM,...}
max_per_slot int DEFAULT 2
max_items_per_slot int DEFAULT 20
advance_days int DEFAULT 14    -- how far ahead consignors can book
booking_notes text             -- instructions shown on /book form
```
RLS: anon SELECT, auth ALL.

#### `intake_appointments`
Consignor drop-off appointments booked via /book.
```
id uuid PK
consignor_name / phone / email text
slot_date date
slot_time text
item_count int
notes text
status text DEFAULT 'pending'  -- pending | confirmed | completed | cancelled
created_at timestamptz
```
RLS: anon INSERT (public booking), auth ALL.

### Sequences
- `receipt_seq` — starts at 1001, backs receipt numbers

### RPC Functions
- `next_receipt_number()` — increments sequence, returns `'R-' || nextval`
- `upload_item_photos(token text, photos text[])` — SECURITY DEFINER; validates token expiry, updates `items.photos`
- `complete_pos_sale(...)` — SECURITY DEFINER; atomically creates transaction + all sale records + marks items sold; returns transaction JSONB
- `count_slot_bookings(p_date date, p_slot text)` — SECURITY DEFINER; returns booking count for a slot; anon-accessible so public booking form can check capacity without reading PII

### Indexes
Added on `items(status)`, `items(contract_id)`, `items(contract_num)`, `sales(contract_id)`, `sales(sale_date)`, `sales(transaction_id)`.

---

## 8. Key Conventions

### File Organization
```
src/
  components/   Reusable UI (Layout, Modal, Toast, Badge, EditItemModal, AIIntake, HelpChat)
  lib/          Data layer (supabase.js, db.js, aiParse.js, photos.js, tagPrinter.js, receiptPrinter.js)
  pages/        Route-level components (one file per page)
  styles/       global.css only — no per-component CSS files
```

### Routing (App.jsx)
- All routes under a single `<Layout>` wrapper
- Public routes (no auth required): `/upload/:token`, `/display`, `/book`, `/reset-password`
- Authenticated but NOT behind force-password-change check: `/change-password`, `/profile`
- All other routes go through `CheckPasswordChange` wrapper — redirects to `/change-password` if `force_password_change = true`
- No lazy loading currently

### Role-Based Access
- `RoleProvider` (src/components/RoleContext.jsx) wraps all authenticated routes
- `useRole()` returns an object: `{ role, isAdmin, isManager, displayName, userId, userEmail, setDisplayName, loaded, forcePasswordChange }`
- Three roles: `'admin'` | `'manager'` | `'staff'`; **no row in user_roles = owner account = admin** (do not break this)
- `isAdmin`: role === 'admin' — `isManager`: role === 'admin' || 'manager' — all authenticated users have staff permissions
- Layout.jsx nav: staff see Dashboard, POS, Contracts, Inventory, Layaway; manager+ add Payouts, Reports, Sale History, Store Credit, Admin; admin-only: Users + Permissions tabs within Admin
- Admin → Users tab is the only place to manage users; Admin tab itself is visible to manager+
- `setDisplayName(name)` exposed from context — call after saving a name so the nav bar updates immediately without reload

### Appointment Booking Pattern
- Public-facing config that consignors need is stored in DB (`appointment_settings`), NOT localStorage — localStorage is device-local and unreachable from other browsers
- Anon INSERT policy allows public form submissions without auth
- Security-definer RPCs expose only aggregated data (counts) to anon, never PII
- **Staff-side scheduling** lives on the Contracts page (📅 Schedule Appointment button) — not in Admin
- Admin → Appointments shows the list, settings, and "📋 Contract →" button to convert an appointment to a contract
- Dashboard shows the next 7 days of non-cancelled appointments; today's appointments show as a blue banner

### Phone Number Formatting
- All phone inputs across the app use `formatPhone()` from `src/lib/phone.js`
- Format: `(xxx) xxx-xxxx` — applied live as the user types, caps at 10 digits
- `isValidPhone()` validates exactly 10 digits before save; `digitsOnly()` strips formatting for DB queries
- Never store raw user input for phone — always pass through `formatPhone()` before setting state

### User Management
- Admin → Users tab (manager-only) — lists all Supabase Auth users with role + last sign-in
- Invite, role change, and delete go through `api/admin-users.js` (Vercel serverless, service role key)
- Requires `SUPABASE_SERVICE_ROLE_KEY` in Vercel env vars — never expose client-side
- No row in `user_roles` = owner account = manager by default; staff must have explicit row

### Vercel API Routes
- `api/send-receipt.js` — POST, sends email receipt via Resend; uses `RESEND_API_KEY`
- `api/admin-users.js` — GET/POST, user management via Supabase Admin API; uses `SUPABASE_SERVICE_ROLE_KEY`
- Both validate the caller's JWT before acting; server-side keys never reach the browser bundle

### Data Access Pattern
- **All Supabase queries live in `src/lib/db.js`** — pages import named functions, never call `supabase` directly (exception: Realtime subscriptions and a few select queries inline in pages)
- Functions follow: `getX()`, `createX()`, `updateX()`, `deleteX()` naming
- Most return `{ data, error }` from Supabase or throw on error

### State Management
- No global state library (no Redux, no Zustand)
- Component-local `useState` / `useEffect` for all state
- Data is re-fetched on page load or after mutations
- `useCallback` used in a few places for performance

### Styling Conventions
- CSS variables defined in `global.css` under `:root`
- BEM-ish class names: `.stat-card`, `.card-header`, `.badge-active`
- Status badges: `.badge .badge-{status}` — classes auto-map to color
- No component-scoped CSS; all styles global

### Printing Conventions
- Tag printer: ZPL, configured via Admin, stored in `localStorage`
- Receipt printer: ESC/POS, configured via Admin, stored in `localStorage`
- Dry-run mode for both — returns formatted string, doesn't print
- QZ Tray connection is checked before printing; error shown if not running
- Batch tag print: max 40 per spool call

### AI Intake Conventions
- System prompt is rebuilt when item_types or designers change in DB
- In-memory cache refreshes at 4 minutes to stay within Anthropic's 5-minute prompt cache TTL
- Call `invalidateListCache()` from Admin after adding/removing item types or designers
- Voice input uses Web Speech API (Chrome/Edge only)

### IDs & Numbers
- `contract_num`: auto-incrementing integer, human-facing (~3000+)
- `item_id`: `{contract_num}-{seq:02d}` — e.g., `3042-07`
- `layaway_ref`: human-readable string (format not fixed in schema)
- `receipt_number`: `R-{seq}` — sequence starts at 1001

### Environment Variables
```
VITE_SUPABASE_URL           Supabase project URL
VITE_SUPABASE_ANON_KEY      Supabase anon/publishable key
VITE_ANTHROPIC_API_KEY      Claude API key (used client-side — private intranet only)
RESEND_API_KEY              Resend transactional email key (server-side only) — ACTIVE in Vercel
RECEIPT_FROM_EMAIL          Sender address for email receipts — ACTIVE in Vercel
SUPABASE_SERVICE_ROLE_KEY   Supabase service role key (server-side only) — ACTIVE in Vercel
```
VITE_ prefix vars are embedded in the browser bundle at build time. All others (no VITE_ prefix) are server-side only and safe for secrets.

**Vercel env var status (as of 2026-05-15):** `RESEND_API_KEY`, `RECEIPT_FROM_EMAIL`, and `SUPABASE_SERVICE_ROLE_KEY` are all configured and active in Vercel production. Email receipts and Admin → Users tab are fully operational.

---

## 9. What Not To Do

- **Don't add Tailwind, shadcn, or any CSS framework.** Styles are plain CSS with variables in `global.css`. Adding a framework would require a full rewrite of all styles.
- **Don't add a global state library.** Component-local state works fine for this scale. Redux/Zustand would be premature.
- **Don't move Supabase queries out of `src/lib/db.js` into components.** The pattern of centralized data functions is intentional — easier to maintain and test.
- **Don't expose the Anthropic API key publicly.** It's acceptable here because the app runs on a private intranet, but never deploy this to a public URL without moving AI calls to a server-side proxy.
- **Don't deploy to a public URL without moving the Anthropic API key to a server-side API route.** Currently exposed client-side via `VITE_` prefix — visible in the browser bundle to anyone who loads the page.
- **Don't change `item_id` format.** The `{contract_num}-{seq:02d}` format is barcoded on physical tags. Changing it would invalidate printed inventory.
- **Don't modify `supabase/schema.sql` and forget to tell the user to re-run it.** The file is the source of truth but changes don't auto-apply. Always remind: "Re-run `supabase/schema.sql` in the Supabase SQL editor."
- **Don't add complexity for its own sake.** This is a one-user desktop app for a small store. Optimize for maintainability and clarity, not scalability.
- **Don't add server routes or a backend.** Everything runs client-side against Supabase. If something requires a server, discuss with the user first.
- **Don't change the ZPL tag format** without checking the physical tag dimensions (3"×1.5", 203 DPI = 609×304 dots). The barcode height, font sizes, and field positions are calibrated to the label stock.
- **Don't use `git add .` or `git add -A`** — the `.env.local` file contains real API keys and must not be committed.
- **Don't break the reference implementation contract.** If in doubt about a business rule, check `consignment-store.html` — it is the authoritative source.
- **Don't upsert `user_roles` without explicitly including the `role` column.** The table default is now `'staff'`, so an upsert that omits `role` will silently demote the owner account to Staff. Always specify the role, or use `update` + conditional `insert` (see Profile.jsx handleSaveName pattern).
- **Don't use `update` alone when writing to `user_roles` for the owner account.** The owner has no row, so `update` silently touches nothing. Pattern: try `update`, check if 0 rows affected, then `insert` with `role='admin'`.

---

## 10. Schema Change Reminder

After **any** change to `supabase/schema.sql`:

> **Action required:** Re-run `supabase/schema.sql` in the Supabase SQL editor to apply these changes to your database.

The file uses `IF NOT EXISTS` and `DROP IF EXISTS` guards — it is safe to re-run in full.

---

## 11. Build & Run Commands

```bash
npm run dev        # Start dev server at http://localhost:5173
npm run build      # Production build → dist/
npm run preview    # Preview production build
npm run lint       # ESLint
npm run test       # Run Vitest (unit tests)
npm run test:watch # Watch mode
```

---

## 12. Reference Files

| File | Purpose |
|---|---|
| `consignment-store.html` | Original single-file implementation — source of truth for business logic |
| `supabase/schema.sql` | Database schema — source of truth for all tables |
| `src/lib/db.js` | All Supabase queries |
| `src/lib/aiParse.js` | Claude item parsing logic |
| `src/lib/tagPrinter.js` | ZPL thermal tag printing |
| `src/lib/receiptPrinter.js` | ESC/POS receipt printing |
| `src/lib/emailReceipt.js` | Client helper to call /api/send-receipt |
| `src/lib/calc.js` | Pure business logic functions (payout, tax, markdown, layaway, IDs) |
| `src/lib/phone.js` | Phone formatting utilities (formatPhone, isValidPhone, digitsOnly) |
| `src/styles/global.css` | All CSS variables and layout classes |
| `api/send-receipt.js` | Vercel serverless — email receipts via Resend |
| `api/admin-users.js` | Vercel serverless — full user management (admin role required) |
| `api/auth-events.js` | Vercel serverless — public lockout tracking (no auth required) |
| `public/test-plan.html` | Printable 97-test pre-launch manual test plan |
| `public/staff-guide.html` | Printable staff operations guide |

---

## 13. Session Log

_Use this section to record significant decisions, changes, or context from each working session._

| Date | Summary |
|---|---|
| 2026-05-02 | Initial comprehensive CLAUDE.md generated from full codebase review |
| 2026-05-07 | Added Self-Update Protocol and Pending items block to CLAUDE.md; added 4 Known Gaps rows (customer display, pos_sessions, Shopify sync, AI Shopify descriptions); added API key deployment warning to What Not To Do; made POS cart price override prominent (amber Negotiated badge, larger input, row highlight, strikethrough original price); added POS Price Negotiation business rule |
| 2026-05-07 | Full senior dev review; created 22 GitHub issues. Fixed 19: atomic POS sale RPC (complete_pos_sale), held cart localStorage persistence, ErrorBoundary component, session expiry warning, DB indexes, AI rate limiting + output validation, speech cleanup, dirty-state warning on EditItemModal, audit_log table + logAudit(), await onSaved() for loading feedback. Installed GitHub CLI (gh). Closed #16 (offline POS) as deferred. Open: #18–#22 (lows). |
| 2026-05-07 | Commercial app gap analysis; created 16 feature issues (#23–#38) + payout price adjustment (#39). Replaced POS inline price input with ✎ Override button + modal (select-all on open). First Vercel production deployment (`consignment-store-psi.vercel.app`). Updated deployment status in CLAUDE.md. |
| 2026-05-09 | Added `pos_sessions` table (anon-readable, single row `session_key='main'`), `/display` public route (customer-facing tablet, Realtime subscription, idle/cart/payment states), POS cart publishing useEffect, and pagination to Inventory (50/page) and Contracts (25/page). Schema requires re-run in Supabase SQL editor. |
| 2026-05-09 | F17: Sale price adjustment in payout finalize modal. Inline ✎ editing per sold item, live-updating gross/payout/batch totals, audit_log on save, adjusted prices in print output. Closes GitHub #39. |
| 2026-05-09 | 7 high/medium issues closed: #24 cash drawer kick, #25 store credit (table+page+POS), #26 role-based access (user_roles+RoleContext+nav filtering), #27 1099 report (Reports page), #28 markdown rules config (Admin tab), #30 shift reconciliation (Dashboard modal+shifts table), #31 split tender (POS+receipt+transactions). Schema requires re-run in Supabase SQL editor. |
| 2026-05-10 | F16: Appointment scheduling — appointment_settings + intake_appointments tables, /book public 3-step booking form (no auth), count_slot_bookings() RPC (anon capacity check without PII), Admin → Appointments tab (settings + upcoming/past list + status management). Schema requires re-run. |
| 2026-05-11 | #29: Email receipts — Resend integration via `api/send-receipt.js` Vercel serverless function. Email field added to POS right panel (optional, fires non-blocking after sale). Server-side RESEND_API_KEY + RECEIPT_FROM_EMAIL env vars. `src/lib/emailReceipt.js` client helper. |
| 2026-05-12 | Pre-launch prep: (1) 171-test automated suite — `src/lib/calc.js` pure business logic functions + `src/lib/calc.test.js` covering payout, tax, markdown, layaway, status transitions, receipt numbers, item IDs. All pass. (2) Printable 97-test manual test plan at `public/test-plan.html`. (3) In-app `/help` route (Help.jsx) with sidebar nav + search + 13 workflow sections. (4) Printable staff guide at `public/staff-guide.html` with quick ref card, full workflows, command table, troubleshooting, contacts. |
| 2026-05-12 | Bug fixes + features: Fixed Appointments tab infinite spinner (`.single()` → `.maybeSingle()` with default fallback). Added Admin → Users tab (invite/role/remove via `api/admin-users.js` Vercel serverless + SUPABASE_SERVICE_ROLE_KEY). Moved appointment scheduling to Contracts page (📅 Schedule Appointment button); Admin → Appointments retained for list/settings/contract-creation. Added upcoming appointments widget to Dashboard (7-day lookahead, today highlighted blue). Added "📋 Contract →" button on each appointment row — navigates to New Contract pre-filled with consignor name/phone/email. Added Pull Items section to Help guide. Created `src/lib/phone.js` with `formatPhone()`/`isValidPhone()`/`digitsOnly()`; applied to all phone inputs app-wide with (xxx) xxx-xxxx format and 10-digit validation. |
| 2026-05-13 | Added step 6 to Self-Update Protocol (copy CLAUDE.md to claude-context repo and push). Updated test-plan.html (97→114 tests: §13 Store Credit, §14 Appointments, email receipt, split tender, 1099, markdown rules, user management, role access). Updated staff-guide.html (9→11 sections: §6 Store Credit, §7 Appointments, split tender + email receipt in §1, sections renumbered). |
| 2026-05-14 | Full AI voice intake rebuild (AIIntake.jsx + aiParse.js). New parsing rules: Designer→Item Type→Description→Size→Price→"Enter" field order. New Claude system prompt returns {designer, item_type, item_type_flagged, description, size, price, raw_input}. Real-time live parsing panel with color-coded field states (green/amber/gray/red). Session list with fade-out animation. 10 voice commands: Enter/Next, Clear last/Remove last/Undo, Submit all, Correct [field] [value], Repeat last, Pause/Resume listening, Full/Brief/No feedback, Test. Web Audio API tone system (10 distinct tones). Web Speech API readback with 3 modes (full/brief/none) stored in localStorage. Mute toggle stored in localStorage. Session warmup message. PAUSED banner overlay. Processing indicator tick + 5s/10s timeout handling. Item count milestones every 5 items. getLists() exported from aiParse.js for real-time parser. Contracts.jsx updated with onRemoveLast + onSubmitAll props. |
| 2026-05-14 | Fixed real-time parsing panel designer/item_type preview. Added normStr(), matchDesigner(), matchItemType() helpers to AIIntake.jsx. matchDesigner tries 1–4 word prefixes against designer names + aliases with 60% coverage threshold to avoid false positives. matchItemType scans first 3 word positions so leading adjective/color words don't block item type detection. getLists() called on mic start and stored in listsRef (hits existing 4-min in-memory cache — no extra DB calls). Designer and item type fields now show actual matched names in green during speech instead of "listening…". |
| 2026-05-15 | Vercel environment variables confirmed active: RESEND_API_KEY + RECEIPT_FROM_EMAIL (email receipts now live), SUPABASE_SERVICE_ROLE_KEY (Admin → Users tab fully operational). Documented store website (www.samirasupsacleboutique.com) and future domain work: verify domain in Resend for branded sender (receipts@samirasupsacleboutique.com), point app.samirasupsacleboutique.com to Vercel. Updated Known Gaps and Pending sections accordingly. |
| 2026-05-15 | Full three-tier user management system. Roles: admin/manager/staff (hierarchy; no row = admin/owner). New: Admin → Users tab (create, role change, suspend/reactivate, force logout, unlock, remove; last-admin protection). Add User form: display name + email + role + temp password + strength meter + welcome email via Resend + force_password_change flag. Forced first-login password change flow (/change-password). Profile page (/profile — display name edit, change password). Forgot password (/reset-password — Supabase email recovery). Login: lockout check (5 failures → 30-min lock), suspension check. api/auth-events.js public endpoint for lockout tracking. Permissions tab (role/feature matrix). User menu in topbar (display name, role badge, My Profile). schema.sql extended: user_roles + display_name, status, force_password_change, failed_attempts, locked_until, created_at, updated_at; default role → 'staff'. ACTION REQUIRED: re-run schema.sql in Supabase. |
| 2026-05-15 | Bug fixes post user management launch. (1) Inline ✏️ name edit added to Admin → Users tab rows. (2) Profile save name: fixed upsert-without-role bug that demoted owner account to Staff — now uses update + conditional insert with role='admin'. (3) RoleContext now exposes setDisplayName() so nav bar updates immediately after name save without page reload. INCIDENT: first buggy upsert deployed briefly and demoted the owner account to Staff — recovered by manually setting role='admin' in Supabase Table Editor. Root cause documented in What Not To Do. |
| 2026-05-20 | User ran `supabase/schema.sql` in Supabase SQL editor — user_roles table extensions (display_name, status, force_password_change, failed_attempts, locked_until, created_at, updated_at; default role → 'staff') now applied to production database. Removed schema re-run from Pending. |
| 2026-05-20 | Three new features. (1) **Markdown Scanner Mode** rebuilt: Inventory toolbar 📉 button now opens overlay directly (no tier-picker modal). Tier selector lives inside overlay; scan input disabled until tier picked. Calculations always from `original_price` (falls back to `price` and saves it as `original_price` on first scan). New price rounded to whole dollar. Skip if `price` already equals computed value (mid tone + "Skip" speech + yellow). Success path: writes `price`, `original_price` (if null), `markdown_applied = true`, `needs_new_tag = false` (staff hand-updates physical tag); logs `audit_log` action `markdown` with from/to/original/tier/userId. Audio system added with `priceToWords()` helper — speaks new price as natural English ("one sixty", "seventy five", "one twenty five") via SpeechSynthesis at rate 0.9. Respects `intake_audio_enabled` localStorage mute. Session list persisted in `markdown.scanner.session` (date-scoped). "New Session" clears list but keeps tier. "Print Report" generates printable floor sheet. (2) **Bulk Tag Reprint** added to Admin → Data Management: lists items where `needs_new_tag = true`, filters by markdown-applied vs manually flagged + consignor + category, Print Selected / Print All via existing `printTags()` (ZPL/QZ Tray, max 40 batch). Post-print prompts "Mark as printed?" — if yes sets `tag_printed = true, needs_new_tag = false`. (3) **Database Health** cards added to Admin → Data Management: Photo Storage (count + MB estimate + purge old photos for sold/returned/pulled items, parses public URLs and removes from `item-photos` bucket); Database Summary (live counts per table, items grouped by status); Items Missing Photos (count + "View Items" → `/inventory?filter=nophotos`). Inventory page now reads `?filter=nophotos` URL param and applies a "Missing Photos" filter with a dismissable chip in the toolbar. Imports: Inventory.jsx now imports `useRole` + `logAudit`; Admin.jsx now imports `printTags`. Build clean, 171 tests pass. |
| 2026-05-20 | Self-Update Protocol step 7 reworked to handle jsDelivr CDN throttling: if first purge returns `throttled`, wait 60s and retry once; if still throttled, add `CDN purge pending` to Pending section and move on (throttle resets overnight). No more unbounded retry loops. |
| 2026-05-20 | Step 7 hardened again: escalating retry schedule (first attempt → 2-minute retry → 5-minute retry → defer to Pending), and every session log entry must now end with one of four explicit purge-result tokens (`finished` / `throttled — retried after 2m, finished` / `throttled — retried after 2m + 5m, finished` / `throttled — deferred to Pending`) so we can audit purge success after the fact. CDN purge: finished. |
| 2026-05-20 | Added **How to Share With claude.ai** section at top of CLAUDE.md: copy/paste the file directly into the chat for same-day context (most reliable). CDN URL kept as backup for next-day reads since jsDelivr caches aggressively and purges are throttled. Replaces the old bare "CLAUDE.md Fetch URL" section. CDN purge: finished. |
