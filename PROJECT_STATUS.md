# Source S4 — Product Status
*Last updated: 2026-05-08*

## What's live in production

- **Invoice Processing** — Users upload a PDF invoice and the system automatically reads it, identifies the vendor, finds the right account number, and saves all the data. No manual data entry required for standard invoices.
- **Contract Processing** — Users upload a PDF contract and the system extracts key fields (vendor, dates, value, renewal terms, cancellation notice) and stores them in a searchable database.
- **Vendor Match Resolution** — When the system can't confidently identify a vendor from a document, the user can confirm the correct vendor or add a new one directly from the UI. The system learns from each correction.
- **Reconciliation** — Matches processed invoices against expected charges to flag discrepancies.

## What's in staging (not yet in production)

- **Inventory Upload** — Batch upload pipeline for contracts and invoices. Users submit a set of PDF files; the system processes each independently (OCR, extraction, vendor matching) and reconciles vendor status across the batch when all items finish. Edge functions: `process-inventory-upload`, `process-inventory-document`, `check-batch-complete`, `reconcile-inventory-batch`. Migration 20260506000002 (queued status constraint on `inventory_upload_items`) applied to staging.
- **Vendor Merge** — Merges a duplicate inventory-created vendor into a canonical vendor, moving all related records (contracts, invoices, accounts, aliases) and adding the duplicate name as an alias for future auto-matching. Edge function: `merge-vendors`.

**Pending before production deploy:** full end-to-end staging validation of the inventory upload flow, then deploy all four inventory functions and `merge-vendors` to production.

## What's in development

- Vendor match status badges and warning indicators in the invoice and contract views, so users can see at a glance which records need vendor review.
- UI workflow for confirming or overriding a vendor match directly from the invoice detail page.

## What's coming next

- Support for contracts that span multiple documents (e.g. an Order Form that references a separate Master Agreement and Terms). Currently the system only reads one PDF at a time.
- Usage tracking per client — visibility into how many documents each client has processed and estimated costs.
- Second client onboarding — the system is architected for multiple clients but has only been configured for one so far.

## Where your input would help

- **Inventory upload staging validation** — the pipeline auth bug has been fixed (see recent changes). Ready for a full end-to-end test before promoting to production.
- **Vendor match thresholds** — when the system is 50–84% confident on a vendor match, it flags it as "pending" for manual review. Is that the right cutoff, or should the team review everything below 85%?
- **Contract description quality** — for simple order forms, the AI tends to describe the document rather than the underlying service. Worth fixing before showing contracts to more users?
- **Reconciliation scope** — what does a "discrepancy" mean to the business? The module exists but the rules for what counts as a match vs. a mismatch need business input.
- **Second client readiness** — are there specific data fields or workflow differences the next client will need that we should design for now?

## Recent changes

- Fixed a critical bug in the inventory upload pipeline that was preventing documents from being processed — items were getting stuck in "pending" status and never moving forward.
- Root cause was a bad internal authentication token stored in Supabase project settings that couldn't be corrected (Supabase won't allow overwriting reserved key names via CLI). Resolved by introducing a dedicated internal secret (`INTERNAL_WORKER_SECRET`) that all four inventory pipeline functions now use to authenticate with each other.
- Also fixed a separate issue where the dispatch step was running in the background after the initial response was sent — the function could shut down before dispatching workers to any documents. Dispatch is now guaranteed to complete before the response is returned.
- Confirmed that `inventory_uploads.status` is correctly set to `completed` at the end of the pipeline (by `reconcile-inventory-batch`) — no logic was missing there.
- Confirmed that `merge-vendors` edge function exists and has a working UI in the frontend (`MergeVendorDialog`).
- Reviewed the full contracts, vendors, and invoices table schemas. Fields for licenses/seats, active users, delivery method, business sponsor, and business group do not exist yet — these would need new migrations if required.
- Inventory upload pipeline built and deployed to staging — batch PDF processing with orchestrator + worker pattern, polling UI, and per-item error display.
- Vendor merge function built and deployed to staging — merges inventory-created vendors into canonical vendors.
- Vendor matching now works for both invoices and contracts — the system identifies vendors on upload and flags ones it's unsure about.
- Admin vs. member roles are now enforced — only admins can delete records or change system settings.
- The backend codebase now has version control and is backed up to a private GitHub repository.
- Security hardening completed — unauthorized users can no longer access internal functions or other clients' data.
