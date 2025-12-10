Market Sentinel – Project Plan (PoC on Mac + Azure OpenAI)

This document describes the phased implementation plan for the Market Sentinel PoC. It is intended as a hand-off to a development team.

The goal is to build a small but realistic risk-analysis agent for Indian markets, running locally on a MacBook, with Azure OpenAI as the AI backbone. Future GCP deployment can reuse the same architecture.

⸻

Guiding Principles
	•	Build in layers: data → semantic → API → interface.
	•	Each phase produces a runnable MVP, not just scaffolding.
	•	Use simple, inspectable logic for event detection (no over-engineering).
	•	Treat the PoC as a mini-product, not a throwaway script.

⸻

Phases Overview

Phase	Name	Main Focus
0	Environment + skeleton	Repo structure, libraries, basic OHLC fetch
1	Data storage + event detection	DuckDB store, simple detection over past data
2	Semantic layer (text-only briefings)	Azure OpenAI wiring, risk briefing generator
3	News enrichment	News client integration, richer prompts
4	API / orchestration layer	FastAPI endpoints for scan + briefing
5	Multimodal charts + PDF	Candlestick images, vision, PDF risk briefing
6	CLI + minimal UI	Usable demo flow for non-engineers

Team assumption: 1–2 Python-centric engineers.

⸻

Phase 0 – Environment + Skeleton

Objective: Get the repo into a usable shape and verify we can fetch Indian market OHLC data locally.

Tasks:
	1.	Repo scaffold
	•	Create directory structure as described in README.
	•	Add requirements.txt, .env.example, basic __init__.py files.
	2.	Market data client (Yahoo Finance)
	•	Implement sentinel_data/data_sources/yahoo_client.py:
	•	get_ohlc(symbol: str, interval: str, start: str, end: str) -> pd.DataFrame
	•	Use yfinance.download or equivalent.
	•	Focus on:
	•	^NSEI (NIFTY 50), ^BSESN (Sensex), and 1–2 large-cap NSE stocks (for example RELIANCE.NS).
	3.	Backfill script
	•	Implement scripts/backfill_history.py:
	•	Hardcode a small list of symbols.
	•	Pull last 5–30 days of 5-minute OHLC bars.
	•	Print basic stats (number of rows, date ranges).

Deliverable / Exit Criteria:
	•	Developer can run:

python scripts/backfill_history.py

and see confirmation that OHLC data is fetched.

⸻

Phase 1 – Data Storage + Event Detection

Objective: Persist OHLC data locally and detect simple “risk events” over historical data.

Tasks:
	1.	Local store (DuckDB)
	•	Implement sentinel_data/store/duckdb_store.py:
	•	write_ohlc(symbol: str, df: pd.DataFrame) -> None
	•	read_ohlc(symbol: str, start: datetime, end: datetime, interval: str) -> pd.DataFrame
	•	Use a single DuckDB file in data/market_sentinel.duckdb.
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
	•	drop_pct: float
	•	event_type: Literal["index_drop", "stock_drop"]
	3.	Event detection
	•	Implement sentinel_data/event_detector.py:
	•	detect_events(df: pd.DataFrame, window_minutes: int, drop_threshold_pct: float) -> list[RiskEvent]
	•	Algorithm:
	•	Use a sliding window over OHLC close prices.
	•	For each window:
	•	Compute percentage change from first close to last close.
	•	If change <= -drop_threshold_pct, create a RiskEvent.
	4.	Batch detection script
	•	Implement scripts/detect_events_batch.py:
	•	For each tracked symbol:
	•	Read recent OHLC (for example last trading day).
	•	Run detect_events with window_minutes=10, drop_threshold_pct=1.0.
	•	Print list of events as a table.

Deliverable / Exit Criteria:
	•	Running python scripts/detect_events_batch.py prints simulated “risk events” for NIFTY and selected stocks.

⸻

Phase 2 – Semantic Layer (Text-Only Risk Briefings)

Objective: Connect to Azure OpenAI and generate structured text-only risk briefings for a given event.

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
	•	Design build_briefing_prompt(event: RiskEvent, ohlc_slice: pd.DataFrame, news_items: list[NewsItem] | None) -> list[dict]:
	•	Encode:
	•	Symbol, window, drop percentage.
	•	High/low in that window vs day high/low.
	•	Volume vs prior average (basic metrics).
	3.	Response schema
	•	Define sentinel_semantic/prompts/briefing_schema.json or Pydantic model RiskBriefing:
	•	classification: Literal["news_driven", "technical", "uncertain"]
	•	summary_text: str
	•	technical_context: str
	•	drivers: list[str]
	•	confidence: float
	•	caveats: list[str]
	4.	Briefing generator
	•	Implement sentinel_semantic/briefing_generator.py:
	•	generate_briefing(event: RiskEvent, ohlc_slice: pd.DataFrame, news_items: list[NewsItem] | None) -> RiskBriefing
	•	Handles:
	•	Prompt construction.
	•	Azure call.
	•	JSON parsing and validation / fallback.
	5.	Demo script
	•	Implement scripts/demo_run_briefings.py:
	•	Calls detect_events_batch logic.
	•	For the most recent event:
	•	Fetches OHLC slice around the event.
	•	Calls generate_briefing.
	•	Prints briefing fields in markdown.

Deliverable / Exit Criteria:
	•	Single command:

python scripts/demo_run_briefings.py

detects at least one event and prints a structured AI-generated briefing to the console.

⸻

Phase 3 – News Enrichment

Objective: Incorporate recent news headlines into the prompt to improve classification and interpretability.

Tasks:
	1.	News client
	•	Implement sentinel_data/data_sources/news_client.py:
	•	get_recent_news(symbol: str, lookback_minutes: int) -> list[NewsItem]
	•	Model NewsItem in sentinel_data/models/news.py:
	•	source: str
	•	headline: str
	•	published_at: datetime
	•	url: str
	•	summary: str | None
	2.	Integration
	•	Modify demo_run_briefings.py to:
	•	Fetch news_items for the event window.
	•	Pass them into generate_briefing.
	3.	Prompt refinement
	•	Update base prompt to instruct the model:
	•	Consider both price action and news.
	•	Reference specific headlines in the explanation when relevant.
	•	Implement fallbacks:
	•	If news_items is empty, steer the model away from unjustified “news-driven” classifications.

Deliverable / Exit Criteria:
	•	Console demo shows:
	•	Event summary.
	•	Top news headlines.
	•	Risk briefing that references these headlines where appropriate.

⸻

Phase 4 – API / Orchestration Layer

Objective: Wrap the pipeline in a REST API to enable UI, integrations, and automated demos.

Tasks:
	1.	FastAPI app
	•	Implement sentinel_api/main.py with routes:
	•	POST /events/scan
	•	Request: { "symbols": [...], "start": "...", "end": "...", "window_minutes": 10, "drop_threshold_pct": 1.0 }
	•	Response: list of RiskEvent.
	•	POST /events/{event_id}/briefing
	•	Request: optional parameters:
	•	regenerate: bool
	•	Behaviour:
	•	Load event (or accept full event in body).
	•	Fetch OHLC slice and news.
	•	Generate or retrieve briefing.
	•	Response: RiskBriefing.
	•	GET /briefings/recent
	•	Response: list of recent events with classification and timestamps.
	2.	Services
	•	sentinel_api/services/event_service.py:
	•	Encapsulates reading OHLC from DuckDB and calling detect_events.
	•	sentinel_api/services/forensic_service.py:
	•	Encapsulates:
	•	Reading OHLC slice.
	•	Fetching news.
	•	Calling generate_briefing.
	•	Persisting briefing JSON to briefings/.
	3.	Persistence
	•	Optional: store RiskEvent and RiskBriefing metadata in DuckDB or a simple JSON index for easy lookup by event_id.

Deliverable / Exit Criteria:
	•	API running via:

uvicorn sentinel_api.main:app --reload


	•	You can:
	•	POST /events/scan to get events.
	•	POST /events/{id}/briefing to get JSON briefings via HTTP.

⸻

Phase 5 – Multimodal Charts + PDF Briefings

Objective: Generate candlestick charts, feed images into the model (if supported), and produce a PDF “Risk Briefing.”

Tasks:
	1.	Chart generation
	•	Implement sentinel_data/charts/generator.py:
	•	generate_candlestick_chart(symbol: str, df: pd.DataFrame, event: RiskEvent) -> Path
	•	Use mplfinance:
	•	Plot a 30–60 minute window around the event.
	•	Save PNG to charts/ with a stable naming convention.
	2.	Multimodal model usage
	•	Update azure_client and briefing_generator to support image input (if available for the chosen Azure model).
	•	Prompt:
	•	Instruct the model to use both the chart and the numerical and news context to explain the event.
	3.	PDF service
	•	Implement sentinel_api/services/pdf_service.py:
	•	Build an HTML template (Jinja) including:
	•	Event metadata.
	•	AI summary and drivers.
	•	Top news headlines with URLs.
	•	Embedded chart image.
	•	Render HTML to PDF with weasyprint (or reportlab if needed).
	•	Save PDF to briefings/ as <event_id>_briefing.pdf.
	4.	API extension
	•	Extend /events/{event_id}/briefing to accept format:
	•	json (default) or pdf.
	•	For pdf:
	•	Return metadata and file path (or base64) for demo.

Deliverable / Exit Criteria:
	•	For at least one event:
	•	API call generates a candlestick chart, multimodal AI reasoning, and a PDF file with chart and text.

⸻

Phase 6 – CLI + Minimal UI

Objective: Make the PoC demoable to non-engineers with a simple, predictable flow.

Tasks:
	1.	CLI interface
	•	Implement sentinel_interface/cli.py using typer or click:
	•	market-sentinel scan --days 5 --symbol ^NSEI
	•	Calls API /events/scan, prints events in a table.
	•	market-sentinel brief --event-id <id> --pdf
	•	Calls /events/{event_id}/briefing?format=pdf.
	•	Prints a short summary.
	•	Shows where the PDF was written.
	2.	Minimal web UI (optional)
	•	Use FastAPI with Jinja templates:
	•	/ – list recent events (symbol, time, drop, classification).
	•	/event/{event_id} – show structured briefing, link to PDF.
	•	Keep it intentionally simple.
	3.	Demo script
	•	Add docs/demo-script.md summarising:
	•	Commands to run.
	•	Expected outputs.
	•	Suggested narrative when demoing to stakeholders.

Deliverable / Exit Criteria:
	•	A non-developer can:
	•	Run a CLI command to scan and brief events.
	•	Open a PDF risk briefing produced by the system.

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

