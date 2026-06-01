# Source S4 — Product Status
*Last updated: 2026-06-01*

## What's live in production

- **Invoice Processing** — Users upload a PDF invoice and the system automatically reads it, identifies the vendor, finds the right account number, and saves all the data. Now also automatically attempts to link the invoice to its matching service record after every upload. PDFs are now saved to secure storage and a link is stored on each invoice record.
- **Contract Processing** — Users upload a PDF contract and the system extracts key fields (vendor, dates, value, renewal terms, cancellation notice) and stores them in a searchable database. Extracts 9 product catalog fields per service line. PDFs are now saved to secure storage and a link is stored on each contract record.
- **Vendor Match Resolution** — When the system can't confidently identify a vendor from a document, the user can confirm the correct vendor or add a new one directly from the UI. Now also supports manually linking an invoice to a specific contract (`link_contract` action), which writes the contract link, service link, and vendor assignment in one step and optionally saves the invoice vendor name as an alias for future matching.
- **Reconciliation** — Matches processed invoices against expected charges to flag discrepancies.
- **Service Match** — After every invoice is processed, the system automatically scores and links the invoice to the best matching service record. The best-scoring service is now always saved on the invoice even when the confidence is low, so no invoice is left with a blank service link.
- **PDF Storage** — Every contract and invoice PDF uploaded through the batch pipeline is now stored in a private Supabase Storage bucket, organised by client and document type. Each record holds a secure link to its source PDF.
- **Inventory Upload — full batch pipeline** — Full inventory batch pipeline live in production. Invoice-to-contract matching for new vendors now works end-to-end: invoice links to Draft contract automatically, vendor is created on user confirmation, invoice backfilled in one step.
- **Billing accounts** — Accounts can be flagged as billing accounts. Each allocation row in the contract detail shows a billing account dropdown. Accounts with no vendor assigned act as defaults across all vendors. Users can create a new billing account inline from the allocation row.
- **Inventory table** — Simplified to 6 columns (Vendor, Service, Start Date, End Date, Status, Cost). Collapsed vendor rows show aggregated data instead of empty dashes.
- **Contract detail improvements** — Action Date and Total Contract Value are now editable inline. Service name and annual value are editable in both the Contract tab (SERVICES section) and the Allocations tab. New services inherit vendor and currency from the contract.
- **Allocation user management** — Users can be created directly from the allocation picker (name, last name, department) without leaving the contract detail page.
- **Approved contract navigation** — Clicking an approved contract in the Upload Results screen now navigates to the correct contract detail page.
- **Vendor name normalization** — Vendor matching now handles `&` vs `"and"` correctly. Legal suffixes (LLC, Inc, Ltd, etc.) are stripped before comparison. Applied to both environments.
- **PDF viewer in contract and invoice detail** — "Review with PDF" now works in production. PDFs are downloaded via the Supabase client and displayed in a split-pane viewer. If the download fails, a fallback "Open in new tab" button appears.

## What's in staging (not yet in production)

- **Contract pricing derivation improvements** — The system now extracts year-by-year contract values and computes the total as a deterministic sum. Service annual value is always Year 1 only. Conservative service splitting deployed. Validated on Moody's contracts. Not yet promoted to production.
- **Contract Extraction V2** — Core extraction pipeline deployed to both environments. Full branch merge to main still pending — eval data, scripts, and golden test cases not yet merged.
- **User roster and invoice seat allocation** (`org_users`, `invoice_allocations`) — Database tables added. No edge function or UI built yet.

## What's in development

- **Monthly pricing normalization (P1)** — When a contract states prices in monthly terms (e.g. "$799/month"), the AI incorrectly uses the monthly price as the annual value instead of multiplying × 12. Confirmed broken on OPIS/CB Information contract in production. Review with Santiago before fixing.
- **Missing External Document Contracts report** — A cross-contract view showing all contracts where one or more fields point to an external document that hasn't been uploaded yet. Backend detection already works; only the list view and filtering need to be built.
- **Service split/merge UI** — Deferred. Spec is ready. Waiting for Santiago to validate the 4 extraction cases before deciding if manual split/merge is needed.
- **Vendor list page bug** — Two known filter issues: (1) auto-created vendors have no `source` tag and are excluded; (2) the vendor list only shows vendors with Active contracts. Fix pending — awaiting decision on correct scope.
- UI for manually linking an invoice to a contract (backend action is live; UI trigger not yet built).
- Service history tracking — snapshots recorded automatically, UI not yet built.

## What's coming next

1. **Monthly pricing normalization (P1)** — Fix extraction prompt so `unit_cost = monthly price` and `annual_value = unit_cost × 12` when contract states monthly rates. Review with Santiago first.
2. **Internal owner in Product Catalog** — Santiago requested this field appear in Product Catalog section. Verify if already present or needs adding.
3. **Service sum validation** — Sum of service annual values should not exceed contract total. Deferred — validate approach with Santiago first.
4. **N3: Supporting document flag + upload** — when a field requires a supplemental document, show a flag in the contract detail UI and allow the user to upload the doc.
5. **N5: Reports section** — new page: "Contracts missing supporting docs", "Missing allocations", "Missing product catalog."
6. **Large batch test** — Upload ~50 Stone X contracts+invoices to stress-test the allocation flow.
7. Build UI trigger for `link_contract`.
8. Fix vendor list page filters (source tag and contract status).
9. Merge `feat-extraction-v2` to main (housekeeping).

## Where your input would help

- **Monthly pricing fix** — Review the OPIS/CB Information contract in production and confirm the expected behavior before we change the extraction logic.
- **Santiago validation** — Re-upload DTCC, S&P, Refinitiv BDC+LPC, and Moody's via Inventory Upload and confirm the service split counts are correct. This gates whether the service split behavior is signed off.
- **Service sum validation** — Confirm whether users should be blocked or just warned when service values exceed the contract total.
- **Vendor list — source filter decision** — Auto-created vendors don't appear in the Vendors page. Fix options: (a) tag them at creation, or (b) remove the source filter. Decision needed.
- **`link_contract` UI** — Worth deciding where the UI trigger should live (batch review screen, invoice detail page, or both).
- **Stone X batch** — Santiago to share ~50 contracts+invoices for large-scale allocation testing.

## Recent changes

### Contract detail + allocation UX overhaul — June 1 session (part 2)

**Contract detail editing:**
- Action Date and Total Contract Value are now editable inline — users can correct values if extraction got them wrong.
- SERVICES section in the Contract tab now supports inline editing of service name and annual value, matching the Allocations tab behavior.
- New services created from the Allocations tab now inherit the vendor and currency from the contract automatically.

**Billing accounts:**
- Accounts with no vendor assigned (`DEFAULT-BILL`) now appear in the billing account dropdown for every contract — acts as a global default.
- When an allocation row has no billing accounts available, a "+ Add" link appears inline. Clicking it opens a mini-form to create and assign a billing account in one step without leaving the page.
- Add Account form now accepts accounts without a vendor ("All vendors") for creating global accounts.
- Placeholder billing accounts added to all staging vendors with Active contracts for testing.

**Allocation user management:**
- "+ Create new user" option added at the bottom of the user picker dropdown. Shows a mini inline form with first name, last name, and department. Creates the user and adds them to the allocation in one step.

**Inventory table:**
- Table simplified to 6 columns: Vendor, Service, Start Date, End Date, Status, Cost.
- Collapsed vendor rows now show: service name (or "N services" if multiple), "Active" status, date range, and total cost — no more empty dashes.

**Navigation fix:**
- Clicking an approved contract in the Upload Results screen now correctly navigates to the contract detail page instead of the old vendor page.

**Known bug identified:**
- Contracts with monthly pricing (e.g. OPIS: "$799/month") show the monthly price as the annual value instead of the yearly total. Fix scheduled for after Santiago review.

---

### Invoice-to-contract matching + billing accounts — June 1 session (part 1)

**Invoice-to-contract matching for new vendors:**
- When a contract and invoice for the same new vendor are uploaded together, the system now links them automatically — even when the vendor doesn't exist yet in the database.
- The invoice is marked as "contract pending" and linked to the Draft contract. When the user clicks Confirm on the contract, the vendor is created and the invoice is updated in one step.
- Works for both batch uploads (contract and invoice in the same batch) and mixed flows (contract in batch, invoice uploaded separately afterwards).
- Race conditions where the invoice arrives before the contract are handled by the reconciliation step that runs after the batch completes.
- Fixed a bug where contract items in a batch were incorrectly staying in "pending" state instead of "contract pending", causing them to show up under Needs Attention in the UI even when everything matched correctly.
- Validated end-to-end twice: once with Dremio (batch), once with City Falcon (mixed flow).

**Billing accounts:**
- Accounts can now be flagged as billing accounts. This distinguishes accounts that appear on invoices from platform/access accounts.
- The Accounts page now shows a Billing column with a toggle. All existing accounts default to billing = on.
- Each allocation row in the contract detail now has a billing account dropdown, showing only the billing accounts for that vendor.
- Validated: City Falcon test accounts created, selected in UI, confirmed saved in database.

---

### Inventory allocation editing + UX polish — May 29–31 session

**Allocation UI — full editing support:**
- Service name is now editable inline (click the name, pencil icon appears on hover → edit in place, save on Enter or blur).
- Annual value is now editable inline (click the value to edit).
- Start date and end date are shown per allocation row and editable with a native date picker — renamed to "Billing Start" / "Billing End".
- "Remove" now soft-deletes (sets `status = 'inactive'`, `end_date = today`) instead of hard-deleting.
- Inactive allocations are shown collapsed under "Show inactive (N)" toggle, rendered at reduced opacity.
- "+ Add service" inline form added to the Allocations tab — no modal required.

---

### Contract pricing derivation + service split — May 26 session

**Pricing derivation (ported from feat-extraction-v2):**

- The system now asks the AI to extract contract-level yearly values (Year 1, Year 2, Year 3) alongside regular fields.
- A deterministic rule computes `contract_value` as the sum of those yearly values — overriding the AI's arithmetic.
- `services.annual_value` always reflects Year 1 only, not a multi-year total.
- Validated on two Moody's contracts.

**Conservative service splitting:**

- Services are now separated only when the contract clearly distinguishes them by pricing, users/seats, billing terms, dates, license scope, delivery method, or materially different product purpose.
- Deterministic post-processing consolidates services that share the same pricing and billing terms.
- Validated 4 cases: DTCC (2 services ✅), S&P (1 service ✅), Refinitiv BDC+LPC (2 services ✅), Moody's (1 service ✅). Pending sign-off from Santiago.

---

### Contract Extraction V2 — May 20 session (staging only)

| Checkpoint | Contracts | Accuracy |
|---|---|---|
| Baseline (before this sprint) | 45 | 64.8% |
| After golden corrections | 44 | 66.8% |
| After vendor knowledge base | 8 (smoke) | 75.3% |
| After concept-based scoring | 8 (smoke) | 78.7% |
| After bug fixes + expanded eval framework | 10 (smoke) | **93.6%** |
| New vendors — zero tuning | 16 contracts | **86.4%** |
