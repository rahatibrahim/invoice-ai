# invoice-ai

AI-powered invoice processing that extracts structured data (vendor, date, amounts, tax, line items) from invoices using OCR and LLMs, and flags results that look wrong instead of silently accepting them.

> **Status: planning.** Documentation is in progress; no working code yet. Release 1 development is planned for December 2026 – April 2027. See the [roadmap](docs/04-roadmap.md).

## What it will do

- Take an invoice as **PDF** (text-based or scanned) or **image** (PNG, JPG, phone photo)
- Extract key fields: vendor, registration number, invoice number, dates, currency, subtotal, tax (per rate), total, line items
- Normalize values (dates including Japanese era dates, amounts with `¥` / `円` / full-width digits)
- Validate the data: totals, tax per rate, required fields — and flag problems
- Check whether a Japanese invoice contains the items required for a qualified invoice (適格請求書)
- Output JSON or CSV
- Measure its own accuracy against a labeled test set

The first release targets **Japanese invoices**. Other countries and languages may follow.

## Releases

| Release | Scope | Status |
|---|---|---|
| Release 1 | Extraction pipeline, command line | Planned: Dec 2026 – Apr 2027 |
| Release 2 | Web application (accounts, upload, review, export) | After Release 1 |

## Planned usage (Release 1)

These commands do not work yet; they show the intended interface.

```bash
# Extract one invoice to JSON
invoice-ai extract path/to/invoice.pdf

# Run the evaluation on the dev set
python -m eval.run --split dev
```

## Setup (planned)

Requires Python 3.12+. The stack is proposed in [architecture](docs/02-architecture.md) and will be confirmed after the feasibility spike.

```bash
git clone https://github.com/rahatibrahim/invoice-ai.git
cd invoice-ai
cp .env.example .env      # then add your API keys to .env
uv sync                   # install dependencies
```

## Documentation

| Document | Contents |
|---|---|
| [Requirements](docs/01-requirements.md) | Purpose, users, scope, functional and non-functional requirements |
| [Architecture](docs/02-architecture.md) | Pipeline design, output schema, tech stack, repository structure |
| [Evaluation plan](docs/03-evaluation-plan.md) | Test set, labels, metrics, how accuracy is measured |
| [Roadmap](docs/04-roadmap.md) | Phases, dates, and progress |
| [Decisions](docs/decisions/) | Records of significant technical decisions |

## Data and privacy

- This repository is public. It never contains real company or customer invoices
- Test invoices are synthetic or publicly licensed samples, kept in `data/`, which is excluded from Git
- API keys go in `.env`, which is excluded from Git. Only `.env.example` with placeholders is committed

## License

[MIT](LICENSE)
