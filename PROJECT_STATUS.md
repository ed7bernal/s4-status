# Source S4 — Product Status
*Last updated: 2026-08-21 (dataset demo ficticio en staging + proceso de entrega revisado + CI en ambos repos; sin cambios de producto — ver Recent changes)*

---

## What's live in production

- **Inventario manual (vendor-first), desplegado 2026-08-20** — nueva RPC `create_manual_vendor(org, nombre, force)` y botón **"Add vendor"** en Inventory: crea el vendor con su **contrato y servicio placeholder** de una vez, sin necesidad de ningún documento. Resuelve el caso de onboarding en el que un cliente llega solo con facturas y no aparece en ninguna pantalla de Inventory (Vendors agrupa por contrato, Documents consulta `contracts`, y `get_billing_invoices` exige `service_match_status='matched'`, que a su vez necesita un `service`). El servicio placeholder es lo que permite que una factura enganche: sin él, `match-invoice-service` sale con *"no services found for vendor"*.
  **Avisos en dos niveles, ninguno bloqueante** — el sistema no puede saber si dos entidades con nombres parecidos son la misma empresa: `same_name` (normaliza igual, p. ej. `BLOOMBERG FINANCE` vs `Bloomberg Finance L.P.`) y `similar` (candidato de `match_vendor` ≥ 0.5), ambos saltables con confirmación; `exact_name_exists` es el único sin salida y lo impone el índice único. Se descartó el bloqueo duro al medir que `normalize_vendor_name('ICE Data LLC')` y `('ICE Data Inc.')` dan las dos `ice data` y pueden ser entidades legales distintas.
  **`GRANT EXECUTE ON match_vendor TO authenticated`** — el camino manual usa el mismo matcher que el automático en vez de copiar su lógica (una cuarta implementación condenada a divergir). Seguro por `can_access_org()`, endurecida en G-29; verificado en producción que anon recibe `42501` y que un usuario contra un org del que no es miembro recibe `unauthorized`.
  **Sin backfill de facturas huérfanas, a propósito**: subir la factura desde la ficha manda `contract_id`, que se mapea a `linked_contract_id`, y entonces `process-inventory-document:903` ni llama a `match-invoice-service` — la factura nace enganchada, y toma `billing_period_id` del período abierto en la ingesta.
  **Además, un arreglo sin relación con la feature**: el icono de borrar servicio pasa a deshabilitado con tooltip cuando el servicio tiene allocations o facturas. `services.delete` era un `DELETE` pelado; producción tiene **13 facturas** ya en estado incoherente por eso (ver `TECH_DEBT.md` S-2). Y la pestaña Invoice del contrato ahora muestra el aviso de mismatch que el worker escribía y ninguna pantalla leía.
  **Un enfoque previo *contract-first* se construyó, se probó y se descartó** tras un code review acotado que encontró 5 problemas reales (un backfill que robaba facturas de otros contratos del vendor, unicidad byte-exacta esquivable, guarda de idempotencia inútil…). El rediseño vendor-first eliminó 3 de los 5 en vez de parchearlos. Deuda derivada documentada en `TECH_DEBT.md` **S-2 a S-8**; el diseño de la fase 2 (adoptar la ficha cuando llegue el PDF real) en `docs/specs/manual-contract-adopt-phase2.md`.
  ⚠️ **`client 1` sigue sin resolverse**: sus 4 vendors ya existen sin contrato, así que "Add vendor" los rechaza con `same_name` — es el caso S-7, pendiente.

- **Inventory Upload — block invoice-only batches (R-053), pushed 2026-07-22** — the general `/inventory/upload` page now refuses invoice-only batches (button disabled, message points to Billing → Upload Invoices instead); contract-only and contract+invoice batches unaffected. Frontend-only — the shared `process-inventory-upload` Edge Function is untouched since `BillingInvoiceUpload.tsx` legitimately still submits invoice-only batches through it. Edgar validated on staging before push. **Pushed to `s4sourceio` `main` — Lovable Publish not confirmed in this session, check before assuming it's live.**
- **Reports Overview cross-filter + monthly pivot table (R-049), deployed 2026-07-20** — clicking a vendor bar, department slice, or pivot row now filters every other visual on the Overview tab to that entity (click again to clear); a chart whose own dimension matches the active filter self-highlights instead of collapsing to one bar. New monthly pivot table (rows = vendor or department, columns = months in the selected range, sticky name column) reuses `report_spend_aggregate`'s existing `group_by='month'`+`split_by` shape — **zero migration or RPC changes**, verified byte-identical on staging/production before building. Vendors/Users tabs (`ReportsDimension.tsx`) also gained a richer drill-down panel (monthly trend chart, summary metrics, tabbed breakdown) above the table when a row is selected. Modeled after a reference Power BI dashboard Santiago referenced; dual-axis combo charts and full Power-BI-style page-wide cross-filter were deliberately not replicated (see R-049 in the backlog for the analysis). Verified against direct RPC ground truth on staging (HIG Testing — Capital Economics vendor filter and "Unallocated" user filter both matched to the cent).
- **Admin page (R-029), deployed 2026-07-15** — S4-staff-only `/admin` route (gated by new `profiles.is_s4_staff`, not `role` or email domain — both were found unreliable): Organizations tab (create org + module checkboxes, member counts, delete with a zero-data guard across 8 tables), Modules tab (toggle `client_modules` per org), Members tab (invite by email via `auth.admin.inviteUserByEmail`, list members, change role, block demoting an org's last admin). 6 new Edge Functions. Replaces the old manual-SQL `client_modules` workflow entirely.
- **Ask AI POC (R-033), deployed 2026-07-17** — New "Ask AI" tab in Reports. New `ask-report-ai` Edge Function calls Anthropic's Messages API with tool-use, restricted to calling the existing `report_spend_aggregate` RPC only (never raw SQL, never lets the model compute its own sums) — same OLTP/OLAP reasoning as the rest of Reports: `inventory_period_snapshots` already is the analytical layer, no separate warehouse needed at this scale. Single-turn Q&A (chat-style multi-turn deliberately deferred). Rate-limited 20 questions/org/day via new `ask_ai_queries` table (doubles as audit trail). Validated against real data on both staging (HIG Testing) and production (S4 Market Data) — answers matched the RPC ground truth exactly.
- **Billing/IRW fixes (R-044, R-039, R-027), deployed 2026-07-17** — R-044: same-vendor contract disambiguation in orphan-invoice matching now breaks name-similarity ties using billing-date overlap (fixed a real Third Bridge mismatch). R-039: `get_invoice_reconciliation`/`create_invoice_distributions`'s account-based service inclusion now requires `annual_value > 0`, excluding unpriced/unallocated placeholder services (fixed on Mimecast, regression-checked against the legitimate Third Bridge cross-contract case); also fixed "+Add Service" showing with nothing addable. R-027: a new trigger syncs `invoice_adjustments` into `inventory_period_snapshots` when the adjustment targets an already-closed period — mirrors what Senthio does manually via `Adj Inv`/`Adj Year`/`Adj Month`; deliberately did **not** add automatic date-to-period inference elsewhere (tried it, reverted — that's a human judgment call there too).
- **Vendor matching fix (R-046), deployed 2026-07-15** — `match_vendor`'s first-token signal used fuzzy similarity, treating any shared first word the same whether generic ("The", "Capital") or distinctive ("Clarksons", "ICE") — caused 11 real false-positive matches in production. Now requires an exact first-token match gated by a new `vendor_name_stopwords` table, or full-name similarity ≥ 0.5.
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

### S4 Market Data - Demo — dataset 100% ficticio (staging only, 2026-08-21)
- Org: `56b7eba9-f0af-411f-8b88-f4faed18835c` — *S4 Market Data - Demo*
- **Para qué existe**: mostrarle el producto en vivo a un cliente potencial y alimentar el proyecto de video. No clona ni reutiliza datos de ningún cliente real, a diferencia de `scripts/demo-seed/`, que clonaba HIG.
- **Producción está fuera de alcance por decisión explícita.** Los guards de los scripts lo imponen: abortan si no ven el marcador de staging o si ven el de prod. Cuando se decida cargar prod, se cambian conscientemente y en su propio commit.
- **Contenido**: 10 vendors ficticios (4 compartidos entre 2-3 departamentos, 6 exclusivos), 14 contratos, 18 servicios, 30 `org_users` de padrón, 78 `service_subscriptions`, 8 períodos (ene–jul 2026 cerrados + agosto abierto), 82 facturas, 427 snapshots, 409 distribuciones, **96 PDFs ficticios en Storage (cobertura completa: los 14 contratos y las 82 facturas)**. Valor anual activo $2.434.200, repartido entre los 5 departamentos.
- **Accesos**: solo Ed (admin). Santi no existe en staging y no lo necesita. No se crea ningún usuario: el paso 1 solo inserta la membresía resolviendo el `user_id` por email.
- **Cabos sueltos deliberados**, para que haya algo que resolver en cámara: 4 facturas con variance exacta (1875 / 900 / 3200 / 640), 1 caso ya resuelto con ajuste retroactivo, 2 facturas con `service_match` pendiente (que por eso no aparecen en Billing — `get_billing_invoices` filtra por `matched`), 9 snapshots `missing_invoice`, 2 contratos con vendor sin confirmar, 1 sin fecha de fin.
- **Scripts**: `scripts/demo-seed-fictional/`, con README. Idempotentes, con guard de entorno, y `00_revert.sql` + `00_revert_storage.mjs` probados **sobre la org ya poblada** — borran todo incluida la org y dejan las otras 5 orgs con sus conteos intactos.
- **Verificado**: las 84 facturas originales con `net_variance` 0.0000 por los dos caminos de `get_invoice_reconciliation` (snapshots congelados y suscripciones vivas); las signed URLs resuelven sin autenticación devolviendo `application/pdf`; validación visual de Edgar el 2026-08-21.
- **Cobertura de PDFs completa desde 2026-08-25.** No queda ningún "Review with PDF" deshabilitado ni ningún panel gris en el modal de comparación. El paso 9 selecciona por hueco (`pdf_url IS NULL`), así que es reanudable y volver a correrlo no repite nada.
- **El paso 9 no usa service key**: las policies ya autorizan a cualquier miembro de la org a escribir en `documents/<org_id>/…` y a actualizar `pdf_url`, así que va con un `access_token` de usuario. Demostrado con escrituras reales, no leyendo `pg_policy`.
- ⚠️ **Pendiente si se lleva a producción**: volver a medir los nombres de vendor contra el padrón de prod (que suma los 42 de TRG). El par más cercano al umbral en staging fue *Ironwood Reference Data Ltd.* ~ *ICE Data Pricing & Reference Data, LLC*, 0.421 contra un umbral de 0.5.

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

- **Grupo C — validating Source against Senthio (test data on staging only)** — HIG Testing's test data was wiped and rebuilt with 23 real contracts + invoices, matched vendor-by-vendor and service-by-service against Senthio's own records so the two systems can be compared apples-to-apples. Along the way this surfaced and fixed 3 real calculation bugs — **those fixes reached production on 2026-07-12**, but the rebuilt test data itself and the 23-invoice comparison remain staging-only. 17 of the 23 test invoices are now fully approved; 6 are deliberately on hold pending Santiago's input on how unusual billing situations (one-time charges, usage-based vendors, hardware purchases) should be handled — see "Where your input would help." The real cent-by-cent comparison against Senthio's own records for June still hasn't been done (see "Coming next").
- **Vendor grouping for new clients** — Spec/prompt ready, implementation pending.
- **Monthly pricing normalization (P1)** — When contract states prices monthly, AI uses monthly price as annual value. Fix pending Santiago review.
- **Service split/merge UI** — Deferred. Waiting for Santiago validation.
- **`link_contract` UI** — Backend action live, UI trigger not built.
- **Not a match button** — Hidden for now. Flow and requirements need validation before re-enabling.

---

## Coming next (priority order)

*Re-sequenced 2026-07-20; partially superseded since — see the note below before trusting this list at face value.*

**Since 2026-07-20:** the Santiago call happened and produced a whole new shipped feature (R-050, Billing Bulk Invoice Upload, deployed 2026-07-21) plus its follow-ups R-052 through R-059 — none of that is reflected in the numbered list below yet, since it wasn't sequenced into it. Check `PRODUCT_REVIEW_BACKLOG.md` directly for the current state of R-050–R-059 rather than trusting this section for that range. Notably: **R-055 (shared billing-account over-inclusion) is now closed** (2026-07-23, resolved by splitting the account per-service, no code needed) — R-054 (auto-populate `billing_account_id` on high-confidence matches) is no longer blocked by it. R-052/R-053/R-054 have no open dependencies; R-053 shipped this session (see Recent changes above), R-052/R-054 are still unbuilt.

1. **Santiago call happened 2026-07-20** — transcript to be processed in a separate session (backlog items TBD from that conversation, not yet captured here). R-049 was built same-day partly in response to a reference dashboard Santiago shared before the call.
2. **Unblocked, can start now:**
   - R-052 — no way back into a Billing invoice-upload batch's results page once you navigate away.
   - R-054 — auto-populate `billing_account_id` when the account match is already high-confidence (`exact`/`single_account`); no longer blocked by R-055.
   - R-048 — full-name-similarity false positives in vendor matching not fixed by R-046 (e.g. "Real Deals"/"The Real Deal" 0.571) — deliberately deferred pending confirmation of real-world impact.
   - App-wide E2E pass (started 2026-07-14) — still ongoing; findings continue to flow through `/app-review` into the backlog.
   - Align R-031 phase 3+ (Budget vs Actuals, vendor deep-dive) scope — informed by whatever comes out of the Santiago transcript above, don't build ahead of it.
3. **Blocked:**
   - R-028 (edit archived-period metadata) — depends on R-003 (reopen a closed period), which depends on Edgar's still-pending conversation with his manager about whether closed periods should be editable at all. Real operational cost of not having this: two separate sessions now (Grupo C on 2026-06-07, and this session's R-055 dry-run on 2026-07-22) had to hand-reset a closed period directly via SQL because no product feature exists for it.
4. **Data work, not product:**
   - Deep Tree data cleanup + Monthly pricing normalization (P1) — close out with Santiago (the 2026-07-02 call confirmed the expected-value derivation, just needs the staging data cleaned).
   - Grupo C — line-by-line dollar comparison vs Senthio's `Inventory_R`/`Invoices_Adj` for June, plus the "missing invoices in an already-closed period" report (R-027's mechanism now supports this).
5. **Backlog, not yet scoped:** Cost per user view · Renewal alerts (`cancel_lead_time_days`) · Bloomberg Terminal Upload (CSV) · Spend/Inventory report with forecast · Bloomberg Terminal Reconciliation · Ask AI chat-style multi-turn memory (deliberately deferred, see R-033).
6. **Small loose ends from recent sessions:**
   - Orphan `invoice_distributions` row for GLG on staging (leftover from Grupo C data prep) — decide whether to clean up.
   - Confirm whether "Processing History" (the old standalone Invoice Processing module) is still an active flow — determines whether `InvoiceDetail.tsx` stays dual-purpose or `/invoices` gets deprecated.
   - ~~Anthropic API key now live as an `ANTHROPIC_API_KEY` secret on both staging and production — consider rotating~~ — now formally tracked as **R-058** (found identical on both environments during this session's parity audit; Edgar's call is to migrate to OpenAI rather than just separate the Anthropic keys).

**R-003 note (still relevant):** the Grupo C session had to hand-reset HIG Testing's June period directly in the database (no product feature exists for this) to redo the close live with Santiago — it's a real operational gap, not just a hypothetical, and a hard dependency for R-028. Raise it with him directly.

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
- ~~**Pre-existing TypeScript errors**~~ — ✅ Done (R-032, 2026-07-16). All 8 fixed; 2 were real bugs (not just type gaps), see Recent changes.
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

## Recent changes (2026-08-21 session — proceso de entrega revisado, script del Step 5.2, CI en ambos repos)

**Sin cambios de producto.** Toda la sesión fue sobre *cómo* se entrega, no sobre qué se entrega. No se desplegó ninguna Edge Function ni migración.

**Diagnóstico que la originó.** Un invite fallido a un cliente nuevo en producción (Fortress Beacon) resultó ser un typo de dominio — `s4maketdata.com` sin la `r`, sin MX ni A — que `invite-user` presentaba como un 500 genérico. De ahí salieron **G-68** (producción no tiene SMTP propio: `smtp_host = null`, `rate_limit_email_sent = 2`, afecta a todo el email transaccional y es bloqueante para onboarding real de un cliente) y **G-69** (el 500 opaco esconde `email_address_invalid`). Ninguno de los dos arreglado todavía.

**El proceso de entrega, evaluado contra práctica de continuous delivery.** Lo que ya estaba bien no se tocó: migraciones como código sin drift (215 archivos / 215 en staging / 215 en producción), staging antes de producción, lotes chicos, trazabilidad. Lo que estaba mal era **el propio documento**: `how-we-build.md` Step 5 y `CLAUDE.md` regla 3 decían commitear *después* de desplegar a producción, contradiciendo a la regla 1 del mismo CLAUDE.md. Los dos modos de falla que eso invita ya habían pasado acá (`match-invoice-service` corriendo 6 semanas desde código inexistente en git; G-28 revirtiendo el workstream TRG en ambos entornos). Se corrigió a **commit antes del deploy**, y se agregaron **Step 7 (verificar en producción)** y una sección de **Rollback**, ninguno de los cuales existía. `how-we-build.md` quedó como una secuencia numerada de 10 pasos sin decisiones abiertas.

**`scripts/diff-live-functions.sh`** — el Step 5.2 (diffear el bundle vivo antes de desplegar) era obligatorio, el más caro de los 10, y tenía un footgun: `supabase functions download` escribe sobre `./supabase/functions/<slug>/` y pisa el fuente propio. Ese perfil es el de una regla que se termina salteando, y una regla salteada es peor que ninguna porque el documento afirma una cobertura que no existe. El script baja a un workdir temporal y **también diffea los helpers de `_shared/`** — no es un extra: G-41 cambió un helper compartido, y un diff por `index.ts` habría estado ciego justo ahí.

**Resultado de correrlo, que cierra un punto que estaba abierto:** las **24 bundles vivas son idénticas byte a byte al fuente local en staging Y producción**, `_shared` incluido. Hasta ahora la paridad se infería de timestamps de deploy; ahora está medida.

**CI, por primera vez (G-70, cerrado).** Había 200 tests pasando en el backend que nadie corría en un push. `.github/workflows/ci.yml` en ambos repos: `s4-backend` = `deno check` sobre las 24 funciones + `deno test` (~2m40s); `s4sourceio` = `tsc --noEmit -p tsconfig.app.json` + `vitest` (~35s). **Verificado en las dos direcciones**, no sólo en verde: se pusheó un test roto a propósito → CI en rojo; se borró → verde. Un gate que sólo se vio pasar no está probado.
**Lo que CI no compra, anotado explícitamente en el Step 3 para que no se malinterprete:** no encuentra la clase de bug que este proyecto tiene. G-41, G-59, G-67 y G-69 fueron de lógica y de permisos, y los encontró revisión humana. `/code-review` sigue siendo el paso que más importa; CI garantiza que la parte mecánica ocurrió y evita que los tests se pudran sin correr.
**`eslint` quedó deliberadamente afuera** — 237 errores hoy, sería rojo permanente, y un gate siempre en rojo se ignora junto con los rojos legítimos (**G-71**, abierto).

**Higiene de ramas.** Se borraron **11 refs remotas** de sesiones anteriores (`g41`, `g50`, `g50b`, `g52`, `g52b`, `g60`, `g61`, `g66`, `g66b`) más las 3 de esta sesión. Todas verificadas como ancestros de `main` antes de borrar — cero commits perdidos, sus SHA quedan en el historial de `main`. No era cosmética: una rama vieja que está *detrás* de lo que está vivo es exactamente el material de **G-28**, donde desplegar desde `main` desactualizado revirtió en silencio el workstream TRG en ambos entornos. **Ambos repos quedan con sólo `main`**, local y remoto.

⚠️ **Salvedad honesta: el proceso todavía no se corrió entero.** Estos cambios ejercitaron los Steps 0-4 y 8-10, pero **los Steps 5, 6 y 7 — deploy a staging, a producción y verificación — siguen sin usarse nunca**, porque ninguno despliega nada. El primer cambio real de backend será el estreno de verdad, y es esperable que ahí aparezcan roces.

---

## Recent changes (2026-07-22 session — R-053 shipped, backlog hygiene, full parity audit, live R-055 dry-run)

**R-053 — block invoice-only uploads on the general Inventory Upload page.** Now that Billing has its own dedicated invoice-only entry point (R-050), the general `/inventory/upload` page no longer accepts invoice-only batches — contract-only and contract+invoice-together still work exactly as before. Frontend-only change (`InventoryUpload.tsx`); validated on staging, pushed to production.

**Backlog hygiene, three small gaps closed:**
- A migration from the R-057 session (`20260721000002`, adds `billing_period_id` to `get_billing_invoices`) had been applied to both staging and production via `supabase db push` but never actually committed to git — verified byte-identical against both live databases before committing it, so this is a paperwork fix, not a re-deploy.
- The R-018 backlog row (Security & Compliance) still said "Triaged," but that work has long since moved to `docs/security/security-compliance-roadmap.md` and is nearly all done there — row updated to point at the roadmap instead of duplicating stale status.
- Logged **R-058**: found during the parity audit below that `ANTHROPIC_API_KEY` (used by Ask AI) is the exact same key on staging and production — Edgar's own personal key, used as a placeholder when R-033 was built. Edgar's call: migrate Ask AI from Anthropic to OpenAI (not just get separate Anthropic keys) — deliberately deferred, not scoped yet.

**Full staging vs. production parity audit** — the "deep" version of the Step 6 session-close check (hash-diffing every live object, not just migration history), last run 2026-07-12. Compared: all tables/columns, RLS policies, indexes, triggers, extensions, grants, and every function's source (hash-diffed) — all identical. Downloaded and hash-compared the actual deployed bundle for all 20 Edge Functions between staging and production — byte-for-byte identical despite staging/production having different internal version counters (explained by production accumulating extra deploys before staging existed as a separate project). Confirmed no unpushed commits in either repo. Only real gap found: R-058 above.

**Live dry-run of a Billing bulk-invoice batch on staging (HIG Testing), ahead of replicating it with Santiago.** Edgar uploaded a 5-invoice July batch twice (first run reverted after validation, second run kept and closed) to rehearse the full flow end-to-end before doing it live. Both times fully validated directly against the database (not just the UI) — vendor/contract match, reconciliation net_variance, allocation, approval, and (second run) the actual period close. Two real findings came out of this:
- **R-059** — a GLG invoice didn't auto-link to its one and only contract because `matchInvoicesToContracts` requires both a billing-date overlap *and* an amount match (±5%) to auto-link regardless of how many contracts the vendor has; with exactly one contract, an unmatched amount most likely means the contract's stored value is stale, not that the invoice belongs elsewhere. Logged for later analysis, not resolved now.
- **A live, deliberate reproduction of R-055** — set Third Bridge's invoice to the shared `G-00024274` billing account on purpose, which pulled Forum Package's expected cost into an invoice that only actually covers Expert Network, forcing a $10,555.33 adjustment to approve. Found an extra wrinkle beyond what R-055 already had documented: once the period closes, that adjustment gets permanently misattributed to the wrong service in the archived snapshot (Expert Network shows an unexplained negative adjustment; Forum Package still shows `missing_invoice`, as if unpaid) — with no way to fix it afterward since Reopen Period (R-003) doesn't exist yet. This concrete example, plus two refinements to Edgar's prepared question list for Santiago, fed directly into the Santiago call the next day that closed out R-055 (see that row in the backlog — resolved by splitting the shared account per-service, no new code).
- After each dry run, the added invoices/adjustments/distributions/snapshots/upload record were deleted and the period reopened by hand via direct SQL — there's still no product feature for undoing a period close (same gap R-003 would fix), so this had to be done the same way the Grupo C session did it on 2026-06-07.

---

## Recent changes (2026-07-20 session — R-049: Reports cross-filter + monthly pivot table)

Edgar shared screenshots of a reference Power BI dashboard (the one Santiago pointed to before today's call) — comparing it against Source's Reports module found two real gaps: no month-by-month pivot table anywhere, and no click-to-filter cross-filtering (clicking one visual doesn't filter the others). Three options were presented (static table / one-way highlight / full cross-filter); Edgar chose full cross-filter and asked for the work split across backend/design/frontend, matching this project's usual multi-agent flow.

**Backend verification (no migration)** — confirmed both dimensions needed (`split_by='vendor'`, the whole `filter_by`/`filter_value` mechanism) already exist in production, proven safe by `ReportsDimension.tsx`'s existing drill-down. Re-confirmed RLS independently gates org access (function is `SECURITY INVOKER`, `inventory_period_snapshots` has its own RLS policy on top), whitelist byte-identical on staging/production, `EXPLAIN ANALYZE` healthy (caveated: HIG Testing's 237 rows is too small to be a real stress test).

**Design spec** — a 7-item spec covering the cross-filter state model (a "self-highlight family" rule: a chart whose own dimension matches the active filter stays unfiltered and highlights the selection instead of collapsing to one bar — except the new pivot table, which always reflects the filter even on its own dimension), the pivot table's sticky-column/layout details, the active-filter chip, selected/dimmed chart coloring, a lightweight (non-full-page) loading state for filter clicks, and a Split-By/filter collision guard.

**Implementation** — built by Edgar with Codex against the spec (the frontend-impl-agent hit a session limit mid-task), then reviewed here: typecheck clean, build clean, logic verified against all 7 items — two real improvements found beyond the spec (request-sequencing guards preventing stale async responses from clobbering state on rapid clicks; a cleaner single-effect resolution of the two loading states that sidesteps a stale-closure risk the spec had flagged). Edgar separately asked Codex to also rework `ReportsDimension.tsx`'s (Vendors/Users) drill-down into a richer panel (monthly trend chart, summary metrics, tabbed breakdown) — outside this item's original scope but same quality; one review concern (panel renders above the table, not inline with the clicked row) was checked live against the real edge case that mattered (Capital Economics, 35 users) and confirmed clean.

**Validation** — numbers cross-checked against direct `report_spend_aggregate` calls on staging (HIG Testing): Capital Economics vendor filter ($3,715.92/35 users/2 services) and "Unallocated" user filter ($1,126,417.23/9 vendors/14 services) both matched Edgar's screenshots to the cent. Deployed: pushed to `s4sourceio` `main`, published in Lovable, zero backend changes. See `PRODUCT_REVIEW_BACKLOG.md` R-049.

---

## Recent changes (2026-07-14 to 2026-07-17 sessions — Admin page, quick wins, R-044/R-039/R-027 billing fixes, Ask AI POC)

Multi-day session covering a full `/app-review` pass (Inventory module end-to-end, R-034 through R-048 logged) plus working through the resulting backlog one item at a time, each validated on staging then production before moving to the next. Full per-item detail lives in `PRODUCT_REVIEW_BACKLOG.md`; this is the summary.

**R-029 — Admin page.** Built after an explicit critical-analysis pass with Edgar on scope (S4-staff-only vs. per-org admin). New `profiles.is_s4_staff` flag (deliberately not reusing `role` or email domain — both proved unreliable signals). `/admin` route with Organizations/Modules/Members tabs, 6 new Edge Functions following the existing `requireS4Staff` pattern. Found and fixed a production bug post-deploy: `Admin.tsx`'s `fetch()` calls were missing the `VITE_SUPABASE_URL` fallback that every other page already has for Lovable's build — orgs appeared to save (toast showed success) but silently 404'd.

**R-046 — vendor matching false positives.** Empirically validated every same-org vendor pair scoring ≥0.5 on the first-token signal, in both staging and production: found the fuzzy first-token match couldn't tell "shared generic word" (the/capital/business — false positives) from "shared distinctive brand" (Clarksons/ICE/CoStar — true positives). Fixed with an exact-match + stopword-table gate. R-048 logged as a related, deliberately deferred follow-up (full-name-similarity false positives, different root cause).

**R-036/R-037/R-040 — quick wins**, done one at a time per Edgar's request: Account dropdown now scopes to the selected vendor (`InventoryBilling.tsx`); invoice detail panel shows a loading spinner instead of a `null` flash while data is still loading; Reports' "+N more" became a real expand/collapse toggle (validated against a real 35-user case).

**R-032 — TypeScript cleanup.** Of 8 pre-existing errors, fixed all; 2 were real bugs, not just type gaps — `InventoryUploadDetail.tsx` was never selecting `contract_id`, silently degrading the manual contract-assignment picker's fallback label.

**R-044 — same-vendor contract disambiguation.** `reconcile-inventory-batch`'s orphan-invoice matcher picked the single best name-similarity score with no tiebreaker; when multiple contracts of the same vendor cleared the threshold within a small margin (a real Third Bridge case), now disambiguates by billing-date overlap — same signal `planInvoiceContractMatches` already used for a different code path. 2 new Deno tests, full 26-test suite passing.

**R-039 — services vs. billing-account UX**, analyzed against Senthio's equivalent (Platform Account/Billing Account/AccountMaps) before deciding: the account-based cross-contract inclusion mechanism is legitimate and stays (needed for R-026), but found and fixed two real bugs — "+Add Service" rendered with nothing left to add, and the account-based inclusion path pulled in unpriced/unallocated placeholder services (3 of 7 on a real Mimecast invoice) that could never be removed. Fixed by requiring `annual_value > 0` on that inclusion path; regression-checked the legitimate Third Bridge cross-contract case still works.

**R-027 — retroactive adjustments to closed periods.** `invoice_adjustments` already nets into an invoice's variance regardless of period status, but nothing synced that correction into `inventory_period_snapshots` (what Reports/exports actually read) — `close_billing_period`'s adjustment-folding only ran at close time. New trigger mirrors that same insert logic whenever an adjustment targets an already-closed period. Also tried and **reverted** automatic date-to-period inference in `match-invoice-service` — Senthio's own reference confirmed this is a deliberate human judgment call there too (`Adj Inv`/`Adj Year`/`Adj Month`, manually set), not something to automate.

**R-033 — Ask AI POC.** Edgar asked for expert reasoning on OLTP vs. OLAP first: is it safe to build a chat feature (or any dashboard) directly on the transactional schema? Answer — Reports already queries `inventory_period_snapshots`, a denormalized fact table populated at month-end close, not the live OLTP tables; that's already the right separation at this scale, no separate warehouse needed. Built accordingly: new `ask-report-ai` Edge Function using Anthropic's Messages API with tool-use, hard-restricted to calling `report_spend_aggregate` (never raw SQL). Validated on staging (HIG Testing) with real questions — all matched RPC ground truth exactly. One bug found live by Edgar and fixed same-session: answers rendered literal `**Markdown**` asterisks since the frontend shows plain text — fixed via a system-prompt instruction. Deployed to production and re-validated (S4 Market Data). Chat-style multi-turn memory discussed and deliberately deferred — single-turn Q&A for now.

Also ran the Step 6 parity check at session close: staging and production fully in sync (migrations, function list). Found substantial uncommitted local changes to `TECH_DEBT.md` and `docs/security/security-compliance-roadmap.md` from a separate, parallel security-review session — left untouched per Edgar (tracked in that other session, not here).

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
