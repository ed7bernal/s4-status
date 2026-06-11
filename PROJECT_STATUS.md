# Source S4 — Product Status
*Last updated: 2026-06-11 (Needs Information report rework shipped to production)*

---

## What's live in production

- **Invoice Processing** — PDF upload → OCR → LLM extraction → vendor match → account match → service match → saved to DB with PDF stored securely.
- **Contract Processing** — PDF upload → OCR → LLM extraction → vendor match → services created → PDF stored. Now derives `annual_value` automatically from extracted contract value.
- **Vendor Match Resolution** — Confirm vendor, create new, or link invoice to contract manually. Vendor backfill: approving a contract auto-links all invoices with matching vendor name.
- **Reconciliation** — Matches processed invoices against expected charges.
- **Service Match** — Every invoice auto-scored and linked to best matching service after processing.
- **PDF Storage** — All PDFs in private Supabase Storage bucket, secure signed URLs on each record.
- **Inventory Upload (batch pipeline)** — Full batch pipeline live. Contract+invoice same-batch matching works end-to-end.
- **Billing accounts** — Accounts flagged as billing accounts. Allocation rows show billing account dropdown. Inline account creation. Fixed: only accounts belonging to the contract's vendor are shown.
- **Contract detail editing** — All contract fields editable inline. Vendor name edit is smart: updates the vendor entity when a vendor is confirmed, updates the raw text field when still unmatched.
- **Allocation user management** — Multi-select modal: add multiple users in one batch with evenly redistributed allocations. "Select all" support. Bulk change billing account and billing dates.
- **Vendor name normalization** — Handles `&`/`and`, strips legal suffixes.
- **PDF viewer** — Split-pane viewer in contract and invoice detail. Now correctly loads PDFs for both batch-uploaded and individually-uploaded files.
- **Invoice detail** — Subtotal and Tax visible and editable alongside Total, with currency formatting.
- **Supplemental documents on contracts** — Documents tab in contract detail. Upload Terms & Conditions, MSA, Schedules, Exhibits, Addendums.
- **GAP 1 — HR/Users CSV upload** — Upload HR CSV monthly → upserts `org_users` with cost_center, building, job_category, investment_strategy.
- **GAP 2 — Auto-advance billing period** — Closing a period automatically creates and activates the next month.
- **GAP 3 — FX/Currencies** — `exchange_rates` table + `user_cost_usd` in snapshots. Exchange rates entered monthly in Periods UI.
- **GAP 4 — Bulk subscription update** — Multi-select allocations → change billing account or billing dates for multiple users at once.
- **GAP 5 — Snapshot enrichment** — `cost_center` and `building` from `org_users` populated into snapshots at period close.
- **Dashboard redesign** — Full dashboard with KPI cards, renewals bar chart, Needs Attention panel, Auto-Renewals table, Top Vendors chart. All data scoped to org.
- **Sidebar navigation** — Dashboard as primary nav item. Processing tools in a TOOLS section, visible only when modules are enabled.
- **Reports section** — Needs Information (formerly "External Documents Required"), Renewal Calendar, Period Snapshots with cost columns.
- **Supplemental document flags auto-clear** — Typing in a contract value that was flagged as missing auto-marks it resolved.
- **Full-width layout** — All inventory pages use full available width.
- **Invoice delete** — Users can delete a wrongly-assigned invoice and re-upload it to the correct contract.
- **Billing period snapshot cost fix** — Avg Monthly uses `unit_cost ?? annual_value/12` so user_cost is never null when annual_value is set.
- **Rate-limit retry hardening** — Batch upload dispatch retries on 429s and thrown rate-limit exceptions with jitter.
- **linked_contract_id fix** — Invoice items in a batch now correctly link to their matched contract after reconciliation.
- **Batch-scoped contract matching** — Invoices only match contracts from the same batch. No cross-batch false positives.
- **Security hardening (P1–P11)** — Internal fields stripped from API responses, legacy RLS policies removed, UUID validation added to Edge Functions, org_id indexes added, audit trail hardened, client_modules RLS fixed.
- **Multi-org support** — A user can belong to multiple organizations and switch active client without re-login. Single-org users see no change. See Recent Changes for details.
- **Expense Type dropdown** — Product Catalog "Expense Type" field is now a fixed-option dropdown (Market Data, Research, Technology, Trade Execution) instead of free text.
- **Upload Results cards collapsed by default** — On entering Upload Results, contract cards now show collapsed (name, service, dates, score) by default, reducing clutter for large batches.
- **Inventory Active/Cancelled/All filter** — Main Inventory list now has a pill filter (Active · Cancelled · All) with counts; Cancelled contracts no longer clutter the default Active view but remain reachable.
- **Contract Invoice tab fixed** — A contract's "Invoice" tab now shows only the invoice(s) actually linked to that contract, not every invoice from the same upload batch. Backfilled `linked_contract_id` for older batches on staging and production.
- **Upload Results unified processing view** (R-004) — During batch processing, all items show a consistent "Processing N documents" state; final per-item statuses (matched, no vendor match, etc.) only appear once the whole batch completes.
- **Confirm with outstanding issues** (R-005) — "Needs Attention" items now have a Confirm action (previously only "Looks Good"/"Needs Review"). Confirming a "Needs Review"/"Needs Attention" item shows a warning dialog listing unresolved issues; the contract carries a "Needs Review"/"Needs Attention" badge in the Inventory vendor list afterward.
- **Delete action on Upload Results** (R-008) — Items not yet approved (status != Active, no review_status, no allocations/snapshots) can be deleted from the review page via `delete_inventory_review_item` RPC, removing the contract and its linked invoice(s). Hidden once a contract is Approved.
- **Review-pending indicator on Recent Uploads** (R-014/R-015) — The Recent Uploads table has separate "Status" and "Review" columns; "Review" shows "Review pending (N)" or "Fully reviewed" based on how many contract items in the batch are still not Active.
- **Inventory Users — explicit Active/Inactive action** (R-001) — Row-level action plus bulk multi-select to set users active/inactive (replaces the old inline toggle).
- **Exchange Rates — dynamic per-org currency list** (R-002) — Periods > Exchange Rates now supports adding new currencies (code + initial rate) and disabling/hiding currencies an org doesn't use, plus FX carry-forward on period close.
- **"Needs Information" report rework** — Renamed from "External Documents Required" and broadened to also surface approved contracts flagged `needs_review`/`needs_attention` (not just ones missing supplemental docs). New pill filters: Not Approved · Needs Review · Needs Attention · Resolved · All. Added a "Mark as resolved" action on report cards that clears `review_status`; contracts with nothing left to track drop off the report entirely.

---

## What's in staging (not yet in production)

### CDR — Test client for CDNR demo prep
- User: `edbernal@cdr.com` / password: `12345`
- Org: CDR (`b986d4d7-ca78-4326-998e-56682352b0e2`) + HIG Testing (`eb63c19f`)
- This user has 2 orgs and sees the client selection screen on login
- Full E2E multi-org flow validated: select client → data isolation confirmed → switch client works

### CDNR — First real client POC
- New small client (~50–75 contracts), inventory starts from June 2026 (no historical data needed)
- Demo with Stephanie de Lucía completed 2026-06-10 (Santiago, Edgar, Bernardo attending)
- Follow-up feedback now flows through `/app-review` sessions into `PRODUCT_REVIEW_BACKLOG.md`

### E2E Demo Dataset (HIG Testing org — staging only)
- Org: `eb63c19f-a8dd-4f28-8638-b8c522fe4e18`
- 15 contracts (Draft, unmatched) + 24 invoices (pending) ready to approve from scratch
- 15 org_users from Senthio HR data, cost_center and building populated
- HR CSV at `~/Downloads/hig-test-hr.csv`

---

## Inventory Module — Senthio Feature Parity

### Goal
Replicate Senthio (H.I.G. Capital's Access DB) in Source so HIG can stop using Access. PoC target: **end of June 2026** — Ricky (internal user) using it for real feedback.

### Senthio reference
Full documentation: `.claude/skills/senthio-reference.md` (all 19 tables + queries + month-end close workflow)

### GAP status
| GAP | Spec | Status |
|---|---|---|
| GAP 1 | `gap1-hr-users-csv.md` | ✅ Production |
| GAP 2 | `gap2-auto-advance-period.md` | ✅ Production |
| GAP 3 | `gap3-fx-currencies.md` | ✅ Production |
| GAP 4 | `gap4-billing-account-bulk-update.md` | ✅ Production |
| GAP 5 | `gap5-snapshot-enrichment.md` | ✅ Production |

### Next to build
1. **Missing invoices view** — services with active subscriptions but no invoice in current period
2. **Cost per user view** — allocation_pct × invoice.total per user/service/period
3. **Vendor grouping** — group unmatched vendors by name when a new client uploads their first batch
4. **Bloomberg Terminal Upload (structured CSV import)** — separate upload type from core subscription PDFs
5. **Spend/Inventory report** — monthly spend view: vendor → contracts → services → users/departments
6. **Bloomberg Terminal Reconciliation** — compare Bloomberg inventory files vs Source

---

## What's in development

- **Vendor grouping for new clients** — Spec/prompt ready, implementation pending.
- **Monthly pricing normalization (P1)** — When contract states prices monthly, AI uses monthly price as annual value. Fix pending Santiago review.
- **Service split/merge UI** — Deferred. Waiting for Santiago validation.
- **`link_contract` UI** — Backend action live, UI trigger not built.
- **CBInsights/ProSights Dice coefficient false positive** — "sights" suffix causes ~0.48 score (above 0.30 threshold), causing an invoice to link to the wrong contract. Tracked as `R-017` in `PRODUCT_REVIEW_BACKLOG.md`. Threshold fix and one-off data correction deferred.
- **Not a match button** — Hidden for now. Flow and requirements need validation before re-enabling.

---

## Coming next (priority order for CDNR demo)

1. Missing invoices view (services without invoice in current period)
2. Cost per user view
3. Renewal alerts (contracts approaching cancel_lead_time_days)
4. Bloomberg Terminal Upload (CSV-based, after HR file already live)
5. Spend/Inventory report with adjustments + forecast
6. Bloomberg Terminal Reconciliation (review with Santi)
7. Monthly pricing fix — after Santiago review

---

## Technical debt

- **Batch upload scalability** — In-process polling + sequential dispatch breaks at ~50 docs. See `docs/tech-debt/batch-upload-scalability.md`
- **Service split/merge UI** — See `docs/tech-debt/service-split-merge.md`
- **Invoice variance/adjustments** — When invoice ≠ inventory expected amount, need adjustment records (Senthio: `Invoices_Adj`). Deferred post-MVP.
- **Soft/Hard dollar classification** — Senthio tracks Hard$ vs Soft$ per user. Deferred.
- **GL Accounts** — Senthio routes expenses to GL accounts via AccountMaps. Deferred.
- **Multi-year contract pricing** — `service.annual_value` is set at Year 1 price. Full fix requires `ServicesFP` equivalent. See `docs/tech-debt/service-pricing-schedule.md`
- **Allocations inline editing** — Custom onBlur pattern still in place in `InventoryContractDetail.tsx` ~line 1898. Should use `InlineEditableField`.
- **Pre-existing TypeScript errors in InventoryUploadDetail.tsx** — `contract_id` not in `ContractData` type (line 849+), several unused state vars. Not causing runtime issues.
- **HIG Testing exchange rates** — Only June 2026 rates configured. Need EUR/GBP rates for July 2026+ before next close.
- **Admin invite-user-by-email flow** — Approved Gate 1 plan, deferred until onboarding volume justifies it. See `docs/tech-debt/admin-invite-user-flow.md`.

---

## Environments

| | Supabase ref | Frontend |
|---|---|---|
| Production | `fdcxcivjhobreuseacot` | https://s4source.io |
| Staging | `fntpcrpmkwyruzplbewq` | https://s4sourceio.lovable.app |

---

## Recent changes (2026-06-11 session — Needs Information report rework shipped to production)

- **Reports tab "External Documents Required" renamed to "Needs Information"** and broadened: it now shows any contract that's missing something — either a required supplemental document, or an approved contract still flagged "Needs Review" / "Needs Attention". Contracts that are fully complete no longer appear in this report at all.
- **New filter pills**: Not Approved · Needs Review · Needs Attention · Resolved · All, each with a live count.
- **New "Mark as resolved" button** directly on each report card — lets the user clear a "Needs Review"/"Needs Attention" flag once they've fixed the issue, without confusing buttons on the contract detail page itself. A short note explains the contract will drop off the report once nothing is left to track.
- Validated on staging (CDR account) and confirmed live in production.

---

## Recent changes (2026-06-11 session — Phase 3 (R-011) Pricing Model shipped to production)

- **Pricing model field (Per User vs Shared)** for contract services is now live: a "Pricing Model" dropdown in Product Catalog (with a short explanation), and an editable "Shared"/"Per User" badge on each service in the Allocations tab.
- **Calculation change**: closing a billing period now computes each user's cost based on the service's pricing model — `Shared` (default, unchanged behavior) splits the monthly cost by allocation %; `Per User` charges each allocated user the full monthly cost regardless of allocation %.
- No retroactive recalculation — only periods closed from now on use the new logic; previously-closed snapshots are untouched.
- Code review and full review-agent checklist (security, data architecture, code quality, performance) passed clean before deploy.
- Validated end-to-end on staging and again on production (closed a real period, confirmed both Shared-split and Per User-full-cost rows in the snapshot, including FX conversion).
- Phase 3 of the `/app-review` action plan is now closed (R-011 marked Done).

---

## Recent changes (2026-06-11 session — CDR user accounts + invite-by-email plan)

- **New CDR users added to production**: Bernardo Santiago and Trey Guevara now have accounts and can log in to https://s4source.io. Stephanie de Lucía's password was also reset. All three were sent the same temporary password by email and can change it themselves from Profile Settings.
- **Looked at how new users are added today** (currently a manual, by-hand process) and put together a plan for a proper "invite by email" flow — admin sends an invite, the new user gets an email, clicks a link, sets their own password, and is in. Decided to **defer building this for now** since it's just a handful of internal CDR users during this proof-of-concept phase. The full plan is written up and ready to pick up later (`docs/tech-debt/admin-invite-user-flow.md`).
- No code changes shipped to staging or production this session — only account setup (production database) and documentation.

---

## Recent changes (2026-06-11 session — Phases 1 + 2 shipped to production)

- **Upload Results review flow improvements** are now live for everyone: a clearer "still processing" view while a batch is running, the ability to confirm an item even when it has outstanding issues (with a warning so nothing gets missed), the ability to delete an uploaded item that hasn't been approved yet, and clear indicators on the uploads list showing which batches still need review.
- **User management improvements**: org admins can now mark users active/inactive individually or in bulk from the Users page.
- **Currency settings improvements**: orgs can now add new currencies and hide ones they don't use in Periods > Exchange Rates.
- Before going live, a security review caught and fixed an issue where the new "delete uploaded item" feature could have been pointed at another organization's data — fixed before production deploy.
- Edgar reviewed and validated all of the above directly on the live site.
- Next up: a pricing model option (Per User vs Shared) for contract services, requested by Santiago — planning is done, implementation starts in a future session.

---

## Recent changes (2026-06-10 session — Phase 0 of action plan shipped)

- **Phase 0 (4 quick wins) completed and deployed to staging and production**, validated by Edgar at each step:
  - **Expense Type dropdown** (`R-010`) — fixed-option dropdown instead of free text in Product Catalog.
  - **Upload Results cards collapsed by default** (`R-016`) — less clutter when reviewing a batch.
  - **Inventory Active/Cancelled/All filter** (`R-013`) — Cancelled contracts hidden from the default view but available via filter, plus a new "Cancelled contracts" view.
  - **Contract Invoice tab fix** (`R-006`) — a contract's Invoice tab now shows only its own linked invoice(s), not the whole batch. Includes a one-time data backfill on staging and production.
- **New issue found and logged during validation (`R-017`)** — while validating the Invoice tab fix on the CDR staging batch, found one invoice ("CB Insights") pointing to the wrong contract ("ProSights Labs"), caused by a pre-existing vendor-name matching bug (Dice coefficient false positive on shared substrings). Not caused by today's changes — logged for a future fix, data not corrected yet.
- **New design principle**: future UI proposals must reuse existing components/patterns already in the app (e.g. the pill-button filter style) rather than introducing new designs.
- Phase 1 of the action plan (Upload Results review flow) is queued for a separate session.

---

## Recent changes (2026-06-10 session — first product review + action plan)

- **First `/app-review` session completed** — Edgar walked through Inventory Users, Periods/Exchange Rates, Inventory Upload, and Contract detail (plus feedback from a manager call with Santiago) and logged 16 pieces of feedback (`R-001`–`R-016`) in `PRODUCT_REVIEW_BACKLOG.md` — covering things like deleting users, adding new currencies, reopening a closed billing period, fixing a bug where invoices show on the wrong contract after a batch upload, and adding a "Per User vs. Shared" pricing option for services.
- **New: `/triage-backlog` command** — A follow-up session type to score backlog items by effort/impact/risk and group them into "quick win / big bet / fill-in" buckets.
- **Action plan with 6 phases** — The 16 items were grouped into phases by shared screens/dependencies (not just priority), each phase to be worked in its own chat session with: an upfront plan Edgar approves, validation after each individual fix, and a code-quality review before deploying. Deploys happen at the end of each phase, staging first as always. Phase 0 (quick wins) is next.

---

## Recent changes (2026-06-10 session — multi-org cleanup, environment health check, new feedback process)

- **Multi-org support fully verified** — Consultants who work with more than one client can switch clients and now have EVERY feature — including batch document uploads and the monthly HR file import — correctly scoped to whichever client is currently selected, not just their home client. Verified live in production with a real multi-client user.
- **Staging vs. production health check** — Compared every database table, security rule, and backend function between the test environment and the live environment. Everything matches, with two small internal-only items noted for later cleanup (no user impact).
- **CDNR demo completed** — Demo with Stephanie de Lucía took place today. Follow-up items will flow into the new review process below.
- **New: structured product review process** — Added an `/app-review` session. Edgar walks through the app and gives feedback out loud; each item is logged as a clear, prioritized to-do in `PRODUCT_REVIEW_BACKLOG.md`, ready to be picked up and worked on safely one at a time.

---

## Recent changes (2026-06-09 session — multi-org support)

**Multi-org support — deployed to staging and production:**

- **Client switching without re-login** — A user can now belong to multiple organizations (e.g. Edgar belongs to both CDR and HIG Testing in staging). On login, users with multiple orgs see a client selection screen. Users with a single org go directly to the dashboard — no change in their experience.
- **Switch client button** — Appears in the sidebar only for multi-org users. Returns to the client selection screen instantly without logging out.
- **Data isolation verified** — Switching clients correctly scopes all data (contracts, invoices, vendors, billing periods, snapshots) to the selected org. Deep DB validation confirmed zero cross-org contamination.
- **Database** — New `user_org_memberships` table (user_id + org_id + role). Backfill ran automatically for all existing users. Index on user_id keeps RLS queries fast.
- **RLS migration** — 48 access policies across 16 tables updated to check org membership via the new memberships table instead of the profiles table.
- **Edge Functions** — 7 functions updated to read the active org from a request header (`X-Active-Org-Id`) and validate the user's membership before processing. Backward-compatible: single-org users without the header fall back to the old behavior.
- **Frontend** — 22 files updated. All direct `profiles.org_id` fetches removed. Every Edge Function call now sends the active org header. New `SelectOrg` page and routing logic added.
