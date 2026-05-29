# Source S4 — Product Status
*Last updated: 2026-05-29*

## What's live in production

- **Invoice Processing** — Users upload a PDF invoice and the system automatically reads it, identifies the vendor, finds the right account number, and saves all the data. Now also automatically attempts to link the invoice to its matching service record after every upload. PDFs are now saved to secure storage and a link is stored on each invoice record.
- **Contract Processing** — Users upload a PDF contract and the system extracts key fields (vendor, dates, value, renewal terms, cancellation notice) and stores them in a searchable database. Extracts 9 product catalog fields per service line. PDFs are now saved to secure storage and a link is stored on each contract record.
- **Vendor Match Resolution** — When the system can't confidently identify a vendor from a document, the user can confirm the correct vendor or add a new one directly from the UI. Now also supports manually linking an invoice to a specific contract (`link_contract` action), which writes the contract link, service link, and vendor assignment in one step and optionally saves the invoice vendor name as an alias for future matching.
- **Reconciliation** — Matches processed invoices against expected charges to flag discrepancies.
- **Service Match** — After every invoice is processed, the system automatically scores and links the invoice to the best matching service record. The best-scoring service is now always saved on the invoice even when the confidence is low, so no invoice is left with a blank service link.
- **PDF Storage** — Every contract and invoice PDF uploaded through the batch pipeline is now stored in a private Supabase Storage bucket, organised by client and document type. Each record holds a secure link to its source PDF.
- **Inventory Upload — full batch pipeline** — `check-batch-complete`, `reconcile-inventory-batch`, and `merge-vendors` promoted to production. The full inventory batch pipeline is now live in production.
- **Inventory Upload — Upload Invoice button** — The "Upload Invoice" button on contract cards in the batch review screen is now fully wired in both "Needs Review" and "Looks Good" sections. Was previously disabled on "Looks Good" cards.
- **Vendor name normalization** — Vendor matching now handles `&` vs `"and"` correctly. Legal suffixes (LLC, Inc, Ltd, etc.) are stripped before comparison. Applied to both environments.
- **PDF viewer in contract and invoice detail** — "Review with PDF" now works in production. PDFs are downloaded via the Supabase client and displayed in a split-pane viewer. If the download fails, a fallback "Open in new tab" button appears.
- **Contract and invoice PDFs migrated to production** — All 93 contracts and 77 invoices from staging have their PDFs copied to production storage with valid signed URLs.

## What's in staging (not yet in production)

- **Contract pricing derivation improvements** — The system now extracts year-by-year contract values and computes the total as a deterministic sum. Service annual value is always Year 1 only. Conservative service splitting deployed. Validated on Moody's contracts. Not yet promoted to production.
- **Contract Extraction V2** — Core extraction pipeline deployed to both environments. Full branch merge to main still pending — eval data, scripts, and golden test cases not yet merged.
- **User roster and invoice seat allocation** (`org_users`, `invoice_allocations`) — Database tables added. No edge function or UI built yet.

## What's in development

- **Missing External Document Contracts report** — A cross-contract view showing all contracts where one or more fields point to an external document that hasn't been uploaded yet. Backend detection already works; only the list view and filtering need to be built.
- **Service split/merge UI** — Deferred. Spec is ready. Waiting for Santiago to validate the 4 extraction cases before deciding if manual split/merge is needed.
- **Vendor list page bug** — Two known filter issues: (1) auto-created vendors have no `source` tag and are excluded; (2) the vendor list only shows vendors with Active contracts. Fix pending — awaiting decision on correct scope.
- UI for manually linking an invoice to a contract (backend action is live; UI trigger not yet built).
- Service history tracking — snapshots recorded automatically, UI not yet built.
- **InventoryDocuments page** — File exists and route is wired. Content and full functionality not yet validated.

## What's coming next

1. **Missing External Document Contracts report** — list view of all contracts with supplemental document flags.
2. **CoStar monthly billing normalization** — extractor returns monthly fees; need a rule to derive annual value from monthly pricing.
3. **Merge `feat-extraction-v2` to main** — housekeeping, no user-facing change.
4. Fix vendor list page filters (source tag and contract status).
5. Build UI trigger for `link_contract`.
6. Build UI for `org_users` and `invoice_allocations`.
7. Support for contracts that span multiple documents.
8. Second client onboarding.

## Where your input would help

- **Santiago validation** — Re-upload DTCC, S&P, Refinitiv BDC+LPC, and Moody's via Inventory Upload and confirm the service split counts are correct. This gates whether the service split behavior is signed off.
- **Vendor list — source filter decision** — Auto-created vendors don't appear in the Vendors page. Fix options: (a) tag them at creation, or (b) remove the source filter. Decision needed.
- **Vendor list — contract status filter decision** — Should the page show all contracts or only Active ones?
- **`link_contract` UI** — Worth deciding where the UI trigger should live (batch review screen, invoice detail page, or both).
- **InventoryDocuments page** — Confirm what this page should show and whether the current content is correct.

## Recent changes

### UI fixes and production stabilization — May 28–29 session

**PDF viewer now works in production:**
- All 93 contract and 77 invoice PDFs from staging were copied to production storage.
- Signed URLs (valid for 10 years) were generated and saved on each record.
- The "Review with PDF" split-pane viewer was fixed: PDFs are now downloaded via the Supabase client (bypassing browser iframe restrictions), then displayed in a local blob URL. A "Open in new tab" button shows as fallback if the download fails.

**Upload Invoice button fixed:**
- In the batch upload results screen, the "Upload Invoice" button on "Looks Good" cards was visually disabled. Fixed by passing the required props — it now works the same as "Needs Review" cards.
- In the contract detail page accessed directly from Inventory (not from a batch), the button now shows a clear error message instead of silently doing nothing.

**Inventory page renamed:**
- Sidebar label and page heading changed from "Vendors" to "Inventory".
- Clicking a vendor now expands it inline instead of navigating to a separate page.
- Clicking a contract navigates to the contract detail page.

**Back navigation fixed:**
- "Back to Upload Results" now correctly shows "Back to Inventory" when reaching a contract detail from the Inventory page directly.

**Repo cleanup:**
- All feature branches merged to main and deleted. Both backend and frontend repos are on a single clean `main` branch.
- Staging and production DB schemas are fully in sync (both at migration `20260528000004`).
- All edge functions deployed to both environments on the same day.

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

### P0 extraction fixes deployed to both environments — May 25 session

- Batch contract uploads now use the V2 prompt.
- JSON reliability fix applied to batch pipeline.
- `internal_owner` priority corrected.

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
