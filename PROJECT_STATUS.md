# Source S4 — Product Status
*Last updated: 2026-05-11*

## What's live in production

- **Invoice Processing** — Users upload a PDF invoice and the system automatically reads it, identifies the vendor, finds the right account number, and saves all the data. Now also automatically attempts to link the invoice to its matching service record after every upload. PDFs are now saved to secure storage and a link is stored on each invoice record.
- **Contract Processing** — Users upload a PDF contract and the system extracts key fields (vendor, dates, value, renewal terms, cancellation notice) and stores them in a searchable database. Extracts 9 product catalog fields per service line. PDFs are now saved to secure storage and a link is stored on each contract record.
- **Vendor Match Resolution** — When the system can't confidently identify a vendor from a document, the user can confirm the correct vendor or add a new one directly from the UI. The system learns from each correction.
- **Reconciliation** — Matches processed invoices against expected charges to flag discrepancies.
- **Service Match** — After every invoice is processed, the system automatically scores and links the invoice to the best matching service record.
- **PDF Storage** — Every contract and invoice PDF uploaded through the batch pipeline is now stored in a private Supabase Storage bucket, organised by client and document type. Each record holds a secure link to its source PDF.
- **Inventory Upload — Upload Invoice button** (`process-inventory-upload`, `process-inventory-document`) — The "Upload Invoice" button on contract cards in the batch review screen is now fully wired end-to-end in production. When a user uploads an invoice for a specific contract, the system links the invoice directly to that contract, skips the org-wide vendor search and assigns the vendor from the contract instead, and validates that the invoice amount, vendor name, and billing period match the contract. Mismatches are flagged with a warning and placed in the Needs Review section rather than silently accepted.

## What's in staging (not yet in production)

- **Inventory Upload — full batch pipeline** (`check-batch-complete`, `reconcile-inventory-batch`) — Batch completion detection and vendor-level flag reconciliation remain staging-only. Full end-to-end staging validation still pending before promoting these to production.
- **Vendor Merge** — Merges a duplicate vendor into a canonical one, moving all related records and adding the duplicate name as an alias for future auto-matching. Edge function: `merge-vendors`.

**Pending before full production deploy:** end-to-end staging validation of the complete inventory batch flow, then promote `check-batch-complete`, `reconcile-inventory-batch`, and `merge-vendors` to production.

## What's in development

- Vendor match status badges and warning indicators in the invoice and contract views.
- UI workflow for confirming or overriding a vendor match directly from the invoice detail page.
- Product Catalog fields on services — database columns and extraction logic are complete. UI for viewing and editing these fields is partially built in the inventory contract detail page.
- Service history tracking — snapshots of service changes recorded automatically. UI not yet built.
- Service match review UI — users will need a way to confirm or override service links on pending-scored invoices.
- PDF viewer — storage is in place but the "View Contract PDF" and "View Invoice PDF" buttons are still disabled pending a UI implementation.

## What's coming next

- Surface stored PDF links in the UI — wire the View PDF buttons in the contract and invoice detail pages to the stored signed URLs.
- Support for contracts that span multiple documents (Order Form + Master Agreement + Terms). Currently the system only reads one PDF at a time.
- Usage tracking per client — visibility into how many documents each client has processed and estimated costs.
- Second client onboarding — the system is architected for multiple clients but has only been configured for one so far.

## Where your input would help

- **Upload Invoice flow — end-to-end test** — The full chain (Upload Invoice button → append mode → linked contract matching → Needs Review / Looks Good grouping) is deployed to production but has not been tested with a real PDF. Worth running a real invoice upload against a known contract to confirm vendor assignment, match chips, and section placement all behave correctly.
- **Auto-create vendor quality** — Contracts with no existing vendor match (score < 0.50) now automatically create a new vendor record using the extracted vendor name. Worth monitoring whether the extracted names are clean enough to use as canonical vendor names, or whether they need normalisation before creation.
- **PDF storage signed URLs** — current signed URLs are generated with a 10-year expiry at upload time and stored on the record. This is simple but means URLs can't be revoked. Alternative: store only the storage path and generate short-lived signed URLs on demand in the frontend. Worth deciding before building the View PDF UI.
- **Product catalog field quality** — the system now extracts delivery method, data frequency, asset class, billing frequency, unit cost, currency, bill-to entity, and licensed seats from contracts. Worth testing with a few real contracts to see how often these populate correctly vs. need manual correction.
- **Drop old column timing** — the old `created_record_id` column on `inventory_upload_items` is still present. Once staging validation confirms the new typed FK columns are working correctly in the UI, it can be dropped.

## Recent changes

- **Upload Invoice flow wired end-to-end** — When a user clicks "Upload Invoice" on a contract card in the batch review screen, the invoice is now processed against that specific contract rather than running a general org-wide vendor search. The system assigns the vendor directly from the linked contract, then checks whether the invoice vendor name, amount (within 5%), and billing period align. If all checks pass the invoice is marked as matched; if any check fails, the item is placed in Needs Review with a warning listing which fields didn't match.
- **Contract vendor auto-create** — When a contract is uploaded through the batch pipeline and no existing vendor in the org matches the extracted vendor name (score below 0.50), the system now automatically creates a new vendor record rather than leaving the contract with no vendor assigned. Scores between 0.50 and 0.84 still go to Pending for user review.
- **Database** — Added `linked_contract_id` column to `inventory_upload_items` (links appended invoice items back to the contract they were uploaded against). Added `needs_review` as a valid value for the `match_state` column. Both migrations applied to staging and production.
- **Frontend** — Contract detail page in the inventory review screen now shows multiple invoices with expand/collapse and an Add Invoice button. Invoice lookup now falls back to `linked_contract_id` search when no vendor-matched invoice is found in the same upload batch.
- **Deployed to production** — `process-inventory-upload` and `process-inventory-document` promoted to production with all of the above changes.
- Contracts already marked Active in the database now appear in the Approved section of the inventory upload review screen on first load — previously they would show with a Confirm button even though they were already approved.
- Created a private PDF storage bucket in Supabase Storage. Every PDF processed through the batch upload pipeline is now saved automatically at a path organised by client and document type. A secure link is written back to the contract or invoice record. Storage failures are logged but do not fail the document — processing continues normally.
