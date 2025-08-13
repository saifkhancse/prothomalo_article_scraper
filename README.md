# Prothom Alo Article Scraper (Bangla) — v6

Collect full-text Bangla news articles from **Prothom Alo** with metadata, resume safely with state files, and run quick EDA with Bangla-aware plots.

> **Use case:** Bangla NLP (topic modeling, classification, NER), time-series trend analysis, media studies, etc.  
> **Output:** JSONL — one article per line with `url`, `title`, `section`, `author`, `published_iso`, `published_text`, `tags`, `body`, `word_count`.

---

## What’s in this repo

- `prothomalo_streaming_v6.ipynb` — the main **scraper notebook** (streaming + month/date range collection).
- `prothomalo EDA.ipynb` — **exploratory analysis** notebook (helps you validate coverage & data quality).
- `prothomalo_stream_state.json` — streaming **cursor/state** (resume from last position).
- `prothomalo_monthly_state.json` — **per-month progress** (which months are finished).
- `prothomalo_seen_links.json` — **dedupe set** (URLs already saved).
- `prothomalo_session_stats.json` — session counters, error rates, timings.
- `kalpurush.ttf`, `NotoSansBengali-Regular.ttf` — fonts for **Bangla rendering** in plots.

---

## Output schema (JSONL)

Each line is a JSON object like:

```json
{
  "url": "https://www.prothomalo.com/bangladesh/2l51vik8wk",
  "title": "...",
  "section": "বাংলাদেশ",
  "author": "নিজস্ব প্রতিবেদক",
  "published_iso": "2025-08-12T10:30:18+06:00",
  "published_text": "আপডেট: ১২ আগস্ট ২০২৫, ০৪: ৩০",
  "tags": ["মালয়েশিয়া", "সমঝোতা"],
  "body": "… full article text …",
  "word_count": 410
}
```

> `word_count` may be absent for some rows. You can estimate from `body` during EDA if needed.

---

## Quick start

### 1) Clone & create environment
```bash
git clone https://github.com/saifkhancse/prothomalo_article_scraper.git
cd prothomalo_article_scraper

# (recommended) create a virtualenv
python -m venv .venv
# Windows:
.venv\Scripts\activate
# macOS/Linux:
source .venv/bin/activate

# install deps
pip install -U pip
pip install -r requirements.txt  # if you have one
```

If you don’t have a `requirements.txt`, install the common deps:
```bash
pip install pandas requests beautifulsoup4 lxml tqdm dateparser tenacity
```

### 2) Open the scraper
Launch Jupyter and open **`prothomalo_streaming_v6.ipynb`**:
```bash
jupyter lab
# or
jupyter notebook
```

---

## Running the scraper (v6)

The v6 notebook supports **three modes**:

1. **Monthly range** (crawl archives from start month to end month)  
2. **Exact date/time range** (collect only items whose `published_iso` falls in a window)  
3. **Streaming** (poll new items continuously; safe to stop/restart)

> The notebook has a **Config** cell near the top. If it doesn’t, create one with the snippet below and run it *before* the main run cell.

### Config cell (add/edit in the notebook)
```python
# ==== SCRAPER CONFIG (edit this cell) ====

# Choose one mode: "monthly", "range", or "stream"
MODE = "monthly"           # "monthly" | "range" | "stream"

# Monthly mode (inclusive)
START_MONTH = "2022-07"    # YYYY-MM
END_MONTH   = "2025-08"    # YYYY-MM

# Exact date/time mode (inclusive, ISO-like)
START_DATE = "2024-07-01T00:00:00+06:00"
END_DATE   = "2024-07-31T23:59:59+06:00"

# Streaming mode
POLL_SECONDS = 60          # how often to poll for new articles (in seconds)
MAX_BATCH = 200            # optional throttle per cycle

# Common options
SAVE_PATH = "prothomalo_articles.jsonl"   # output file
STATE_DIR = "."                           # where state JSONs live
RATE_LIMIT_SECONDS = 1.0                  # polite delay between requests
MAX_PAGES_PER_MONTH = None                # set int to cap pages/month

# Networking
USER_AGENT = ("Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
              "AppleWebKit/537.36 (KHTML, like Gecko) "
              "Chrome/126.0 Safari/537.36 "
              "(prothomalo-scraper/v6; github.com/saifkhancse)")

# Dedup/Resume
RESUME = True             # keep using state files to resume safely
OVERWRITE_OUTPUT = False  # set True to start a fresh output (also delete state files)
```

> The rest of the notebook reads these variables to decide what to crawl.

### Run it
- Set `MODE` and the corresponding dates/months in the **Config** cell
- Run the subsequent “Run” cell (or the whole notebook)

---

## Date & time control — how it works

- **Monthly mode** uses `START_MONTH` → `END_MONTH` (e.g., `"2023-01"` to `"2023-12"`).  
  It visits monthly archive pages and pulls everything in that span.

- **Exact range mode** uses `START_DATE` → `END_DATE` (full timestamps recommended, with `+06:00`).  
  Only items whose parsed `published_iso` fall inside the window are saved.

- **Streaming mode** ignores the above windows; it **polls forward** from the saved cursor in `prothomalo_stream_state.json`.  
  - Stop anytime; restart will continue from where it left off.  
  - To **start fresh**, delete `prothomalo_stream_state.json` and `prothomalo_seen_links.json`.

---

## State files (resume safely)

- `prothomalo_stream_state.json` — stores the last processed cursor/URL/time for streaming.
- `prothomalo_monthly_state.json` — tracks which months (and pages) are already done.
- `prothomalo_seen_links.json` — hash set of URLs to avoid duplicates across runs.
- `prothomalo_session_stats.json` — counters (requests, saved rows, errors, durations).

**Resume behavior**
- Keep state files in place and set `RESUME = True` → the scraper skips what’s already done.
- Want a **clean re-crawl**? Set `OVERWRITE_OUTPUT = True` **and** delete the state files above.

---

## Output & simple dedup

The main output is `prothomalo_articles.jsonl`. To dedup by URL from the command line (quick & dirty):

```bash
# Python one-liner
python - <<'PY'
import json, sys
seen=set()
with open("prothomalo_articles.jsonl","r",encoding="utf-8") as inp, \
     open("prothomalo_articles_dedup.jsonl","w",encoding="utf-8") as out:
    for line in inp:
        try:
            o=json.loads(line)
        except Exception:
            continue
        u=o.get("url")
        if u and u not in seen:
            seen.add(u)
            out.write(json.dumps(o, ensure_ascii=False)+"\n")
print("Wrote", len(seen), "unique rows")
PY
```

---

## EDA (plots with Bangla labels)

Open **`prothomalo EDA.ipynb`** and run all cells.  
If labels don’t render in Bangla, set Matplotlib to use one of the included fonts:

```python
import matplotlib.pyplot as plt
from matplotlib import font_manager as fm

# pick one:
font_path = "kalpurush.ttf"  # or "NotoSansBengali-Regular.ttf"

fm.fontManager.addfont(font_path)
font_name = fm.FontProperties(fname=font_path).get_name()
plt.rcParams["font.family"] = font_name
plt.rcParams["font.sans-serif"] = [font_name]
plt.rcParams["axes.unicode_minus"] = False
```

---

## Headless runs (optional)

If you want to run the notebook headlessly with parameters (e.g., on a server/cron), use **papermill**:

```bash
pip install papermill

papermill \
  prothomalo_streaming_v6.ipynb out.ipynb \
  -p MODE "range" \
  -p START_DATE "2024-07-01T00:00:00+06:00" \
  -p END_DATE   "2024-07-31T23:59:59+06:00" \
  -p SAVE_PATH "prothomalo_articles.jsonl" \
  -p RATE_LIMIT_SECONDS 1.0 \
  -p RESUME True
```

> If your notebook doesn’t have `papermill` parameters yet, keep using the **Config** cell approach.

---

## Tips & troubleshooting

- **HTTP 403 / blocks**: lower `MAX_BATCH`, increase `RATE_LIMIT_SECONDS`, rotate `USER_AGENT`.  
- **Incomplete months**: re-run in monthly mode; the state files ensure you won’t re-download finished pages.  
- **Timezone**: `published_iso` keeps site timezone (`+06:00`). Convert later if needed.  
- **Large runs**: write to disk frequently; don’t keep everything in memory.

---

## Terms & ethics

- Data is collected from the publicly accessible **Prothom Alo** site.  
- This repo is for **research and educational purposes**.  
- Respect the site’s **Terms of Service** and robots directives.  
- Use **polite rate limits** and avoid excessive load.

---

## Related

- **Kaggle dataset (public dump)**:  
  https://www.kaggle.com/datasets/saifkhancse/prothomalo-dataset

---

## Credits

Built by **@saifkhancse**. Fonts © their respective authors:  
- Kalpurush, Noto Sans Bengali (included for visualization only).

Contributions & issues welcome!
