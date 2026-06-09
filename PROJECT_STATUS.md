# Source S4 — Product Status
*Last updated: 2026-06-09 (evening session)*

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
- **Sidebar navigation** — Dashboard as primary nav item. Processing tools (Invoice Processing, Contracts, Bloomberg Recon.) in a TOOLS section, visible only when modules are enabled for the user.
- **Reports section** — Three tabs: External Documents Required, Renewal Calendar (search, urgency filters, sortable columns), and Period Snapshots (browse closed billing-period history by month, with search, status filters, sortable columns, and new Allocation / Avg Monthly / User Cost / Cost USD columns).
- **Supplemental document flags auto-clear** — When a user manually types in a contract value that was flagged as "needs supporting document," the system automatically marks it resolved.
- **Full-width layout** — All inventory pages now use full available width.
- **Invoice delete** — Users can delete an incorrectly assigned invoice from the contract detail page and re-upload it to the correct contract.
- **Billing period snapshot cost fix** — Snapshot now uses Avg Monthly cost (unit_cost ?? annual_value/12) so user_cost is never null when annual_value is set.
- **Rate-limit retry hardening** — Batch upload dispatch retries on both HTTP 429 and thrown Supabase rate-limit exceptions, with jitter to prevent thundering herd.
- **linked_contract_id fix** — Invoice items in a batch now correctly link to their matched contract after reconciliation.
- **Batch-scoped contract matching** — Invoices in a batch now only match against contracts from the same batch. Previously, a vendor name match (e.g. "AlphaSights") could incorrectly link to a Draft contract from a completely different prior batch (e.g. "ProSights") due to a shared word in the vendor name. Both the real-time processing phase and the reconciliation phase are now fully batch-isolated.

---

## What's in staging (not yet in production)

### CDR — Test client for CDNR demo prep
- User: `edbernal@cdr.com` / password: `12345`
- Org: CDR (`b986d4d7-ca78-4326-998e-56682352b0e2`)
- Only inventory module — no TOOLS section shows
- Full E2E flow validated: batch upload → vendor confirm → HR CSV → allocations → billing period close → snapshot with correct costs

### CDNR — First real client POC
- New small client (~50–75 contracts), inventory starts from June 2026 (no historical data needed)
- Demo meeting with Stephanie de Lucía planned for ~2026-06-10
- Santiago, Edgar, Bernardo + Stephanie attending
- Goal: introduce them to the system and collect real feedback

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
3. **Vendor grouping** — group unmatched vendors by name when a new client uploads their first batch. Prompt ready.
4. **Bloomberg Terminal Upload (structured CSV import)** — separate upload type from core subscription PDFs.
5. **Spend/Inventory report** — monthly spend view: vendor → contracts → services → users/departments, with invoice adjustments and forecast.
6. **Bloomberg Terminal Reconciliation** — compare Bloomberg inventory files vs Source.

---

## What's in development

- **Vendor grouping for new clients** — Spec/prompt ready, implementation pending.
- **Monthly pricing normalization (P1)** — When contract states prices monthly, AI uses monthly price as annual value. Fix pending Santiago review.
- **Service split/merge UI** — Deferred. Waiting for Santiago validation.
- **`link_contract` UI** — Backend action live, UI trigger not built.
- **CBInsights/ProSights Dice coefficient false positive** — "sights" suffix causes ~0.53 score (above 0.30 threshold). Threshold fix deferred.
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

- **Batch upload scalability** — In-process polling + sequential dispatch breaks at ~50 docs. See `.claude/memory/project_batch_scalability_debt.md`
- **Service split/merge UI** — See `.claude/memory/project_service_split_merge_debt.md`
- **Invoice variance/adjustments** — When invoice ≠ inventory expected amount, need adjustment records (Senthio: `Invoices_Adj`). Deferred post-MVP.
- **Soft/Hard dollar classification** — Senthio tracks Hard$ vs Soft$ per user. Deferred.
- **GL Accounts** — Senthio routes expenses to GL accounts via AccountMaps. Deferred.
- **Multi-year contract pricing** — `service.annual_value` is set at Year 1 price. For escalating multi-year contracts, user must manually update Annual Value each year. Full fix requires `ServicesFP` equivalent. See `.claude/memory/project_service_pricing_schedule_debt.md`
- **Allocations inline editing** — Custom onBlur pattern still in place for service name/annual_value in the allocations panel (`InventoryContractDetail.tsx` ~line 1898). Should use `InlineEditableField`.
- **Pre-existing TypeScript errors in InventoryUploadDetail.tsx** — `contract_id` not in `ContractData` type (line 849), several unused state vars. Not causing runtime issues but should be cleaned up.

---

## Environments

| | Supabase ref | Frontend |
|---|---|---|
| Production | `fdcxcivjhobreuseacot` | https://s4source.io |
| Staging | `fntpcrpmkwyruzplbewq` | https://s4sourceio.lovable.app |

---

## Recent changes (2026-06-09 session)

**Batch upload reliability — deployed to production:**
- Fixed a silent failure where the system would hit rate limits and stop dispatching documents without retrying. Now retries on both HTTP 429 responses and thrown rate-limit exceptions from Supabase, with random jitter so all workers don't retry at exactly the same time
- Reduced concurrent worker slots from 5 to 3 to reduce pressure on rate limits
- Fixed invoices in a batch not showing as linked to their matched contract after processing

**Billing period snapshot costs — deployed to production:**
- Fixed: user cost in a closed period snapshot was null whenever a contract had an annual value but no per-seat price set. Now uses annual value ÷ 12 as the monthly cost (matching what the contract detail page shows as "Avg Monthly")
- Period Snapshots table now shows Allocation %, Avg Monthly, and User Cost columns in addition to Cost (USD)
- Removed the Status column from the snapshot table (redundant)
- Fixed money formatting — small amounts like €3.15 were displaying as €3 (rounded to integer)

**PDF viewer fix — deployed to production:**
- Fixed: contracts and invoices uploaded via batch showed "No PDF available" in the review panel. The viewer was looking for the file at a hardcoded path that only applies to individually-uploaded documents. Now reads the actual storage path from the signed URL stored on each record

**Invoice management — deployed to production:**
- Added a delete button (trash icon) to the invoice table in contract detail. Allows users to remove a wrongly-assigned invoice and re-upload it to the correct contract
- Confirmed end-to-end: delete → re-upload → correct vendor match works

**UI cleanup — deployed to production:**
- Removed "Not a match" button from invoice match review modal (deferred — flow not yet validated)
- Removed "Merge vendors" button from Upload Results page — it belongs in Inventory where duplicate vendors are visible in context

**Batch-scoped contract matching — deployed to production:**
- Fixed a cross-batch false positive: when uploading a new batch, invoices were being incorrectly linked to Draft contracts from previous batches due to vendor name similarity (e.g. "AlphaSights" matching "ProSights" because both contain "sights")
- Contract fallback matching during document processing now only considers contracts from the same batch
- Both processing and reconciliation phases are now fully batch-isolated
