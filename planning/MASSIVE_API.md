# Massive API Reference (formerly Polygon.io)

Reference documentation for the Massive REST API as used in FinAlly for real market data.

## Overview

| Detail | Value |
|--------|-------|
| **Base URL** | `https://api.massive.com` (legacy `https://api.polygon.io` still works) |
| **Python package** | `massive` — install via `uv add massive` or `pip install -U massive` |
| **Min Python** | 3.9+ (project uses 3.12) |
| **Auth method** | Bearer token: `Authorization: Bearer <API_KEY>` |
| **Auth in Python** | Reads `MASSIVE_API_KEY` env var automatically, or `RESTClient(api_key=...)` |

Polygon.io rebranded to Massive on 2025-10-30. All existing API keys, endpoints, and client libraries are backward-compatible. The Python package was renamed from `polygon-api-client` to `massive`.

---

## Rate Limits

| Tier | Limit | Recommended Poll Interval |
|------|-------|---------------------------|
| **Free** | 5 requests/minute | 15 seconds (default in FinAlly) |
| **Paid (all tiers)** | Unlimited (stay under 100 req/s) | 2-5 seconds |

### Data Availability by Plan

Not all response fields are available on all plans:

| Field | Required Plan |
|-------|---------------|
| End-of-day aggregates, previous close | Free |
| `day` (current day OHLCV) | Free |
| `prevDay` (previous day OHLCV) | Free |
| `lastTrade` (real-time trade price) | Paid (Stocks Starter+) |
| `lastQuote` (real-time NBBO) | Paid (Stocks Starter+) |
| `fmv` (fair market value) | Business plan only |

On the free tier, `lastTrade` and `lastQuote` may be absent or null. The FinAlly implementation handles this gracefully via try/except on `AttributeError`.

---

## Client Initialization

```python
from massive import RESTClient

# Reads MASSIVE_API_KEY from environment automatically
client = RESTClient()

# Or pass explicitly
client = RESTClient(api_key="your_key_here")
```

The RESTClient is **synchronous**. In async code (FastAPI), wrap calls in `asyncio.to_thread()` to avoid blocking the event loop.

---

## Primary Endpoint: Snapshot — All Tickers

This is the main endpoint FinAlly uses. It returns current prices for multiple tickers in a **single API call** — critical for staying within free-tier rate limits.

### REST Request

```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT,AMZN,TSLA
Authorization: Bearer <API_KEY>
```

### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tickers` | string (comma-separated) | No | Case-sensitive ticker symbols. Max 250 per request. If omitted, returns all tickers. |
| `include_otc` | boolean | No | Include OTC securities. Default: `false` |

### Python Client

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    print(f"{snap.ticker}: ${snap.last_trade.price}")
```

### Response JSON

```json
{
  "count": 5,
  "status": "OK",
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChange": -4.54,
      "todaysChangePerc": -3.50,
      "updated": 1675190399000,
      "day": {
        "o": 129.61,
        "h": 130.15,
        "l": 125.07,
        "c": 125.07,
        "v": 111237700,
        "vw": 127.35
      },
      "prevDay": {
        "o": 128.50,
        "h": 130.00,
        "l": 128.00,
        "c": 129.61,
        "v": 95000000,
        "vw": 129.10
      },
      "min": {
        "o": 125.10,
        "h": 125.12,
        "l": 125.05,
        "c": 125.07,
        "v": 50000,
        "vw": 125.08,
        "av": 111237700,
        "n": 120,
        "t": 1675190340000
      },
      "lastTrade": {
        "p": 125.07,
        "s": 100,
        "x": 11,
        "i": "TAPE_A",
        "c": [14, 41],
        "t": 1675190399000
      },
      "lastQuote": {
        "p": 125.06,
        "P": 125.08,
        "s": 500,
        "S": 1000,
        "t": 1675190399500
      },
      "fmv": 125.07
    }
  ]
}
```

### Response Field Reference

**Bar objects** (`day`, `min`, `prevDay`):

| Field | Description |
|-------|-------------|
| `o` | Open price |
| `h` | High price |
| `l` | Low price |
| `c` | Close price |
| `v` | Volume (shares traded) |
| `vw` | Volume-weighted average price (VWAP) |
| `av` | Accumulated volume (`min` bar only) |
| `n` | Number of transactions (`min` bar only) |
| `t` | Timestamp in Unix **milliseconds** |

**`lastTrade` object** (requires paid plan):

| Field | Description |
|-------|-------------|
| `p` | Trade price |
| `s` | Trade size (shares) |
| `x` | Exchange code |
| `i` | Tape ID |
| `c` | Condition codes array |
| `t` | Timestamp in Unix **milliseconds** |

**`lastQuote` object** (requires paid plan):

| Field | Description |
|-------|-------------|
| `p` | Bid price |
| `P` | Ask price |
| `s` | Bid size |
| `S` | Ask size |
| `t` | Timestamp in Unix **milliseconds** |

**Top-level per-ticker fields**:

| Field | Description |
|-------|-------------|
| `todaysChange` | Dollar change from previous close |
| `todaysChangePerc` | Percentage change from previous close |
| `updated` | Last updated timestamp (Unix ms) |
| `fmv` | Fair market value (Business plan only) |

### Python Client Object Mapping

The Python client maps the raw JSON to objects with Pythonic attribute names:

| Raw JSON | Python attribute |
|----------|-----------------|
| `lastTrade.p` | `snap.last_trade.price` |
| `lastTrade.t` | `snap.last_trade.timestamp` |
| `lastTrade.s` | `snap.last_trade.size` |
| `lastQuote.p` | `snap.last_quote.bid_price` |
| `lastQuote.P` | `snap.last_quote.ask_price` |
| `day.c` | `snap.day.close` |
| `day.o` | `snap.day.open` |
| `prevDay.c` | `snap.prev_day.close` |
| `todaysChange` | `snap.todays_change` |
| `todaysChangePerc` | `snap.todays_change_percent` |

### Important Notes

- Data resets daily at ~3:30 AM EST, repopulates around ~4:00 AM EST
- During market closed hours, `lastTrade.p` reflects the last traded price (may include after-hours)
- All timestamps are Unix **milliseconds** — divide by 1000 for Python `time.time()` seconds
- The `tickers` parameter accepts a maximum of **250 symbols** per request

---

## Other Useful Endpoints

### Single Ticker Snapshot

```
GET /v2/snapshot/locale/us/markets/stocks/tickers/{stocksTicker}
```

```python
snapshot = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)
print(f"Price: ${snapshot.last_trade.price}")
```

Response structure is identical to a single element from the multi-ticker snapshot.

### Previous Close

```
GET /v2/aggs/ticker/{ticker}/prev
```

```python
prev = client.get_previous_close_agg(ticker="AAPL")
for agg in prev:
    print(f"Previous close: ${agg.close}")
    print(f"OHLC: O={agg.open} H={agg.high} L={agg.low} C={agg.close}")
    print(f"Volume: {agg.volume}")
```

### Aggregates (Historical Bars)

```
GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}
```

```python
for a in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2024-01-01",
    to="2024-01-31",
    limit=50000,
):
    print(f"O={a.open} H={a.high} L={a.low} C={a.close} V={a.volume}")
```

### Last Trade / Last Quote

```python
# Last trade for a ticker
trade = client.get_last_trade(ticker="AAPL")
print(f"Last trade: ${trade.price} x {trade.size}")

# Last NBBO quote
quote = client.get_last_quote(ticker="AAPL")
print(f"Bid: ${quote.bid} x {quote.bid_size}")
print(f"Ask: ${quote.ask} x {quote.ask_size}")
```

---

## Error Handling

| HTTP Code | Meaning | Action |
|-----------|---------|--------|
| 401 | Invalid API key | Check `MASSIVE_API_KEY` env var |
| 403 | Insufficient permissions (plan doesn't include endpoint) | Upgrade plan or use free-tier endpoints only |
| 429 | Rate limit exceeded (free tier: 5 req/min) | Increase poll interval |
| 5xx | Server error | Client retries automatically (3 retries by default) |

---

## How FinAlly Uses the API

The implemented code is in `backend/app/market/massive_client.py`. The polling loop:

1. Collects all tickers from the active watchlist
2. Calls `get_snapshot_all()` with those tickers — **one API call for all tickers**
3. Runs the synchronous client call via `asyncio.to_thread()` to avoid blocking
4. Extracts `snap.last_trade.price` and `snap.last_trade.timestamp` from each snapshot
5. Converts timestamps from milliseconds to seconds
6. Writes each price to the shared `PriceCache`
7. Sleeps for the poll interval (default 15s), then repeats

```python
async def _poll_once(self) -> None:
    if not self._tickers or not self._client:
        return
    try:
        snapshots = await asyncio.to_thread(self._fetch_snapshots)
        for snap in snapshots:
            try:
                price = snap.last_trade.price
                timestamp = snap.last_trade.timestamp / 1000.0  # ms -> seconds
                self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
            except (AttributeError, TypeError) as e:
                logger.warning("Skipping snapshot for %s: %s", getattr(snap, "ticker", "???"), e)
    except Exception as e:
        logger.error("Massive poll failed: %s", e)
```

Errors are logged but never crash the poll loop — it retries on the next interval.
