# Source S4 — Product Status
*Last updated: 2026-06-09 (security hardening + production deploy)*

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
- **Reports section** — External Documents Required, Renewal Calendar, Period Snapshots with cost columns.
- **Supplemental document flags auto-clear** — Typing in a contract value that was flagged as missing auto-marks it resolved.
- **Full-width layout** — All inventory pages use full available width.
- **Invoice delete** — Users can delete a wrongly-assigned invoice and re-upload it to the correct contract.
- **Billing period snapshot cost fix** — Avg Monthly uses `unit_cost ?? annual_value/12` so user_cost is never null when annual_value is set.
- **Rate-limit retry hardening** — Batch upload dispatch retries on 429s and thrown rate-limit exceptions with jitter.
- **linked_contract_id fix** — Invoice items in a batch now correctly link to their matched contract after reconciliation.
- **Batch-scoped contract matching** — Invoices only match contracts from the same batch. No cross-batch false positives.
- **Security hardening (P1–P11)** — Internal fields stripped from API responses, legacy RLS policies removed, UUID validation added to Edge Functions, org_id indexes added, audit trail hardened, client_modules RLS fixed. See Recent Changes for full list.

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
3. **Vendor grouping** — group unmatched vendors by name when a new client uploads their first batch
4. **Bloomberg Terminal Upload (structured CSV import)** — separate upload type from core subscription PDFs
5. **Spend/Inventory report** — monthly spend view: vendor → contracts → services → users/departments
6. **Bloomberg Terminal Reconciliation** — compare Bloomberg inventory files vs Source

---

## What's in development

- **Multi-org architecture** — Consultants need to belong to multiple orgs (e.g. HIG and CDNR) and switch active client without re-login. Design planned but not started. See prompt in memory for next chat.
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

- **Multi-org user model** — Current model is 1 user = 1 org. Consultants need N orgs. Full redesign needed: `user_org_memberships` table, RLS overhaul, `active_org_id` in request headers, all Edge Functions updated. See `.claude/memory/project_multiorg_debt.md`
- **Batch upload scalability** — In-process polling + sequential dispatch breaks at ~50 docs. See `.claude/memory/project_batch_scalability_debt.md`
- **Service split/merge UI** — See `.claude/memory/project_service_split_merge_debt.md`
- **Invoice variance/adjustments** — When invoice ≠ inventory expected amount, need adjustment records (Senthio: `Invoices_Adj`). Deferred post-MVP.
- **Soft/Hard dollar classification** — Senthio tracks Hard$ vs Soft$ per user. Deferred.
- **GL Accounts** — Senthio routes expenses to GL accounts via AccountMaps. Deferred.
- **Multi-year contract pricing** — `service.annual_value` is set at Year 1 price. Full fix requires `ServicesFP` equivalent. See `.claude/memory/project_service_pricing_schedule_debt.md`
- **Allocations inline editing** — Custom onBlur pattern still in place in `InventoryContractDetail.tsx` ~line 1898. Should use `InlineEditableField`.
- **Pre-existing TypeScript errors in InventoryUploadDetail.tsx** — `contract_id` not in `ContractData` type (line 849+), several unused state vars. Not causing runtime issues.

---

## Environments

| | Supabase ref | Frontend |
|---|---|---|
| Production | `fdcxcivjhobreuseacot` | https://s4source.io |
| Staging | `fntpcrpmkwyruzplbewq` | https://s4sourceio.lovable.app |

---

## Recent changes (2026-06-09 session — security hardening)

**Security hardening — deployed to production (P1–P11):**

- **API response filtering (P1, P2)** — Internal fields like raw extraction data, confidence scores, PDF hashes, and processing logs are now stripped from API responses before reaching the frontend. These were never meant to be client-visible.
- **Legacy RLS policies removed (P3)** — Dropped 8 old database access policies on invoices, accounts, and vendors that used the user's individual ID instead of the organization ID. These were safe today but would have exposed cross-org data in a multi-user setup.
- **Input validation hardened (P5, P6)** — Two backend functions that accept IDs from internal callers now validate that those IDs are properly formatted UUIDs before using them in database queries.
- **Dead error path removed (P7)** — Removed a code branch in vendor matching that could never be reached but would have returned an error message revealing org ownership information if it somehow was.
- **Database indexes added (P8)** — Added missing indexes on the org_id column for two frequently-queried tables (vendors, inventory_uploads). Queries that filter by organization are now faster.
- **Audit trail hardened (P9)** — The billing period close function now uses the session's verified user identity for the audit log instead of trusting a user ID passed in the request body. A direct API call can no longer forge who closed a period.
- **User/org cross-check added (P10)** — Batch document processing now verifies that the user attributed to a document actually belongs to the organization being processed. Prevents mis-attribution by a misconfigured internal caller.
- **Access policy standardized (P11)** — The client_modules table had an access policy that checked org membership via an indirect user_id join instead of the org_id column directly. Standardized to match all other tables.

**Performance — deployed to production (P12):**
- **Upload Results polling reduced ~92%** — The Upload Results page was re-fetching 8–10 database queries every 5 seconds while a batch was processing, regardless of whether anything had changed. Now polls every 10 seconds and skips the heavy queries entirely when item statuses haven't changed since the last check. Validated via DevTools: 113 requests for a 5-minute batch vs ~660 estimated before the fix.

**E2E validation — confirmed on staging:**
- Batch upload → polling behavior correct (fingerprint skip working)
- Confirm contract → single Edge Function call, no polling
- Allocation → direct CRUD, no polling
