# Market Sentinel (PoC)

Market Sentinel is a proof-of-concept “event-driven risk analyst” for Indian markets.

When a significant price move occurs in an index or stock (for example, NIFTY 50 drops 1% in 10 minutes), the system:

1. Detects the move from historical or near-real-time market data.
2. Extracts an OHLC window around the event.
3. Optionally generates a candlestick chart image.
4. Pulls recent news related to the instrument or index.
5. Calls Azure OpenAI (GPT-4 class models) to produce a structured **Risk Briefing**.
6. Exposes the briefing via:
   - A Python CLI, and
   - A FastAPI backend (for later UI / integrations).

This PoC is designed to run entirely on a MacBook. Cloud deployment (GCP / Azure) can be added later using the same layered architecture.

---

## High-Level Architecture

We use a layered pattern so the PoC can evolve cleanly.

**1. Interface layer**

- CLI commands to:
  - Scan for events in recent history.
  - Generate and view risk briefings for selected events.
- Optional minimal web UI on top of the FastAPI backend.

**2. API / orchestration layer**

- FastAPI service that exposes:
  - Event detection endpoints.
  - Forensic / briefing generation endpoints.
- Orchestrates:
  - Data access,
  - News enrichment,
  - Azure OpenAI calls,
  - Chart / PDF generation (later phase).

**3. Semantic / AI layer**

- Azure OpenAI adapter:
  - Text-only briefing generation (Phase 1).
  - Multimodal briefing generation with chart images (Phase 2).
- Prompt templates and response schemas:
  - Classification (news-driven / technical / uncertain).
  - Summary, drivers, technical context, caveats, confidence scores.

**4. Data layer**

- Market data ingestion and storage:
  - Free / batch-friendly APIs for Indian markets (for example `yfinance` for NIFTY, Sensex, large-cap NSE stocks).
- Event detection:
  - Simple rule-based anomaly detection (for example “drop > 1% in 10 minutes”).
- News retrieval:
  - Free news API for recent headlines.
  - Stubbed data is acceptable in early phases.

---

## Repository Structure (Proposed)

This is the intended structure as the implementation progresses.

```text
market-sentinel/
  README.md
  PROJECT_PLAN.md
  requirements.txt
  .env.example

  sentinel_config/
    __init__.py
    settings.py               # symbols, thresholds, API keys, etc.

  sentinel_data/
    __init__.py
    data_sources/
      yahoo_client.py         # yfinance wrapper for OHLC
      news_client.py          # news API / stub client
    store/
      duckdb_store.py         # DuckDB / parquet interaction
    models/
      ohlc.py                 # Pydantic models for OHLC bars
      events.py               # RiskEvent model
      news.py                 # NewsItem model
    event_detector.py         # logic to detect abnormal moves
    charts/
      generator.py            # candlestick chart generation

  sentinel_semantic/
    __init__.py
    prompts/
      base_prompt.txt         # system / role prompt
      briefing_schema.json    # expected structured output schema
    azure_client.py           # Azure OpenAI wrapper
    briefing_generator.py     # builds prompts, parses responses

  sentinel_api/
    __init__.py
    main.py                   # FastAPI application entrypoint
    routes/
      events.py               # /events endpoints
      briefings.py            # /briefings endpoints
    services/
      event_service.py        # orchestrates detection
      forensic_service.py     # orchestrates full briefing pipeline
      pdf_service.py          # HTML → PDF rendering

  sentinel_interface/
    __init__.py
    cli.py                    # Typer/Click-based CLI
    web/
      templates/
        index.html
        event_detail.html
      static/

  scripts/
    backfill_history.py       # download + persist OHLC history
    detect_events_batch.py    # offline event detection
    demo_run_briefings.py     # simple scripted demo helper

  data/                       # DuckDB / parquet data (gitignored)
  charts/                     # generated chart PNGs (gitignored)
  briefings/                  # generated PDF / JSON briefings (gitignored)

---

## Phased Roadmap (summary)

- Phase 0: Environment + skeleton (repo structure, Yahoo Finance OHLC fetch).
- Phase 1: Local store + simple event detection (DuckDB + sliding window drops).
- Phase 2: Semantic layer (Azure OpenAI text briefings).
- Phase 3: News enrichment (headlines in prompts).
- Phase 4: API / orchestration (FastAPI endpoints).
- Phase 5: Charts + PDF briefings (candlesticks, multimodal if available).
- Phase 6: CLI + minimal UI (demoable flow).

See `PROJECT_PLAN.md` for detailed tasks and exit criteria—build in order, do not skip phases.

---

## Getting Started

Prereqs: Python 3.11+ on macOS, `pip`, optional `uvicorn` for API dev.

1) Install deps
```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

2) Environment
```bash
cp .env.example .env
# Required for Azure OpenAI
AZURE_OPENAI_ENDPOINT=...
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_DEPLOYMENT_NAME=...
# Optional for news client
NEWS_API_KEY=...
```

3) Config (planned)
- `sentinel_config/settings.py` will hold symbols, thresholds, data paths.
- Defaults: DuckDB at `data/market_sentinel.duckdb`; generated artifacts under `charts/` and `briefings/` (gitignored).

---

## Running the PoC (by phase)

- Phase 0: Backfill recent OHLC (Yahoo Finance)
  ```bash
  python scripts/backfill_history.py
  ```
  Fetches 5–30 days of 5m bars for a small set of symbols (NIFTY, Sensex, large-cap NSE stocks) and prints row counts / date ranges.

- Phase 1: Detect simple drop events
  ```bash
  python scripts/detect_events_batch.py
  ```
  Runs a 10-minute window with a 1% drop threshold by default and prints detected events.

- Phase 2: Generate risk briefings (Azure required)
  ```bash
  python scripts/demo_run_briefings.py
  ```
  Reuses detection logic, fetches an OHLC slice around the most recent event, calls Azure OpenAI, and prints a structured markdown briefing. News items are included when available.

---

## API Surface (planned FastAPI)

- `POST /events/scan`
  - Body: `{ "symbols": [...], "start": "...", "end": "...", "window_minutes": 10, "drop_threshold_pct": 1.0 }`
  - Returns: list of RiskEvent objects.
- `POST /events/{event_id}/briefing`
  - Body: optional `{ "regenerate": false }` or full event payload.
  - Returns: RiskBriefing JSON; later may support `format=pdf`.
- `GET /briefings/recent`
  - Returns recent events with classification/timestamps.

---

## CLI Sketch (planned)

- `market-sentinel scan --days 5 --symbol ^NSEI`
- `market-sentinel brief --event-id <id> --pdf`

---

## Data & Storage

- OHLC persisted to `data/market_sentinel.duckdb` (gitignored).
- Generated assets:
  - Charts in `charts/`
  - Briefings (JSON/PDF) in `briefings/`
- News client can stub or use a free API during early phases.

---

## Testing Notes

- Add small unit tests for:
  - Event detection logic (sliding window drop calculation).
  - Prompt construction and Azure response parsing (schema validation/fallbacks).

---

## Housekeeping

- Add `.env.example` and ensure `data/`, `charts/`, `briefings/` stay gitignored.
- Keep layers separated: data (no Azure), semantic (no DuckDB), API/interface (orchestration only).

