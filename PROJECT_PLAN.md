Market Sentinel – Project Plan (PoC on Mac + Azure OpenAI)

This document describes the phased implementation plan for the Market Sentinel PoC. It is intended as a hand-off to a development team.

The goal is to build a small but realistic risk-analysis agent for Indian markets, running locally on a MacBook, with Azure OpenAI as the AI backbone. Future GCP deployment can reuse the same architecture.

⸻

Guiding Principles
	•	Build in layers: data → semantic → API → interface.
	•	Each phase produces a runnable MVP, not just scaffolding.
	•	Use simple, inspectable logic for event detection (no over-engineering).
	•	Treat the PoC as a mini-product, not a throwaway script.
	•	Optimize for “why did this happen?” with evidence (headlines + stats).
	•	Enable reproducible, demo-friendly flows (curated “demo pack”).

⸻

Phases Overview

Phase	Name	Main Focus
0	Environment + skeleton	Repo structure, libraries, basic OHLC fetch (daily + intraday)
1	Data storage + event detection	DuckDB store, multi-detector events + catalog
2	Semantic layer (text-only briefings)	Azure OpenAI with evidence blocks + deterministic/demo mode
3	News enrichment (historical-friendly)	Historical-capable client, curated cache for demos
4	API / orchestration layer	FastAPI endpoints for scan + briefing + event catalog
5	Multimodal charts + PDF	Candlestick images with annotations, vision/PDF briefing
6	CLI + minimal UI	Usable demo flow + “demo pack” replay

Team assumption: 1–2 Python-centric engineers.

⸻

Phase 0 – Environment + Skeleton

Objective: Get the repo into a usable shape and verify we can fetch Indian market OHLC data locally (daily 5y + recent intraday).

Tasks:
	1.	Repo scaffold
	•	Create directory structure as described in README.
	•	Add requirements.txt, .env.example, basic __init__.py files.
	2.	Market data client (Yahoo Finance)
	•	Implement sentinel_data/data_sources/yahoo_client.py:
	•	get_ohlc(symbol: str, interval: str, start: str, end: str) -> pd.DataFrame
	•	Use yfinance.download or equivalent.
	•	Focus on:
	•	^NSEI (NIFTY 50), ^BSESN (Sensex), and 3–5 large-cap NSE stocks (for example RELIANCE.NS, TCS.NS, INFY.NS, HDFCBANK.NS).
	3.	Backfill script
	•	Implement scripts/backfill_history.py:
	•	Hardcode a small list of symbols.
	•	Pull last 5–30 days of 5-minute OHLC bars and ~5 years of daily bars.
	•	Print basic stats (number of rows, date ranges).

Deliverable / Exit Criteria:
	•	Developer can run:

python scripts/backfill_history.py

and see confirmation that OHLC data (daily + intraday) is fetched.

⸻

Phase 1 – Data Storage + Event Detection

Objective: Persist OHLC data locally and detect a range of “risk events” over historical data; build an event catalog for quick lookup and demos.

Tasks:
	1.	Local store (DuckDB)
	•	Implement sentinel_data/store/duckdb_store.py:
	•	write_ohlc(symbol: str, df: pd.DataFrame, interval: str) -> None
	•	read_ohlc(symbol: str, start: datetime, end: datetime, interval: str) -> pd.DataFrame
	•	Use a single DuckDB file in data/market_sentinel.duckdb; partition by symbol/interval.
	2.	Data models
	•	In sentinel_data/models/ohlc.py:
	•	Pydantic model OHLCBar (symbol, ts, open, high, low, close, volume).
	•	In sentinel_data/models/events.py:
	•	Pydantic model RiskEvent:
	•	event_id: str
	•	symbol: str
	•	start_ts: datetime
	•	end_ts: datetime
	•	window_minutes: int
	•	move_pct: float
	•	event_type: Literal["index_drop", "index_rise", "stock_drop", "stock_rise", "gap_up", "gap_down", "vol_spike"]
	•	stats: dict (e.g., vol_zscore, gap_size_pct, move_sigma)
	3.	Event detection
	•	Implement sentinel_data/event_detector.py:
	•	detect_events(df: pd.DataFrame, detectors: list[DetectorSpec]) -> list[RiskEvent]
	•	Support detectors: drop/rise, gap up/down, volatility spike, volume z-score spike.
	•	Compute and store stats needed for prompts (pct move, sigma vs rolling vol, gap size, volume z-score).
	4.	Event catalog
	•	Implement an index (DuckDB table or JSON) to persist RiskEvents with tags: symbol, interval, type, severity, start/end.
	•	Support listing/filtering by symbol, date range, type, severity.
	5.	Batch detection script
	•	Implement scripts/detect_events_batch.py:
	•	Run detectors over daily (multi-year) and intraday windows.
	•	Insert events into the catalog; print a table of notable events.

Deliverable / Exit Criteria:
	•	Running python scripts/detect_events_batch.py populates the event catalog and prints notable events for NIFTY, Sensex, and selected stocks.

⸻

Phase 2 – Semantic Layer (Text-Only Risk Briefings)

Objective: Connect to Azure OpenAI and generate structured, evidence-backed text-only risk briefings for a given event; support deterministic/demo mode.

Tasks:
	1.	Azure client
	•	Implement sentinel_semantic/azure_client.py:
	•	Uses environment variables:
	•	AZURE_OPENAI_ENDPOINT
	•	AZURE_OPENAI_API_KEY
	•	AZURE_OPENAI_DEPLOYMENT_NAME
	•	Provides generate_completion(messages: list[dict]) -> dict for a ChatCompletions-style API.
	2.	Prompt design
	•	Create sentinel_semantic/prompts/base_prompt.txt:
	•	System prompt: senior market risk analyst, concise, explicit classification.
	•	Design build_briefing_prompt(event: RiskEvent, ohlc_slice: pd.DataFrame, news_items: list[NewsItem] | None, stats: dict) -> list[dict]:
	•	Encode:
	•	Symbol, window, move percentage, gap size, vol/volume stats.
	•	High/low in that window vs day/week high/low.
	•	Volume vs rolling average.
	•	Include matched headlines with timestamps/urls when available.
	•	Support deterministic mode (fixed seeds + supplied news/metrics) for demo reproducibility.
	3.	Response schema
	•	Define sentinel_semantic/prompts/briefing_schema.json or Pydantic model RiskBriefing:
	•	classification: Literal["news_driven", "technical", "structural", "uncertain"]
	•	summary_text: str
	•	technical_context: str
	•	drivers: list[str]
	•	confidence: float
	•	caveats: list[str]
	•	evidence: list[HeadlineEvidence] (source, headline, published_at, url) plus numeric stats used
	4.	Briefing generator
	•	Implement sentinel_semantic/briefing_generator.py:
	•	generate_briefing(event: RiskEvent, ohlc_slice: pd.DataFrame, news_items: list[NewsItem] | None, stats: dict, deterministic: bool = False) -> RiskBriefing
	•	Handles:
	•	Prompt construction.
	•	Azure call.
	•	JSON parsing and validation / fallback.
	•	Deterministic/demo mode (seeded and/or cached responses for curated events).
	5.	Demo script
	•	Implement scripts/demo_run_briefings.py:
	•	Calls detect_events_batch logic or reads from event catalog.
	•	Supports selecting from a curated “demo pack” of historical events (e.g., COVID crash, conflict shocks, AI rallies).
	•	Fetches OHLC slice around the event.
	•	Fetches news (live or cached).
	•	Calls generate_briefing.
	•	Prints briefing fields in markdown with evidence.

Deliverable / Exit Criteria:
	•	Single command:

python scripts/demo_run_briefings.py

detects or selects an event, fetches slice + news, and prints a structured, evidence-backed AI-generated briefing to the console (works in deterministic demo mode).

⸻

Phase 3 – News Enrichment (Historical-Friendly)

Objective: Incorporate news headlines (live and historical) into prompts to improve classification, interpretability, and demo reliability over 5 years of history.

Tasks:
	1.	News client
	•	Implement sentinel_data/data_sources/news_client.py:
	•	get_news(symbol: str, start: datetime, end: datetime, mode: Literal["live", "historical", "cache"]) -> list[NewsItem]
	•	Model NewsItem in sentinel_data/models/news.py:
	•	source: str
	•	headline: str
	•	published_at: datetime
	•	url: str
	•	summary: str | None
	•	keywords: list[str] | None
	2.	Historical and cache strategy
	•	Support historical-capable sources (e.g., GDELT/NewsAPI with from/to).
	•	Add curated cached headlines (Parquet/JSON) for “demo pack” periods (COVID crash, conflicts, tariffs, AI rallies).
	•	Map tickers to keywords for better headline matching; allow fuzzy matching and deduping.
	3.	Integration
	•	Modify demo_run_briefings.py to:
	•	Fetch news_items in live/historical/cache modes.
	•	Pass them into generate_briefing along with stats.
	4.	Prompt refinement
	•	Update base prompt to instruct the model:
	•	Consider both price action and news.
	•	Reference specific headlines with timestamps/urls.
	•	Implement fallbacks:
	•	If news_items is empty, steer the model away from unjustified “news-driven” classifications and favor technical/uncertain with caveats.

Deliverable / Exit Criteria:
	•	Running python scripts/demo_run_briefings.py in live and cache modes produces briefings that cite headlines (when present) and degrade gracefully when none are available.

⸻

Phase 4 – API / Orchestration Layer

Objective: Wrap the pipeline in a REST API to enable UI, integrations, event catalog queries, and automated demos.

Tasks:
	1.	FastAPI app
	•	Implement sentinel_api/main.py with routes:
	•	POST /events/scan
	•	Request: { "symbols": [...], "start": "...", "end": "...", "window_minutes": 10, "drop_threshold_pct": 1.0, "detectors": [...] }
	•	Response: list of RiskEvent.
	•	GET /events/catalog
	•	Query params: symbol, start, end, type, severity, interval, limit; supports demo pack presets.
	•	POST /events/{event_id}/briefing
	•	Request: optional parameters:
	•	regenerate: bool
	•	mode: "live" | "historical" | "cache" (news mode)
	•	deterministic: bool
	•	Behaviour:
	•	Load event (or accept full event in body).
	•	Fetch OHLC slice and news (chosen mode).
	•	Generate or retrieve briefing.
	•	Response: RiskBriefing.
	•	GET /briefings/recent
	•	Response: list of recent events with classification and timestamps.
	2.	Services
	•	sentinel_api/services/event_service.py:
	•	Encapsulates reading OHLC from DuckDB and calling detect_events; manages event catalog queries.
	•	sentinel_api/services/forensic_service.py:
	•	Encapsulates:
	•	Reading OHLC slice.
	•	Fetching news (live/historical/cache).
	•	Calling generate_briefing.
	•	Persisting briefing JSON to briefings/.
	•	sentinel_api/services/pdf_service.py:
	•	HTML → PDF rendering with charts and evidence.
	3.	Persistence
	•	Store RiskEvent and RiskBriefing metadata in DuckDB or a JSON index for lookup by event_id and for demo pack navigation.

Deliverable / Exit Criteria:
	•	API running via:

uvicorn sentinel_api.main:app --reload


	•	You can:
	•	POST /events/scan to get events.
	•	GET /events/catalog to browse events.
	•	POST /events/{id}/briefing to get JSON briefings via HTTP (with selected news mode).

⸻

Phase 5 – Multimodal Charts + PDF Briefings

Objective: Generate candlestick charts, feed images into the model (if supported), and produce a PDF “Risk Briefing” with evidence and annotations.

Tasks:
	1.	Chart generation
	•	Implement sentinel_data/charts/generator.py:
	•	generate_candlestick_chart(symbol: str, df: pd.DataFrame, event: RiskEvent) -> Path
	•	Use mplfinance:
	•	Plot a 30–60 minute (intraday) or multi-day window around the event.
	•	Annotate start/end, gap bars, and notable highs/lows.
	•	Save PNG to charts/ with a stable naming convention.
	2.	Multimodal model usage
	•	Update azure_client and briefing_generator to support image input (if available for the chosen Azure model).
	•	Prompt:
	•	Instruct the model to use chart, numerical stats, and headlines to explain the event.
	3.	PDF service
	•	Implement sentinel_api/services/pdf_service.py:
	•	Build an HTML template (Jinja) including:
	•	Event metadata.
	•	AI summary and drivers.
	•	Top news headlines with URLs and timestamps.
	•	Embedded chart image with annotations.
	•	Evidence block (stats used).
	•	Render HTML to PDF with weasyprint (or reportlab if needed).
	•	Save PDF to briefings/ as <event_id>_briefing.pdf.
	4.	API extension
	•	Extend /events/{event_id}/briefing to accept format:
	•	json (default) or pdf.
	•	For pdf:
	•	Return metadata and file path (or base64) for demo.

Deliverable / Exit Criteria:
	•	For at least one event:
	•	API call generates a candlestick chart, (optionally) multimodal AI reasoning, and a PDF file with chart, evidence, and text.

⸻

Phase 6 – CLI + Minimal UI

Objective: Make the PoC demoable to non-engineers with a simple, predictable flow and curated demo packs.

Tasks:
	1.	CLI interface
	•	Implement sentinel_interface/cli.py using typer or click:
	•	market-sentinel scan --days 5 --symbol ^NSEI
	•	market-sentinel brief --event-id <id> --pdf
	•	market-sentinel demo --pack five-year-highlights --limit 5 --mode cache
	•	Calls API /events/scan or /events/catalog, prints events in a table, and can generate PDFs.
	2.	Minimal web UI (optional)
	•	Use FastAPI with Jinja templates:
	•	/ – list recent and curated events (symbol, time, move, classification).
	•	/event/{event_id} – show structured briefing, evidence, chart, link to PDF.
	•	Include filters for symbol, type, severity, and demo packs.
	3.	Demo script
	•	Add docs/demo-script.md summarising:
	•	Commands to run (live vs demo pack).
	•	Expected outputs and narrative beats.
	•	Suggested “highlight reel” events.

Deliverable / Exit Criteria:
	•	A non-developer can:
	•	Run a CLI command (or UI) to scan, browse cataloged events, and brief them (JSON/PDF).
	•	Replay a curated demo pack that answers “why” with charts and headline evidence.

⸻

Roles and Handover Notes

Skills required:
	•	Strong Python (data, APIs, packaging).
	•	Familiarity with FastAPI and environment management.
	•	Comfort with Azure OpenAI APIs.

Handover notes:
	•	Start implementing from Phase 0 upward. Do not skip phases.
	•	Keep clear separation between:
	•	Data layer (no Azure calls here).
	•	Semantic layer (no DuckDB knowledge here).
	•	API and interface layers (orchestration only).
	•	As features stabilise, add small unit tests around:
	•	Event detection (pure logic).
	•	Prompt building and response parsing (semantic layer).

