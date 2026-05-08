# Source S4 — Product Status
*Last updated: 2026-05-08*

## What's live in production

- **Invoice Processing** — Users upload a PDF invoice and the system automatically reads it, identifies the vendor, finds the right account number, and saves all the data. No manual data entry required for standard invoices.
- **Contract Processing** — Users upload a PDF contract and the system extracts key fields (vendor, dates, value, renewal terms, cancellation notice) and stores them in a searchable database.
- **Vendor Match Resolution** — When the system can't confidently identify a vendor from a document, the user can confirm the correct vendor or add a new one directly from the UI. The system learns from each correction.
- **Reconciliation** — Matches processed invoices against expected charges to flag discrepancies.

## What's in staging (not yet in production)

- **Inventory Upload** — Batch upload pipeline for contracts and invoices. Users submit a set of PDF files; the system processes each independently (OCR, extraction, vendor matching) and reconciles vendor status across the batch when all items finish. Edge functions: `process-inventory-upload`, `process-inventory-document`, `check-batch-complete`, `reconcile-inventory-batch`.
- **Vendor Merge** — Merges a duplicate inventory-created vendor into a canonical vendor, moving all related records (contracts, invoices, accounts, aliases) and adding the duplicate name as an alias for future auto-matching. Edge function: `merge-vendors`.
- **Service Match** — New `match-invoice-service` edge function that automatically links a processed invoice to its matching service record. Scores each candidate service using four criteria: service name similarity (40%), amount within 5% tolerance (25%), billing date overlap (20%), and bill-to entity match (15%). Sets `service_match_status` to matched/pending/unmatched and stores the score on the invoice row. Deployed to staging. Now called automatically after every invoice is processed — both from `process-invoice` and `process-inventory-document`.

**Pending before production deploy:** end-to-end staging validation of the inventory upload flow and service matching, then deploy all three updated functions (`process-invoice`, `process-inventory-document`, `match-invoice-service`) to production.

## What's in development

- Vendor match status badges and warning indicators in the invoice and contract views, so users can see at a glance which records need vendor review.
- UI workflow for confirming or overriding a vendor match directly from the invoice detail page.
- Product Catalog fields on services (delivery method, asset class, billing frequency, licensed seats, etc.) — database columns added, UI not yet built.
- Service history tracking — the database now automatically records a snapshot whenever a service's key fields change (price, dates, renewal terms). UI not yet built.
- Service match review UI — once `match-invoice-service` is in production, users will need a way to confirm or override service links on pending-scored invoices.

## What's coming next

- Support for contracts that span multiple documents (e.g. an Order Form that references a separate Master Agreement and Terms). Currently the system only reads one PDF at a time.
- Usage tracking per client — visibility into how many documents each client has processed and estimated costs.
- Second client onboarding — the system is architected for multiple clients but has only been configured for one so far.

## Where your input would help

- **Service match staging validation** — `match-invoice-service` is deployed to staging and wired into both invoice pipelines. Worth running a real invoice upload (via the UI or `/test-invoice`) to confirm the service match fires correctly and `service_match_score` / `service_match_status` populate on the invoice row.
- **Service match thresholds** — current thresholds are matched ≥ 0.85, pending 0.50–0.84, unmatched < 0.50. In the Bloomberg Terminal test, the score landed at 0.5946 (pending) due to name similarity dilution from the service description. Are these thresholds right for the business, or should they be adjusted?
- **Service name quality** — the name similarity score compares the invoice vendor name against the service name. If service names are generic (e.g. "Premium Communication Services" instead of "Bloomberg Terminal"), scores will be low. Worth auditing service names in staging before going live.
- **Inventory upload staging validation** — `process-inventory-document` was updated today. Ready for a test batch upload in staging to confirm service matching fires correctly on inventory-processed invoices.
- **Drop old column timing** — the old `created_record_id` column on `inventory_upload_items` is still present as a safety net. Once staging validation confirms everything is correct, it can be dropped.
- **Vendor match thresholds** — when the system is 50–84% confident on a vendor match, it flags it as "pending" for manual review. Is that the right cutoff, or should the team review everything below 85%?

## Recent changes

- Built and deployed `match-invoice-service` edge function to staging. Automatically scores and links invoices to services using weighted criteria: service name similarity (40%), invoice amount vs annual value prorated to billing period (25%), billing date overlap with service contract dates (20%), and bill-to entity match (15%).
- Wired `match-invoice-service` as a fire-and-forget step in both `process-invoice` and `process-inventory-document` — runs after every invoice is successfully created, only when a vendor was matched. Errors are logged but never block the invoice response.
- Fixed a scoring bug discovered during testing: the name similarity was being computed against the full service description text rather than just the service name, which diluted the score significantly for services with long descriptions.
- Fixed 6 pre-existing TypeScript type errors in `process-invoice` (the `insertErrorRow` function had an incompatible Supabase client parameter type — no runtime impact, purely cosmetic).
- Updated `client_modules` in staging to set the correct user ID for the invoices and contracts modules for the test org.
- Deployed `process-invoice`, `match-invoice-service`, and `process-inventory-document` to staging.
