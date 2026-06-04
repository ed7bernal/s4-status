# Source S4 — Product Status
*Last updated: 2026-06-04*

---

## What's live in production

- **Invoice Processing** — PDF upload → OCR → LLM extraction → vendor match → account match → service match → saved to DB with PDF stored securely.
- **Contract Processing** — PDF upload → OCR → LLM extraction → vendor match → services created → PDF stored.
- **Vendor Match Resolution** — Confirm vendor, create new, or link invoice to contract manually.
- **Reconciliation** — Matches processed invoices against expected charges.
- **Service Match** — Every invoice auto-scored and linked to best matching service after processing.
- **PDF Storage** — All PDFs in private Supabase Storage bucket, secure signed URLs on each record.
- **Inventory Upload (batch pipeline)** — Full batch pipeline live. Contract+invoice same-batch matching works end-to-end. Two-phase dispatch: contracts processed first, invoices after, fixing race condition.
- **Billing accounts** — Accounts flagged as billing accounts. Allocation rows show billing account dropdown. Inline account creation.
- **Contract detail editing** — Action Date, Total Contract Value, service name, annual value all editable inline.
- **Allocation user management** — Create users inline from allocation picker.
- **Vendor name normalization** — Handles `&`/`and`, strips legal suffixes.
- **PDF viewer** — Split-pane viewer in contract and invoice detail.

---

## What's in staging (not yet in production)

### Inventory Module — GAPs 1-4 implemented and validated

- **GAP 1 — HR/Users CSV upload** — Edge Function `upload-hr-csv` + RPC `reconcile_hr_csv`. Upload a CSV from HR each month → upserts `org_users` with cost_center, building, job_category, investment_strategy. Deactivation guard: if CSV has < 50% of active users, blocks deactivation and returns warning.
- **GAP 2 — Auto-advance billing period** — `close_billing_period()` now automatically creates and activates the next month's period when a period is closed.
- **GAP 3 — FX/Currencies** — `exchange_rates` table + `user_cost_usd` column in `inventory_period_snapshots`. Exchange rates entered monthly in the Periods UI; applied at period close.
- **GAP 4 — Bulk update subscriptions** — Multi-select users in the Allocations tab → change billing account or billing dates for N users at once via `bulk_update_subscriptions` RPC.
- **GAP 5 — Snapshot enrichment** — `cost_center`, `building`, `entity` columns added to snapshots, populated from `org_users` at period close.
- **Vendor backfill on contract approval** — When a contract is approved and a vendor is created, all unmatched invoices with the same vendor name are automatically linked to the new vendor.
- **Invoice amount match fix** — Amount badge now compares invoice subtotal (pre-tax) vs service annual_value, instead of invoice total vs contract total value. Fixes false Amount ✗ for EU/GBP vendors with VAT and for multi-year contracts.
- **Invoice detail shows subtotal + tax** — Subtotal and Tax fields now visible (and editable) alongside Total in the invoice review panel, with currency formatting.
- **Supplemental documents on contracts** — New Documents tab in contract detail. Upload Terms & Conditions, MSA, Schedules, Exhibits, Addendums, and other supporting docs. Multiple files per tag allowed. Documents uploaded from the global Documents page also appear here. Full review (PDF viewer) and delete support.

### E2E Test Dataset (HIG Testing org — staging)
- Org: `eb63c19f-a8dd-4f28-8638-b8c522fe4e18`
- 15 org_users loaded via HR CSV (real HIG employees from Senthio)
- 15 contracts uploaded and validated against Senthio data
- 21 invoices uploaded and matched
- Data covers monthly, quarterly, and annual billing frequencies; USD, GBP, EUR currencies

---

## Inventory Module — Senthio Feature Parity

### Goal
Replicate Senthio (H.I.G. Capital's Access DB) in Source so HIG can stop using Access. PoC target: **end of June 2026** — Ricky (internal user) using it for real feedback.

### Senthio reference
Full documentation: `.claude/skills/senthio-reference.md` (all 19 tables + queries + month-end close workflow)
Senthio DB version analyzed: 2026-06-02 (latest)

### Specs ready to implement
`.claude/specs/` — each spec has problem, what/how to build, risks, validation, UI spec.

| GAP | Spec | Status |
|---|---|---|
| GAP 1 | `gap1-hr-users-csv.md` | ✅ Implemented & validated |
| GAP 2 | `gap2-auto-advance-period.md` | ✅ Implemented & validated |
| GAP 3 | `gap3-fx-currencies.md` | ✅ Implemented & validated |
| GAP 4 | `gap4-billing-account-bulk-update.md` | ✅ Implemented & validated |
| GAP 5 | `gap5-snapshot-enrichment.md` | ✅ Implemented & validated (2026-06-04) |

### Next to build
1. **Vendor grouping** — when a new client uploads contracts + invoices with no vendors yet, the system should group them by extracted vendor name and present them as "Fitch Solutions — 1 contract + 1 invoice → Confirm?" instead of showing all invoices as "No vendor match". Prompt ready in PROJECT_STATUS context.
2. **Snapshot viewer UI** — browse closed period snapshots with matched/missing status per service
3. **Missing invoices view** — services with active subscriptions but no invoice in current period
4. **Cost per user view** — allocation_pct × invoice.total per user/service/period
5. **Bloomberg Terminal Reconciliation** — compare Bloomberg's inventory files vs Source. Review with Santi.

### Questions for Santi
1. ¿Confirmar que `billing_start_date`/`billing_end_date` en `service_subscriptions` son las fechas en que el usuario empezó/terminó de aparecer en la factura de esa cuenta? (GAP 4 — ya implementado como tal)

---

## What's in development

- **Vendor grouping for new clients** — see above. Spec/prompt ready, implementation pending.
- **Monthly pricing normalization (P1)** — When contract states prices monthly, AI uses monthly price as annual value. Fix pending Santiago review.
- **Missing External Document Contracts report** — Backend detection works, UI not built.
- **Service split/merge UI** — Deferred. Waiting for Santiago validation.
- **`link_contract` UI** — Backend action live, UI trigger not built.

---

## Coming next (after Vendor Grouping)

1. Snapshot viewer UI
2. Missing invoices view (services without invoice in current period)
3. Cost per user view
4. Renewal alerts (contracts approaching cancel_lead_time_days)
5. Bloomberg Terminal Reconciliation (review with Santi Wednesday)
6. Monthly pricing fix — after Santiago review
7. Merge feat-extraction-v2 to main

---

## Technical debt

- **Batch upload scalability** — In-process polling + sequential dispatch breaks at ~50 docs. See `.claude/memory/project_batch_scalability_debt.md`
- **Service split/merge UI** — See `.claude/memory/project_service_split_merge_debt.md`
- **Invoice variance/adjustments** — When invoice ≠ inventory expected amount, need adjustment records (Senthio: `Invoices_Adj`). Deferred post-MVP.
- **Soft/Hard dollar classification** — Senthio tracks Hard$ vs Soft$ per user. Deferred.
- **GL Accounts** — Senthio routes expenses to GL accounts via AccountMaps. Deferred.
- **Multi-year contract pricing** — `service.annual_value` is set at Year 1 price. For escalating multi-year contracts (e.g. Gartner: $64k/$66.6k/$69.3k), user must manually update Annual Value each year. Full fix requires `ServicesFP` equivalent. See `.claude/memory/project_service_pricing_schedule_debt.md`

---

## Environments

| | Supabase ref | Frontend |
|---|---|---|
| Production | `fdcxcivjhobreuseacot` | https://s4source.io |
| Staging | `fntpcrpmkwyruzplbewq` | https://s4sourceio.lovable.app |

---

## Recent changes (2026-06-03 session)

**Senthio full analysis completed:**
- Complete documentation of all 19 tables, 70+ queries, and month-end close workflow in `.claude/skills/senthio-reference.md`
- Verified against latest Senthio DB (2026-06-02): schema unchanged, Anthropic added as new vendor
- Feature parity analysis: 8 gaps identified, 5 specs written and reviewed by architecture + UX agents

**Inventory module GAPs 1-5 implemented and validated:**
- HR CSV upload with monthly reconciliation (deactivation guard, audit trail, ACID transaction)
- Billing period auto-advance after close
- FX/Currency exchange rates with USD normalization
- Bulk update of billing accounts and dates for multiple users
- Snapshot enrichment with cost_center and building from org_users

**Vendor backfill fix:**
- When a contract is approved, all invoices with matching vendor name are now automatically linked — no more manual SQL needed

**E2E test dataset prepared:**
- HIG Testing org created in staging with 15 real HIG users (from Senthio HR data)
- 15 contracts + 21 invoices uploaded and cross-validated against Senthio
- Contract data completed using Senthio as source of truth (bill_frequency, cancel_lead_time_days, renewal terms)

**Amount match fixes:**
- Invoice subtotal (pre-tax) now used instead of total for Amount ✓/✗ badge
- Comparison against service.annual_value instead of contract total value
- Subtotal and Tax now visible in invoice detail panel with currency formatting
- Applied in both InventoryContractDetail and InventoryUploadDetail

**Pending:**
- Vendor grouping for new clients (prompt ready, needs implementation in new chat)
- feat/gap1-hr-users branch has all frontend changes — needs merge to main
