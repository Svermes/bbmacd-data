# bbmacd-data

Public, read-only data mirror for the BBMACD swing scanner.

Everything here is **generated** — it is pushed automatically by the
`Mirror history.csv to public data repo` workflow in the private scanner repo
after each daily scan. Do not edit these files by hand; the next scan overwrites
them.

The only reason this repo exists is that sandboxed consumers can reach
`raw.githubusercontent.com` and nothing else.

## Files

| File | Size | Contents |
|---|---|---|
| [`history_recent.csv`](history_recent.csv) | ~7 MB | Trailing **300 trading sessions**. Use this one. |
| [`history.csv`](history.csv) | ~31 MB | Full archive, every session since 2021-05-24. |
| [`manifest.json`](manifest.json) | <1 KB | Row counts, date range, and the source blob the files were built from. |
| [`position_calculator.html`](position_calculator.html) | — | Unrelated; predates the mirror. |

Prefer `history_recent.csv` unless you genuinely need history before 2025-07.
300 sessions is comfortably more lookback than a 200-day moving average
requires, and it downloads in a fraction of the time.

## Schema

```
date,symbol,open,high,low,close,volume
2026-09-15,ZTS,72.91,73.78,72.72,73.43,4079011
```

- `date` — `YYYY-MM-DD`, the session date. Rows are grouped by symbol, then
  ascending by date.
- `open,high,low,close` — **rounded to 2 decimal places.** The private source
  keeps Yahoo's raw float64 (`72.91000366210938`); that extra precision is
  noise, and dropping it halves the file. Values are exact to the cent, so
  worst-case error on any price is half a cent.
- `volume` — integer, copied through unchanged.
- Prices are **not** split- or dividend-adjusted beyond whatever Yahoo returned
  at fetch time.

~511 symbols (S&P 500 constituents plus recent changes).

## Fetching

```python
import urllib.request, io
import pandas as pd

URL = "https://raw.githubusercontent.com/Svermes/bbmacd-data/main/history_recent.csv"
req  = urllib.request.Request(URL, headers={"User-Agent": "python/3.12"})
raw  = urllib.request.urlopen(req, timeout=30).read().decode("utf-8")
df   = pd.read_csv(io.StringIO(raw), parse_dates=["date"])

aapl = df[df.symbol == "AAPL"].sort_values("date")
```

Give the request a realistic timeout — 30s, not 10s. `history.csv` is ~31 MB
and can take a while on a slow link.

## Freshness

`manifest.json` records `generated_at` and the `source_blob_sha` these files
were built from, so you can tell whether a scan has landed yet:

```python
import json, urllib.request
m = json.load(urllib.request.urlopen(
    "https://raw.githubusercontent.com/Svermes/bbmacd-data/main/manifest.json"))
print(m["generated_at"], m["recent"]["last_date"])
```

Note that `raw.githubusercontent.com` caches for about five minutes, so a fresh
push can take that long to become visible.
