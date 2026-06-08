# Developer Guide — ThrustVault Scraper

**Document Version**: 1.1.0  
**Target Audience**: Lead Engineer, Backend Developers, Quality Assurance Engineers

---

## 1. Directory Structure Deep-Dive

```
motor_scraper/
│
├── api.py                    # Flask server, SSE routing, and job runner
├── config.py                 # Central configurations & env variable loader
├── run.py                    # CLI script for offline extraction
├── requirements.txt          # Python dependency list
│
├── scrapers/                 # Web Scraping Modules
│   ├── base_scraper.py       # Scraper interface & HTTP connection fallbacks
│   ├── tmotor_scraper.py     # T-Motor crawler (pre-filtering, table parser)
│   ├── getfpv_scraper.py     # GetFPV crawler (Playwright fallback, Magento)
│   ├── emax_scraper.py       # EMAX crawler (Shopify)
│   ├── speedybee_scraper.py  # Speedybee crawler (Shopify)
│   └── rcbenchmark_scraper.py# RCBenchmark crawler (CSV test log downloads)
│
├── parsers/                  # Spec extraction & normalization
│   ├── motor_parser.py       # String cleanup & unit extraction
│   └── groq_parser.py        # Groq Llama3 spec extraction pipeline
│
├── storage/                  # Persistence handlers
│   └── csv_storage.py        # CSV/JSON writers with UTF-8 BOM
│
├── utils/                    # Utility components
│   ├── logger.py             # Custom colored logger
│   ├── rate_limiter.py       # Per-domain random jitter throttle
│   └── dedup.py              # Record deduplication engine
│
└── output/                   # Auto-generated CSV and JSON reports
```

---

## 2. Setting Up the Development Environment

### 2.1 Dependencies Installation
Ensure Python 3.10+ is installed on your system. Run:
```bash
# Clone the repository and navigate to the project
cd motor_scraper

# Create a virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: .\venv\Scripts\activate

# Install requirements
pip install -r requirements.txt

# Install Playwright browser binaries
playwright install chromium
```

### 2.2 Environment Configuration
Create a `.env` file in the root of the `motor_scraper/` folder:
```ini
# Groq API key for spec extraction & batch summaries
GROQ_API_KEY=gsk_your_real_api_key_here

# Groq Model
GROQ_MODEL=llama-3.3-70b-versatile

# Rate Limit Overrides (use 0.0 for development/tests)
REQUEST_DELAY_MIN=0.1
REQUEST_DELAY_MAX=0.3

# Playwright Browser Timeouts (in milliseconds)
PLAYWRIGHT_GOTO_TIMEOUT=12000
PLAYWRIGHT_SELECTOR_TIMEOUT=8000
```

---

## 3. How to Write a New Scraper

To add a new crawler source (e.g. `hobbyking`), follow this recipe:

### 3.1 Create the Scraper Module
Create `scrapers/hobbyking_scraper.py`. Inherit from `BaseScraper` and implement the `scrape` method:

```python
from scrapers.base_scraper import BaseScraper
from utils.logger import get_logger

log = get_logger(__name__)

class HobbyKingScraper(BaseScraper):
    name = "hobbyking"
    base_url = "https://hobbyking.com"

    def scrape(self, query: str = "") -> list[dict]:
        results = []
        # 1. Fetch content
        url = f"{self.base_url}/search?q={query}"
        html = self.fetch(url) # Automatically uses fallbacks, headers & rate limiters
        
        if not html:
            # Fallback to Playwright if plain HTTP gets blocked
            html = self.fetch_with_browser(url, wait_selector=".product-card")
            
        if not html:
            return results

        # 2. Parse content
        soup = self.parse(html)
        products = soup.select(".product-card")
        
        for item in products:
            # Extract and parse specs into a standard dict
            # ...
            results.append({
                "motor_name": "...",
                "company": "HobbyKing",
                "max_thrust": "...",
                "recommended_esc": "...",
                "recommended_propeller": "...",
                "link_motor": "...",
                "source": "hobbyking",
            })
            
        return results
```

### 3.2 Register the Scraper
Add your new scraper to `config.py` in `SOURCES` and `_get_scraper` in `api.py` / `run.py`:
```python
# In config.py:
SOURCES = {
    # ...
    "hobbyking": "HobbyKing listings",
}

# In api.py:
def _get_scraper(name: str):
    # ...
    elif name == "hobbyking":
        from scrapers.hobbyking_scraper import HobbyKingScraper
        return HobbyKingScraper()
```

---

## 4. Verification & Testing Workflows

### 4.1 CLI Scrape Jobs
Verify scraper output directly from the terminal:
```bash
# Dry run a specific scraper with query filtering
python run.py --source tmotor --query "MN3508" --dry-run
```

### 4.2 API Integration Testing
Use the programmatic test suite to verify the REST backend:
```bash
# Start Flask locally
python api.py

# Run the test job script (in a separate terminal)
python C:\Users\bhara\.gemini\antigravity-ide\brain\44303753-c3e5-445a-9617-21fb9f378a2c\scratch\debug_scrape_job.py
```

---

## 5. Performance Tuning & Best Practices

1. **Keep Rate Limits Low in Dev**: Set `REQUEST_DELAY_MIN=0.0` and `REQUEST_DELAY_MAX=0.0` inside your dev `.env` file to accelerate iteration.
2. **Pre-Filter Aggressively**: Always verify titles/links against query terms before dispatching detail page crawls.
3. **Use Thread Pools**: When writing loops that hit multiple URLs or API endpoints, use `concurrent.futures.ThreadPoolExecutor` to execute them in parallel.
4. **Clean up DOM Elements**: When parsing complex tables, propagate rowspans and colspans to ensure specifications arrays remain dense and align correctly with performance records.
