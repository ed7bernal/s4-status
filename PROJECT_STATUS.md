# Source S4 — Product Status
*Last updated: 2026-05-10*

## What's live in production

- **Invoice Processing** — Users upload a PDF invoice and the system automatically reads it, identifies the vendor, finds the right account number, and saves all the data. Now also automatically attempts to link the invoice to its matching service record after every upload.
- **Contract Processing** — Users upload a PDF contract and the system extracts key fields (vendor, dates, value, renewal terms, cancellation notice) and stores them in a searchable database. Now also extracts product catalog fields on each service line (delivery method, data frequency, asset class, billing frequency, licensed seats, and more).
- **Vendor Match Resolution** — When the system can't confidently identify a vendor from a document, the user can confirm the correct vendor or add a new one directly from the UI. The system learns from each correction.
- **Reconciliation** — Matches processed invoices against expected charges to flag discrepancies.
- **Service Match** — After every invoice is processed, the system automatically scores and links the invoice to the best matching service record. Sets a match status of matched/pending/unmatched and stores the confidence score on the invoice row.

## What's in staging (not yet in production)

- **Inventory Upload** — Batch upload pipeline for contracts and invoices. Edge functions: `process-inventory-upload`, `process-inventory-document`, `check-batch-complete`, `reconcile-inventory-batch`.
- **Vendor Merge** — Merges a duplicate vendor into a canonical one, moving all related records and adding the duplicate name as an alias for future auto-matching. Edge function: `merge-vendors`.

**Pending before production deploy:** end-to-end staging validation of the full inventory upload flow, then promote `process-inventory-upload`, `check-batch-complete`, `reconcile-inventory-batch`, and `merge-vendors` to production.

## What's in development

- Vendor match status badges and warning indicators in the invoice and contract views.
- UI workflow for confirming or overriding a vendor match directly from the invoice detail page.
- Product Catalog fields on services — database columns and extraction logic are complete. UI for viewing and editing these fields not yet built.
- Service history tracking — snapshots of service changes recorded automatically. UI not yet built.
- Service match review UI — users will need a way to confirm or override service links on pending-scored invoices.
- Inventory Upload detail page — actively being improved. Classification logic, vendor resolution flow, and invoice matching all updated in recent sessions.

## What's coming next

- Support for contracts that span multiple documents (Order Form + Master Agreement + Terms). Currently the system only reads one PDF at a time.
- Usage tracking per client — visibility into how many documents each client has processed and estimated costs.
- Second client onboarding — the system is architected for multiple clients but has only been configured for one so far.

## Where your input would help

- **Product catalog field quality** — the system now extracts delivery method, data frequency, asset class, billing frequency, unit cost, currency, bill-to entity, and licensed seats from contracts. Worth testing with a few real contracts to see how often these populate correctly vs. need manual correction.
- **Service match thresholds** — matched ≥ 0.85, pending 0.50–0.84, unmatched < 0.50. In testing, Bloomberg Terminal scored 0.5946 (pending) due to org name in staging not matching bill-to entity. In production with real org names, scores should be higher. Worth monitoring the first real batch.
- **Service name quality** — name similarity compares invoice vendor name against service name. Generic names produce low scores. Worth auditing service names before inventory upload goes to production.
- **Drop old column timing** — the old `created_record_id` column on `inventory_upload_items` is still present. Once staging validation confirms the new typed FK columns are working correctly in the UI, it can be dropped.
- **Inventory upload staging validation** — ready for a real batch test in staging to confirm contract cards show correct vendor match state, invoice links, and classification.

## Recent changes

- Added 9 product catalog fields to the contract extraction prompt: delivery method, data frequency, asset class, expense type, billing frequency, unit cost, currency, bill-to entity, and licensed seats. Each field includes specific extraction hints (e.g. look for "Billing Terms" field for bill frequency, default currency to USD when $ symbol is present).
- Added a rule requiring the AI to always output every service field in its response, even when the value is unknown — prevents fields from being silently omitted.
- Increased the AI response token limit from 2,000 to 4,000 to accommodate the larger prompt and richer output.
- Synced all changes to the inventory upload pipeline — the batch contract processing path now extracts and saves the same 9 new fields as the single-contract upload path.
- Removed temporary debug logging from both the contract processing function and the inventory upload detail page.
- Deployed updated `process-contract` and `process-inventory-document` to **both staging and production**.
