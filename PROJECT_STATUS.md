# Source S4 — Product Status
*Last updated: 2026-05-08*

## What's live in production

- **Invoice Processing** — Users upload a PDF invoice and the system automatically reads it, identifies the vendor, finds the right account number, and saves all the data. Now also automatically attempts to link the invoice to its matching service record after every upload.
- **Contract Processing** — Users upload a PDF contract and the system extracts key fields (vendor, dates, value, renewal terms, cancellation notice) and stores them in a searchable database.
- **Vendor Match Resolution** — When the system can't confidently identify a vendor from a document, the user can confirm the correct vendor or add a new one directly from the UI. The system learns from each correction.
- **Reconciliation** — Matches processed invoices against expected charges to flag discrepancies.
- **Service Match** — `match-invoice-service` is now live in production. After every invoice is processed (both direct upload and batch inventory upload), the system automatically scores and links the invoice to the best matching service record. Sets a match status of matched/pending/unmatched and stores the confidence score on the invoice row.

## What's in staging (not yet in production)

- **Inventory Upload** — Batch upload pipeline for contracts and invoices. Edge functions: `process-inventory-upload`, `process-inventory-document`, `check-batch-complete`, `reconcile-inventory-batch`.
- **Vendor Merge** — Merges a duplicate vendor into a canonical one, moving all related records and adding the duplicate name as an alias for future auto-matching. Edge function: `merge-vendors`.

**Pending before production deploy:** end-to-end staging validation of the full inventory upload flow, then promote `process-inventory-upload`, `check-batch-complete`, `reconcile-inventory-batch`, and `merge-vendors` to production.

## What's in development

- Vendor match status badges and warning indicators in the invoice and contract views.
- UI workflow for confirming or overriding a vendor match directly from the invoice detail page.
- Product Catalog fields on services (delivery method, asset class, billing frequency, licensed seats, etc.) — database columns added, UI not yet built.
- Service history tracking — snapshots of service changes recorded automatically. UI not yet built.
- Service match review UI — users will need a way to confirm or override service links on pending-scored invoices.
- Inventory Upload detail page — actively being improved. Classification logic, vendor resolution flow, and invoice matching all updated this session.

## What's coming next

- Support for contracts that span multiple documents (Order Form + Master Agreement + Terms). Currently the system only reads one PDF at a time.
- Usage tracking per client — visibility into how many documents each client has processed and estimated costs.
- Second client onboarding — the system is architected for multiple clients but has only been configured for one so far.

## Where your input would help

- **Service match thresholds** — matched ≥ 0.85, pending 0.50–0.84, unmatched < 0.50. In testing, Bloomberg Terminal scored 0.5946 (pending) due to the org name in staging ("Test Client") not matching the bill-to entity ("H.I.G. Capital"). In production with real org names, scores should be higher. Worth monitoring the first real batch.
- **Service name quality** — name similarity compares the invoice vendor name against the service name. Generic names ("Premium Communication Services" instead of "Bloomberg Terminal") produce low scores. Worth auditing service names before the inventory upload goes to production.
- **Drop old column timing** — the old `created_record_id` column on `inventory_upload_items` is still present. Once staging validation confirms the new typed FK columns (`created_contract_id` / `created_invoice_id`) are working correctly in the UI, it can be dropped.
- **Bill-to matching** — the inventory detail page shows a "Bill-to" chip on each contract card but it always shows as unmatched because the `bill_to_match_status` column doesn't exist yet on the invoices table. Is this worth building, or should the chip be hidden until it can be real?
- **Inventory upload staging validation** — the upload detail page had several data bugs fixed today. Ready for a real batch test in staging to confirm contract cards show correct vendor match state, invoice links, and classification.

## Recent changes

- Deployed `match-invoice-service`, `process-invoice`, and `process-inventory-document` to **production**. Service matching is now live — every new invoice automatically gets a service match score written to it.
- Fixed the Inventory Upload detail page to use the new typed FK columns (`created_contract_id` and `created_invoice_id`) instead of the old ambiguous `created_record_id` column. The vendor resolution modal now correctly handles both contract items and invoice items, and passes the right source type to the backend.
- Fixed the "Looks Good" vs "Needs Review" classification on the inventory upload review screen. It was checking whether a vendor ID was present, which incorrectly promoted low-confidence matches to "Looks Good." Now uses `match_state` directly — only `matched` (auto, high confidence) and `resolved` (manually confirmed) count as good.
- Removed a reference to a non-existent column (`bill_to_match_status`) from the invoices query on the inventory detail page, which would have caused a silent data error.
- Cleaned up test data in staging for the test org.
