# Evaluation Plan — invoice-ai

| Item | Value |
|---|---|
| Status | Draft |
| Owner | Ibrahim |
| Last updated | 2026-11-03 |
| Related | `01-requirements.md` (FR-8 to FR-13, FR-21, NFR-1 to NFR-4), `02-architecture.md` (JSON schema) |

> Items marked **TBD** are open decisions.

## 1. Purpose

This plan defines how the extraction pipeline's quality is measured, so that:
- every change (prompt, model, OCR engine, preprocessing) can be compared against a fixed baseline
- "good enough" is decided by numbers, not impressions
- results can be reproduced by anyone with the repository and the test set

## 2. What is evaluated

| Component | Evaluated here | Requirement |
|---|---|---|
| Field extraction | Yes | FR-8 |
| Normalization (dates, era dates, amounts, currency) | Yes, through the extracted values and through unit tests | FR-9 |
| Validation flags | Yes | FR-10, FR-11 |
| Qualified-invoice check (missing items) | Yes | FR-12 |
| Registration number format check | Yes, unit tests | FR-13 |
| Processing time and cost | Yes | NFR-3, NFR-4 |
| Web UI, accounts, export | No — covered by normal application tests | — |

## 3. Test set

### 3.1 Size and split

- Total: 30–50 invoices to start
- **Dev set** (~60%): used freely while improving prompts and code
- **Held-out test set** (~40%): run only at milestones (end of a roadmap phase). Never used to tune prompts
- The split is fixed once created and recorded in `data/split.json`

Why the split: with a small set it is easy to tune the prompt until the dev set looks perfect without the pipeline actually getting better. The held-out set shows the real accuracy.

### 3.2 Coverage

The set must include each of these cases at least a few times (exact targets **TBD**):

| Dimension | Cases |
|---|---|
| File type | Text-based PDF, scanned PDF, phone photo (PNG/JPG) |
| Quality | Clean, skewed, low resolution, shadows / glare (photos) |
| Pages | Single-page, multi-page |
| Invoice type | Qualified invoice (適格請求書), non-qualified invoice, qualified invoice missing one or more required items |
| Tax | Single rate (10%), mixed rates (10% + 8% reduced), tax-exclusive and tax-inclusive line amounts |
| Layout | Table-based, free-form, vertical text, company seal (印鑑) overlapping text |
| Dates | Western (`2026年11月2日`, `2026/11/02`), Japanese era (`令和8年11月2日`) |
| Numbers | `¥`, `円`, commas, full-width digits |
| Currency | JPY, at least one foreign currency |
| Errors | Invoices with an intentional arithmetic error (e.g. total ≠ subtotal + tax), to test validation |

### 3.3 Sources

- Synthetic invoices generated from templates with varied layouts and values
- Publicly available sample invoices whose license allows use
- **Never** real company or customer invoices (the repository is public)

### 3.4 Storage and naming

```
data/                       # git-ignored
├── invoices/
│   ├── inv-0001.pdf
│   └── inv-0002.jpg
├── labels/
│   ├── inv-0001.json
│   └── inv-0002.json
├── metadata.csv            # id, file type, quality, layout, tax type, ... (used for slicing)
└── split.json              # {"dev": [...], "test": [...]}
```

If the synthetic generator and its output are safe to publish, a small subset may later be committed under `samples/` so others can run the evaluation.

## 4. Labels

### 4.1 Format

One JSON file per invoice, using the same schema as the pipeline output (defined in `02-architecture.md`), plus the expected validation and qualified-invoice results.

```json
{
  "id": "inv-0001",
  "fields": {
    "vendor_name": "株式会社サンプル商事",
    "vendor_registration_number": "T1234567890123",
    "recipient_name": "山田太郎",
    "invoice_number": "INV-2026-0042",
    "invoice_date": "2026-11-02",
    "due_date": "2026-11-30",
    "currency": "JPY",
    "subtotal": 10000,
    "tax_amount": 920,
    "tax_breakdown": [
      {"rate": 0.10, "taxable_amount": 6000, "tax_amount": 600},
      {"rate": 0.08, "taxable_amount": 4000, "tax_amount": 320}
    ],
    "total_amount": 10920,
    "line_items": [
      {"description": "コピー用紙", "quantity": 2, "unit_price": 3000, "amount": 6000, "tax_rate": 0.10},
      {"description": "お茶 ※", "quantity": 10, "unit_price": 400, "amount": 4000, "tax_rate": 0.08}
    ]
  },
  "expected_flags": [],
  "expected_missing_qualified_items": []
}
```

### 4.2 Labeling rules

- Label what is **printed on the invoice**, normalized to the schema (ISO dates, decimal amounts, ISO currency codes)
- A field not present on the invoice is `null`. The pipeline is correct only if it also returns `null` (no guessing, FR-11)
- Strings are copied as written, including legal entity markers (株式会社, (株)); comparison rules in section 5.2 handle formatting differences
- If an invoice contains an error (e.g. wrong total), label the printed value and list the expected flag in `expected_flags`
- Every label is checked twice: once when written, once on a later day. Labeling mistakes make the metrics meaningless

## 5. Metrics

### 5.1 Primary metrics

| Metric | Definition | Target |
|---|---|---|
| Field-level accuracy | Correct fields ÷ all fields, over all invoices (scalar fields only) | NFR-1 (**TBD**, e.g. ≥ 90%) |
| Required-field accuracy | Same, for `vendor_name`, `invoice_number`, `invoice_date`, `total_amount` | NFR-2 (**TBD**, e.g. ≥ 95%) |
| Document-level accuracy | Invoices where **all** required fields are correct ÷ all invoices | Reported, no target yet |
| Per-field accuracy | Accuracy of each field separately | Reported, used to find weak fields |

`currency` is a required field (`02-architecture.md` §2.5) but is left out of required-field accuracy: it is almost always JPY, so including it would inflate the score. It is still measured in field-level and per-field accuracy.

### 5.2 Comparison rules

Before comparing, both expected and actual values are normalized:

| Type | Rule |
|---|---|
| String | Unicode NFKC (full-width → half-width), trim, collapse whitespace. `(株)` and `株式会社` are treated as equal. Otherwise exact match |
| Date | Exact match after conversion to ISO 8601 |
| Amount | Exact match (yen has no fractional part; foreign currencies to 2 decimals) |
| Currency | Exact ISO 4217 code |
| Registration number | Exact match, `T` + 13 digits |
| `null` | Correct only if both are `null`. A value where `null` is expected counts as a **hallucination** and is reported separately |

### 5.3 Line items and tax breakdown

Lists are matched item by item:
- Items are paired by best match on description and amount
- Report **precision** (extracted items that are correct), **recall** (expected items that were found), and per-column accuracy for paired items
- `tax_breakdown` is matched by `rate`

### 5.4 Validation and qualified-invoice checks

| Metric | Definition |
|---|---|
| Flag recall | Of the invoices/fields that should be flagged, how many were flagged |
| Flag precision | Of the flags raised, how many were justified |
| Wrong-but-unflagged rate | Fields that are wrong **and** not flagged ÷ all fields. This is the most important safety metric: these errors reach the user silently |
| Qualified-invoice check accuracy | Invoices where the list of missing items matches `expected_missing_qualified_items` exactly |

### 5.5 Operational metrics

| Metric | Target |
|---|---|
| Processing time per invoice (median and max) | NFR-3 (**TBD**, e.g. < 15 s) |
| Cost per invoice (average, from API usage) | NFR-4 (**TBD**, e.g. < ¥5) |
| Failure rate (crash, timeout, invalid JSON) | Reported; target 0% on the test set |

## 6. Slicing

Every metric is also reported per slice from `metadata.csv`: file type, quality, layout, tax type, qualified/non-qualified. An overall average can hide that, for example, all phone photos fail.

## 7. Running the evaluation

```bash
# command name TBD once the stack is chosen
python -m eval.run --split dev
python -m eval.run --split test   # milestones only
```

Each run produces a report in `eval/results/<timestamp>/` (git-ignored) containing:
- `summary.md`: metrics from section 5 and slices from section 6
- `details.csv`: one row per invoice × field — expected, actual, correct, flagged
- `run.json`: git commit hash, model name and version, prompt version, OCR engine, parameters, date, total cost

`run.json` makes every result traceable to the exact code and settings that produced it.

## 8. Process

1. **Baseline:** run the first working pipeline on the dev set and record the result
2. **One change at a time:** change one thing (prompt, model, preprocessing), re-run the dev set, compare with the previous run
3. **Keep or revert:** keep the change only if it improves the target metrics without making the wrong-but-unflagged rate worse
4. **Log decisions:** record significant changes and their results in `docs/decisions/`
5. **Milestones:** at the end of each roadmap phase, run the held-out test set once and record the result in `docs/evaluation-results.md`

### Interpreting small differences

With 30–50 invoices, one invoice is 2–3% of the set. Differences of a few percentage points between runs may be noise. LLM output can also vary between identical runs, so:
- set temperature to 0 (or the lowest available) for evaluation runs
- for important comparisons, run each configuration **3 times** and report the average and range
- prefer conclusions supported by several metrics or slices, not one number

## 9. Limitations

- A small, partly synthetic test set may not represent real users' invoices. Accuracy on real invoices may be lower
- The test set should grow over time, especially with cases the pipeline fails on. New cases go into the dev set; the held-out set is changed only deliberately and the change is recorded
- Exact-match rules may count acceptable variants as wrong (e.g. different spacing in a vendor name). Review `details.csv` before trusting a low score

## 10. Open questions

- Exact coverage targets per case in section 3.2
- Final values for NFR-1 to NFR-4
- Tool for creating synthetic invoices
- Should a small synthetic sample be committed so others can run the evaluation?
- Is fuzzy matching (e.g. edit distance) needed for `description` in line items, or is exact match after normalization enough?
