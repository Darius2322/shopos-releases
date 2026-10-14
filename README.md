# ShopOS app — Round 18 (added on top of Round 17)

- **Sign-in screens**: Login, Register, Check status and Forgot password share one split-screen layout (blue brand panel left, form right on desktop). Logo panel fixed to match the phone version.
- **Link and restore tile** on Login: *Link phone* (desktop shows a QR/code) or *Link desktop* (phone scans). *Restore* needs the backup password AND the owner's sign-in. Inside the account: More menu -> Link to phone / Link to desktop (signed-in only). Camera scanning needs Chrome on Android; iPhone/Safari users type the 8-character code.
- **Register** also has Check status and extra business details (type and town).
- **Security** now holds account deletion authorization.
- **Sales / Profile hubs**: redesigned main tabs (`components/HubTabs.tsx`). **Guide**: Demo button.
- **Charts**: recharts pie/bar replaced by a shared one-colour kit (`components/charts/kit.tsx`) in Analytics.
- **About / Services** public pages rewritten (business types come from `lib/businessTypes.ts`).
- **Installers are zipped** by the GitHub workflow before publishing (`desktop-build.yml`, `publish-installers.mjs`).
- Backend: migration `round18_*` and edge function `device-link` v3 are already live.

# ShopOS app — Round 17

## Menus
- **Business Profile** now has two tabs: *Profile* and *Branding & receipts* (logo, business type, receipt format, numbering). Old links `/branding` and `/printer-settings` redirect there.
- **Receipts & Printer** settings now sit at the bottom of Branding & receipts → *Receipt format* (the existing printer page is embedded unchanged).
- **End of Day** is now a tab inside **Sales**. Old `/end-of-day` links redirect to `/sales?view=end-of-day` and keep the today / calendar / days tab. The Dashboard shortcut and closing notifications point there.
- **Users** is renamed **Staff** (menu, page title, counts, guide text). Routes stay `/users`.
- NEW `features/settings/BusinessProfileHub.tsx`, `features/sales/SalesHub.tsx`. CHANGED `App.tsx`, `AppShell.tsx`, `BrandingPage.tsx` (prop `embedded`, reads `?section=receipt`), `EndOfDayHub.tsx` (keeps `view` in the address), `QuickActions.tsx`, `notificationFeed.ts`, guide text.

## Friendly errors
- NEW `lib/friendlyError.ts`: every on-screen error goes through it. Database, network, permission, duplicate and session errors become short plain sentences; anything technical (table names, SQL codes, URLs, stack traces) is replaced by "Something went wrong. Please try again." Your own plain-language messages pass through.
- Applied to sign-in, password reset, account setup, debts, invoices, quotations, sales requests, staff, inventory, batches, bundles, stock adjust, backup, branding, share sheets, reviews, device log and the sync status banner. Logic that reads raw messages (sync retry rules, permission checks) is untouched.

## Error reporting to your Monitor
- NEW `lib/errorReporter.ts`: unexpected errors and sync failures are sent (emails, phones and tokens blanked; no query strings; business taken from the session on the server) to `report_client_event`. Max 12 per session, silent offline and in the demo. They appear in the admin Monitor.

## Needs you
- Run the Vercel build once; nothing was compiled here (syntax-checked only).
- `PrinterSettingsPage` and `BusinessProfile` were not visible to me. They are used as-is; if the printer page shows its own big title inside the tab, tell me and I will add a compact mode.
- Supabase's free plan cannot raise the 50 MB single-file upload limit, and installers are larger. Publish installers by linking them (admin → Installers → "Link an installer hosted elsewhere"), or upgrade the Supabase plan.

---
# ShopOS – this round (drop-in files, same paths under `src/`)

Database (already applied live to project ShopOs via migrations phase30–32):
- `inventory_movements`: + `note`, `quantity_before`; insert guard (needs inventory.adjust, same-business product, actor forced to auth.uid()).
- `businesses.document_template` (classic|modern|compact|ledger|bold) – only owner / business.settings can change (trigger).
- `document_shares` + `create_document_share` / `revoke_document_share` / `get_public_document_full` (quotation + invoice links; cross-business creation rejected – tested).
- `client_activity_events` + `record_client_activity()` – offline sessions uploaded with real timestamps; advances last_active_at / last_login_at / login counts.
- Admin RPCs: `admin_transactions_per_day`, `admin_top_businesses`, `admin_activity_feed`, upgraded `admin_business_usage` (all check is_platform_admin).

## shopos-app (`app/src`)
- NEW `lib/documentTemplates.ts`, `components/DocumentView.tsx` – 5 ShopOS-branded templates (footer cannot be removed).
- NEW `components/ShareDocumentSheet.tsx`, `lib/documentShare.ts`, `lib/documentData.ts` – "Send digital receipt/quotation/invoice": WhatsApp / SMS / copy / native share as link, or share as picture (works offline).
- NEW `features/documents/PublicDocument.tsx` (route `/d/<token>`), `DocumentTemplatesPage.tsx` (More → Document Templates).
- NEW `lib/clientActivity.ts` – offline-safe session log. `lib/auth.ts` records app opens.
- NEW `features/inventory/StockAdjustSheet.tsx`; `lib/products.ts` `adjustStock()` – Add / Remove / Set, reason, note, before→after, audit event; product page history shows who/why/note.
- `InventoryList.tsx`: category menu (chips + counts + inline "New category"), richer dashboard (units, retail value, potential profit, expiring, stock-health bar, top categories), sort + supplier + stock-status filters, quick adjust per row.
- `lib/sync.ts`: finished items leave the queue and counters immediately; duplicate queue entries collapse; activity flushed after sync.
- `index.css`: visible hover/pressed states in dark mode (desktop).
- `MoreMenu.tsx` / `AppShell.tsx`: items flagged `comingSoon` (M-Pesa) are inert with a "Coming soon" badge.
- Routes added in `App.tsx`; types in `lib/types.ts`.

## shopos-admin (`admin/src/pages`)
- `Dashboard.tsx`: transactions per day chart (7/30/90d) + top performing shops with growth vs prior period.
- `BusinessDetail.tsx`: last login, last inventory update, last offline session, logins today/7d/30d, merged activity timeline (flags offline sessions).

## Not verified / needs you
- I only had the change bundles, not the full repos, so nothing was compiled here. Run `tsc`/build once after copying.
- `lib/db.ts`, `lib/audit.ts` weren't in the bundle; I used the exports the existing code already imports (`db`, `newRecordBase`, `enqueueSync`, `recordAuditEvent`). Confirm `db.suppliers` and `db.businesses` exist.
- M-Pesa is already flagged comingSoon in AppShell; the nav item type now allows it.
- Receipt popup/print (`ReceiptModal`, `PublicReceipt`) still use their existing layout; templates are applied to the new share sheet, public `/d/` links and the preview.
- Test on a real Android device: share-as-picture uses SVG foreignObject; logos from other domains need CORS, otherwise the logo is omitted from the picture.

---
# Round 3 (soft delete, consent, deposits, search, avatars, categories, admin tabs)

## Already live in Supabase (project ShopOs)
- Migration `phase33`: soft-delete columns on businesses/profiles (+ `auth_business_id()` now ignores deleted businesses, so RLS hides them), `account_deletions` log, `consent_documents` / `account_consents`, `customer_deposits` ledger + `record_customer_deposit()` / `customer_deposit_balance()` (tested: overdraw blocked, other business blocked), unused `product-images` bucket (can be ignored/removed).
- Edge functions: `delete-my-account` v2 (SOFT: bans sign-ins, revokes sessions, keeps data 30 days), `admin-delete-business` v2 (soft by default; `purge:true` only for already-deleted), NEW `admin-restore-business`, `claim-owner-account` v8 (records consent).
- Consent notice v1.0 is seeded with `enforced = false`. **After you wire `<ConsentBox>` into the account-setup screen, turn enforcement on:**
  `update consent_documents set enforced = true where is_current;`
  (Until then nobody is blocked; existing users get a one-time prompt once enforced.) Have a lawyer review the notice text.

## App files
NEW: `components/UniversalSearch.tsx`, `components/layout/UserMenu.tsx`, `components/ProductAvatar.tsx`, `components/DeleteAccountDialog.tsx`, `components/ConsentBox.tsx`, `features/dashboard/QuickActions.tsx`, `features/customers/DepositCard.tsx`, `features/categories/CategoryDetailPage.tsx`
CHANGED: `AppShell.tsx` (search + avatar menu + quick actions on dashboard), `MoreMenu.tsx` (Theme tile + Categories tile removed), `App.tsx` (route /categories/:id, ConsentGate), `InventoryList.tsx` (product pictures, add to category: new + existing), `CustomerDetailPage.tsx` (deposit card), `lib/products.ts`.

## Admin files
`pages/BusinessDetail.tsx` (tabs: Overview, Activity, Users, Access & branches, Consent, Account; soft delete / restore / permanently erase), `pages/Businesses.tsx` (deleted filter + badge).

## Needs you
- Wire `<ConsentBox onChange={(accepted, version) => ...}/>` into the owner account-setup page and send `{ acceptedTerms, termsVersion }` to `claim-owner-account`. I couldn't see that page.
- `DeleteAccountDialog.tsx` REPLACES your existing one (same props). Check it still matches your look.
- Dashboard shortcuts render above the dashboard via AppShell (I couldn't see Dashboard.tsx).
- Deposits are online-only on purpose (to prevent two devices spending the same balance); using a deposit as payment at the POS is not built yet.
- Product pictures are stored as small compressed images on the product (work offline, sync with the product).


---
# Round 4

## Already live in Supabase (migrations phase34, phase35)
- Secure links: every receipt/quotation/invoice link now EXPIRES (7/30/90/365 days, default 30), can be TURNED OFF, counts views, and stays 192-bit unguessable. Links already sent keep working for 90 more days. Pages send `noindex` + no-referrer.
- Templates: 10 layouts (5 new: Minimal, Elegant, Stripe, Corporate, Soft) + a brand colour per business (`businesses.document_accent`).
- `get_shopos_contact()`: ShopOS phone / WhatsApp / email / slogan from `platform_settings` (admin-editable). The slogan is a PLACEHOLDER ("Smart tools for every shop"); change it with:
  `update platform_settings set slogan = 'YOUR SLOGAN' where id = 'default';`
- Unique offline numbers: `reserve_document_numbers()` gives each device its own block of real numbers (tested: blocks never overlap).

## App
NEW: `lib/numberBlocks.ts` (offline-unique numbers, installed in App.tsx), `lib/brand.ts`, `components/PersonAvatar.tsx`, `RecordDebtSheet.tsx`, `AddDepositModal.tsx`, `features/closing/ClosingHistoryPage.tsx` (More > Previous Days), `features/profile/ProfilePage.tsx` (avatar menu > My profile), `features/debts/DepositsTab.tsx`, `features/expenses/ExpensesList.tsx` + `ExpenseDetailPage.tsx`, `features/public/ReviewForm.tsx`.
CHANGED: dashboard shortcuts (3 groups + clickable Recent sales / Low stock / Owed to you), customer page (avatar, quick actions: Record debt, Add deposit, New sale, Call, WhatsApp; tabs), debts list/detail (avatars, timestamps, tabs, Call/WhatsApp), Documents group in More now holds Templates, live search from the first letter, public receipt/quotation/invoice page (Close, Call/WhatsApp shop, Call ShopOS, logo + slogan), share sheet (expiry + Turn off link + Close).

## Needs you
- `ExpensesList.tsx` REPLACES your file (I couldn't see it). Rows open details; the repeat button re-records the same expense for today. No edit/delete: expenses have no delete column on the server.
- Landing page review form: I couldn't see it. Replace its form with `<ReviewForm />` (blank name = Anonymous; the database already does this).
- `/r/` receipt links now use the new public page (the old `PublicReceipt` is no longer routed).
- Dashboard.tsx wasn't available: shortcuts and clickable records are added above it via AppShell. If you send Dashboard.tsx I can make its own tiles clickable and remove duplicates.
- Deposits and Previous Days need a connection.


---
# Round 5

## Fixed in Supabase (live)
- **Deposit error ("Couldn't load the deposit balance")**: the new tables had no SELECT grant for signed-in users. Granted (row-level security still limits everyone to their own business) on customer_deposits, account_consents, account_deletions, consent_documents, document_shares, client_activity_events. Retested: deposit saved, row visible, balance 100.
- ShopOS slogan set to "Run Your Business Smarter" (the wording in your app's title). Change anytime: `update platform_settings set slogan = '...' where id = 'default';`

## App
- Dashboard shortcut buttons + records now sit AFTER the dashboard content (bottom).
- Bottom bar: Debts replaced by Inventory (Home, POS, Customers, Inventory).
- Inventory split into tabs: Products | Overview | Categories | Stock history. Categories tab: tap a category (or the + button) to add existing products to it or create a new product inside it.
- Documents/receipts/quotes/invoices and the customer link page now show the real ShopOS emblem (/logo-emblem.png) + slogan + contact.
- NEW Business Reports (More > Documents > Business Reports): Profit & loss, Sales summary, Expenses, Top customers, Low & out of stock, Expired & expiring, Whole store stock, Debts owed. Time frames: today, yesterday, this/last week, this/last month, this year, custom. Print/PDF, share as picture, CSV. Profit and stock-value reports need cost access.
- Debt page: "Register as customer" for people who owe but aren't customers (also offers "Link to <customer>" when the same phone exists) and links their other unlinked debts.
- Customer page: Call and WhatsApp are now round icons beside the name, same as the debt page.
- DMN Solutions link now https://www.dmnsolutions.co.ke in PublicLayout.tsx. Search your repo for `dmn-solutions.vercel.app`: `usePlatformContact.ts` (which I could not see) defines DMN_URL and may be used elsewhere.

## Round: reports, barcode, end of day, suppliers, notifications, updates

Already live in Supabase (nothing to run): last-active triggers and backfill, 11:59 pm auto end-of-day (pg_cron, Nairobi time), hourly notification generator, release notifications, `barcode_cache`, edge functions `lookup-product` v2 and `github-latest-release`.

App files: `lib/reports.ts` (9 new reports), `features/closing/EndOfDayHub.tsx` (Close today / Calendar / Previous days), `features/suppliers/SupplierListPage.tsx` + `SupplierDetailPage.tsx` (old screen kept at `/suppliers/manage`), `lib/supplierImport.ts` + `features/inventory/SupplierImportSheet.tsx` (CSV/TSV only), `components/layout/NotificationsBell.tsx`, `lib/notificationFeed.ts`, `features/notifications/NotificationsPage.tsx`, `components/layout/MobileBranchChip.tsx`, `components/ShareShopos.tsx`, `components/motion/Motion.tsx` + motion CSS in `index.css`, `vite.config.ts` (build id + version.json), `lib/updates.ts`, `lib/pwaUpdate.ts`, `vite-env.d.ts`.
Admin: `pages/AppReleases.tsx` has the "Publish from GitHub" card.

To finish by hand: put `<ReviewForm />` in the landing review section (the page was not visible to me); optional secret `GITHUB_TOKEN` in Supabase if a repo is private; set the admin repo name in the GitHub card if it is not `Darius2322/shopos-admin`.

## Round 4 (documents, logs, reviews, dark mode)
- Invoice and quotation pages now preview the real document (business name, logo, title, ShopOS footer) in the chosen template. "Download PDF" and "Print" open the print dialog on a clean print layer (`components/PrintPortal.tsx`); choose "Save as PDF". Reports and supplier statements use the same layer.
- Dark mode: light-grey utilities (bg-slate-100 etc.) are themed, so the search, bell, filter chips and review text are readable.
- Sticky bar shows the business name on phones.
- Logs menu: Activity log + Device log (`features/logs`). Owners can sign out unknown devices (Supabase function `revoke_device_session`, column `device_sessions.revoked_at`); the signed-out device logs itself out within about a minute (`lib/deviceGuard.ts`).
- Landing page: Share ShopOS lives in the hamburger menu (and footer), reviews appear on the home page with an "Add a review" button. `features/public/ReviewsPage.tsx` is replaced (no anonymous checkbox).

## Round 5
- Shared document links: a near-white brand colour made the header and Total invisible. Colours too light to read are now ignored (template default used), and the template page refuses them. The public page prints/saves PDF through the print layer (it printed blank before).
- Customers: "From contacts" (multi-select, de-duplicated by phone) and a single-contact button in Add customer. Uses the Android contact picker, or a .vcf file where the picker is missing.
- Install prompt (`components/InstallPrompt.tsx`): shows only when ShopOS is not installed; iPhone gets Add-to-Home-Screen steps. Dismiss hides it for 7 days.
- Update flow: "Update now" shows Updating, restarts on the new version, then shows an animated "Update complete" (`components/UpdateSuccess.tsx`).
- Favicon and home-screen icon are set from the logo files already in /public (logo-emblem.png, shopos-192-v3.png).
- New Terms and Privacy text (`features/public/LegalPages.tsx`) served at /terms and /privacy. Have it reviewed by a lawyer before relying on it.

## Round 6
- Home page: features carousel (3 s, icons + short text) and reviews carousel (3 s), both pause on touch/hover and loop (`components/AutoCarousel.tsx`). "Add a review" opens the form on the home page. Review avatars show two initials; anonymous reviews show a person icon.
- Add customer: "Choose from phone contacts" fills name, phone and email; all stay editable. Email field added.
- Search in the top bar: desktop has a real search field with live dropdown results; phones get a search bar with live results beneath it and a Cancel/clear button.
- Notifications panel has a close (X) button.
- Products: unit is chosen before quantity; quantity and min stock accept decimals (5, 2.5) and show the unit. Category list in Add/Edit product now reads the business's categories (the old query filtered on a boolean index, which IndexedDB cannot match). Barcode lookup appends the pack size to the name, maps bottle/box/pack packaging to a unit, and matches the suggested category to your own categories by name or related words.

## Round 7
- Barcode: lookup-product v3 (sign-in checked inside the function, tries 12/13-digit forms) + direct fallback to Open Food/Products/Beauty Facts from the device.
- Products: photo picker removed; "Size per item" field added (Quantity unchanged: 25 pieces of 500 g salt); starter products get an Autofilled badge, can be removed one by one or all together, and lose the mark when edited (new column products.auto_filled, synced).
- More is now a page (/more) and the bottom bar highlights the section you are in, including pages reached through More.
- Avatar menu has a close button; Customers header no longer has the contacts button (pick from contacts inside Add customer).
- Reviews carousel: fixed touch-hold bug that froze it on phones; carousel on the Reviews page; home carousels are added under the landing page if it does not draw them.

## Round 8
- Reviews page: carousel only (no list, no progress bar). Add product: Quantity (size, becomes part of the name) is separate from Count (stock, drives low-stock).
- Sync: activity-log rows are sent with the signed-in user id (fixes the stuck "row-level security" item). Update now activates the new version and restarts by itself; Install app button in the update banner.
- /legal shows Terms and Privacy in full with logo, slogan, last-updated + viewed time and contact. Customer display has a readable background and a close button.

## Round 9
- Loyalty page rebuilt: Enable/Turn off button (no checkbox), earning rules, and a Members tab listing everyone enrolled with their points (plus quick Register for others).
- Add/Edit product: many more units (packet, sachet, crate, tray, bag, sack, bundle, roll, tonne, gallon, yard...) and more size units (mg, oz, lb, m, cm).
- Inventory: new Bundles tab (collections such as 12 x 500 g flour, how many bundles; open into stock) and Batches tab; each product page also has its own Batches list (date received, count, expiry, cost; optionally adds the count to stock). New Supabase tables product_batches and product_bundles (synced, per-business RLS); local database version 31.

## Round 10
- Sync: pulling no longer stops at the first table that fails (the error now names the table); payment matching can't block syncing; real error messages instead of "Sync failed".
- Batches: automatic numbers (B-YYMMDD-n), several products per batch, tap a batch for details (products, expiry, cost) with Download (CSV), Add product, Delete. Bundles: whole row opens details with actions.
- Inventory Overview: last 7 days, batches/bundles, catalogue health, stock by category, restocking list, most valuable stock.

## Round 11
- Sync: the "permission denied for table product_batches / product_bundles" error was missing database grants on the two new tables; fixed live in Supabase (no app change needed).
- Menu "Profit & Loss" renamed "Analytics". New Analytics tabs: Profit & Loss (this period vs the last one: comparison table, bar chart, daily profit line graph, revenue and expense pies) and Insights (best weekdays and hours, payment-method and category pies, top products).
- Batches: Download now opens a proper ShopOS document (business header, ShopOS logo/slogan/contact, products table, totals, timestamp) with Print / PDF and Picture, instead of a CSV.
- Legal page redesigned (hero header per document, numbered section cards, tabs that follow your scroll); content unchanged.
- Update banner: "Install app" button when the app is not installed (iPhone shows the Add to Home Screen steps) and "Open in app" when you are in a browser tab with the app installed. The manifest now declares the app so Chrome can tell it is installed.

## Round 12
- Legal: back to the earlier look; Terms & Conditions and Privacy Policy are now two tabs under Legal (one shows at a time; /privacy and /terms open the right tab).
- Update now: the app now takes over the new version, clears the old cached files and restarts itself; if it ever comes back still on the old version it resets itself again on start. The service worker also claims open pages (clientsClaim).
- App Updates page: new "ShopOS app" card with Install app (iPhone shows Add to Home Screen steps) or Open in app.
- Analytics > Profit & Loss: shows one day (today by default) with day arrows and a date box; switch on "Compare with another day" for a side-by-side table, bars and an hourly line graph.
- New More > Guide (route /guide): ~45 topics with steps, examples, tips and drawn illustrations, section chips, and a search over every word with highlighting.

## Round 13 — Desktop app
- New `desktop/` folder: ShopOS Desktop (a locked-down Electron window around the live web app; sandboxed, navigation limited to ShopOS, camera/notifications/copy only, dev tools off, single instance, offline page). `desktop/desktop-build.yml` (copy it to `.github/workflows/` on the GitHub website, because a command-line token without the `workflow` permission cannot push that folder) builds the Windows .exe, Mac .dmg and Linux AppImage on GitHub (Actions → "Build desktop installers" → Run workflow). Installers are unsigned (see desktop/README.md).
- More → Desktop app (/desktop): owner asks for the setup file per computer type → ShopOS admin approves (time-limited) → owner downloads through a private 2-minute link. "Continue with the web app" is always offered.
- More → Android app: shown as "SOON" (route /android has the coming-soon page).
- Guide: new topics "The desktop app" and "Android app".
- Supabase (live): private bucket desktop-installers, tables desktop_releases and desktop_setup_requests (RLS, one open request per shop and computer type), functions request_desktop_setup and admin_decide_desktop_request, edge function desktop-setup-link (owner + approved + unexpired + published, 2-minute signed link, counts downloads).

## Round 14 — Link a device + How to register
- Sign in a second device without a password, like WhatsApp Web: sign-in page → "Link with a phone (scan a code)" shows a QR code and an 8-character code (2 minutes, one use). On a signed-in device open More → Link a device, scan or type the code, check the device name, tap Approve. The new device signs in by itself and the link is recorded in the security log.
- Security: the new device's secret never leaves it (only its hash is stored); the one-time sign-in token is handed out once; only an active member of an active business can approve; creating codes is rate-limited. Supabase (live): table device_link_requests, edge function device-link. QR codes are drawn by our own generator (src/lib/qr.ts), no outside service. Camera scanning needs Chrome on Android; elsewhere type the code.
- Landing page: new "How to register" tab (/how-to-register): 6 animated, drawn steps (auto-playing, pausable, respects reduced-motion) from Get started to first sign-in.
- Guide: new topic "Link a computer or phone with a code".

## Round 15
- **New sign-in screen**: calmer, modern layout (brand panel on computers, clean card on phones). Every earlier button is kept: Sign in, Forgot password, biometrics, Link with a phone, Register, Check status, Legal links, support contact, theme switch. New: **Try the demo**.
- **Demo mode** (`src/lib/demo.ts`, `demoSeed.ts`): a sample shop ("Demo Duka": 24 products, customers, 14 days of sales, debts, expenses) on its own separate browser database (`shopos-demo`). It never talks to the server (sync, activity, device log and number reservation are all off), never mixes with a real shop on the device, and disappears when the app closes (flag in sessionStorage, database deleted at next start, 3-hour limit). An amber "Demo" tag with Leave is on every screen. Switched off with a clear note ("works in a real account"): Security / change password, Delete account, Staff accounts, Link a device, Desktop setup, Support, Sync, Backup.
- **How to register** is now 8 animated steps with a checklist, a status explainer and FAQ. New **How to install** (`/how-to-install`, tabs: Android, iPhone, computer browser, desktop program, POS terminals by USB). Both live under a "Guides" tab in the landing menu.
- **Desktop installers for every till**: Windows 64-bit, 32-bit and ARM; Mac (Intel + Apple); Linux x64 and ARM. Owners with an approval see one download button per version and can download again for each terminal until the approval ends.
- **Backup & restore** (More → Backup & restore, owner only): the shop's records are collected on the device, compressed and locked with the owner's password (AES-256-GCM, PBKDF2), then saved to the owner's own Google Drive ("ShopOS Backups" folder, narrowest `drive.file` permission) or as a file / via the share sheet. Restore: pick a Drive backup or file, type the password, choose "Fill in what is missing" or "Put everything back", confirm. Nothing is deleted; restored records re-sync. **To switch Google Drive on**: Google Cloud Console → create a project → enable "Google Drive API" → OAuth consent screen → create an OAuth client ID (Web) with authorised JavaScript origin `https://shopos-app.vercel.app` → add its ID in Vercel as `VITE_GOOGLE_CLIENT_ID` and redeploy. Until then the file backup works. If a Content-Security-Policy is ever added, allow `accounts.google.com` and `www.googleapis.com`.
- Guide: new topics "Try the demo", "Link a computer or phone with a code", "Backup and restore".

---
# Round 16 (owner downloads, visitor analytics, logo, business type, receipt formats, numbering)

## Already live in Supabase (project frhhldkotckfexsegtjx)
- Migrations `round16_business_type_numbering_branding` and `round16_site_analytics_and_installer_ingest`.
- `businesses.business_type`, `number_formats`, `receipt_settings`; trigger `guard_business_branding` (only owner / business.settings may change them; a logo must come from our own `business-logos` bucket).
- Public bucket `business-logos` (1 MB, PNG/JPG/WebP, writes only into the caller's own business folder).
- `format_document_number`, `reserve_document_numbers_v2`, `next_document_number_v2` (old function names still work and follow the same format). Tested: default format unchanged (`REC-2026-00042`, `QT-2026-00032`); custom `BAR/MAI/26/10/0100` with monthly reset.
- `site_visits` + admin-only `admin_site_overview / timeseries / breakdown / hourly / recent / funnel`, `admin_business_types` (tested with synthetic rows, then rolled back).
- Edge functions: `desktop-setup-link` v3 (owners download directly, no approval), `desktop-ingest` v1 (GitHub build publishes installers), `track` v1 (anonymous visit counter).
- `desktop_downloads` log; `ci_ingest_tokens` (hash only).

## App (`src/`)
- `features/desktop/DesktopSetupPage.tsx`: no request step; pick computer type and download.
- `lib/siteAnalytics.ts` (+ `main.tsx`, `lib/demo.ts`): anonymous visit counter, respects Do Not Track, off in demo.
- `lib/logo.ts`, `features/branding/BrandingPage.tsx` (More → Branding & receipts): logo upload, business type, receipt format with live preview and test print, number formats with live preview.
- `lib/businessTypes.ts`, `lib/businessTypeActions.ts`, `components/BusinessTypePicker.tsx`, `features/onboarding/BusinessTypeStep.tsx`: first step of new-business setup; existing shops are asked once a day until they choose.
- `lib/receiptFormats.ts`, `components/ReceiptSheet.tsx`, `lib/printReceipt.ts`, `components/PrintPortal.tsx`: 80 mm / 58 mm thermal, A6, A5, half letter, A4, Letter; compact / standard / detailed; 1-3 copies.
- `lib/numberFormat.ts`, `lib/numberBlocks.ts`, `lib/sync.ts`: numbers follow each business's format (per branch when the branch code is included).
- `features/sales/SaleDetailPage.tsx`: "Print on <paper>" button.
- `desktop/publish-installers.mjs`, `desktop/desktop-build.yml`: the GitHub build publishes installers into the admin library.

## One-time setup for installer publishing
GitHub repo (shopos-app) → Settings → Secrets and variables → Actions → New repository secret: name `SHOPOS_INGEST_TOKEN`, value = the token given in chat. Then Actions → Build desktop installers → Run workflow.

## Not verified / needs you
- Nothing here was run in a browser or compiled with the project's dependencies (syntax-checked only). Run the Vercel build and open Branding & receipts once.
- `features/pos/ReceiptModal` and the existing "Receipts & Printer" page were not available, so the POS "sale complete" print still uses them. The new formats are used by Sales → sale → "Print on …" and the test print.
- The Supabase free plan caps single uploads at 50 MB; installers are larger. If the build's publish step fails with a size error, raise the limit in Supabase → Storage → Settings.

## Round 19
- **Packages**: admin creates packages (name, cost, what it includes, validity, details, services); owners see Current plan, History and Upgrade/Renew under More → Package; expiry notices; expired plans freeze writes in the database; open-app offers for a period. Public Packages page on the landing page (menu and phone hamburger).
- **Sign-in**: related buttons grouped into two menus (New to ShopOS, Link & restore); Google account restore; password always required after a restore.
- **Auth panel**: logo and slogan drawn straight on the blue as a white silhouette (no white box), subtle motion (off with reduced-motion).
- **Security** has submenus: Password & sign-in, Devices, Delete account.
- **Linked devices**: Link a device → Linked devices tab (date added, last active, sign out). Max 4 linked devices per account, enforced in the database (`device_link_limit` trigger, `my_device_slots()`).
- **External devices** (More → External devices): barcode scanner test, camera presence, fingerprint guidance, serial/USB/HID permissions (Chrome/Edge).

## Round 22
- Backend done and DB-tested: branch limits, requisitions, stock transfers, POS sessions, price rules, business-type themes, release-note sanitising. See admin `supabase/round20/NOTES.md`.
- App: business-type theme (`lib/businessTheme.ts`), `BusinessTypePicker`, release notes component with security filtering.
- Android: `android-wrapper/` + `.github/workflows/android-build.yml`. **Not built, not tested** - read `android-wrapper/README.md` (needs your live URL and signing secrets in GitHub).
- Still needs files from you: POS/sales screens (discount & override UI, bundles, scrollable mobile cart), `index.html`/logo files (WhatsApp preview, favicon, PWA icons), requisition/transfer/POS-session screens, product min-price field, public document link check.
- Nothing in this round was compiled with the project's dependencies (package install was unavailable here); run the Vercel build and fix any type errors it reports.

## Round 23
- **Mobile apps** (More → Mobile apps, replaces "Android app: coming soon"): Android app download (owner only, short-lived link) and iPhone/iPad (TestFlight/App Store link when published; otherwise Safari Add to Home Screen steps). Build and publish steps: `android-wrapper/README.md` (also `MOBILE_APPS_GUIDE.md`). Workflows: `.github/workflows/android-build.yml`, `ios-build.yml`. Not built or tested yet.
- **Theme & look** (Business Profile → Theme & look, or More → Theme & look): owner picks a full look (colour, corners, spacing, lettering, cards), "Match my business" uses the look for the business type, with fine-tuning. Saved on the business, so web, desktop and phone apps match. Admin still sets each type's colour in Business types.
- Admin → Installers now handles Android and iPhone/iPad.

## Round 23b
- One menu **More → Apps & updates** with tabs: App updates, Desktop app, Android & iPhone (old links /updates, /desktop, /mobile-apps, /android open the right tab). Admin: **Apps & Updates** menu with sub-menus App Updates and Installers (desktop, Android APK, iPhone link).
- Desktop build fixed: removed the unsupported `nsis.buildUniversalInstaller` option.
- Still waiting for the POS/sales source files for: discounts and price overrides, scrollable mobile cart, sellable bundles.
