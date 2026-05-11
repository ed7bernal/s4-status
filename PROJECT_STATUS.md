# Source S4 — Product Status
*Last updated: 2026-05-11*

## What's live in production

- **Invoice Processing** — Users upload a PDF invoice and the system automatically reads it, identifies the vendor, finds the right account number, and saves all the data. Now also automatically attempts to link the invoice to its matching service record after every upload. PDFs are now saved to secure storage and a link is stored on each invoice record.
- **Contract Processing** — Users upload a PDF contract and the system extracts key fields (vendor, dates, value, renewal terms, cancellation notice) and stores them in a searchable database. Extracts 9 product catalog fields per service line. PDFs are now saved to secure storage and a link is stored on each contract record.
- **Vendor Match Resolution** — When the system can't confidently identify a vendor from a document, the user can confirm the correct vendor or add a new one directly from the UI. The system learns from each correction.
- **Reconciliation** — Matches processed invoices against expected charges to flag discrepancies.
- **Service Match** — After every invoice is processed, the system automatically scores and links the invoice to the best matching service record.
- **PDF Storage** — Every contract and invoice PDF uploaded through the batch pipeline is now stored in a private Supabase Storage bucket, organised by client and document type. Each record holds a secure link to its source PDF.

## What's in staging (not yet in production)

- **Inventory Upload** — Batch upload pipeline for contracts and invoices. Edge functions: `process-inventory-upload`, `process-inventory-document`, `check-batch-complete`, `reconcile-inventory-batch`.
- **Vendor Merge** — Merges a duplicate vendor into a canonical one, moving all related records and adding the duplicate name as an alias for future auto-matching. Edge function: `merge-vendors`.

**Pending before production deploy:** end-to-end staging validation of the full inventory upload flow, then promote `process-inventory-upload`, `check-batch-complete`, `reconcile-inventory-batch`, and `merge-vendors` to production.

## What's in development

- Vendor match status badges and warning indicators in the invoice and contract views.
- UI workflow for confirming or overriding a vendor match directly from the invoice detail page.
- Product Catalog fields on services — database columns and extraction logic are complete. UI for viewing and editing these fields is partially built in the inventory contract detail page.
- Service history tracking — snapshots of service changes recorded automatically. UI not yet built.
- Service match review UI — users will need a way to confirm or override service links on pending-scored invoices.
- Inventory Upload detail page — actively being improved. Contract status is now read from the database on load — contracts already marked Active are shown in the Approved section immediately without requiring a Confirm action.
- PDF viewer — storage is in place but the "View Contract PDF" and "View Invoice PDF" buttons are still disabled pending a UI implementation.

## What's coming next

- Surface stored PDF links in the UI — wire the View PDF buttons in the contract and invoice detail pages to the stored signed URLs.
- Support for contracts that span multiple documents (Order Form + Master Agreement + Terms). Currently the system only reads one PDF at a time.
- Usage tracking per client — visibility into how many documents each client has processed and estimated costs.
- Second client onboarding — the system is architected for multiple clients but has only been configured for one so far.

## Where your input would help

- **PDF storage signed URLs** — current signed URLs are generated with a 10-year expiry at upload time and stored on the record. This is simple but means URLs can't be revoked. Alternative: store only the storage path and generate short-lived signed URLs on demand in the frontend. Worth deciding before building the View PDF UI.
- **Product catalog field quality** — the system now extracts delivery method, data frequency, asset class, billing frequency, unit cost, currency, bill-to entity, and licensed seats from contracts. Worth testing with a few real contracts to see how often these populate correctly vs. need manual correction.
- **Service match thresholds** — matched ≥ 0.85, pending 0.50–0.84, unmatched < 0.50. Worth monitoring the first real production batch.
- **Drop old column timing** — the old `created_record_id` column on `inventory_upload_items` is still present. Once staging validation confirms the new typed FK columns are working correctly in the UI, it can be dropped.
- **Inventory upload staging validation** — ready for a real batch test in staging to confirm contract cards show correct vendor match state, invoice links, and classification.

## Recent changes

- Contracts already marked Active in the database now appear in the Approved section of the inventory upload review screen on first load — previously they would show with a Confirm button even though they were already approved.
- Created a private PDF storage bucket in Supabase Storage. Every PDF processed through the batch upload pipeline is now saved automatically at a path organised by client and document type. A secure link is written back to the contract or invoice record. Storage failures are logged but do not fail the document — processing continues normally.
- Added `pdf_url` column to both the contracts and invoices tables.
- Applied all storage changes to both staging and production.
