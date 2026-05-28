# Consignment Store — Claude Code Master Reference

> **Trim note:** This file is kept lean on purpose. Full column-level schema lives in `supabase/schema.sql` (source of truth). Old session-log detail is collapsed to one-liners. Don't re-expand either — point to the code instead.

---

## Self-Update Protocol

At the end of every session Claude Code must:
1. Check off completed items in Build Status
2. Add new business rules or conventions established
3. Add new routes, tables, or integrations created
4. Add a one-line session log entry with date and summary (keep it short — detail belongs in the code/commits)
5. Update Known Gaps if anything was resolved or added
6. Push to both repos:
   a. Commit and push CLAUDE.md in the consignment-store repo (private)
   b. Copy CLAUDE.md to `C:\claude-context\CLAUDE.md` and push to the claude-context repo (public — backup fetch source)
7. Purge the CDN, then **log the result** in today's entry with one of the exact tokens below:
   `curl "https://purge.jsdelivr.net/gh/greghussey62/claude-context@main/CLAUDE.md"`
   - `{"status":"finished"}` → log `CDN purge: finished`
   - `throttled` → wait 2 min, retry once → log `CDN purge: throttled — retried after 2m, finished`
   - still throttled → wait 5 more min, retry once → log `...retried after 2m + 5m, finished`
   - still throttled → add `CDN purge pending — run manually before next session` to Pending, log `CDN purge: throttled — deferred to Pending`, and stop (throttle resets overnight)

Do this automatically without being asked.

## How to Share With claude.ai
Paste this file's contents directly into the chat for same-day context — the jsDelivr CDN caches aggressively and purges are throttled, so the fetch URL is only reliable next-day. In VS Code: open `CLAUDE.md`, `Ctrl+A`, `Ctrl+C`, paste.
Backup fetch URL: `https://cdn.jsdelivr.net/gh/greghussey62/claude-context@main/CLAUDE.md` (consignment-store repo is private; claude.ai fetches the public claude-context mirror.)

## Pending From Last Session
- Move Anthropic API key to server-side before any public deployment
- Tech debt open: #19 (QZ Tray cert), #20–#21 (accessibility), #22 (archival)
- Feature backlog open: #23 (consignor sale email — skipped), #32–#36
- **Tag text alignment:** text still prints right-justified in read-orientation. Left-justify approach attempted (Y_L flip + ^FB L/R swap) but reverted — the axis behaviour wasn't what was expected. Resume next session with fresh diagnostic prints.
- **Tag PT_ID=30 centering:** physical print pending to confirm item_id looks balanced in the above-hole strip.

---

## 1. What This App Is & Who Uses It

Consignment store management for a small fashion shop (**Samira's**). The store takes items from consignors, sells them, pays out the consignor's share later.

**Primary user:** non-technical store owner who does every task herself (intake, selling, printing, payouts) on a Windows machine with thermal printers. Not a developer.
**Design imperative:** simplicity and plain language over technical sophistication. Short workflows, minimal clicks, no jargon.
**Secondary user:** consignors — no login, view their account via a public portal lookup.

---

## 2. Tech Stack

React 19 + Vite 8 · React Router v7 (client-side SPA) · Supabase (Postgres + Auth + Realtime + Storage) · Anthropic Claude API (Haiku, model `claude-haiku-4-5-20251001`) · QZ Tray printing · qrcode.react · Vitest · ESLint · plain CSS with variables (no Tailwind, no CSS-in-JS) · Vercel (prod: `consignment-store-psi.vercel.app`).

Key versions: react 19.2.5 · react-router-dom 7.14.2 · @supabase/supabase-js 2.104.1 · @anthropic-ai/sdk 0.91.0 · qz-tray 2.2.6 · vite 8.0.10 · vitest 4.1.5.

---

## 3. Hardware Context

Runs on a physical retail counter. Devices:
- **Barcode scanner** (Tera HW0002, USB keyboard-wedge — emits text + Enter): POS lookup, markdown scanning, reconciliation
- **Thermal tag printer** (iDPRT iF4, 3"×1.5" die-cut hang-tag): ZPL at 203 DPI via QZ Tray
- **Receipt printer** (Epson TM-T20III): ESC/POS, 42-char wide, via QZ Tray
- **Cash drawer** (APG Vasario): RJ11 off the Epson — opens on `CMD:CASH` only
- **Counter PC:** Beelink Mini S12 Pro · **Staff monitor:** MUNBYN 12" touchscreen (video + touch over one HDMI)
- **Customer display:** Amazon Fire HD 8 tablet (WiFi only, no PC link — relies entirely on Supabase Realtime to `/display`)
- **Back office:** Brother MFC-L2820DW laser
- **Phone camera:** QR-based photo uploads (mobile browser, no app)

**QZ Tray** is a local desktop app (`localhost:8181`) bridging browser → raw printer access. Must be running. Printer names in `localStorage` (set via Admin). Dry-run mode previews without hardware.

**Web presence:** store site `www.samirasupsacleboutique.com`. Future: verify domain in Resend for branded sender (`receipts@samirasupsacleboutique.com`); point `app.samirasupsacleboutique.com` → Vercel.

---

## 4. Business Rules

### Contract Lifecycle
Create contract → sequential `contract_num` (~3000+). Add items → each gets `item_id` = `{contract_num}-{seq:02d}` (e.g. `3042-01`). Items on floor until sold/returned. At payout: calc consignor share, issue check, return unsold items.

### Split / Payout
Default split 50/50 (`split_pct` = consignor's %, configurable per contract). Consignor gets `split_pct/100 × sale_price`; store gets the rest. Per-item `base_amount` = guaranteed payout, overrides split%. No-tax cash toggle at POS.

### Item Status Flow
`active → sold` (POS) · `active → layaway` · `active → returned` (consignor takes back) · `active → pulled` (store claims after deadline) · `sold → active` (return/refund).

### Markdown — two paths
- **Batch (Reports → Run Markdown):** items 60+ days on floor; applies tier %, stores `original_price`, sets `markdown_applied = true`, `tag_printed = false`. Floor Sheet prints for physical re-pricing. (Markdowns are hand-written with a red sharpie — no tag reprint.)
- **Scanner Mode (Inventory → 📉):** full-screen overlay; pick tier (20%/50%) inside overlay before scan input enables. Always calcs from `original_price` (if null on first scan, saves current `price` as `original_price` first). New price rounded to whole dollar. Skip if `price` already equals computed value (mid tone + "Skip" + yellow flash, no write). Success: writes `price`, `markdown_applied = true`, **`needs_new_tag = false`** (staff hand-update tag with red sharpie); high beep + speaks new price as words via SpeechSynthesis. Logs `audit_log` action `markdown` (from/to/original/tier/userId). Audio respects `intake_audio_enabled` localStorage mute. Session list in `markdown.scanner.session` (date-scoped). "New Session" clears list, keeps tier. "Print Report" → floor sheet.
- **Tiers:** multi-tier rules in `localStorage` key `markdown.rules` as `[{days, pct}]`; configured Admin → Markdown Rules; default 60d → 20%.
- ⚠️ Scanner sets `needs_new_tag = false` (harmless dead write — see Tag Printing; markdowns are red-sharpie, never reprinted).

### Payout Batches
Each contract has 1+ batches (pickup windows). First auto-created on first detail view. Items assigned via dropdown. Payout includes all batches. After: confirm pickup or claim unclaimed as store property.

### Layaway
POS cart → items status `layaway` + `layaway_ref`. Partial payments in `payments` jsonb. Fully paid → Complete → creates sales records. Not picked up → Forfeit → store keeps deposit, items return to `active`.

### POS Command Cards (scan codes)
`CMD:CASH` (cash screen — also the ONLY drawer-open trigger) · `CMD:CARD` · `CMD:APPROVED` · `CMD:DECLINED` · `CMD:VOID` (remove last) · `CMD:CLEAR` · `CMD:HOLD` · `CMD:RESUME` · `CMD:10OFF/20OFF/50OFF` (discount last item).

### Tax
Default 6% (Admin → Receipt Printer). No-tax cash option at POS.

### Receipt Numbers
`R-{seq}` from Postgres sequence `receipt_seq` (starts 1001), via RPC `next_receipt_number()`.

### POS Price Negotiation
Per cart row ✎ Override button → modal (shows tag price, discount %, Enter to confirm). Override recorded as `sales.sale_price` + `items.sold_price`. Original `price` unchanged (shows strikethrough). After override: amber "Negotiated" badge, yellow row. `sales.listed_price` always keeps original tag price for reporting.

### Split Tender
Cash + card in one transaction. Cashier enters cash portion; card = total − cash. Stored `payment_method='split'` + `payment_method_2='card'` + `amount_tendered_2`. Both lines on receipt.

### Store Credit
Accounts keyed by phone, looked up at POS. Balance deducted from `store_credit` after `complete_pos_sale`. Debit logged in `store_credit_transactions` (reason: sale/return/adjustment). Partial credit → toast with remainder, collect rest cash/card. Returns issue credit via Admin → Store Credit.

### Shift Reconciliation
Dashboard → Close Shift. Expected cash = opening float + cash sales − cash returns (cash filtered to `payment_method='cash'`, `sale_date=today`). Over/short → `shifts` table.

### Appointment Scheduling
Config in `appointment_settings` DB table (NOT localStorage — must be reachable from any device). Anon INSERT on `intake_appointments`; `count_slot_bookings()` RPC checks capacity without exposing PII. Public booking URL `{origin}/book`. Staff-side scheduling on Contracts page (📅 button); Admin → Appointments shows list/settings + "📋 Contract →" convert. Dashboard shows next 7 days.

### Photo Management
Max 4 photos/item. Path `items/{contractNum}/{itemId}/photo_{slot}.jpg`. Client compress before upload (max 1200px, JPEG 82%). QR phone upload: token → QR → scan → camera upload; token expires 15 min. Realtime in EditItemModal picks up new photos live. Purge old photos via Admin → Data Management (sold/returned/pulled items; deletes Storage files + sets `photos = []`, preserves record).

### Tag Printing (two paths only — simplified 2026-05-23)
Real workflow: all of a consignor's tags print once at intake, get attached, items go to floor. Tags are permanent — markdowns are hand-written (red sharpie), never reprinted. Only reprint scenario is a rare damaged/lost tag.
- **Path 1 — Print All Tags for a contract (the 99% case):** button in the contract detail toolbar (`Contracts.jsx` `ContractModal` footer). Gathers active items → `printTags()` (shared ZPL/QZ Tray, dry-run honored). On real (non-dry-run) print, sets `tag_printed = true` on those items. This is the **only** writer of `tag_printed = true` — it keeps the Inventory "Needs Tag" tab + Dashboard "needs tag" alert working.
- **Path 2 — Single-item reprint:** "Print Tag" button on the item detail view (`ItemDetail.jsx` → `printTag()`), for the rare damaged/lost tag.
- **Also kept:** Inventory multi-select "Thermal Tags" button (ad-hoc batch via `printTags()`); Admin → Tag Printer test/config (`getZplPreview`/`testPrint`).
- **`tag_printed`:** written `true` only by Path 1; reset `false` by the markdown paths (Reports Run Markdown, Inventory markdown modal). Read by Inventory "Needs Tag" tab + Dashboard alert.
- **`needs_new_tag`:** column retained but has **no readers** since the old sheet/Bulk-Tag-Reprint system was removed. Remaining writes (`db.js` payout keep-active = true; markdown scanner = false) are harmless dead writes.

### Database Health (Admin → Data Management, manager+ only)
- **Database Health cards:** Photo Storage (counts + MB estimate + purge button), Database Summary (rows per table, items by status), Items Missing Photos ("View Items" → `/inventory?filter=nophotos`).

---

## 5. Build Status

### Fully Built & Working
Auth · Contracts CRUD · Items (AI + manual add, edit, status) · AI Intake (voice + text → Haiku) · Inventory (filter/sort/bulk, 50/page) · POS (scan, cart, payment, receipts, price negotiation, split tender, store credit) · Layaway · Payouts (+ batches, + inline price adjustment w/ audit_log) · Thermal tag print (ZPL) · Receipt print (ESC/POS) · Cash drawer kick · Photo uploads (direct + QR mobile) · Markdown (batch + Scanner Mode) · Markdown rules config · Reports & analytics · 1099 report · Sales history + reprint · Consignor statements · CSV export · Admin (item types, designers, printer config) · Help chat (Claude) · Dashboard (stats, Z-report, alerts, shift reconciliation, 7-day appt widget) · Camera Mode (Realtime) · Customer display `/display` (Realtime) · Contracts pagination (25/page) · Role-based access (admin/manager/staff) · Three-tier user management (CRUD, lockout, suspend, welcome email, force password change) · Profile page · Forgot/reset password · Account lockout (5 fails → 30-min) · Appointment scheduling (public /book + Admin) · Phone formatting (`src/lib/phone.js`) · Email receipts (#29, Resend) · Tag printing (Print All Tags per contract + single-item reprint) · Database Health cards · Automated tests (171, passing) · In-app Help (/help) · Printable test plan + staff guide (public/*.html).

### Known Gaps / Placeholders
- **Consignor portal (public view):** `getPortalData()` exists in db.js, no public route wired
- **Declined items:** DB table exists, no UI
- **Consignor sale email notifications:** skipped (#23), needs Resend wired to sale events
- **Appointment confirmation email:** Resend infra in place, needs wiring to appointment insert
- **Shopify product sync (#33)** + **AI Shopify descriptions (#34):** not built
- **Offline POS mode:** deferred (#16 closed)
- **Accessibility (#20, #21), data archival (#22), QZ Tray cert (#19):** open issues

---

## 6. Integrations (high level)

**Supabase:** Auth (email/password). All queries via `@supabase/supabase-js` in `src/lib/db.js`. RLS on every table — authenticated get full access via `auth_all`; `photo_upload_sessions` has anon read for unexpired tokens; anon SELECT on `pos_sessions`/`appointment_settings`, anon INSERT on `intake_appointments`. Storage bucket `item-photos` (public). Realtime in EditItemModal/ItemDetail/display. RPCs: `next_receipt_number()`, `upload_item_photos(token, photos)` (SECDEF), `complete_pos_sale(...)` (SECDEF, atomic sale), `count_slot_bookings(date, slot)` (SECDEF, anon).

**Anthropic Claude (Haiku):** `src/lib/aiParse.js` parses spoken/typed items → JSON; system prompt built from DB item_types + designers, cached 4 min (ephemeral prompt cache, 5-min TTL); `max_tokens 300`. `src/components/HelpChat.jsx` streaming assistant with full app docs in system prompt. ⚠️ Key exposed client-side via `VITE_` — OK for private intranet, NOT for public deploy.

**QZ Tray:** see Hardware. Tag = ZPL, receipt = ESC/POS. Config in localStorage. Dry-run available. Max 40 tags/spool.

---

## 7. Database Schema

**Full column-level definitions live in `supabase/schema.sql` — that is the source of truth. Re-run it in the Supabase SQL editor after any change (uses `IF NOT EXISTS`/`DROP IF EXISTS` guards, safe to re-run in full).**

Tables (purpose):
- `contracts` — consignor agreements (`contract_num` serial, `split_pct`, status)
- `items` — inventory; PK `item_id` `{contract_num}-{seq:02d}`; `item_type` (the kind, e.g. "Dress") + `description` (free-text detail, e.g. "black floral"), `brand`, `size`, `price`/`original_price`/`sold_price`, `status`, `markdown_applied`, `needs_new_tag`, `photos[]`, `base_amount`. (Renamed 2026-05-23: old `color` → `description`; old `description` held the type → now `item_type`. No more swap.)
- `sales` — line items (one per item sold/returned; negative `sale_price` for returns); carries `item_type` + `description` snapshot; `listed_price` keeps original tag price
- `transactions` — receipt groupings; `receipt_number` `R-{seq}`; split-tender fields `payment_method_2`/`amount_tendered_2`
- `payouts` — check runs (check#, pickup deadline, void/reissue tracking)
- `payout_batches` — pickup windows per contract
- `layaways` — holds; `items`/`payments` jsonb
- `item_types` — category vocab (~150) · `designers` — brand names + phonetic `aliases` for AI matching
- `declined_items` — small UI in Contracts (add/list); column `description` stores free-text (e.g. "Prada bag, water damage"); on-screen label is "Description" — column name and label agree
- `audit_log` — immutable change log (price/status/payout/delete/markdown); `details` jsonb
- `photo_upload_sessions` — QR upload tokens (15-min expiry; anon SELECT if unexpired)
- `user_active_item` — Camera Mode active item per user (own-row RLS)
- `pos_sessions` — single row `session_key='main'`, Realtime cart sync for `/display` (anon SELECT)
- `user_roles` — role per user + display_name/status/force_password_change/failed_attempts/locked_until. **No row = owner = admin.**
- `shifts` — end-of-day cash reconciliation
- `store_credit` + `store_credit_transactions` — balances + ledger
- `appointment_settings` (single row, anon SELECT) + `intake_appointments` (anon INSERT)

Sequence `receipt_seq` (1001+). Indexes on `items(status/contract_id/contract_num)`, `sales(contract_id/sale_date/transaction_id)`.

---

## 8. Key Conventions

**Files:** `src/components/` (reusable UI) · `src/lib/` (data layer: supabase.js, db.js, aiParse.js, photos.js, tagPrinter.js, receiptPrinter.js, calc.js, phone.js, emailReceipt.js) · `src/pages/` (one per route) · `src/styles/global.css` only (no per-component CSS).

**Routing (App.jsx):** all routes under one `<Layout>`. Public (no auth): `/upload/:token`, `/display`, `/book`, `/reset-password`. Auth but outside force-password-change: `/change-password`, `/profile`. Everything else through `CheckPasswordChange` (redirects to `/change-password` if `force_password_change`). No lazy loading.

**Roles:** `RoleProvider` (RoleContext.jsx) wraps authed routes. `useRole()` → `{ role, isAdmin, isManager, displayName, userId, userEmail, setDisplayName, loaded, forcePasswordChange }`. Three roles admin/manager/staff; **no user_roles row = owner = admin (don't break)**. Nav: staff see Dashboard/POS/Contracts/Inventory/Layaway; manager+ add Payouts/Reports/Sale History/Store Credit/Admin; admin-only: Users + Permissions tabs in Admin. Call `setDisplayName()` after saving a name so nav updates without reload.

**Phone:** all inputs use `formatPhone()` from `src/lib/phone.js` → `(xxx) xxx-xxxx`, caps 10 digits. `isValidPhone()` before save, `digitsOnly()` for DB queries. Never store raw input.

**Vercel API routes:** `api/send-receipt.js` (Resend, `RESEND_API_KEY`) · `api/admin-users.js` (Supabase Admin API, `SUPABASE_SERVICE_ROLE_KEY`) · `api/auth-events.js` (public lockout tracking). First two validate caller JWT; server-side keys never reach browser.

**Data access:** all Supabase queries in `src/lib/db.js` (`getX/createX/updateX/deleteX`), pages import named functions — never call `supabase` directly (except Realtime + a few inline selects).

**State:** no global store. Component-local `useState`/`useEffect`, re-fetch on load/mutation.

**Styling:** CSS vars in `global.css` `:root`. BEM-ish (`.stat-card`, `.badge-active`). Status badges `.badge .badge-{status}` auto-map color. All styles global.

**Printing:** tag ZPL / receipt ESC/POS, config in localStorage via Admin, dry-run for both, QZ checked before print, max 40 tags/batch.

**Markdown Scanner reporting:** Scanning is pure DB input (audit_log is the system of record). The on-screen session list (`marked` state / `localStorage` `markdown.scanner.session`) is a cosmetic per-device scratchpad only. "Print Report" queries `getMarkdownAuditReport(date)` in db.js — filters `audit_log` by `action='markdown'` + date, joins `items` (active only), groups by user. Name resolution: `details.markdown_applied_by_name` (durable, written on each scan) → `user_roles.display_name` → current user's `email.split('@')[0]` (matches RoleContext for owner-admin who has no role row) → `'Admin'`. Date picker + All Stations / My Scans Only filter. Runs from any machine. Note: batch markdown path (Reports → Run Markdown) does NOT write audit_log, so the report covers scanner activity only.

**AI Intake:** system prompt rebuilt when item_types/designers change; in-memory cache refresh at 4 min (within 5-min prompt-cache TTL). Call `invalidateListCache()` from Admin after vocab edits. Voice = Web Speech API (Chrome/Edge only). AI returns `item_type` + `description`; these flow **straight through** to the matching `items` columns — `AIIntake.jsx` no longer swaps them (the old `item_type→color`, `description→` bug is gone, see What Not To Do).

**IDs:** `contract_num` int ~3000+ · `item_id` `{contract_num}-{seq:02d}` · `receipt_number` `R-{seq}` (1001+) · `layaway_ref` human-readable.

**Env vars** (VITE_ = in browser bundle; others server-side only): `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`, `VITE_ANTHROPIC_API_KEY` (client-side, intranet only), `RESEND_API_KEY`, `RECEIPT_FROM_EMAIL`, `SUPABASE_SERVICE_ROLE_KEY`. All confirmed active in Vercel as of 2026-05-15.

---

## 9. What Not To Do

- **No CSS framework** (Tailwind/shadcn) — plain CSS vars in `global.css`. **No global state library.** **No backend/server routes** without discussing first (everything's client-side against Supabase; the few Vercel functions are the exception).
- **Don't move Supabase queries out of `src/lib/db.js`** into components.
- **Don't expose the Anthropic key publicly / deploy to a public URL** without moving AI calls server-side (currently `VITE_`-exposed, visible in the bundle).
- **Don't change `item_id` format** `{contract_num}-{seq:02d}` — it's barcoded on physical tags.
- **Don't reintroduce the item_type/description swap.** Columns are now correctly named: `items.item_type` holds the kind ("Dress"), `items.description` holds free-text detail ("black floral"). AI `item_type`→`item_type`, AI `description`→`description`, straight through. The pre-2026-05-23 code stored the type in `description` and the detail in `color`, and `AIIntake.jsx` swapped on the way in — that's all gone. `db.test.js` asserts the no-swap mapping.
- **Don't change ZPL tag format** without checking physical dims (3"×1.5", 203 DPI = 609×304 dots).
- **Don't edit `supabase/schema.sql` and forget to tell the user to re-run it** in the SQL editor.
- **Don't use `git add .` / `git add -A`** — `.env.local` has real keys.
- **Don't upsert `user_roles` without an explicit `role`** — table default is `'staff'`, so an omitted role silently demotes the owner to Staff. (Past incident: owner demoted this way, recovered manually in Table Editor.) Pattern: try `update`, if 0 rows then `insert` with `role='admin'` (see Profile.jsx `handleSaveName`).
- **Don't add complexity for its own sake** — one-user desktop app for a small store; optimize for clarity, not scale.
- **Keep this file lean** — full schema in `supabase/schema.sql`, session detail in commits. Don't let CLAUDE.md balloon past ~40k chars (Claude Code flags it + it burns usage faster every request).

---

## 10. Build & Run

```
npm run dev        # dev server :5173
npm run build      # prod build → dist/
npm run preview    # preview prod build
npm run lint       # ESLint
npm run test       # Vitest
npm run test:watch # watch mode
```

---

## 11. Reference Files

`consignment-store.html` (original single-file impl — business-logic source of truth) · `supabase/schema.sql` (schema source of truth) · `src/lib/db.js` (all queries) · `aiParse.js` (Claude parsing) · `tagPrinter.js` (ZPL) · `receiptPrinter.js` (ESC/POS) · `emailReceipt.js` · `calc.js` (pure business logic — payout/tax/markdown/layaway/IDs) · `phone.js` · `global.css` · `api/send-receipt.js` · `api/admin-users.js` · `api/auth-events.js` · `public/test-plan.html` · `public/staff-guide.html`.

---

## 12. Session Log

Keep entries to ONE LINE. Older detail collapsed intentionally — full history is in git.

| Date | Summary |
|---|---|
| 2026-05-02 → 05-13 | Initial CLAUDE.md; senior-dev review (22 issues, 19 fixed incl. atomic `complete_pos_sale`, audit_log, indexes); commercial gap analysis (#23–#39); first Vercel deploy; `pos_sessions` + `/display`; payout price adjustment; cash drawer, store credit, role-based access, 1099, markdown rules, shift reconciliation, split tender; appointment scheduling; email receipts (#29); 171-test suite + /help + printable plan/guide; Admin → Users tab; phone formatting. |
| 2026-05-14 | AI voice intake rebuild (AIIntake.jsx + aiParse.js): new field order + Claude prompt, live parsing panel, 10 voice commands, Web Audio tones, speech readback (3 modes), session list. Designer/item-type live matching helpers (normStr/matchDesigner/matchItemType). |
| 2026-05-15 | Vercel env vars confirmed active (email receipts + Users tab live). Documented store site + future domain work. Full three-tier user management (admin/manager/staff, CRUD, lockout, suspend, force password change, /change-password, /profile, /reset-password, Permissions tab). schema.sql extended (user_roles). INCIDENT: buggy upsert demoted owner to Staff → recovered manually; root cause in What Not To Do. |
| 2026-05-20 | Ran schema.sql in prod (user_roles extensions applied). Three features: Markdown Scanner Mode (Inventory overlay, tier selector, audio+speech via priceToWords(), audit_log, date-scoped session), Bulk Tag Reprint (Admin → Data Management), Database Health cards (Admin → Data Management). 171 tests pass. Reworked Self-Update step 7 (escalating CDN-purge retry + result tokens). Added "How to Share With claude.ai". Slimmed CLAUDE.md back under 40k (collapsed schema dump + old session log). Admin tabs grouped into five labeled sections. CDN purge: finished. |
| 2026-05-21 | Decoupled Markdown Scanner reporting from per-device localStorage. Print Report now queries audit_log (getMarkdownAuditReport in db.js): date picker, All Stations / My Scans Only filter, groups by user, active items only, prices from audit log details. Added details.markdown_applied_by_name for durable attribution. Name resolution falls back through stored name → user_roles → current user's email prefix → 'Admin' (consistent with RoleContext for owner with no role row). On-screen session list kept as cosmetic scratchpad; New Session still clears it. 171 tests pass. CDN purge: finished (×2). |
| 2026-05-23 | System-wide field-naming fix (pre-launch, data disposable → drop-and-recreate). Renamed `items`/`sales` columns: old `color`→`description` (free-text detail), old `description`(held the type)→`item_type`. Updated `complete_pos_sale` RPC in lockstep. Removed AIIntake.jsx swap (AI item_type/description now flow straight through). Standardized all UI labels to "Item Type"/"Description"; eliminated every "Color"/"Category" label. Left `item_types.category_group` taxonomy untouched. ~20 files swept; db.test.js gains no-swap assertion; 171 tests pass; live round-trip verified. Gotcha: `CREATE TABLE IF NOT EXISTS` skips existing tables — must `DROP … CASCADE` first. Follow-up: reverted `declined_items.item_type`→`description` (stores free-text, not a type; label was already "Description"). CDN purge: finished. |
| 2026-05-23 (2) | iDPRT iF4 hang-tag template rewrite (tagPrinter.js). New portrait layout: item_id (large) → item_type → brand → description (2-line ^FB word-wrap) → price/size row → DataMatrix (^BX, encodes item_id, centered). All content rotated via single ROT constant. First print: ROT='R' printed upside-down → corrected to ROT='B' (270°CW); physically confirmed correct orientation. Fixed pre-existing Admin Bulk Tag Reprint query selecting deleted `color` column + missing `item_type`. 171 tests pass. CDN purge: finished. |
| 2026-05-23 (3) | Confirmed working iF4 tag template (all fields print + DataMatrix scans). Reverted uncommitted broken changes left by prior stuck session. Remaining TO DO: flip 180° so item_id is at hole end, center text. Hardware doc corrected: Zebra ZD421 → iDPRT iF4. CDN purge: finished. |
| 2026-05-23 (4) | Tag subsystem simplified to two paths (real workflow: batch-print at intake, permanent tags, red-sharpie markdowns). Built "Print All Tags" in contract detail (`Contracts.jsx` ContractModal footer → `printTags()`, sets `tag_printed=true`, now sole true-writer). Deleted obsolete sheet system: `Tags.jsx` page + `/tags` route + 3 nav links (Inventory toolbar/Sheet Tags, Dashboard ×2). Removed Bulk Tag Reprint (`BulkTagReprintSection`) from Admin Data Management + trimmed unused `printTags` import. Removed dead `needs_new_tag=true` set in Reports Run Markdown. Kept: `printTags()`/`printTag()`/ZPL template (untouched), ItemDetail single reprint, Inventory "Thermal Tags", scanner `needs_new_tag=false`. `needs_new_tag` column retained (now reader-less). Help.jsx wording aligned. Build clean, 171 tests pass. CDN purge: finished. |
| 2026-05-24 | Added `^POI` after `^XA` in renderMerchandiseTag() (tagPrinter.js) to test hardware 180° flip — item_id at hole end, DataMatrix at far end. Labelary ignores ^POI; physical print result pending next session. If iF4 also ignores it, will do coordinate-transform approach instead. CDN purge: finished. |
| 2026-05-26 | No code changes. ^POI physical test result still pending — report at next session start. CDN purge: finished. |
| 2026-05-28 | Tag template polish sprint: ^POI confirmed working on iF4. Font hierarchy (FH_ID 24, FH_TYPE 44, FH_BRAND 40, FH_DESC 26, FH_PRICE 52) physically verified. Circled size in price row (^FO415,15^GC70,3,B) physically verified. item_id moved to above-hole strip (PT_ID=30, PT_TYPE=166 hardcoded). Price now whole-dollar (Math.round, no cents). Left-justify ongoing — attempted Y_L flip reverted. CDN purge: finished. |
