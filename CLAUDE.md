# Carrier Invoice Check
 
Demo tool. It reads carrier PDF invoices with an LLM, extracts structured data, and checks each invoice against the matching transport order. Discrepancies are flagged for a human to approve or reject.
 
Context: order entry at logistics companies is often already automated. This demo targets the next manual step: checking incoming carrier invoices against orders (accounts payable / freight audit).
 
## Core principle
 
**AI only reads. Python decides.**
 
- The LLM extracts data from the invoice into a strict schema. It never judges whether an invoice is correct.
- All checks live in `src/rules.py` as plain, deterministic, tested Python.
- Every flag must be explainable: a code, a human-readable message, and the amount involved.
- A human always confirms a rejection. The tool recommends, it does not pay or reject on its own.
## Ownership — read this first
 
I'm building this to learn. Split of work:
 
- **I write:** `src/models.py`, `src/extract.py`, `src/rules.py`. Do not write or rewrite these files. When asked, review them: point out bugs, edge cases and improvements, and explain why. Short snippets to illustrate a point are fine.
- **You write:** data generator, tests, eval script, Streamlit UI, config, README, and other boilerplate.
For any task bigger than one file, show a plan first and wait for my OK.
 
## Stack
 
- Python 3.12, managed with `uv`
- `pdfplumber` for PDF → text (layout-preserving, so table columns stay readable)
- `groq` SDK (default provider, free tier)
- `anthropic` SDK (optional second provider, used for comparison)
- Pydantic v2
- Streamlit (UI)
- pandas (CSV / Excel export)
- fpdf2 (generating synthetic invoices)
- pytest
## Extraction providers
 
Provider is chosen in `.env`, never hard-coded:
 
```
EXTRACTION_PROVIDER=groq          # groq | anthropic
GROQ_API_KEY=
GROQ_MODEL=openai/gpt-oss-120b
ANTHROPIC_API_KEY=                # optional
ANTHROPIC_MODEL=claude-haiku-5-5  # optional
```
 
`src/extract.py` exposes one public function:
 
```
extract(pdf_path: Path) -> ExtractionResult   # Invoice + provider, model, seconds, error
```
 
Internally it branches per provider:
 
- **groq:** PDF → text with pdfplumber → chat completion with a JSON schema in strict mode → validate with Pydantic. On validation failure, retry once.
- **anthropic:** send the PDF directly as a document block → `client.messages.parse(..., output_format=Invoice)`. Check `stop_reason` (e.g. `max_tokens`, refusal).
Both providers must return the same `Invoice` model, so nothing downstream knows or cares which one ran.
 
Free-tier handling: retry with exponential backoff on HTTP 429, and respect the cache in `eval.py`.
 
## Project structure
 
```
carrier-invoice-check/
├── CLAUDE.md
├── .env.example
├── data/
│   ├── orders.csv        # transport orders
│   ├── carriers.csv      # carrier master data
│   └── invoices/         # generated PDF invoices
├── ground_truth/         # correct JSON for each invoice (same file stem as the PDF)
├── src/
│   ├── models.py         # Pydantic models            (mine)
│   ├── extract.py        # LLM extraction             (mine)
│   ├── rules.py          # validation rules           (mine)
│   └── generate_data.py  # synthetic data generator
├── app.py                # Streamlit UI
├── eval.py               # extraction accuracy vs ground truth
└── tests/
    └── test_rules.py
```
 
## Data
 
All data is synthetic. Never use real company, carrier or customer data.
 
**orders.csv:** `order_id` (e.g. `Q-10423`), `carrier_id`, `origin`, `destination`, `pickup_date`, `delivery_date`, `agreed_price`, `currency`, `agreed_surcharges` (semicolon-separated types, may be empty)
 
**carriers.csv:** `carrier_id`, `name`, `country`, `vat_id`, `iban`
 
**Invoice model (`src/models.py`), for reference:**
- `Invoice`: `invoice_number`, `invoice_date`, `carrier_name`, `carrier_vat_id`, `iban`, `order_id` (nullable), `origin`, `destination`, `transport_date`, `currency`, `line_items: list[LineItem]`, `net_amount`, `vat_amount`, `total_amount`
- `LineItem`: `description`, `type` (enum: `freight`, `waiting_time`, `adr`, `toll`, `fuel_surcharge`, `other`), `amount`
- Fields missing on the invoice are `None`. The extraction prompt says: return null, never guess.
- Compare enum values case-insensitively.
## Synthetic data generator (`src/generate_data.py`)
 
- ~10 fictional carriers from HR, SI, RO, SK, each with its own invoice layout and language (HR/EN/DE mix is fine).
- ~40 orders in `orders.csv`.
- One PDF invoice per order in `data/invoices/`, plus the correct JSON in `ground_truth/`.
- PDFs must have a real text layer (no images of text), so pdfplumber can read them.
- About 1/3 of invoices contain exactly one planted error, one per rule below. Record the planted error in the ground-truth JSON as `"planted_error": "<RULE_CODE>"` (or `null`).
- Deterministic: use a fixed random seed.
## Validation rules (`src/rules.py`)
 
Each rule returns zero or more `Finding(code, severity, message, amount_diff)`. Messages are in Croatian, e.g. `"Naplaćeno čekanje 85 € koje nije u nalogu Q-10423"`.
 
| Code | Check | Severity |
|---|---|---|
| `ORDER_NOT_FOUND` | order_id missing or not in orders.csv | reject |
| `PRICE_HIGHER` | freight > agreed price + tolerance (2%) | review; reject above 10% |
| `UNAGREED_SURCHARGE` | surcharge line item not in the order's agreed_surcharges | review |
| `DUPLICATE` | same invoice_number + carrier already processed | reject |
| `IBAN_CHANGED` | IBAN differs from carrier master data (fraud signal) | reject |
| `SUM_MISMATCH` | line items + VAT ≠ total (±0.01) | review |
| `DATE_OUT_OF_RANGE` | transport date outside the order's pickup/delivery window | review |
| `CURRENCY_MISMATCH` | currency ≠ order currency | review |
 
Overall status: any reject → **Odbij**, else any review → **Provjeri**, else **Odobri**.
 
Tolerances and thresholds live in one config dict at the top of `rules.py`, not scattered as magic numbers.
 
## Streamlit UI (`app.py`)
 
- Upload multiple PDFs, or a "Load demo set" button that processes `data/invoices/`.
- Results table: carrier, invoice number, order, total, status (colored), number of findings.
- Selecting a row opens a detail view: PDF preview on the left, extracted fields on the right, findings below.
- Buttons: Approve / Reject / Edit field. Decisions are kept in session state.
- Top metrics: invoices processed, flagged, total € potentially overcharged, average seconds per invoice.
- Small label showing which provider and model did the extraction.
- Export results to Excel.
- UI text in Croatian.
## Eval (`eval.py`)
 
- Runs extraction on every PDF in `data/invoices/` and compares each field with `ground_truth/`.
- Prints per-field accuracy as a table.
- Also reports rule detection: planted errors caught / missed, and false positives.
- Caches extraction results to `.cache/<provider>/` so reruns don't call the API again (flag `--fresh` to bypass).
- `--provider groq|anthropic` overrides `.env`. `--compare` runs both (when both keys exist) and prints a side-by-side table: accuracy, avg seconds per invoice, failures.
## Commands
 
```
uv run python src/generate_data.py
uv run streamlit run app.py
uv run python eval.py [--provider groq|anthropic] [--compare] [--fresh]
uv run pytest
```
 
## Conventions
 
- Type hints everywhere; small functions; no classes where a function will do.
- Code, identifiers and comments in English. User-facing UI text and finding messages in Croatian.
- Secrets only from `.env`. Never commit `.env` or print API keys.
- Extraction errors never crash the app: return a result with an error message and show it in the UI.
- Each rule gets at least two tests: one that passes and one that triggers the finding.
- Keep dependencies minimal. Ask before adding a new package.
## Later (not v1)
 
- Scanned invoices: render page to image + OCR (Tesseract) or a vision model, then the same extraction path.
- Batch processing for overnight runs.