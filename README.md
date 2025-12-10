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


