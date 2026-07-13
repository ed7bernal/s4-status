# Source S4 — Product Status
*Last updated: 2026-07-13 (Reports module — Overview + Vendors/Users sheets — deployed to production, closes R-030)*

---

## What's live in production

- **Reports module — Overview + Vendors/Users sheets, deployed 2026-07-13 (R-030 done, R-031 phase 1+2)** — new `report_spend_aggregate` RPC (generic, whitelisted `group_by`/`split_by`/`filter_by` over `inventory_period_snapshots`) backs three new tabs in Reports: **Overview** (default tab — KPI tiles, monthly spend chart with a Split-by selector, a department-composition donut, Top 10 Vendors/Departments, range-scoped CSV export), and **Vendors**/**Users** (sortable tables with a Share bar + %, expandable rows showing two side-by-side breakdowns each — lazy-loaded, cached per range — with per-entity CSV export). A shared From/To period-range picker persists across all three sheets. Modeled on a reference BI dashboard Santiago shared, adapted to available data (no FO/MBO field, no PM Teams/Cost Savings concept) — explicitly deferred: Bloomberg Terminal deep-dive (Recon not settled), custom widgets, Ask AI (both lowest priority per Santiago's own words, his reference dashboard is fully static). Validated on staging (HIG Testing) and smoke-tested on production with real data (drill-down filter reconciles exactly). See `PRODUCT_REVIEW_BACKLOG.md` R-030/R-031.
- **Billing Module — deployed 2026-07-12, a full week of accumulated backend work in one batch** (11 migrations, `20260706000001` through `20260710000002` — none of these had reached production before this deploy, staging-only until now): month-count fix v8 (a contract billing period ending on an earlier day-of-month than it starts was counting one extra month — **production audit run post-deploy, zero affected approved invoices found**, no data fix needed); `invoice_services` junction table + v9 RPCs (one invoice can now cover more than one service); `close_billing_period` matching + adjustment-folding fixes (v2); RLS `WITH CHECK` hardening (A54, blocks cross-org `org_id` reassignment on UPDATE); IRW v10 (billing account shown per row) and v11/R-026 (billing account as an invoice-level cross-contract scoping parameter — `invoices.billing_account_id`, eligible services resolved as `invoice_services` ∪ subscriptions sharing that account, across any contract); `invoice_adjustments.tax` fix (was captured but silently ignored in variance/distribution calc); **R-025 — `get_billing_invoices` RPC** (standalone Billing table, cross-contract by vendor/account, computed `reconciled`/`allocated`). **R-026 is now live in production but has not yet been shown to Santiago himself** (he proposed the shape but hasn't validated this implementation — he's back 2026-07-20). See `PRODUCT_REVIEW_BACKLOG.md` R-022–R-026, R-025 for full detail on each.
- **Billing Module — P2.4 (IRW Dataset 1 — historical snapshots)** — `get_invoice_reconciliation` now includes a UNION with `inventory_period_snapshots` (Dataset 1) for months already closed, plus a NOT EXISTS guard on `service_subscriptions` (Dataset 2) to prevent duplication. Invoices spanning closed periods now show frozen snapshot values for archived months and live subscription values for current months. `create_invoice_distributions` also updated with the same UNION pattern so distribution amounts match what the user approved. Fixes: billing_period_id fallback for invoices with no explicit billing dates; ROUND(expected_subtotal, 2) in Dataset 1; GRANT EXECUTE added. Validated on staging and deployed to production.
- **Billing Module — Grupo B (Adjustments + Approve + Distributions)** — Full invoice approval flow live. Users can create adjustments to close the variance (tab "Invoice Adjustments" alongside the IRW), approve the invoice once `net_variance ≈ 0`, and the system generates per-user distributions (`invoice_distributions` table — equivalent of Senthio's `Invoices_Adj`). `get_invoice_reconciliation` v6 returns `adjustments_total` and `net_variance`. `create_invoice_distributions` RPC is idempotent and generates rows from `service_subscriptions` (one per month × user over billing range) plus `invoice_adjustments` rows. Validated end-to-end: Tax Analysts invoice 50217 ($8,751.60), adjustment $1,037.85, approved — 34 distribution rows generated, SUM = $8,751.60.
- **Billing Module — Grupo A (IRW)** — Invoice Reconciliation Worksheet live in production. Given an invoice matched to a service, the system generates a breakdown per month × user (via `get_invoice_reconciliation` RPC), showing Expected vs Invoice vs Variance. IRW panel appears in InvoiceDetail, InventoryContractDetail, and InventoryVendorDetail. Gated by `client_modules.module = 'inventory'` per org. Org isolation bug fixed: batch uploads now correctly scope to the active client in all modes (new batch, append, retry). `invoices.status` now supports `'approved'`; `invoices.approved_at` column added.
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
- **Delete/re-upload active contract (R-009)** — From a contract's detail page, admins can delete (or archive, if it has allocations/history) the existing contract and re-upload a new one from Inventory, linked to the same vendor. `process-contract` now stores the PDF and defaults new contracts to `Draft` status, matching the batch pipeline. `delete_active_contract` and `delete_inventory_review_item` RPCs are now admin-only, matching the table's RLS delete policy.
- **Orphan-invoice contract matching threshold raised (R-017)** — `reconcile-inventory-batch`'s `planOrphanInvoiceLinks` threshold raised from 0.30 to 0.50, fixing a Dice-coefficient false positive (e.g. "CBINSIGHTS" incorrectly linking to a "ProSights Labs" contract on shared "sights" substring).
- **Upload Results stale-data fix (R-007)** — Navigating between different uploads' Results pages now shows a loading skeleton instead of briefly flashing the previous upload's cards/header.

---

## What's in staging (not yet in production)

### CDR — Test client for CDNR demo prep
- User: `edbernal@cdr.com` / password: `12345`
- Org: CDR (`b986d4d7-ca78-4326-998e-56682352b0e2`) + HIG Testing (`eb63c19f`)
- This user has 2 orgs and sees the client selection screen on login
- Full E2E multi-org flow validated: select client → data isolation confirmed → switch client works
- Bloomberg Reconciliation module enabled for this user on both orgs (CDR + HIG Testing) — see Recent changes

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

- **Grupo C — validating Source against Senthio (staging only)** — HIG Testing's test data was wiped and rebuilt with 23 real contracts + invoices, matched vendor-by-vendor and service-by-service against Senthio's own records so the two systems can be compared apples-to-apples. Along the way this surfaced and fixed 3 real calculation bugs (see Recent changes below) and added the ability for one invoice to cover more than one service. 17 of the 23 test invoices are now fully approved; 6 are deliberately on hold pending Santiago's input on how unusual billing situations (one-time charges, usage-based vendors, hardware purchases) should be handled — see "Where your input would help." None of this session's fixes have reached production yet.
- **Vendor grouping for new clients** — Spec/prompt ready, implementation pending.
- **Monthly pricing normalization (P1)** — When contract states prices monthly, AI uses monthly price as annual value. Fix pending Santiago review.
- **Service split/merge UI** — Deferred. Waiting for Santiago validation.
- **`link_contract` UI** — Backend action live, UI trigger not built.
- **Not a match button** — Hidden for now. Flow and requirements need validation before re-enabling.

---

## Coming next (priority order)

*Re-sequenced 2026-07-08 after the live demo + feedback call with Santiago (see Recent changes below and `PRODUCT_REVIEW_BACKLOG.md` R-023–R-030), refined 2026-07-10 after a follow-up call added R-031. Santiago is on vacation from Thu Jul 9, back for a call **2026-07-20** — big-bet items need his input and wait until then; quick wins don't need further validation from him and can proceed now. Target: Santiago wants invoicing + reporting (+ ideally admin) closed out by **end of July**, after which further work is driven by real client feedback in production rather than a fixed roadmap. Edgar's stated order: finish the remaining piece of R-026 (Santiago's own validation) → reporting (R-030 then R-031) → admin page (R-029) last.*

1. **Quick wins — done 2026-07-08:**
   - ✅ R-023 — removed "Add general adjustment (no user)" in both `InventoryContractDetail.tsx` and `InvoiceDetail.tsx`; every adjustment must target a user. Department-as-alternative explicitly deferred (no `departments` concept in schema today).
   - ✅ R-024 — widened the adjustment user picker in `InventoryContractDetail.tsx` to reach unallocated org users, not just those already subscribed to the service.
   - ✅ FIS + Nxt Gen special cases — resolved using Santiago's decided treatment: FIS got a 2nd service ("IDX Subscription Overage Fee") on a separate billing account ("FIS-IDX-Overage"), approved with a $0.05 rounding adjustment; Nxt Gen hardware invoice got its own billing account ("NextGen Hardware") + a one-time $91,989.27 adjustment, approved.
   - ✅ **Bonus finds, same session**: a real regression where `get_invoice_reconciliation`/`create_invoice_distributions` had silently reverted to their pre-`invoice_services` bodies on staging (fixed, Mimecast's variance had been wrong all along as a result — now $0.00 with all 4 services); `invoice_adjustments.tax` was captured but ignored by both RPCs (fixed).
   - ✅ **R-026 — done, ahead of schedule** (see big bets note below — this one didn't actually need to wait).
2. **Medium — scoped builds, target this/next week:**
   - ✅ **R-025 — done 2026-07-10, deployed to production 2026-07-12**: `InventoryBilling.tsx` rewritten as a sortable, filterable table (vendor/account/status filters + search), re-added to nav. Row click deep-links into `InventoryContractDetail.tsx`'s existing invoice-review panel (reused the existing IRW/Approve UI instead of building a duplicate standalone page — a design correction made mid-implementation after Edgar pushed back on the initial approach). Validated end-to-end on staging, merged to `main` in both repos, deployed to production.
   - R-030 — minimal budget export (period-range picker + CSV) on the Period Snapshots report. Santiago asked for this explicitly "for next week." Get the Power BI reference video from Edgar before building the layout. First slice of the bigger reporting push — see R-031.
   - R-027 — retroactive matching of a late invoice against an already-closed period (flips a `missing_invoice` snapshot row to `matched` without a full period reopen). Doesn't block on R-003.
   - R-031 *(new, 2026-07-10)* — Reporting/Dashboards module. Santiago showed a reference BI dashboard (KPI tiles, charts, per-dimension drill-down tables, Excel export, an "Ask AI" chat tab on Claude Opus 4.8) from another client's external tooling and wants equivalent capability inside Source, as reference not spec. Phase after R-030; needs its own scoping/design pass before estimating. "Ask AI" chat is explicitly lowest priority per Santiago (nice-to-have, not required).
3. **Big bets — need Santiago's input before fully closing out:**
   - ✅ **R-026 — billing account as an invoice-level scoping parameter — implemented 2026-07-08, deployed to production 2026-07-12**, sooner than planned (the FIS case made the gap concrete enough to design and build same-day). `invoices.billing_account_id` + both RPCs resolve eligible services as invoice_services ∪ subscriptions sharing that account, across any contract. Validated with a temporary synthetic account across Third Bridge's two real contracts (reverted after) and again live in the UI with Third Bridge's real accounts (Expert Network + Forum Package). Regression-checked against FIS/Nxt Gen/Mimecast. **Still not yet seen by Santiago** — now live in production ahead of his review (bundled into the same deploy as R-025), the shape matches his own description and Senthio's actual model (AccountMaps, confirmed against `.claude/skills/senthio-reference.md`), but show him once back from vacation (2026-07-20) before treating it as fully closed. Also still open: clearer UI distinction between "+ Add Service" (manual, single service) and the new account selector (automatic, cross-contract) — currently both exist side by side without an obvious explanation of when to use which.
   - R-028 — edit archived-period metadata (billing account, product catalog) with historical propagation. Directly coupled to R-003 (reopen closed period) — don't scope in isolation before Edgar's pending conversation with his manager about whether closed periods should be editable at all.
   - R-029 — admin page (users, clients, modules, permissions). Nothing exists in the frontend today; an approved Gate-1 plan already covers the user-invite half (`docs/tech-debt/admin-invite-user-flow.md`), but Santiago's ask is broader (also module/client CRUD, ties to R-020). Largest of the Jul-8 asks — needs its own planning session.
4. **Deep Tree data cleanup + Monthly pricing normalization (P1)** — The $944.50 expected (vs ~$880 in contract) came from a Senthio-loaded `annual_value`. Clean staging data and close out P1 (AI using monthly price as annual value) with Santiago — the 2026-07-02 call effectively confirmed the expected value must derive from the contract's monthly average × months + tax.
5. ✅ **Deploy the month-count fix (v8) to production — done 2026-07-12**, bundled with the R-025 deploy. Post-deploy audit for already-approved production invoices with a similar end-day < start-day billing range: **zero matches found**, no data fix needed.
6. **Grupo C — line-by-line dollar comparison vs Senthio** — the mechanism was validated 2026-07-06/07 (23 contracts/invoices reconciled), but a real cent-by-cent comparison against Senthio's `Inventory_R`/`Invoices_Adj` for June hasn't been done yet, and the "missing invoices in an already-closed period" report (Senthio's `qryInventoryH` equivalent) still needs to be built — see R-027 above, which is the mechanism this needs.
7. **Security & Compliance (~50% of time from week of Jul 13)** — Still pending, now more urgent: Execute `docs/security/security-compliance-roadmap.md`: Track 1 first (**A50 GCP key file — confirmed still sitting exposed in `~/Downloads` since May 4, 2+ months, needs rotating in GCP IAM Console**; A51 JWT toggle; A52 secrets audit; A53 MFA), Track 3 in parallel (A58 asset inventory + data flow, A59 core policies, A60 client one-pager — this is the documentation Santiago asked for, incl. CDR due-diligence/business continuity). Fold in R-018 (audit logging/SSO) rather than running it separately. Edgar is aiming for a first draft of the client security one-pager (Q&A style) by end of this week (2026-07-10). **Immediate pending:** review Supabase free-tier limits → tell Santi whether production needs a plan upgrade (promised 2026-07-02, still overdue).
8. Cost per user view
9. Renewal alerts (contracts approaching cancel_lead_time_days)
10. Bloomberg Terminal Upload (CSV-based, after HR file already live)
11. Spend/Inventory report with adjustments + forecast
12. Bloomberg Terminal Reconciliation (review with Santi)

**Note (from 2026-07-02 weekly, still relevant):** R-003 (reopen closed billing period) is no longer just a hypothetical — the Grupo C session had to hand-reset HIG Testing's June period directly in the database (no product feature exists for this yet) to redo the close live with Santiago. Now also a hard dependency for R-028. Raise it with him directly once he's back.

---

## Where your input would help

Six real invoices from the rebuilt HIG Testing test set were deliberately left unapproved because how to treat them isn't a calculation question — it's a product/business judgment call:

1. **AlphaSights (and similar usage-based vendors)** — the "expected" cost is always going to look different from any single month's invoice, because these vendors bill actual usage that varies month to month, not a fixed subscription. Question: should Source try to mirror Senthio's manual monthly re-allocation process for these vendors, or is a routine Invoice Adjustment the intended answer every month?
2. **DeepTree** — this specific invoice is a one-time "added 2 seats" charge, not the recurring subscription fee. Confirmed Senthio itself never tries to reconcile this type of invoice against the ongoing service cost — it's simply paid and filed. Same question likely applies to any vendor with similar "adding capacity mid-contract" invoices.
3. **FIS** — turned out to be an entirely different kind of charge (an overage fee for exceeding a user limit) than the annual subscription fee we had on file — unrelated dollar amounts, same vendor/contract.
4. **Nxt Gen Technologies (AvePoint)** — this invoice is a one-time hardware purchase (a physical backup appliance), not the AvePoint cloud subscription.
5. **Third Bridge (Forum contract)** — this invoice appears to be the final installment of a contract that started back in 2024, entered into the system two years late; the going rate we have on file may reflect a newer renewal that this old invoice was never meant to match.
6. **PEI** — a small $67.84 gap between the invoice and what's recorded in Senthio, with no obvious explanation (not a rounding issue, not an escalator).

Full detail and evidence for each is documented in `scripts/staging-reset-hig-testing/vendor-pricing-validation.md`.

Two more decisions from earlier in the week that still need an answer:
- **Supabase plan upgrade** — promised an answer to Santiago on 2026-07-02, still pending.
- **GCP key file** — confirmed sitting unprotected in `~/Downloads` since May 4; needs a decision on rotating/revoking it in the GCP console.

---

## Technical debt

- **Batch upload scalability** — In-process polling + sequential dispatch breaks at ~50 docs. See `docs/tech-debt/batch-upload-scalability.md`
- **Service split/merge UI** — See `docs/tech-debt/service-split-merge.md`
- **Invoice variance/adjustments** — ✅ Done (Grupo B). `invoice_adjustments` + `invoice_distributions` in production.
- **Soft/Hard dollar classification** — Senthio tracks Hard$ vs Soft$ per user. Deferred.
- **GL Accounts** — Senthio routes expenses to GL accounts via AccountMaps. Deferred.
- **Multi-year contract pricing** — `service.annual_value` is set at Year 1 price. Full fix requires `ServicesFP` equivalent. See `docs/tech-debt/service-pricing-schedule.md`
- **Allocations inline editing** — Custom onBlur pattern still in place in `InventoryContractDetail.tsx` ~line 1898. Should use `InlineEditableField`.
- **Pre-existing TypeScript errors in InventoryUploadDetail.tsx** — `contract_id` not in `ContractData` type (line 849+), several unused state vars. Not causing runtime issues.
- **HIG Testing exchange rates** — Only June 2026 rates configured. Need EUR/GBP rates for July 2026+ before next close.
- **Admin invite-user-by-email flow** — Approved Gate 1 plan, deferred until onboarding volume justifies it. See `docs/tech-debt/admin-invite-user-flow.md`.
- **Duplicated PDF-upload helper** — `uploadPdfToStorage`/signed-URL logic is now duplicated between `process-contract` and `process-inventory-document`. Should be consolidated into `_shared/`.

---

## Environments

| | Supabase ref | Frontend |
|---|---|---|
| Production | `fdcxcivjhobreuseacot` | https://s4source.io |
| Staging | `fntpcrpmkwyruzplbewq` | https://s4sourceio.lovable.app |

---

## Recent changes (2026-07-13 session — Reports module, R-030 done, R-031 phase 1+2 deployed)

Also recovered a stranded commit (`a259595`, $1 approve-variance threshold + IRW billing-account column in `InvoiceDetail.tsx`) from a stale branch before deleting it — its other 4 commits were already re-implemented on `main` via different commits, but this one never was. Deleted the stale branch plus 5 other fully-merged local branches in both repos as general cleanup.

**Backend** — `report_spend_aggregate(org, from, to, group_by, split_by?, filter_by?, filter_value?)`: one generic aggregation RPC over `inventory_period_snapshots` instead of one query per chart. Every report widget is a `{group_by, split_by}` spec; default sheets are predefined specs now, user-saved/AI-generated specs later reuse the same contract unchanged. Whitelisted dimensions (month/vendor/department/cost_center/building/user/service) — no string interpolation. Validated against direct SQL sums on staging and against real org data on production (a user's total reconciled exactly through the vendor drill-down filter).

**Frontend** — Reports gained 3 new tabs. **Overview** (now the default): 4 KPI tiles, monthly spend bar chart with a Split-by selector (department/cost_center/building — this is the generalization of the reference dashboard's FO/MBO split we don't have a field for), a department-composition donut (fills the reference's FO/MBO donut slot), Top 10 Vendors/Departments bar charts, CSV export. **Vendors** / **Users**: one generic component instantiated twice — sortable table with a Share bar + %, click-to-expand rows revealing two side-by-side breakdowns (a vendor's services + users; a user's vendors + services), lazy-loaded and cached per period range, per-entity CSV export. All three sheets share one From/To period-range picker (`usePeriodRange` hook) so a range chosen in one tab carries to the others.

**Scope discipline, twice**: (1) When asked which charting library would give users "autonomy" to build widgets, pushed back on adding a BI/self-service charting library — the reference dashboard Santiago showed is itself fully static (his own words: *"esto no, esto es todo estático, sí"*), so the recommendation was `recharts` (already installed, already live on the Dashboard) plus the generic RPC contract, which buys the same future extensibility without new infra or a multi-tenancy story to solve. (2) When asked to make the sheets "more visual, not just tables," compared every reference screenshot tab-by-tab: confirmed the existing build already matched the reference's mix (Overview fully visual, dimension sheets are enriched tables with embedded share bars, same as the reference) — added a donut and %-labels as the two genuinely missing pieces, explicitly declined to duplicate the Top-10 charts into the Vendors tab or add a treemap/alternate view.

No new npm dependencies. TypeScript check clean throughout (12 pre-existing errors, zero new). Deployed to production same-session per the new deploy-cadence rule: backend `db push` to prod (dry-run confirmed only these 2 migrations pending, no accumulated drift), frontend pushed to `main` + published in Lovable.

---

## Recent changes (2026-07-12 session — deep staging-vs-prod parity audit, drift fixed)

Full audit comparing live state (not just migration history) between staging (`fntpcrpmkwyruzplbewq`) and production (`fdcxcivjhobreuseacot`): all 141 migrations, every public function definition (hash-diffed), all columns, indexes, RLS policies, all 13 Edge Functions (source downloaded and diffed), verify_jwt settings, and frontend deploy state.

**Already in sync:** migrations, columns, RLS policies, verify_jwt (false × 13 × both), 12 of 13 Edge Functions.

**Drift found and fixed (5 findings, all pre-dating R-025):**
1. **`match-invoice-service`** — staging had been running (since ~2026-05-26) an improved version (resolves `contract_id` + `billing_period_id` on match, auto-matches single-service vendors) that existed **nowhere in git**; repo and prod had the old version. Without `billing_period_id`, prod-matched invoices are invisible to Billing/IRW. Fixed: staging source committed to repo verbatim, deployed to prod (v5).
2. **`audit_log_trigger`** — staging and prod each had a *different* untracked manual patch (both extending the original to resolve user from `app.current_user_id`). Canonicalized on staging's variant via migration `20260712000001`, applied to both.
3. **`idx_invoices_billing_period_id`** — staging had a hand-made composite `(org_id, billing_period_id)` version; prod had the original. Composite adopted as canonical (matches every consumer's filter shape) via the same migration.
4. **`vendors_name_trgm_idx` + `vendors_name_user_id_key`** — existed only in prod (manual, untracked). Added to staging via migration (pre-checked: no duplicate rows blocking the unique index).
5. **`vendor_aliases_alias_user_id_key`** — staging-only legacy constraint from the pre-multi-org model; a latent staging-only bug (same alias in two orgs would fail only there). Dropped via migration. The real constraint (`alias, org_id`) exists in both.

Also converged: `get_invoice_reconciliation`/`create_invoice_distributions` on staging had comment-only diffs from the R-022 manual re-apply — re-applied verbatim from the canonical migration file. **Post-fix verification: every function and index now hash-identical between environments.** Smoke-tested `get_billing_invoices` on both (staging 23 rows, prod 3 rows, no errors).

**Frontend finding:** `s4sourceio.lovable.app` (documented as the staging frontend) just 302-redirects to `s4source.io` — there is no staging frontend; validation is local `npm run dev` + `.env.staging`. Also confirmed with Edgar: **push to `main` alone does not deploy — he must click Publish in Lovable** (the old "never use Publish" guidance was wrong).

**Process changes:** CLAUDE.md — environments table corrected, new deploy rule 4 (no drift accumulation, `db push` only, parity check at session close). `how-we-build.md` — Step 4 rewritten (deploy to prod as part of closing each item, not batched "later"; correct Lovable Publish flow; re-link staging after prod work) and new Step 6 (parity checklist at every session close).

---

## Recent changes (2026-07-12 session — R-025 + accumulated backend work deployed to production)

- **`invoice-agent-mvp`**: `feat/r025-billing-table` merged to `main`, pushed. 11 migrations applied to production (`fdcxcivjhobreuseacot`) via `supabase db push` — `20260706000001` through `20260710000002`. This was the first production deploy for all of them; everything from R-022 (invoice_services multi-match) through R-026 (cross-contract billing account) and the month-count fix (v8) had been staging-only until now. Confirmed via `supabase migration list --linked` that all 11 show a `Remote` timestamp post-push.
- **Post-deploy audit (month-count fix v8)**: queried production for approved invoices with `billing_end_date`'s day-of-month < `billing_start_date`'s day-of-month (the pattern the pre-fix bug mis-counted) — **zero matches**, no data correction needed.
- **`s4sourceio`**: `feat/r025-billing-table` merged to `main`, pushed — triggers the Lovable production deploy.
- **R-026 caveat**: now live in production, but Santiago (who proposed the shape) still hasn't seen this implementation — he's back 2026-07-20. Not a rollback risk (regression-checked against FIS/Nxt Gen/Mimecast before this), but flag it to him explicitly rather than treating it as fully signed off.
- Scope note: what shipped was scoped as "merge R-025 to main," but production had never received the prior week's backend work either — `supabase db push --dry-run` surfaced this before applying anything, confirmed with Edgar to bundle all 11 rather than leave production further behind.

---

## Recent changes (2026-07-10 session — R-025 Billing table implemented, staging only)

- **R-025 shipped to staging** — `InventoryBilling.tsx` rewritten from a flat pending-approval queue into a sortable table: 12 columns (vendor, account, invoice #, amount, 5 dates, status, reconciled, allocated), client-side filters (status pills + vendor/account dropdowns + vendor search) over one fetch, re-added to the sidebar nav. New `get_billing_invoices` RPC (backend) computes `reconciled` by reusing `get_invoice_reconciliation`'s own `net_variance` (same threshold the Approve button uses, not a reimplementation) and `allocated` via an `invoice_distributions` existence check.
- **Design correction mid-build**: the first cut routed row clicks to a new standalone `/invoices/:id` page. Edgar pushed back — that duplicated IRW/Approve UI that already exists in `InventoryContractDetail.tsx`. Reworked to deep-link there instead (`?tab=invoice&invoiceId=X`, added `contract_id` to the RPC); confirmed every Billing-eligible invoice has a non-null `contract_id` before relying on it. `InventoryContractDetail.tsx`'s "Back" link now reads the URL to return to Billing, Upload Results, or Inventory depending on how the page was entered.
- **Side fix**: `InvoiceDetail.tsx` (still used by the separate Processing History flow, confirmed still reachable via `InventoryLayout`'s TOOLS section — not deprecated) was rendering completely outside the Inventory app shell. Now wrapped in `InventoryLayout`; "Back" uses `navigate(-1)` since the page is reached from two different flows with different correct return destinations.
- **Migration tracking gap fully closed** — while starting this, found the R-022 postmortem's flagged gap had grown to 8 untracked migrations (not 3). Verified each one's live state on staging (function bodies, RLS `WITH CHECK`, indexes) before running `supabase migration repair` for all 8. Going forward, new migrations go through `supabase db push`, not manual `db query`.
- **New: R-032** — logged 12 pre-existing TypeScript errors found while typechecking this change (confirmed via `git stash` none were introduced by R-025). Grouped by root cause (missing `bold`/`status` properties, mismatched Supabase join types) for a future cleanup pass.
- Validated end-to-end on staging by Edgar (table/filters/sort, row click into the contract's invoice panel, Approve flow, both Back-link entry points). **Not yet merged to `main` or deployed to production** — everything above is on branch `feat/r025-billing-table` in both `invoice-agent-mvp` and `s4sourceio`.

---

## Recent changes (2026-07-10 session — follow-up call with Santiago, no code changes)

- **R-025 refined** — Santiago wants the Billing list as a table (not cards), with filters by vendor/account/status, bidirectional (from the list or drilling into an account). Shared a reference screenshot (another client's Salesforce-style Invoices table) with concrete columns — captured in the backlog row along with open questions on exact field mapping (Reconciled?/Allocated?/Recovery Period/Processed) to confirm once this is actually scoped for build.
- **New: R-031 — Reporting/Dashboards module.** Santiago showed a full external BI dashboard (another client's tooling, built outside Source) as a reference for what he wants brought into the product: KPI tiles, spend charts, per-dimension drill-down tables (Vendors/Cost Centres/Users) with Excel export, a dedicated single-vendor deep-dive page, and an "Ask AI" chat tab already running on Claude Opus 4.8. Explicitly given as inspiration, not a spec to replicate 1:1. Extends/supersedes R-030 (still the first, minimal slice). The Ask AI chat is explicitly the lowest-priority piece per Santiago himself.
- **Timeline confirmed**: next weekly with Santiago is 2026-07-20 (he's out the week of Jul 13). Edgar's work order for the rest of July: finish the outstanding piece of R-026 (get Santiago's own validation) → reporting (R-030 then R-031) → admin page (R-029) last.
- No code shipped this session — planning/backlog update only.

---

## Recent changes (2026-07-06/07 session — Grupo C data prep, staging only)

This session's goal was to get HIG Testing's test data ready for a real side-by-side comparison against Senthio (the Access database HIG currently uses), and rehearse the full invoice-reconciliation flow before doing it live with Santiago. Everything below happened on staging only — nothing reached production yet.

- **Rebuilt HIG Testing's test data from scratch, using real records.** The old test data was made-up and messy. It's been replaced with 23 real contracts and their matching invoices, with vendor names, service names, and pricing all lined up to match Senthio's own records, so any comparison between the two systems is apples-to-apples.
- **Found and fixed a real calculation bug affecting every client, not just this test data.** The invoice-reconciliation calculation was counting one extra month in specific cases — whenever a contract's billing period ends on an earlier day-of-month than it starts (a common pattern for annual contracts renewing "the day before" the anniversary). This made the system's expected cost look too high for those invoices. Fixed and confirmed against several real examples. **This fix is on staging only — production still has the bug**, so this should be prioritized for a proper review and deploy soon, and any already-approved production invoices with this pattern should be double-checked afterward.
- **An invoice can now cover more than one service.** Previously, if one bill from a vendor covered two different services (for example, a Mimecast invoice that includes both email security and Teams archiving), the system could only "see" one of them and always showed a mismatch. Users can now explicitly tell the system which services a given invoice covers, and the expected-cost calculation adds them all up correctly. Validated live on two real invoices.
- **Manual cost adjustments can now be applied to several people at once**, instead of one at a time — matching how Santiago asked for this back on 2026-07-02. Also found and fixed a small follow-up bug where adjustments couldn't be entered without picking a specific person, even when the adjustment wasn't about any one individual.
- **Closing a billing period now also carries forward manual adjustments into the permanent monthly record**, matching how Senthio itself does it. Before this fix, an adjustment used to approve an invoice would "disappear" from the historical record once the period closed, even though the invoice itself was correctly approved.
- **17 of the 23 real test invoices are now fully approved and reconciled end-to-end** (10 with no adjustment needed, 7 needing a small rounding adjustment). The remaining 6 were left unapproved on purpose — see "Where your input would help" above.
- **Found a genuine, unprotected Google Cloud credentials file** sitting in Downloads since May 4th (2+ months) — flagged for Santiago, needs rotating.
- Two small UI polish items were requested from the frontend session and are still pending: showing the after-adjustment variance number instead of the before-adjustment one in the invoice summary, and clearer labeling on the monthly cost report (which numbers are "the full service cost" vs. "this person's share").

---

## Recent changes (2026-07-02 session — Billing Module Grupo B shipped to production)

- **`invoice_adjustments` table** (`20260701000007`) — new table for per-invoice variance adjustments, RLS via `user_org_memberships`. `contract_id` stored on each adjustment row.
- **`invoice_distributions` table** (`20260701000008`) — equivalent of Senthio's `Invoices_Adj`. Stores per-user, per-month cost distribution rows generated at invoice approval.
- **`create_invoice_distributions` RPC** (`20260701000009`) — SECURITY DEFINER, idempotent. Generates distribution rows from `service_subscriptions` (one per month × user over billing range) plus `invoice_adjustments` (is_adjustment=true). Includes FX conversion via `exchange_rates`.
- **`get_invoice_reconciliation` v6** (`20260701000010`) — adds `adjustments_total` and `net_variance` to the return payload. `net_variance ≈ 0` enables Approve.
- **Frontend: Invoice Adjustments tab + Approve Invoice** — `InventoryContractDetail.tsx` and `InvoiceDetail.tsx` now show two tabs: "Invoice Reconciliation Worksheet" and "Invoice Adjustments". Approve Invoice button enabled when `|net_variance| < 0.01`. Inline add/delete form; Year/Month derived from `billing_start_date`.
- **End-to-end validated** — Tax Analysts invoice 50217 ($8,751.60): adjustment $1,037.85, approved. 34 rows in `invoice_distributions`, SUM = $8,751.60.

---

## Recent changes (2026-06-12 session — Bloomberg Reconciliation activation + nav fixes, staging)

- **Reconciliation module activated for CDR + HIG Testing (staging only)** — Inserted `client_modules` rows (`module = 'reconciliation'`, `enabled = true`) for `edbernal@cdr.com` (`298fa7d4-5c55-4e13-b615-43cc2a0f961f`) across both org_ids (CDR `b986d4d7-ca78-4326-998e-56682352b0e2` and HIG Testing `eb63c19f-a8dd-4f28-8638-b8c522fe4e18`), consistent with the multi-org `user_org_memberships` model. Not yet replicated in production.
- **Duplicate "Bloomberg Recon." nav item fixed** — `InventoryLayout.tsx` built `enabledModules` from one `client_modules` row per org, so a user with the module enabled on 2 orgs saw it listed twice. Deduped with `[...new Set(...)]`.
- **Misleading "account switch" on entering Reconciliation fixed** — `Reconciliation.tsx` sidebar footer showed the static `profile.company_name` instead of `activeOrg?.name`, making it look like the active client changed when it hadn't. Also found and fixed a real scoping bug: `loadRuns()` queried `reconciliation_runs` with no `org_id` filter (relying only on RLS), which would mix runs from all of a user's orgs. Both fixed, committed and pushed to `s4sourceio` main (commit `f268afa`).
- **Sample test CSVs created** for manually exercising the reconciliation upload flow (`ReconciliationModal`) — `bloomberg_sample.csv` (`SID`,`Extended Price`) and `fits_sample.csv` (`KeyPart1`,`CvtCost`), built from real SIDs/prices in HUDSON BAY CAPITAL's (Cust_Num 30656515) subscription list, covering all 5 reconciliation statuses (MATCH, ADJ_POS, ADJ_NEG, ONLY_BLOOMBERG, ONLY_FITS). Saved to `~/Downloads/`.
- **Entitlement gap noted (not yet tracked as tech debt)** — `client_modules.enabled` only gates sidebar visibility per `user_id`; it does not restrict data access by `org_id`. Revisit if this becomes a real security boundary for a second client.

---

## Recent changes (2026-06-12 session — Phase 5 (R-017, R-007) shipped to production)

- **R-017 (vendor matching false positive)** — `reconcile-inventory-batch/index.ts` `CONTRACT_MATCH_THRESHOLD` raised from 0.30 to 0.50. This is a general fix (single constant, not vendor-specific) — before deciding, simulated both thresholds against all 21 real orphan-invoice/Draft-contract pairs on staging plus a sample of 8 contracts. At 0.50, the CBINSIGHTS/ProSights Labs false positive (score ~0.455) is rejected; the only other pair affected was Factiva↔Dow Jones (score ~0.471, a true positive — Factiva is a Dow Jones brand), which now requires manual linking instead of auto-linking. Net improvement accepted by Edgar. Added a regression test. One-off data fix reverted the bad CBINSIGHTS→ProSights Labs link on staging (`invoices.contract_id` → null, `vendor_match_status` → unmatched, corresponding `inventory_upload_items` row reverted). Checked production for the same pattern — only one linked orphan pair exists ("AgFlow"→"AgFlow SA", score ~1.0, a true positive) — no data fix needed in prod. Deployed to staging and production (`verify_jwt` already off on both).
- **R-007 (stale data on Upload Results navigation)** — Root cause was isolated to `InventoryUploadDetail.tsx`: a `hasLoadedOnce` ref stayed `true` across navigation to a different upload, so `setLoading(true)` never re-fired and the previous upload's cards/header briefly stayed on screen. Fixed with a `prevUploadIdRef` that detects a real `upload_id` change (not a poll/refresh on the same upload) and resets `loading`/`detailsReady`/fingerprint state. Audited the other 4 pages with navigable route params (`ContractDetail`, `InvoiceDetail`, `InventoryContractDetail`, `InventoryVendorDetail`) — all already handle this correctly, so this was not a transversal issue and no other pages needed changes. Validated locally against staging data, then pushed to `s4sourceio` main (direct production deploy, no separate frontend staging).
- Both items used a lighter targeted review instead of the full `/code-review` multi-agent flow, given the small diff size (1 constant + 1 test; 1 ref + ~8 lines in one effect).
- Phase 5 of the `/app-review` action plan is now closed. All items from the original 2026-06-10 action plan (R-001 through R-017, except deferred R-003 and discarded R-012) are Done.

---

## Recent changes (2026-06-12 session — Phase 4 (R-009) delete/re-upload contract shipped to production)

- **R-009 (delete/archive + re-upload contract under same vendor)** validated by Edgar on staging, then shipped to production.
- **R-012 (manual contract creation)** discarded — R-009's delete/re-upload flow already covers the use case; marked "Won't Do" in `PRODUCT_REVIEW_BACKLOG.md`.
- **Extraction quality check** — confirmed `process-contract` (individual upload) and `process-inventory-document` (batch) use identical OCR/extraction logic. The perceived quality gap was due to `process-contract` not storing a PDF and defaulting status to `Active` instead of `Draft`. Both fixed and deployed to staging + production.
- **Production hotfix** — a premature frontend deploy (push to `s4sourceio` main = production deploy) exposed the R-009 UI before the backend was synced, causing a "function not found" error when deleting a contract. Fixed by applying the pending migration and redeploying `process-contract` to production, plus a one-off data fix (`source = 'inventory_upload'`) on the re-created contract.
- **Gate 3 security fix** — `delete_active_contract` and `delete_inventory_review_item` only checked org membership (any role); now require `role = 'admin'`, matching the contracts table's DELETE RLS policy. Deployed to staging and production.
- **Operational note**: pushing to `s4sourceio` main deploys directly to production — any frontend change touching a new backend RPC/Edge Function must have that backend already live in prod first.

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
