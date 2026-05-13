# Source S4 — Product Status
*Last updated: 2026-05-14*

## What's live in production

- **Invoice Processing** — Users upload a PDF invoice and the system automatically reads it, identifies the vendor, finds the right account number, and saves all the data. Now also automatically attempts to link the invoice to its matching service record after every upload. PDFs are now saved to secure storage and a link is stored on each invoice record.
- **Contract Processing** — Users upload a PDF contract and the system extracts key fields (vendor, dates, value, renewal terms, cancellation notice) and stores them in a searchable database. Extracts 9 product catalog fields per service line. PDFs are now saved to secure storage and a link is stored on each contract record.
- **Vendor Match Resolution** — When the system can't confidently identify a vendor from a document, the user can confirm the correct vendor or add a new one directly from the UI. Now also supports manually linking an invoice to a specific contract (`link_contract` action), which writes the contract link, service link, and vendor assignment in one step and optionally saves the invoice vendor name as an alias for future matching.
- **Reconciliation** — Matches processed invoices against expected charges to flag discrepancies.
- **Service Match** — After every invoice is processed, the system automatically scores and links the invoice to the best matching service record. The best-scoring service is now always saved on the invoice even when the confidence is low, so no invoice is left with a blank service link.
- **PDF Storage** — Every contract and invoice PDF uploaded through the batch pipeline is now stored in a private Supabase Storage bucket, organised by client and document type. Each record holds a secure link to its source PDF.
- **Inventory Upload — full batch pipeline** — `check-batch-complete`, `reconcile-inventory-batch`, and `merge-vendors` promoted to production this session. The full inventory batch pipeline is now live in production.
- **Inventory Upload — Upload Invoice button** — The "Upload Invoice" button on contract cards in the batch review screen is fully wired end-to-end in production.
- **Vendor name normalization** — Vendor matching now handles `&` vs `"and"` correctly (e.g. "S&P" matches "S and P"). Legal suffixes (LLC, Inc, Ltd, etc.) are stripped before comparison in both the JavaScript matching engine and the PostgreSQL `match_vendor` function. Applied to both environments.

## What's in staging (not yet in production)

- **User roster and invoice seat allocation** (`org_users`, `invoice_allocations`) — Database tables added to staging and production. `org_users` stores the organisation's people directory. `invoice_allocations` links invoices to specific users with seat counts and unit cost. No edge function or UI built yet.

## What's in development

- **Vendor list page bug** — The Vendors page in the inventory section has two known filter issues: (1) auto-created vendors have no `source` tag and are excluded from the vendor list query; (2) the vendor list only shows vendors with Active contracts. Debug logging added. Fix pending — awaiting decision on correct scope.
- Vendor match status badges and warning indicators in the invoice and contract views.
- UI for manually linking an invoice to a contract (the `link_contract` backend action is live; the UI trigger is not yet built).
- Product Catalog fields — database and extraction complete, UI partially built in the inventory contract detail page.
- Service history tracking — snapshots recorded automatically, UI not yet built.
- PDF viewer — storage in place, View PDF buttons not yet wired to signed URLs.

## What's coming next

- Fix vendor list page filters (source tag and contract status).
- Build UI trigger for `link_contract` — allow users to manually link an invoice to a contract from the review screen.
- Surface stored PDF links in the UI — wire the View PDF buttons to the stored signed URLs.
- Build UI for `org_users` — upload/manage the user roster.
- Build UI for `invoice_allocations` — assign invoice seats to users from the invoice detail page.
- Support for contracts that span multiple documents.
- Usage tracking per client.
- Second client onboarding.

## Where your input would help

- **Vendor list — source filter decision** — Auto-created vendors (score < 0.50, created automatically during batch upload) are inserted without a `source` tag and don't appear in the Vendors page. Fix options: (a) tag them `source = 'inventory_upload'` at creation, or (b) remove the source filter entirely and show all org vendors. Decision needed before fixing the page.
- **Vendor list — contract status filter decision** — The vendor list only shows vendors with Active contracts. Vendors with Draft contracts are hidden. Should the page show all contracts, or show all with a status badge?
- **`link_contract` UI** — The backend action for manually linking an invoice to a contract is deployed and tested. The UI trigger (a button on the Needs Review cards or invoice detail) still needs to be built. Worth deciding where it should live — on the contract card in the batch review screen, or on the invoice detail page, or both.
- **`data_frequency` extraction quality** — The GPT prompt for extracting data frequency now maps to canonical values (Real-time, End of Day, Daily, Weekly, Monthly, Quarterly, Annual). Worth testing against a few real contracts to confirm extraction quality improved.
- **Auto-create vendor quality** — Contracts with no existing vendor match automatically create a new vendor. Worth monitoring whether extracted names are clean enough to use as canonical names.
- **Drop old column timing** — `inventory_upload_items.created_record_id` is still present. Can be dropped after staging validation.

## Recent changes

- **Invoice-to-contract manual linking** — New `link_contract` action added to `resolve-vendor-match`. Accepts an invoice ID and contract ID, verifies they belong to the same org, writes the contract link, service link, vendor assignment, and match scores to the invoice in one step. Optionally saves the invoice vendor name as an alias if it differs from the contract's vendor name. Tested end-to-end in staging. Deployed to production.
- **`invoices.contract_id` column** — New column added to the invoices table linking an invoice directly to a contract. Populated by the `link_contract` action. Migration applied to both staging and production.
- **Vendor name normalization improved** — `&` is now expanded to `"and"` before comparison so "S&P" matches "S and P". Applied to both the JavaScript matching engine (used in batch processing) and the PostgreSQL `match_vendor` function (used in single-invoice processing). Deployed to both environments.
- **Service match always saves best candidate** — Even when the confidence score is too low to call a match, the best-scoring service is now written to the invoice. Previously, low-scoring invoices had no service link at all.
- **`data_frequency` extraction rules improved** — The GPT extraction prompt now maps document language to canonical values with explicit rules, reducing inconsistent outputs like "EOD" vs "End of Day".
- **All inventory batch functions promoted to production** — `check-batch-complete`, `reconcile-inventory-batch`, and `merge-vendors` are now live in production. The full inventory pipeline is available in both environments.
- **`org_users` and `invoice_allocations` tables** — Added to both staging and production. Groundwork for cost allocation reporting.
- **`INTERNAL_WORKER_SECRET` rotated** — The secret that authenticates internal calls between `process-inventory-upload`, `process-inventory-document`, and `check-batch-complete` was missing from production. A new secret was generated and set on both staging and production simultaneously so the worker dispatch chain functions correctly in both environments.
- **Migration sync** — `invoices.contract_id` migration was missing from production and has been applied. Both environments are now fully in sync on all 64 migrations.
