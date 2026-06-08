# Features Specification — ThrustVault Scraper

**Document Version**: 1.1.0  
**Target Audience**: Principal Software Architect, Senior Product Manager, Lead Engineer

---

## 1. Multi-Source Search & Pre-Filtering Engine

The core value proposition of ThrustVault is real-time propulsion data aggregation. To prevent massive network overhead, the search engine utilizes target-filtered querying.

### 1.1 Query Tokenization & Smart Matching
Queries are split into alphanumeric tokens to handle spacing variations (e.g. `MN3508 KV380` vs `MN-3508-380KV` vs `MN 3508 380 KV`).
* **Normalization Logic**:
  * Strips punctuation, replaces hyphens with spaces, and converts to lowercase.
  * Extracts numerical stator sizes (e.g. `3508`) and KV values (e.g. `380`).
  * Rejects generic tokens (like `kv`, `motor`, `t-motor`) to prevent false positives.
* **Match Criteria**: Returns `True` if all core search terms (model prefix and numerical identifiers) exist in the candidate text.

### 1.2 Upstream Candidate Pre-Filtering
Before executing detail page fetches (which are expensive), scrapers match the list-view titles and links against the tokenized query:
* **T-Motor Store**: Search returns an average of 12 candidate links (e.g. propellers, motor mounts, screws). Pre-filtering isolates the motor series (e.g. `MN3508`) and skips the remaining 11 detail fetches, saving over 90% of requests.
* **RCBenchmark Database**: Pre-filters the list of 100+ CSV logs by comparing the query keywords directly to the CSV URLs. Only matched files are downloaded.

---

## 2. Advanced HTML Table Rowspan Parser

Propulsion specifications and performance tables on manufacturer websites (especially T-Motor) heavily utilize complex HTML tables with `rowspan` and `colspan` attributes (e.g., repeating the stator size cell for multiple KV configurations, or repeating propeller types across different voltages).

Standard HTML parsing libraries parse row-by-row, leading to missing cells in subsequent rows where a prior row has a `rowspan`. ThrustVault implements a custom 2D grid matrix parser in `TMotorScraper` to solve this:

```python
def _parse_html_table(self, table) -> list[list[str]]:
    rows = table.find_all("tr")
    if not rows:
        return []

    grid = {}
    for r_idx, row in enumerate(rows):
        c_idx = 0
        for cell in row.find_all(["td", "th"]):
            # Advance column pointer if index is already claimed by a rowspan from above
            while (r_idx, c_idx) in grid:
                c_idx += 1

            rowspan = int(cell.get("rowspan", 1))
            colspan = int(cell.get("colspan", 1))
            val = cell.get_text(separator=" ", strip=True)

            # Propagate the cell value across its entire rowspan and colspan dimensions
            for r_offset in range(rowspan):
                for c_offset in range(colspan):
                    grid[(r_idx + r_offset, c_idx + c_offset)] = val
            c_idx += colspan

    num_rows = len(rows)
    num_cols = max(c for r, c in grid.keys()) + 1 if grid else 0

    table_data = []
    for r in range(num_rows):
        row_data = [grid.get((r, c), "") for c in range(num_cols)]
        table_data.append(row_data)

    return table_data
```

---

## 3. Real-Time Performance Test Data Processing

The scraper parses empirical bench test tables containing throttle-step curves.

### 3.1 Data Capture Metrics
For each throttle step (typically 50%, 65%, 75%, 85%, 100%), the system extracts:
* **Throttle (%)**: Input signal level.
* **Voltage (V)**: Supply voltage (e.g. 14.8V, 22.2V).
* **Current (A)**: Steady-state current draw.
* **Power (W)**: Electrical power consumed ($P = V \times I$).
* **Thrust (g)**: Mechanical force generated.
* **RPM**: Motor rotation speed.
* **Efficiency (g/W)**: Grams of thrust per electrical watt.
* **Temperature (°C)**: Thermal reading at step duration limit.

### 3.2 Dynamic UI Grouping
In the frontend, test data points are grouped dynamically using JS:
* **Group Level 1: Motor model / KV**
* **Group Level 2: Propeller configuration**
* Grouped tables are sorted in ascending order of throttle percentage. This structures the data in a clean, comparable layout.

---

## 4. Groq LLM Spec Normalization & AI Enrichment

When raw web scrapers fail to find specific keys (like "Max Thrust" or "Recommended ESC") due to layout variations, the system uses the **Groq AI Enrichment pipeline**.

### 4.1 Prompt Engineering Strategy
The raw product page text (truncated to the first 4000 characters to conserve context windows) is sent to Groq with a system prompt detailing the target schema.
* **Strict Output Constraints**: The model is instructed to output *only* raw JSON matching the schema. No markdown wrappers, conversational text, or metadata are permitted.
* **Validation**: The backend uses regular expressions to find the JSON boundary `{ ... }` and parses it.

### 4.2 Multi-Threaded AI Dispatch
To bypass the latency of sequential LLM completions, the enrichment engine parallelizes the API requests. If a search query yields multiple motors that lack manufacturer data, the engine submits up to 10 parallel completions concurrently.

---

## 5. Deduplication & Data Integrity Heuristics

Data collected across different catalogs can contain duplicate entries or formatting discrepancies (e.g., "T-MOTOR MN3508" vs "MN3508" vs "MN 3508").
* **Deduplication Engine (`utils/dedup.py`)**:
  * Extracts manufacturer names (T-Motor, Emax, etc.).
  * Sanitizes the motor name by stripping branding prefixes, spaces, and casing.
  * Keeps the record with the most comprehensive spec list (e.g., prioritizing detail-scraped specs over catalog-scraped summary specs).
  * Merges proof links (retaining unique URLs for the motor, ESC, and propeller).
