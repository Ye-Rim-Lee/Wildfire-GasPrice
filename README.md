# ⛽️ Scaling Daily Web Scrapes of U.S. Gas Stations for Wildfire Impact Research

## Objective
This project builds a daily data ingestion pipeline and custom web scraper for [GasBuddy](https://www.gasbuddy.com/), designed to monitor daily retail gasoline prices across 88,000+ U.S. stations. The scraper automatically retrieves, parses, and stores structured price data, enabling causal inference analyses of wildfire-driven market disruptions and uncovering short-term price responses and spatial heterogeneity across U.S. regions.

## Workflow Overview
I use a modular, fault-tolerant scraping routine scheduled to run once per day:

- ****Step 1**. Data Ingestion Layer**
  - Iterates through a curated ZIP code list (`ZIP_Code_unique_all.xlsx`)
  - Dynamically constructs GasBuddy search URLs to retrieve nearby station data
  <br>

    <div>
      <figure>
        <img width="600" alt="image" src="https://github.com/user-attachments/assets/a3350951-2ee1-4d30-9695-aa718c3e1880" />
        <br>
        <figcaption><em>Figure. Screenshot of GasBuddy search by zipcode website</em></figcaption>
      </figure>
    </div>
  <br>
- ****Step 2**. Web Scraping and Parsing**
  - Uses `requests` for HTTP GET requests with randomized headers and retry logic
  - Parses DOM elements via `BeautifulSoup` to extract station IDs, displayed prices, and timestamps
- ****Step 3**. Data Structuring and Storage**
  - Converts raw HTML to structured tabular records
  - Persists daily outputs in versioned folders (e.g., `scraping_YYYYMMDD/`)
  - Logs failures to a separate `skipped/` directory for traceability
- ****Step 4**. Job Scheduling and Automation**
  - Orchestrated with `schedule` to execute once per day at a fixed time
  - Generates detailed runtime logs for auditing and performance monitoring

## Key Challenges & Mitigations

| Challenge                      | Description                                          | Resolution                                                                                      |
| ------------------------------ | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Dynamic HTML Markup**        | Front-end element names are obfuscated and unstable. | Used regex-based DOM selection and minimal stable attributes (IDs, class patterns).             |
| **Network Latency & Blocking** | Occasional HTTP errors or throttling.                | Implemented robust retry logic with exponential backoff and random sleep intervals.             |
| **Scalability**                | ~88k stations → long runtimes    | Partitioned input ZIPs by script number to enable parallel execution.                           |
| **Temporal Consistency**       | Price visibility depends on user reports; station sets evolve.  | Always record `collection_time` and the page’s human-readable `posted_time` (e.g. 3 hours ago). Use station IDs as stable keys and keep daily snapshots rather than overwriting. |


## Data Artifacts
**Each daily run produces:**
- `scraping_YYYYMMDD/gas_price_zip_<script_no>_YYYYMMDD.csv`
- `scraping_YYYYMMDD/output_<script_no>_YYYYMMDD.txt` (console+file log)
- `scraping_YYYYMMDD/skipped/gas_price_zip_skip_<script_no>.csv` (if any)

**Schema (CSV):**
- `zipcode` (int): ZIP queried
- `station_id` (str): GasBuddy DOM station identifier
- `price` (str): Displayed price text (e.g., `$3.59`) or `N/A`
- `posted_time` (str): “posted x mins ago” or `N/A`
- `collection_time` (str): UTC timestamp when captured

## Minimal Example
This example demonstrates the core function used to collect station-level retail gasoline prices from a public data source.
The scraper sends controlled HTTP requests, handles transient errors, and parses structured data from HTML pages. <br>
⭐ For reproducibility and ethical compliance, only the simplified logic is shared here — **please contact the author for access to the full research-grade code.**

```python
import re, time, random
import pandas as pd, requests
from datetime import datetime
from bs4 import BeautifulSoup

HEADERS = {"User-Agent": "Mozilla/5.0"}

def process_url(zipcode: int):
    """Fetch and parse GasBuddy station data for a given ZIP code."""
    url = f"https://www.gasbuddy.com/home?search={zipcode:05d}&fuel=1&method=all&maxAge=0"
    while True:
        try:
            res = requests.get(url, headers=HEADERS, timeout=20)
            res.raise_for_status()
            soup = BeautifulSoup(res.text, "html.parser")
            break
        except requests.exceptions.RequestException:
            print(f"Retrying ZIP {zipcode} in 60 seconds...")
            time.sleep(60)

    # Parse station cards
    cards = soup.find_all('div', class_=re.compile(r'StationItem'))
    now = datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S')

    records = []
    for c in cards:
        sid = c.get('id')
        price = c.find('span', class_=re.compile('Price'))
        posted = c.find('span', class_=re.compile('Time'))
        records.append([
            zipcode,
            sid,
            price.text if price else None,
            posted.text if posted else None,
            now
        ])
    return pd.DataFrame(records, columns=["zipcode", "station_id", "price", "posted_time", "collection_time"])
```


## Regarding data access:
According to [GasBuddy’s Terms of Service](https://chatgpt.com/g/g-p-6786dbd705908191a827700a1de5e363-1-wildfire/c/68e2b1ac-22e0-8328-889b-cb7a10a8595b#:~:text=According%20to%20GasBuddy%E2%80%99s,purposes%20(Request%20%231527604).), user-submitted retail gas price data is public, as GasBuddy does not claim ownership of it. Nonetheless, we requested and received explicit permission from GasBuddy to use its data for research purposes (Request `#1527604`).

---

## 🔍 Highlights of Analysis
- Analyzes the 2024 Park Fire in California and its effect on retail gasoline prices.
- Incorporates station-level gas price data, weather conditions, road closures, and proximity to the fire or closed roads.
- Identifies a **price decline** in the affected area during the wildfire period.

## 📑 Abstract
As wildfires become more frequent and intense due to factors such as climate change and dry vegetation accumulation, assessing their economic impact on critical commodities is essential. This study examines the short-term effects of the 2024 Park Fire on local retail gasoline prices in Northern California using **high-frequency station-level data**. 

Applying a **generalized difference-in-differences** approach and event study analysis, we find that gas stations within **20 miles of the wildfire** experienced an average **nine-cent-per-gallon price drop** compared to more distant control stations. This outcome contrasts with the **price spikes** typically observed following hurricanes and floods. 

Our findings contribute to the literature by demonstrating how natural disasters—particularly wildfires—can shape fuel market dynamics in unexpected ways. These insights emphasize the need for **post-disaster price monitoring** and market resilience strategies.

📄 **Latest Draft:** [Draft](Draft.pdf) [Preprint](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=5207948)

📌 **Keywords:** Natural Disasters, Wildfire, Retail Fuel Markets <br>
📚 **JEL Classifications:** Q41, Q54, R11, R32

Authors: **Ye-Rim Lee**, Dusan Paredes, Mark Skidmore, Scott Loveridge <br>
Contact: leeye12@msu.edu

