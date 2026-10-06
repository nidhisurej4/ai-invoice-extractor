# AI Invoice Extractor

Reads invoice emails, pulls out the structured data with an LLM, checks that the numbers actually add up, and exports an Excel file for finance, with anything suspicious routed to a "Needs Review" sheet instead of being passed along silently.

This is the kind of email → Excel workflow I automate as an AI Builder: the LLM does the reading, but deterministic validation decides what is trusted.

## How it works

```
email / .txt ──▶ extract (LLM or rule-based) ──▶ validate ──▶ SQLite ──▶ Excel (Approved | Needs Review)
                                                     │
                                                     └─ subtotal + tax = total? line items = subtotal?
                                                        due date after invoice date? required fields?
```

- **Two extractors.** With `ANTHROPIC_API_KEY` set, Claude extracts fields as JSON. Without it, a regex parser runs, so the project works offline and CI needs no secrets. The rule-based output is also a baseline to measure the LLM against.
- **Validation over trust.** Every invoice is reconciled. A mismatch produces a readable reason, e.g. `subtotal + tax = 528.0, total says 580.0`.
- **Graceful fallback.** If the LLM call fails, the document is still processed by the parser instead of being lost.

## Tech stack

Python · FastAPI · Anthropic API (Claude) · Prompt engineering · SQLite · openpyxl · pytest · GitHub Actions

## Run it

```bash
pip install -r requirements.txt
cp .env.example .env          # optional: add your API key

# Batch mode: process a folder and write an Excel file
python -m invoice_ai.cli samples/ --out invoices.xlsx

# API mode
uvicorn invoice_ai.api:app --reload   # docs at http://localhost:8000/docs
```

Sample output:

```
[OK   ] 01_freight_invoice.txt: Oceanic Freight Services 4372.5
[OK   ] 02_office_supplies.txt: Bright Office Supplies 6018.0
[CHECK] 03_mismatched_total.txt: QuickHaul Logistics 580.0 subtotal + tax = 528.0, total says 580.0; due date is before invoice date
```

## API

| Method | Path | Purpose |
|---|---|---|
| POST | `/documents` | Process raw text `{"text": "..."}` |
| POST | `/documents/upload` | Upload a `.txt` / `.eml` file |
| GET | `/documents?status=needs_review` | List processed documents |
| GET | `/export.xlsx` | Download the Excel report |

## Tests

```bash
pytest
```

## Roadmap (ideas to extend)

- [ ] PDF invoices (pdfplumber) and scanned images (OCR)
- [ ] Read directly from a mailbox (IMAP / Microsoft Graph)
- [ ] Accuracy report: LLM vs rule-based on a labelled sample set
- [ ] Duplicate invoice detection (same vendor + number)
- [ ] Small review UI to correct and approve flagged invoices
- [ ] PostgreSQL + Docker Compose
