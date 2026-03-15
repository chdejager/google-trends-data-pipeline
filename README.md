# google-trends-data-pipeline
Automated pipeline for bulk-collecting Google Trends interest-over-time data with anchor-based rescaling, batching, and auto-resume — built for quantitative and alternative data research.
---

## Overview

Google Trends is a widely used source of alternative data in quantitative finance, capturing retail investor attention and search-based sentiment signals across tickers, sectors, and macroeconomic themes.

This pipeline automates bulk data collection at scale, solving the core limitations of using Google Trends manually:

- **Cross-batch comparability** — Google Trends returns relative scores (0–100) that are recalculated per query, making raw results from separate requests incomparable. This pipeline anchors every batch to a common reference term and rescales accordingly.
- **Rate limiting** — Google aggressively throttles programmatic requests. The pipeline handles this with exponential backoff, jitter, and configurable delays.
- **Fault tolerance** — Results are written to disk after every batch. If the job crashes or gets rate-limited mid-run, it resumes from where it left off.

---

## How it works

Google does not offer an official API for Trends data. This pipeline uses [`pytrends`](https://github.com/GeneralMills/pytrends), an open-source library that programmatically interfaces with `trends.google.com` — the only available method for automated access.

```
Google Trends (trends.google.com)
        ↓
   pytrends — mimics browser requests, returns raw data
        ↓
   pipeline — batches, rescales, retries, writes to disk
        ↓
   output CSV — long-format: (date, keyword, interest)
```

### Anchor-based rescaling

A configurable anchor term (default: `"stock market"`) is included in every batch of keywords. Each keyword's interest score is then divided by the anchor's score, normalising all batches to a common baseline. This makes scores comparable across keywords collected in separate API calls.

---

## Quickstart

```bash
pip install pytrends pandas
```

```bash
# Single keyword, last 10 years
python google_trends_pipeline.py --keywords AAPL --geo US --output aapl_trends.csv

# Bulk keywords from CSV
python google_trends_pipeline.py \
    --input keywords.csv \
    --column keyword \
    --start 2016-01-01 \
    --end   2025-12-31 \
    --geo   US \
    --output trends_output.csv
```

---

## CLI options

| Flag | Default | Description |
|------|---------|-------------|
| `--input` | — | CSV file containing keywords |
| `--column` | `keyword` | Column name in the input CSV |
| `--keywords` | — | Comma-separated keywords (alternative to `--input`) |
| `--start` | 10 years ago | Start date `YYYY-MM-DD` |
| `--end` | Today | End date `YYYY-MM-DD` |
| `--geo` | `US` | Country code (`US`, `GB`, etc.) — leave blank for worldwide |
| `--output` | `trends_final.csv` | Output CSV path |
| `--delay` | `15` | Base delay in seconds between requests |
| `--retries` | `5` | Max retries per batch on failure |
| `--anchor` | `stock market` | Anchor term for rescaling — pass empty string to disable |

---

## Output format

Long-format CSV with one row per `(keyword, date)` pair:

| date | keyword | interest |
|------|---------|----------|
| 2020-01-05 | AAPL | 42.3 |
| 2020-01-05 | MSFT | 31.7 |
| 2020-01-12 | AAPL | 48.1 |

See `examples/sample_output.csv` for a full example.

---

## Requirements

```
pytrends
pandas
```

Or install via:

```bash
pip install -r requirements.txt
```

> **Note:** `pytrends` is an unofficial library and may break if Google changes its internal endpoints. If you encounter persistent failures, check for [upstream updates](https://github.com/GeneralMills/pytrends) or consider paid alternatives such as SerpAPI.

---

## Project structure

```
google-trends-data-pipeline/
├── google_trends_pipeline.py   # Main pipeline
├── requirements.txt
├── examples/
│   ├── sample_keywords.csv     # Example keyword list
│   └── sample_output.csv       # Example output
└── tests/
    └── test_transforms.py      # Unit tests for rescale and melt functions
```

---

## Use cases

- Retail investor attention signals for equity research
- Search-based sentiment as a feature in predictive models
- Sector rotation and thematic trend analysis
- Macro keyword monitoring (e.g. "inflation", "recession", "fed rate")

---

## Notes

- Be mindful of Google's Terms of Service when running at scale
- Recommended delay between requests is 15–30 seconds to avoid rate limiting
- For very large keyword lists (1000+), consider running overnight or in batches across sessions — auto-resume handles interruptions
