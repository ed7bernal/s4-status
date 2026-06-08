# Source S4 — Product Status
*Last updated: 2026-06-08*

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
- **PDF viewer** — Split-pane viewer in contract and invoice detail.
- **Invoice detail** — Subtotal and Tax visible and editable alongside Total, with currency formatting.
- **Supplemental documents on contracts** — Documents tab in contract detail. Upload Terms & Conditions, MSA, Schedules, Exhibits, Addendums.
- **GAP 1 — HR/Users CSV upload** — Upload HR CSV monthly → upserts `org_users` with cost_center, building, job_category, investment_strategy.
- **GAP 2 — Auto-advance billing period** — Closing a period automatically creates and activates the next month.
- **GAP 3 — FX/Currencies** — `exchange_rates` table + `user_cost_usd` in snapshots. Exchange rates entered monthly in Periods UI.
- **GAP 4 — Bulk subscription update** — Multi-select allocations → change billing account or billing dates for multiple users at once.
- **GAP 5 — Snapshot enrichment** — `cost_center` and `building` from `org_users` populated into snapshots at period close.
- **Dashboard redesign** — Full dashboard with KPI cards, renewals bar chart, Needs Attention panel, Auto-Renewals table, Top Vendors chart. All data scoped to org.
- **Sidebar navigation** — Dashboard as primary nav item. Processing tools (Invoice Processing, Contracts, Bloomberg Recon.) in a TOOLS section, visible only when modules are enabled for the user.
- **Reports section** — Renamed from Documents. Two tabs: External Documents Required + Renewal Calendar (all active contracts with action date, sorted by urgency).
- **Full-width layout** — All inventory pages now use full available width. No more fixed max-width constraints.

---

## What's in staging (not yet in production)

### CDR — Test client for CDNR demo prep
- User: `edbernal@cdr.com` / password: `12345`
- Org: CDR (`b986d4d7-ca78-4326-998e-56682352b0e2`)
- Only inventory module — no TOOLS section shows
- Created 2026-06-08 for testing the new dashboard and inventory flow

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
1. **Snapshot viewer UI** — browse closed period snapshots with matched/missing status per service
2. **Missing invoices view** — services with active subscriptions but no invoice in current period
3. **Cost per user view** — allocation_pct × invoice.total per user/service/period
4. **Vendor grouping** — group unmatched vendors by name when a new client uploads their first batch. Prompt ready.
5. **Bloomberg Terminal Upload (structured CSV import)** — separate upload type from core subscription PDFs. Bloomberg provides standardized CSVs (Dash 2, Dash 3 formats). Flow: HR file → Bloomberg CSV → create contract + allocations. UI: upload type dropdown.
6. **Spend/Inventory report** — monthly spend view: vendor → contracts → services → users/departments, with invoice adjustments and forecast.
7. **Bloomberg Terminal Reconciliation** — compare Bloomberg inventory files vs Source.

---

## What's in development

- **Vendor grouping for new clients** — Spec/prompt ready, implementation pending.
- **Monthly pricing normalization (P1)** — When contract states prices monthly, AI uses monthly price as annual value. Fix pending Santiago review.
- **Service split/merge UI** — Deferred. Waiting for Santiago validation.
- **`link_contract` UI** — Backend action live, UI trigger not built.
- **Allocations service name/annual_value editing** — The inline editing pattern around line 1898 in `InventoryContractDetail.tsx` still uses the old custom pattern (onBlur saves, no save/cancel buttons). Should be standardized to `InlineEditableField` like `ServiceBreakdownRow` was this session.

---

## Coming next (priority order for CDNR demo)

1. Snapshot viewer UI
2. Missing invoices view (services without invoice in current period)
3. Cost per user view
4. Renewal alerts (contracts approaching cancel_lead_time_days)
5. Bloomberg Terminal Upload (CSV-based, after HR file already live)
6. Spend/Inventory report with adjustments + forecast
7. Bloomberg Terminal Reconciliation (review with Santi)
8. Monthly pricing fix — after Santiago review

---

## Technical debt

- **Batch upload scalability** — In-process polling + sequential dispatch breaks at ~50 docs. See `.claude/memory/project_batch_scalability_debt.md`
- **Service split/merge UI** — See `.claude/memory/project_service_split_merge_debt.md`
- **Invoice variance/adjustments** — When invoice ≠ inventory expected amount, need adjustment records (Senthio: `Invoices_Adj`). Deferred post-MVP.
- **Soft/Hard dollar classification** — Senthio tracks Hard$ vs Soft$ per user. Deferred.
- **GL Accounts** — Senthio routes expenses to GL accounts via AccountMaps. Deferred.
- **Multi-year contract pricing** — `service.annual_value` is set at Year 1 price. For escalating multi-year contracts, user must manually update Annual Value each year. Full fix requires `ServicesFP` equivalent. See `.claude/memory/project_service_pricing_schedule_debt.md`
- **Allocations inline editing** — Custom onBlur pattern still in place for service name/annual_value in the allocations panel (`InventoryContractDetail.tsx` ~line 1898). Should use `InlineEditableField`.

---

## Environments

| | Supabase ref | Frontend |
|---|---|---|
| Production | `fdcxcivjhobreuseacot` | https://s4source.io |
| Staging | `fntpcrpmkwyruzplbewq` | https://s4sourceio.lovable.app |

---

## Recent changes (2026-06-08 session)

**Dashboard redesign — deployed to production:**
- Dashboard moved into the main inventory layout (sidebar now always visible)
- 4 KPI cards: Total Annual Value, Active Vendors, Renewals This Month, Expiring (90 days)
- Renewals bar chart showing next 12 months of contract renewals by annual value
- Needs Attention panel: vendor matches pending, contracts missing end dates, contracts flagged for review
- Auto-Renewals table: contracts with auto_renew=true, sorted by action date, color-coded urgency
- Top Vendors horizontal bar chart: top 8 vendors by annual contract value, truncated labels with hover tooltip

**Sidebar navigation overhaul — deployed to production:**
- Processing tools (Invoice Processing, Contracts, Bloomberg Recon.) moved from dashboard cards to a TOOLS sidebar section
- TOOLS section only shows when modules are enabled for the user; cached in localStorage to prevent flicker
- "Documents" nav item renamed to "Reports"

**Reports section — deployed to production:**
- New tab layout: "External Documents Required" (existing) + "Renewal Calendar" (new)
- Renewal Calendar shows all active contracts with action_date set, sorted ascending, action dates color-coded (red ≤14 days, amber ≤30 days)

**Layout — deployed to production:**
- Removed fixed max-width from all inventory pages (Inventory, Upload, Users, ContractDetail, VendorDetail, Documents, Processing, UploadDetail)
- InventoryPeriods kept at max-w-3xl (form page)

**Contract detail fixes — deployed to production:**
- Vendor name edit is now context-aware: updates `vendors.name` when vendor is confirmed (cascades to all linked contracts/invoices), updates `contracts.contract_vendor` text field when still unmatched
- `ServiceBreakdownRow` standardized to use `InlineEditableField`: removes onBlur auto-save bug, adds explicit Save/Cancel buttons, async-safe, draft syncs with prop changes
- `InventoryUsers` field typing fixed: `keyof OrgUserRow` throughout the editing chain (was `string`, causing TypeScript error)

**Staging — CDR test client created:**
- User: `edbernal@cdr.com` / `12345`, org CDR, inventory-only (no modules)
