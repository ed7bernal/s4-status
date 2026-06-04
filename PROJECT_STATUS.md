# Source S4 — Product Status
*Last updated: 2026-06-04*

---

## What's live in production

- **Invoice Processing** — PDF upload → OCR → LLM extraction → vendor match → account match → service match → saved to DB with PDF stored securely.
- **Contract Processing** — PDF upload → OCR → LLM extraction → vendor match → services created → PDF stored. Now derives `annual_value` automatically from extracted contract value.
- **Vendor Match Resolution** — Confirm vendor, create new, or link invoice to contract manually. Vendor backfill: approving a contract auto-links all invoices with matching vendor name.
- **Reconciliation** — Matches processed invoices against expected charges.
- **Service Match** — Every invoice auto-scored and linked to best matching service after processing.
- **PDF Storage** — All PDFs in private Supabase Storage bucket, secure signed URLs on each record.
- **Inventory Upload (batch pipeline)** — Full batch pipeline live. Contract+invoice same-batch matching works end-to-end.
- **Billing accounts** — Accounts flagged as billing accounts. Allocation rows show billing account dropdown. Inline account creation. Fixed: only accounts belonging to the contract's vendor are shown (global accounts no longer appear as invalid options).
- **Contract detail editing** — Action Date, Total Contract Value, Annual Value, service name all editable inline.
- **Allocation user management** — Multi-select modal: add multiple users in one batch with evenly redistributed allocations. "Select all" support. Bulk change billing account and billing dates.
- **Vendor name normalization** — Handles `&`/`and`, strips legal suffixes.
- **PDF viewer** — Split-pane viewer in contract and invoice detail.
- **Invoice detail** — Subtotal and Tax visible and editable alongside Total, with currency formatting. Amount badge compares invoice subtotal (pre-tax) vs service annual_value.
- **Supplemental documents on contracts** — Documents tab in contract detail. Upload Terms & Conditions, MSA, Schedules, Exhibits, Addendums. Multiple files per tag. Global Documents page documents also appear here.
- **GAP 1 — HR/Users CSV upload** — Upload HR CSV monthly → upserts `org_users` with cost_center, building, job_category, investment_strategy. Deactivation guard prevents accidental mass deactivation.
- **GAP 2 — Auto-advance billing period** — Closing a period automatically creates and activates the next month.
- **GAP 3 — FX/Currencies** — `exchange_rates` table + `user_cost_usd` in snapshots. Exchange rates entered monthly in Periods UI; applied at period close.
- **GAP 4 — Bulk subscription update** — Multi-select allocations → change billing account or billing dates for multiple users at once.
- **GAP 5 — Snapshot enrichment** — `cost_center` and `building` from `org_users` populated into snapshots at period close.

---

## What's in staging (not yet in production)

### E2E Demo Dataset (HIG Testing org — staging only)
- Org: `eb63c19f-a8dd-4f28-8638-b8c522fe4e18`
- Reset to clean state for Santi demo: 0 vendors, 0 allocations, 0 snapshots
- 15 contracts (Draft, unmatched) + 24 invoices (pending) ready to approve from scratch
- 15 org_users from Senthio HR data, cost_center and building populated
- Invoices pre-linked to July 2026 billing period for post-approval close demo
- HR CSV at `~/Downloads/hig-test-hr.csv`

---

## Inventory Module — Senthio Feature Parity

### Goal
Replicate Senthio (H.I.G. Capital's Access DB) in Source so HIG can stop using Access. PoC target: **end of June 2026** — Ricky (internal user) using it for real feedback.

### Senthio reference
Full documentation: `.claude/skills/senthio-reference.md` (all 19 tables + queries + month-end close workflow)
Senthio DB version analyzed: 2026-06-02 (latest)

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
5. **Bloomberg Terminal Reconciliation** — compare Bloomberg's inventory files vs Source.

### Questions for Santi
1. ¿Confirmar que `billing_start_date`/`billing_end_date` en `service_subscriptions` son las fechas en que el usuario empezó/terminó de aparecer en la factura de esa cuenta?

---

## What's in development

- **Vendor grouping for new clients** — Spec/prompt ready, implementation pending.
- **Monthly pricing normalization (P1)** — When contract states prices monthly, AI uses monthly price as annual value. Fix pending Santiago review.
- **Missing External Document Contracts report** — Backend detection works, UI not built.
- **Service split/merge UI** — Deferred. Waiting for Santiago validation.
- **`link_contract` UI** — Backend action live, UI trigger not built.

---

## Coming next

1. Snapshot viewer UI
2. Missing invoices view (services without invoice in current period)
3. Cost per user view
4. Renewal alerts (contracts approaching cancel_lead_time_days)
5. Bloomberg Terminal Reconciliation (review with Santi)
6. Monthly pricing fix — after Santiago review

---

## Technical debt

- **Batch upload scalability** — In-process polling + sequential dispatch breaks at ~50 docs. See `.claude/memory/project_batch_scalability_debt.md`
- **Service split/merge UI** — See `.claude/memory/project_service_split_merge_debt.md`
- **Invoice variance/adjustments** — When invoice ≠ inventory expected amount, need adjustment records (Senthio: `Invoices_Adj`). Deferred post-MVP.
- **Soft/Hard dollar classification** — Senthio tracks Hard$ vs Soft$ per user. Deferred.
- **GL Accounts** — Senthio routes expenses to GL accounts via AccountMaps. Deferred.
- **Multi-year contract pricing** — `service.annual_value` is set at Year 1 price. For escalating multi-year contracts, user must manually update Annual Value each year. Full fix requires `ServicesFP` equivalent. See `.claude/memory/project_service_pricing_schedule_debt.md`

---

## Environments

| | Supabase ref | Frontend |
|---|---|---|
| Production | `fdcxcivjhobreuseacot` | https://s4source.io |
| Staging | `fntpcrpmkwyruzplbewq` | https://s4sourceio.lovable.app |

---

## Recent changes (2026-06-04 session)

**GAPs 1-5 + all inventory features deployed to production:**
- 29 migrations applied to production (HR CSV, auto-advance, FX rates, bulk subscriptions, snapshot enrichment, annual value, supplemental docs)
- 7 edge functions deployed: `upload-hr-csv` (new), `close-billing-period`, `resolve-vendor-match`, `process-contract`, `process-inventory-upload`, `process-inventory-document`, `reconcile-inventory-batch`
- Frontend branch `feat/gap1-hr-users` merged to `main` and deployed via Lovable
- Staging and production confirmed in full schema parity (372 columns, 51 RPCs — identical)

**Billing account dropdown fix (production):**
- Dropdown now only shows accounts belonging to the contract's vendor
- Global accounts (vendor_id IS NULL) no longer appear as options when a vendor exists — they fail the trigger anyway
- "+ New account" button always visible alongside the dropdown, not only when list is empty

**E2E demo dataset prepared for Santi meeting (staging):**
- HIG Testing org reset to clean state: vendors, allocations, snapshots, billing periods, org_users all cleared
- Contracts reset to Draft/unmatched, invoices reset to pending
- 24 invoices pre-linked to July 2026 billing period for post-approval snapshot demo
- `hig-test-hr.csv` at `~/Downloads/` ready to upload

**Production validated end-to-end:**
- Gain.pro contract + invoice uploaded and processed correctly in production
- `annual_value` derived automatically from contract value ✅
- Vendor name normalization (Gain.AI → Gain.pro) working ✅
- Billing period close generated snapshots with FX conversion (GBP → USD) ✅
- Auto-advance created July period after June close ✅
