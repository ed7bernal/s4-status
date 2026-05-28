# Source S4 — Product Status
*Last updated: 2026-05-26*

## What's live in production

- **Invoice Processing** — Users upload a PDF invoice and the system automatically reads it, identifies the vendor, finds the right account number, and saves all the data. Now also automatically attempts to link the invoice to its matching service record after every upload. PDFs are now saved to secure storage and a link is stored on each invoice record.
- **Contract Processing** — Users upload a PDF contract and the system extracts key fields (vendor, dates, value, renewal terms, cancellation notice) and stores them in a searchable database. Extracts 9 product catalog fields per service line. PDFs are now saved to secure storage and a link is stored on each contract record.
- **Vendor Match Resolution** — When the system can't confidently identify a vendor from a document, the user can confirm the correct vendor or add a new one directly from the UI. Now also supports manually linking an invoice to a specific contract (`link_contract` action), which writes the contract link, service link, and vendor assignment in one step and optionally saves the invoice vendor name as an alias for future matching.
- **Reconciliation** — Matches processed invoices against expected charges to flag discrepancies.
- **Service Match** — After every invoice is processed, the system automatically scores and links the invoice to the best matching service record. The best-scoring service is now always saved on the invoice even when the confidence is low, so no invoice is left with a blank service link.
- **PDF Storage** — Every contract and invoice PDF uploaded through the batch pipeline is now stored in a private Supabase Storage bucket, organised by client and document type. Each record holds a secure link to its source PDF.
- **Inventory Upload — full batch pipeline** — `check-batch-complete`, `reconcile-inventory-batch`, and `merge-vendors` promoted to production. The full inventory batch pipeline is now live in production.
- **Inventory Upload — Upload Invoice button** — The "Upload Invoice" button on contract cards in the batch review screen is fully wired end-to-end in production.
- **Vendor name normalization** — Vendor matching now handles `&` vs `"and"` correctly (e.g. "S&P" matches "S and P"). Legal suffixes (LLC, Inc, Ltd, etc.) are stripped before comparison in both the JavaScript matching engine and the PostgreSQL `match_vendor` function. Applied to both environments.

## What's in staging (not yet in production)

- **Contract pricing derivation improvements** — The system now extracts year-by-year contract values (Year 1, Year 2, Year 3) and computes the total contract value as a deterministic sum — never trusting the AI's arithmetic. Service annual value is always Year 1 only. Conservative service splitting also deployed: services are only separated when the contract clearly distinguishes them by pricing, billing terms, delivery method, or materially different purpose. Deployed to staging on `feat/inventory-period-allocations`. Validated on Moody's contracts. Not yet in production.
- **Contract Extraction V2** — Core extraction pipeline (V2 prompt, deterministic derivation, vendor hints) deployed to both environments. Full branch merge to main still pending — eval data, scripts, and golden test cases not yet merged.
- **User roster and invoice seat allocation** (`org_users`, `invoice_allocations`) — Database tables added to staging and production. No edge function or UI built yet.

## What's in development

- **Missing External Document Contracts report** — Starting next session. A cross-contract view showing all contracts where one or more fields point to an external document (Exhibit A, Schedule, Order Form, MSA, etc.) that hasn't been uploaded yet. Backend detection already works; only the list view and filtering need to be built.
- **Service split/merge UI** — Deferred. Spec is ready (two Supabase RPCs + dialogs). Waiting for Santiago to validate the 4 extraction cases (DTCC=2, S&P=1, Refinitiv BDC+LPC=2, Moody's=1) before deciding if manual split/merge is needed.
- **Vendor list page bug** — Two known filter issues: (1) auto-created vendors have no `source` tag and are excluded from the vendor list query; (2) the vendor list only shows vendors with Active contracts. Fix pending — awaiting decision on correct scope.
- UI for manually linking an invoice to a contract (the `link_contract` backend action is live; the UI trigger is not yet built).
- Service history tracking — snapshots recorded automatically, UI not yet built.
- PDF viewer — storage in place, View PDF buttons not yet wired to signed URLs.

## What's coming next

1. **Missing External Document Contracts report** — list view of all contracts with supplemental document flags. No backend changes needed; query + UI only.
2. **CoStar monthly billing normalization** — extractor returns monthly fees; need a rule to derive annual value from monthly pricing.
3. **Merge `feat-extraction-v2` to main** — housekeeping, no user-facing change.
4. Fix vendor list page filters (source tag and contract status).
5. Build UI trigger for `link_contract`.
6. Surface stored PDF links — wire View PDF buttons to signed URLs.
7. Build UI for `org_users` and `invoice_allocations`.
8. Support for contracts that span multiple documents.
9. Second client onboarding.

## Where your input would help

- **Santiago validation** — Re-upload DTCC, S&P, Refinitiv BDC+LPC, and Moody's via Inventory Upload and confirm: (a) DTCC shows 2 services, (b) S&P shows 1 service, (c) Refinitiv BDC+LPC shows 2 services, (d) Moody's shows 1 service. This gates whether the service split behavior is signed off or needs further tuning.
- **Vendor list — source filter decision** — Auto-created vendors (score < 0.50) are inserted without a `source` tag and don't appear in the Vendors page. Fix options: (a) tag them `source = 'inventory_upload'` at creation, or (b) remove the source filter entirely. Decision needed before fixing the page.
- **Vendor list — contract status filter decision** — The vendor list only shows vendors with Active contracts. Vendors with Draft contracts are hidden. Should the page show all contracts, or show all with a status badge?
- **`link_contract` UI** — The backend action is deployed and tested. Worth deciding where the UI trigger should live — on the contract card in the batch review screen, on the invoice detail page, or both.

## Recent changes

### Contract pricing derivation + service split — May 26 session

**Pricing derivation (ported from feat-extraction-v2):**

- The system now asks the AI to extract contract-level yearly values (Year 1, Year 2, Year 3) alongside regular fields. These are stored in `raw_extraction.pricing_evidence.contract_year_values`.
- A deterministic rule then computes `contract_value` as the sum of those yearly values — overriding the AI's arithmetic, which was shown to be wrong on a real Moody's contract (AI returned $170,441 instead of the correct $167,474).
- `services.annual_value` always reflects Year 1 only, not a multi-year total.
- Validated on two Moody's contracts:
  - 3-year contract: Year 1 $55,771 + Year 2 $52,864 + Year 3 $58,838 → `contract_value` $167,474
  - 2-year contract: Year 1 $52,704 + Year 2 $56,920 → `contract_value` $109,624 (matched explicit document total)

**Conservative service splitting:**

- Updated the AI prompt to avoid over-splitting. Services are now separated only when the contract clearly distinguishes them by pricing, users/seats, billing terms, dates, license scope, delivery method, or materially different product purpose.
- Added a deterministic post-processing step: when the AI splits services that share the same pricing and billing terms, the code consolidates them back into one service automatically.
- Validated 4 cases: DTCC (2 services ✅), S&P (1 service ✅), Refinitiv BDC+LPC (2 services ✅), Moody's (1 service ✅). Pending sign-off from Santiago.

**UI:**

- Contract Detail now shows multiple services as stacked expandable cards when a contract has more than one service. Single-service contracts look identical to before.
- Service split/merge (user-initiated) documented as technical debt — spec ready, deferred until extraction defaults are confirmed correct.

---

### Inventory contract pricing validation — May 26 session (earlier)

Validated the pricing model with Santiago's guidance:

- **Contract Value** means total contract value when clearly stated, or the deterministic sum of clear contract-level yearly values.
- **Service Annual Value** means the Year 1 recurring value for the service.
- **CUSIP counts are not seats** — CUSIP-based licenses leave `licensed_seats` empty unless users/seats are explicitly stated.

---

### P0 extraction fixes deployed to both environments — May 25 session

- **Batch contract uploads now use the V2 prompt** — Contracts uploaded through the batch pipeline were using an older, less accurate prompt. Fixed in both environments.
- **JSON reliability fix** — The AI extraction call in the batch pipeline did not force JSON output mode. Fixed in both environments.
- **`internal_owner` priority corrected** — Request body value now correctly takes priority over org-level default. Fixed in both environments.

---

### Contract Extraction V2 — May 20 session (staging only, branch feat-extraction-v2)

**Accuracy progression:**

| Checkpoint | Contracts | Accuracy |
|---|---|---|
| Baseline (before this sprint) | 45 | 64.8% |
| After golden corrections | 44 | 66.8% |
| After vendor knowledge base | 8 (smoke) | 75.3% |
| After concept-based scoring | 8 (smoke) | 78.7% |
| After bug fixes + expanded eval framework | 10 (smoke) | **93.6%** |
| New vendors — zero tuning | 16 contracts | **86.4%** |

**New vendor baseline (no vendor-specific tuning):**

| Vendor | Accuracy |
|---|---|
| AlphaSense | 100% |
| TSX / Moodys / PitchBook | 90–93% |
| DTCC | 87% |
| CoStar | 75% |
| Deutsche Börse | 46%* |

*\*Master agreement only — no pricing or products to extract. Low score expected by design.*
