# Requirements — invoice-ai

| Item | Value |
|---|---|
| Status | Draft |
| Owner | Ibrahim |
| Last updated | 2026-11-03 |

> Items marked **TBD** are open decisions. Release 1 items are resolved during Release 1, in the phase noted in `04-roadmap.md` (e.g. the extraction approach in P2, the accuracy targets in P6). Release 2 items are resolved before Release 2 design starts.

## 1. Purpose

Small business owners and freelancers receive invoices as PDFs, scans, and photos. Reading them and typing their data into a spreadsheet or accounting tool is slow and error-prone.

invoice-ai is a web application that takes an invoice file and returns its key data as structured output. It flags results that look wrong instead of silently accepting them, lets the user review and correct the data, and exports it for use in other tools.

The first release targets **Japanese invoices** (請求書), including those issued under the Qualified Invoice System (適格請求書等保存方式, "インボイス制度"). Other countries and languages may follow later.

The work is delivered in two releases:
- **Release 1 — Extraction pipeline (CLI):** extraction, validation, qualified-invoice checks, and evaluation, run from the command line. This proves the AI part works and is the core of the project.
- **Release 2 — Web application:** accounts, upload UI, review screen, history, export, deployment, and operations, built on top of the Release 1 pipeline.

Release 2 starts only after Release 1 meets its success criteria (section 10.1).

## 2. Users

### Target users

| User | Need |
|---|---|
| Small business owner in Japan | Digitize received invoices without manual data entry; keep a searchable record; export data for bookkeeping and tax filing |
| Freelancer / sole proprietor (個人事業主) in Japan | Same as above, usually with lower volume and no dedicated bookkeeping staff; check whether received invoices are qualified invoices for input tax credit |

Typical characteristics: low to moderate volume (roughly tens of invoices per month), no technical background, often uploads from a phone. Many will prefer a Japanese UI, which is planned after Release 2.

### Project roles

| Role | Need |
|---|---|
| Developer / contributor | Run the app and pipeline locally, measure accuracy, compare approaches |
| Operator | Deploy, monitor, and maintain the hosted service; respond to errors and incidents |

## 3. Scope

### In scope — Release 1 (Extraction pipeline, CLI)
- Japanese invoices, in yen and in foreign currencies
- Input of single-page and multi-page invoices as **PDF** (text-based and scanned) and **images** (PNG, JPG, including phone photos) through a command-line interface
- Extraction of the fields listed in section 5
- Normalization and validation of extracted data (section 6)
- Qualified-invoice check (FR-12) and registration number format check (FR-13)
- Output as JSON and CSV
- An evaluation script that measures accuracy against a labeled test set

### In scope — Release 2 (Web application)
- Web application in English, usable on desktop and mobile browsers, with all UI text kept in translation files so a Japanese UI can be added without code changes
- User accounts: sign up, log in, log out, password reset
- Upload from the web UI, including camera capture from mobile browsers
- Review screen where the user checks, corrects, and confirms extracted data
- Invoice history per user, with deletion
- Export as CSV and JSON from the UI
- Deployment to a hosted environment with basic monitoring

### Out of scope (for now)
- Handwritten invoices
- Receipts, quotations, purchase orders, and other document types
- Integration with accounting software (candidate for a later release)
- Team or organization accounts (multiple users sharing data)
- Native mobile apps (the web app must work in mobile browsers instead)
- Public API for third-party developers
- Payments and billing (**TBD**, see section 12)
- Invoices from countries other than Japan, and invoices in languages other than Japanese
- Japanese UI (planned for a later release)
- Acting as a storage system compliant with the Electronic Books Maintenance Act (電子帳簿保存法). Users remain responsible for keeping the original invoices (see section 12)

## 4. Input

- One invoice file per upload
- Supported formats: PDF, PNG, JPG
- Maximum file size: **TBD** (e.g. 10 MB)
- Maximum pages: **TBD** (e.g. 5)
- Maximum uploads per user per day: **TBD** (protects against abuse and runaway API cost)

## 5. Output — extracted fields

| Field | Type | Required | Notes |
|---|---|---|---|
| vendor_name | string | yes | Issuer of the invoice |
| vendor_registration_number | string | no | Qualified invoice issuer registration number (登録番号): `T` followed by 13 digits |
| recipient_name | string | no | Addressee (宛名) |
| invoice_number | string | yes | |
| invoice_date | date (ISO 8601) | yes | Converted from formats such as `2026年11月2日` and Japanese era dates (e.g. `令和8年11月2日`) |
| due_date | date (ISO 8601) | no | Payment due date (支払期限) |
| currency | string (ISO 4217) | yes | Usually JPY |
| subtotal | decimal | no | Amount before tax (税抜) |
| tax_amount | decimal | no | Total consumption tax (消費税) |
| tax_breakdown | list | no | One entry per tax rate: rate (10% or 8% reduced), taxable amount, tax amount |
| total_amount | decimal | yes | Amount including tax (税込) |
| line_items | list | no | description, quantity, unit_price, amount, tax_rate (marks reduced-rate items, e.g. `※`) |
| flags | list | yes | Per-field warnings from validation (empty if none) |

The exact JSON schema is defined in `02-architecture.md`.

## 6. Functional requirements

| Release | Requirements |
|---|---|
| Release 1 | FR-5 (applied to CLI input), FR-7 to FR-13, FR-20, FR-21 |
| Release 2 | FR-1 to FR-4, FR-5 (web upload), FR-6, FR-14 to FR-19 |

### Accounts and access

| ID | Requirement |
|---|---|
| FR-1 | Users can sign up, log in, log out, and reset their password |
| FR-2 | Each user can see and act on only their own invoices |
| FR-3 | Users can delete their account, which deletes all their invoices and extracted data |

### Upload and processing

| ID | Requirement |
|---|---|
| FR-4 | Users can upload a PDF or image file from the web UI, including from a phone camera |
| FR-5 | Reject unsupported, oversized, or over-limit files with a clear message |
| FR-6 | Show processing status (queued, processing, done, failed) without the user having to keep the page open |
| FR-7 | Convert input to a form the extraction step can read (text and/or image) |
| FR-8 | Extract the fields in section 5 |
| FR-9 | Normalize values: dates (including Japanese era dates) to ISO 8601; amounts written with `¥`, `円`, commas, or full-width digits to decimals; currency codes to ISO 4217 |
| FR-10 | Validate: subtotal + tax = total (within rounding); sum of line items = subtotal; tax per rate ≈ taxable amount × rate (within ±1 yen rounding); required fields present; dates valid |
| FR-11 | Mark fields that fail validation or are missing, instead of guessing |
| FR-12 | Check whether the invoice contains the items a qualified invoice (適格請求書) must include: issuer name and registration number, transaction date, transaction description, amounts and applicable tax rates per rate, consumption tax per rate, and recipient name. Tell the user which items are missing |
| FR-13 | Check the format of the registration number (`T` + 13 digits). Verifying it against the National Tax Agency's public registry is **TBD** (see section 12) |

### Review, history, and export

| ID | Requirement |
|---|---|
| FR-14 | Show the invoice next to the extracted fields, with flagged fields highlighted |
| FR-15 | Users can correct any field and mark the invoice as confirmed |
| FR-16 | Users can list past invoices and filter them by date, vendor, and status |
| FR-17 | Users can export one or more invoices as CSV or JSON |
| FR-18 | Users can delete individual invoices |
| FR-19 | Uploaded files and extracted data are deleted automatically after the retention period (**TBD**, e.g. 12 months) |

### Operations and evaluation

| ID | Requirement |
|---|---|
| FR-20 | Log each processing run: model/approach used, time taken, cost, errors. Logs must not contain invoice contents |
| FR-21 | Provide an evaluation script that runs the pipeline on the test set and reports accuracy |

## 7. Non-functional requirements

| Release | Requirements |
|---|---|
| Release 1 | NFR-1 to NFR-4, NFR-6 (no secrets in the repository), NFR-11 |
| Release 2 | NFR-5, NFR-6 (all), NFR-7 to NFR-10, NFR-12 |

| ID | Requirement | Target |
|---|---|---|
| NFR-1 | Field-level accuracy on the test set | **TBD** (e.g. ≥ 90%) |
| NFR-2 | Required-field accuracy (total, date, vendor, invoice number) | **TBD** (e.g. ≥ 95%) |
| NFR-3 | Processing time per invoice | **TBD** (e.g. < 15 s) |
| NFR-4 | Cost per invoice (API usage) | **TBD** (e.g. < ¥5) |
| NFR-5 | Availability of the hosted service | **TBD** (e.g. 99% monthly) |
| NFR-6 | Security: HTTPS everywhere; files and data encrypted at rest; passwords never stored in plain text; no API keys or secrets in the repository | Required |
| NFR-7 | Privacy: comply with Japan's Act on the Protection of Personal Information (個人情報保護法, APPI); publish a privacy policy and terms of service before launch (Japanese versions with the Japanese UI) | Required |
| NFR-8 | Backups of user data, with a tested restore procedure | **TBD** (frequency, retention) |
| NFR-9 | Monitoring: errors and processing failures are recorded and alert the operator | Required |
| NFR-10 | Usability: the UI is in English, and the main flow (upload, review, export) works on current desktop and mobile browsers | Required |
| NFR-11 | Reproducibility: any contributor can set up the app locally and run the evaluation with the README instructions | Required |
| NFR-12 | Internationalization: no hard-coded UI text; dates, numbers, and currency are formatted per locale, so a Japanese UI can be added by adding translations | Required |

## 8. Data

### Test data
- Test set: 30–50 Japanese invoices with hand-labeled correct values
- Coverage: qualified and non-qualified invoices; single and mixed tax rates (10% and 8%); text-based PDFs, scans, and phone photos; documents with company seals (印鑑) overlapping text
- Sources: synthetic or publicly available sample invoices only. **No real company or customer invoices**, because this repository is public.
- Location: `data/` (excluded from Git via `.gitignore`)
- Labels format: **TBD** (e.g. one JSON file per invoice with the expected output)

### User data
- Uploaded files and extracted data belong to the user and are stored only for the retention period (FR-19) or until the user deletes them
- User data is never committed to the repository or copied into the test set
- Whether user data or corrections may be used to improve the system is **TBD** (if allowed, only with explicit opt-in)

## 9. Constraints

- Timeline:
  - Release 1: December 2026 – April 2027, about 6–8 hours per week (120–160 hours of capacity; about 140 hours planned in `04-roadmap.md`)
  - Release 2: **TBD**, planned after Release 1 is complete
- Budget: AI/OCR APIs and hosting, **TBD** per month
- Language / stack: **TBD** (decided in `02-architecture.md`)
- Hosting provider: **TBD** (decided in `02-architecture.md`)
- The app must run locally for development and evaluation, and in the hosted environment for production
- The repository is public. All code, documentation, and test data must be safe to publish; secrets and user data must never be committed
- Code and project documentation are written in English; the user interface is in English first, with Japanese added later

## 10. Success criteria

### 10.1 Release 1 is complete when
1. The pipeline processes every invoice in the test set end to end from the CLI
2. Accuracy, time, and cost targets in section 7 (NFR-1 to NFR-4) are met on the test set, or the gap is measured and explained
3. Invalid results are flagged by validation, and the qualified-invoice check reports missing items
4. The README lets a contributor set up the pipeline locally and run the evaluation

### 10.2 Release 2 is complete when
1. The app is deployed at a public URL over HTTPS
2. A new user can sign up, upload an invoice, review and correct it, and export the result without help
3. Tests confirm that one user cannot access another user's data
4. Deleting an invoice or an account removes the associated files and data
5. Monitoring and alerts are in place for errors and failed processing

## 11. Assumptions and risks

| Risk | Mitigation |
|---|---|
| Low accuracy on scanned or low-quality images | Preprocessing; measure separately from clean PDFs; flag low-confidence fields for review |
| Japanese-specific layout issues: seals overlapping text, vertical text, full-width characters, era dates | Include these cases in the test set; measure them separately; normalization rules with unit tests |
| Target users find an English-only UI hard to use | Build the UI for translation from the start (NFR-12); test the main flow with Japanese users; prioritize the Japanese UI after Release 2 |
| Users rely on the app as their legal record of invoices | State clearly in the UI and terms that the app does not replace keeping the originals under 電子帳簿保存法 |
| Users export wrong data without noticing | Highlight flagged fields; require review of flagged fields before an invoice can be confirmed |
| Data breach or leak of invoice data | Access control per user, encryption at rest, minimal logging, retention limit, dependency updates |
| API or hosting cost higher than expected | Track cost per run; per-user upload limits; monthly budget alerts |
| AI provider outage, price change, or model deprecation | Keep the extraction step behind an interface so the provider can be swapped; monitor failure rates |
| Hard to obtain enough varied sample invoices | Generate synthetic invoices with varied layouts |
| The product is not useful enough to real users | Test the main flow with a few small business owners or freelancers before launch |
| Scope creep | Anything not in section 3 goes to a "later" list |

## 12. Open questions

- Should registration numbers be verified against the National Tax Agency's public registry (Web-API, requires an application ID)?
- Should the app later support storage that complies with 電子帳簿保存法 (e.g. 7-year retention, search by date, amount, and counterparty)? This would change the retention requirement (FR-19)
- Must user data be stored in Japan, or is hosting abroad acceptable under APPI with the required disclosures?
- Which approach: LLM with vision only, or OCR + LLM? (decide after the feasibility spike)
- Authentication: build in-house or use an auth provider?
- Hosting provider and deployment setup
- Retention period for uploaded files and extracted data
- Pricing: free beta, free tier plus paid plan, or paid only? When does billing enter scope?
- May user data or corrections be used to improve the system (opt-in)?
- Expected number of users and invoices per month at launch
- Timeline for Release 2
- Exact accuracy, time, cost, and availability targets (section 7)
