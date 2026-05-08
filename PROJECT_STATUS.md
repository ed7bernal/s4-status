# Source S4 — Product Status
*Last updated: 2026-05-08*

## What's live in production

- **Invoice Processing** — Users upload a PDF invoice and the system automatically reads it, identifies the vendor, finds the right account number, and saves all the data. No manual data entry required for standard invoices.
- **Contract Processing** — Users upload a PDF contract and the system extracts key fields (vendor, dates, value, renewal terms, cancellation notice) and stores them in a searchable database.
- **Vendor Match Resolution** — When the system can't confidently identify a vendor from a document, the user can confirm the correct vendor or add a new one directly from the UI. The system learns from each correction.
- **Reconciliation** — Matches processed invoices against expected charges to flag discrepancies.

## What's in staging (not yet in production)

- **Inventory Upload** — Batch upload pipeline for contracts and invoices. Users submit a set of PDF files; the system processes each independently (OCR, extraction, vendor matching) and reconciles vendor status across the batch when all items finish. Edge functions: `process-inventory-upload`, `process-inventory-document`, `check-batch-complete`, `reconcile-inventory-batch`. Two of the four functions (`process-inventory-document`, `reconcile-inventory-batch`) were updated today and deployed to staging — pending validation before deploying to production.
- **Vendor Merge** — Merges a duplicate inventory-created vendor into a canonical vendor, moving all related records (contracts, invoices, accounts, aliases) and adding the duplicate name as an alias for future auto-matching. Edge function: `merge-vendors`.

**Pending before production deploy:** end-to-end staging validation of the inventory upload flow using the new typed FK columns (`created_contract_id` / `created_invoice_id`), then deploy `process-inventory-document` and `reconcile-inventory-batch` to production. After staging validation passes, also drop the old `created_record_id` column manually.

## What's in development

- Vendor match status badges and warning indicators in the invoice and contract views, so users can see at a glance which records need vendor review.
- UI workflow for confirming or overriding a vendor match directly from the invoice detail page.
- Product Catalog fields on services (delivery method, asset class, billing frequency, licensed seats, etc.) — database columns added today, UI not yet built.
- Service history tracking — the database now automatically records a snapshot whenever a service's key fields change (price, dates, renewal terms). UI not yet built.

## What's coming next

- Support for contracts that span multiple documents (e.g. an Order Form that references a separate Master Agreement and Terms). Currently the system only reads one PDF at a time.
- Usage tracking per client — visibility into how many documents each client has processed and estimated costs.
- Second client onboarding — the system is architected for multiple clients but has only been configured for one so far.

## Where your input would help

- **Inventory upload staging validation** — `process-inventory-document` and `reconcile-inventory-batch` were updated today to use typed FK columns. Ready for a test batch upload in staging to confirm the new columns populate correctly before promoting to production.
- **Drop old column timing** — the old `created_record_id` column on `inventory_upload_items` is still present as a safety net. Once staging validation confirms the backfill is correct, it can be dropped with: `ALTER TABLE public.inventory_upload_items DROP COLUMN created_record_id;`
- **Vendor match thresholds** — when the system is 50–84% confident on a vendor match, it flags it as "pending" for manual review. Is that the right cutoff, or should the team review everything below 85%?
- **Contract description quality** — for simple order forms, the AI tends to describe the document rather than the underlying service. Worth fixing before showing contracts to more users?
- **Reconciliation scope** — what does a "discrepancy" mean to the business? The module exists but the rules for what counts as a match vs. a mismatch need business input.

## Recent changes

- Added Product Catalog fields to the services table: delivery method, data frequency, asset class, expense type, billing frequency, unit cost, currency, billing entity, licensed seats, service notes, and a completeness score. These fields are nullable and do not affect existing records. Applied to both staging and production.
- Added service match columns to the invoices table (`service_match_score`, `service_match_status`) to track how confidently an invoice was linked to a specific service. Applied to both staging and production.
- Added a service history table that automatically saves a snapshot of a service record whenever key fields change — useful for tracking price changes and renewals over time. Applied to both staging and production.
- Replaced an ambiguous record ID column (`created_record_id`) on the inventory upload items table with two typed columns — one for contracts and one for invoices. This removes ambiguity about which table the ID points to and adds proper foreign key constraints. The old column is still present temporarily while staging validation completes.
- Added missing database indexes across invoices, services, contracts, and inventory upload items to improve query performance as data volume grows. Applied to both staging and production.
- Updated `process-inventory-document` and `reconcile-inventory-batch` edge functions to use the new typed FK columns. Deployed to staging.
- Fixed a Supabase CLI compatibility issue: `CREATE INDEX CONCURRENTLY` cannot run inside a migration transaction — removed `CONCURRENTLY` from the index migration (indexes are still idempotent via `IF NOT EXISTS`).
- Production database was behind by 6 migrations (the entire inventory upload module had never been pushed). All migrations are now in sync between staging and production.
- Updated Supabase CLI from v2.90.0 to v2.98.2 via Homebrew.
