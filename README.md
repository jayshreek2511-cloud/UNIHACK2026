# CommerceAI — AI-Powered Product Intelligence for Industrial Commerce

Built for **UniHack 2026** (Unilog).

CommerceAI turns messy, incomplete distributor catalog data into clean, standardized, search-ready product records matching Unilog's exact 252-column delivery schema — with every field traceable back to its source.

Given a raw row like `"PDSH4816AF Dishwasher SS - Display Only"` with mostly-empty brand fields, the pipeline:
- Classifies the product into a full **Dept > Class > Fine** category taxonomy
- Resolves the **true manufacturer and brand**, separated from the reselling distributor
- Extracts category-specific structured attributes via LLM reasoning
- Generates **5 consistent description formats** (Invoice, Mobile, Short, Long, Retail) from one shared source of truth
- Flags anything it isn't confident about for **human review**, instead of guessing

## Why it's different

Most catalog enrichment tools either require fully manual data entry, or apply AI silently with no way to tell a verified fact from a guess. CommerceAI is built around explainability as a structural property, not a UI add-on:

- Every field carries an explicit confidence tag: `source-verified` (found directly in text), `llm-inferred` (reasoned from context), or `not-found` — never silently guessed.
- Manufacturer URLs are marked verified **only** after a real, live HTTP fetch confirms the page exists (HTTP 200) and mentions the actual SKU. Bot-blocked (403) pages are never guessed — they're set to `None` and flagged.
- Records below a confidence threshold are automatically routed to a **Human Review Queue** with the exact reason stated (e.g. "Unverified Manufacturer URL", "Missing critical field: Material").
- The final delivery spreadsheet includes a `REMARKS` column and color-coded row highlighting, so a reviewer can audit any record's trustworthiness without touching code.

Tested across 25+ real product categories (appliances, plumbing, electrical, tools, abrasives, and more) — classification and attribute extraction both use LLM reasoning to pick the right schema per product, so new categories don't need new code.

## Tech stack

| Layer | Technology |
|---|---|
| Backend | Python, FastAPI |
| Frontend | React + Vite + Tailwind CSS |
| LLM (primary) | Google Gemini API |
| LLM (fallback) | Groq API |
| URL verification | httpx (live HTTP fetch) |
| Data processing | pandas |
| Export | openpyxl (Excel, with conditional formatting) |
| Testing | pytest |

## Repository structure

```
.
├── api/                    # FastAPI backend — serves dashboard endpoints
│   └── main.py
├── pipeline/                # Core enrichment pipeline
│   ├── stages/               # s01_ingest ... s10_delivery_export
│   ├── llm_client.py         # Gemini + Groq client, with automatic fallback
│   ├── brand_resolver.py     # Manufacturer/brand vs. distributor resolution
│   ├── uom_normalize.py      # Unit-of-measure & fraction normalization
│   └── reference/            # Taxonomy, LOV, UOM standards
├── frontend/                 # React dashboard (Overview, Pipeline, Catalog, Review Queue, Settings)
├── data/
│   ├── input/                 # Sample & test catalogs
│   └── output/                 # Pipeline run artifacts (generated, gitignored)
├── scripts/                  # Diagnostics, audits, one-off utilities
├── tests/                    # pytest suite
├── run_full_pipeline.py      # Run the full S01→S10 pipeline end-to-end
├── run_phase1.py … run_phase5.py   # Run individual pipeline phases
└── requirements.txt
```

## Pipeline stages

| Stage | What it does |
|---|---|
| S01 — Ingestion | Parses raw CSV rows |
| S02 — Classification | Assigns category via Dept > Class > Fine taxonomy |
| S03 — Distributor Normalization | Fuzzy-merges distributor/manufacturer name variants |
| S04 — Attribute Extraction | Resolves brand/manufacturer and extracts structured attributes (Gemini, regex fallback) |
| S05 — Manufacturer Enrichment | Live HTTP verification of manufacturer source pages |
| S06 — Description Generation | Produces 5 consistent description formats |
| S07 — Confidence Scoring | Weighted completeness score per record |
| S08 — Provenance | Builds a full lineage graph — source type + verified URL per field |
| S09 — Review Queue Routing | Routes low-confidence/unverified records to human review |
| S10 — Delivery Export | Formats output into Unilog's 252-column schema (CSV + Excel) |

## Getting started

### Prerequisites
- Python 3.10+
- Node.js 18+
- A [Gemini API key](https://ai.google.dev/) and, optionally, a [Groq API key](https://console.groq.com/) for fallback resilience

### Backend

```bash
pip install -r requirements.txt

# API keys
export GEMINI_API_KEY="your-gemini-key"
export GROQ_API_KEY="your-groq-key"        # optional but recommended — automatic fallback on rate limits

uvicorn api.main:app --reload --port 8000
```

The API will be live at `http://localhost:8000`. Health check: `GET /health`.

### Frontend

```bash
cd frontend
npm install
npm run dev
```

The dashboard will be live at `http://localhost:5173` (proxies `/api` and `/health` to `localhost:8000` in dev).

### Running the pipeline directly (CLI)

```bash
python run_full_pipeline.py
```

Or run individual phases with `run_phase1.py` … `run_phase5.py`. Output lands in `data/output/`.

## Key API endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/health` | Liveness check |
| GET | `/api/stats` | Dashboard overview KPIs |
| GET | `/api/records` | Full enriched catalog |
| GET | `/api/records/{sku}` | Single record detail |
| GET | `/api/review-queue` | Records flagged for human review |
| GET | `/api/analytics/scale` | Cost/throughput/human-burden projections |
| POST | `/api/records/{sku}/approve` | Approve a flagged record |
| POST | `/api/pipeline/run` | Kick off a full pipeline run |
| GET | `/api/pipeline/status/{job_id}` | Poll pipeline job status |
| GET | `/api/pipeline/download/excel` | Download delivery export (Excel) |
| GET | `/api/pipeline/download/csv` | Download delivery export (CSV) |

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `GEMINI_API_KEY` | Yes | Primary LLM provider |
| `GROQ_API_KEY` | Recommended | Fallback LLM provider, used automatically on Gemini rate limits |
| `GEMINI_MODEL` | No | Defaults to `gemini-flash-lite-latest` |
| `GROQ_MODEL` | No | Defaults to `openai/gpt-oss-20b` |
| `GEMINI_MIN_INTERVAL` | No | Minimum seconds between Gemini calls (default `7.0`) |

## Known limitations

- Manufacturer URL verification rate is limited by manufacturer sites' bot-blocking (HTTP 403) — an external constraint, not a pipeline defect. Every unverified case is explicitly flagged, never silently accepted.
- Attribute extraction depth outside a few core fields is bounded by how much information exists in short source descriptions.

## Roadmap

- Deeper attribute extraction via manufacturer spec-sheet/PDF parsing
- Broader manufacturer-site retrieval strategies to raise URL verification rate
- Product-level deduplication stage
- Full webhook/notification integration for pipeline completion events

## Deployment

The frontend (static Vite build) deploys cleanly to Vercel. The backend needs a host that supports a persistent, long-running Python process (it spawns pipeline runs as background jobs and keeps state in memory/local files) — Render, Railway, or Fly.io are good fits. See deployment notes in the project write-up for a step-by-step split-deployment guide.

## License

Built for UniHack 2026. License TBD.
