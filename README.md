# 🚁 ThrustVault Scraper — Drone Motor Intelligence

Self-contained web scraper for drone motor, ESC, and propeller specifications and empirical bench test curves. Part of the **ThrustVault** intelligence ecosystem.

---

## 📖 Product & Engineering Documentation

We maintain detailed engineering specifications, architectural designs, and developer manuals in the `docs/` folder:

* **[CTO Specification: System Architecture](file:///d:/motor%20data/motor_scraper/docs/system_architecture.md)**: Details structural layers, multi-threaded concurrency models, rate-limiting, and connection fallback strategies.
* **[Product Specification: Features Specification](file:///d:/motor%20data/motor_scraper/docs/features_specification.md)**: Deep-dive into query tokenization, upstream pre-filtering algorithms, rowspan HTML table parser, and Groq AI spec extraction prompts.
* **[Developer Manual: Developer Guide](file:///d:/motor%20data/motor_scraper/docs/developer_guide.md)**: Step-by-step setup guides, folder structures, testing scripts, and guides on how to write new crawlers.

---

## ⚡ Core Features

* 🚀 **High-Speed Execution**: Fully concurrent source crawling, pre-filtering of search matches, parallel detail fetches, and parallel AI spec completion.
* 🛡️ **Anti-Bot Resiliency**: Sophisticated multi-tier network connection fallback: `requests` ➔ `curl_cffi` (TLS JA3 fingerprint impersonation) ➔ `cloudscraper` ➔ `Playwright-stealth`.
* 🤖 **Groq LLM Parser**: Parallelizes queries using `llama-3.3-70b-versatile` to translate unstructured manufacturer descriptions into strict JSON specifications.
* 📊 **Empirical Performance Data**: Extracts throttle curves (thrust, current, voltage, RPM, efficiency, temp) and displays them in structured, collapsible tables grouped by motor and propeller configuration.
* 💾 **Reference Exports**: Aggregates datasets into normalized CSV/JSON files with built-in Excel-compatible Byte Order Mark (BOM).

---

## 🛠️ Quick Start Setup

```powershell
# 1. Clone & Navigate
cd motor_scraper

# 2. Initialize Virtual Environment
python -m venv venv
venv\Scripts\activate

# 3. Install Dependencies & Headless Browser
pip install -r requirements.txt
playwright install chromium
```

### Configure Environment variables
Create a `.env` file inside `motor_scraper/`:
```ini
GROQ_API_KEY=your_groq_api_key_here
GROQ_MODEL=llama-3.3-70b-versatile
```

---

## 🏃 Usage

### Start the Interactive Web UI
```powershell
python api.py
# Open http://localhost:5050 in your browser
```

### Run via Command Line Interface (CLI)
```powershell
# Run all scrapers & save CSV/JSON reports
python run.py --all

# Run specific scrapers for a query
python run.py --source tmotor rcbenchmark --query "MN3508"

# Run without AI enrichment (faster)
python run.py --all --no-groq
```
