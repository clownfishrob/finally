# Market Data Interface Design

Unified Python interface for market data in FinAlly. Two implementations — GBM simulator (default) and Massive REST API (optional) — behind one abstract interface. All downstream code (SSE streaming, portfolio valuation, trade execution) is source-agnostic.

**Implementation status:** Complete. Code is in `backend/app/market/` (8 modules, ~500 lines, 73 tests passing).

---

## Architecture

```
                     ┌─────────────────────────┐
                     │  MarketDataSource (ABC)  │
                     │    interface.py          │
                     └─────────┬───────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                                 │
   ┌──────────▼──────────┐          ┌───────────▼──────────┐
   │ SimulatorDataSource  │          │  MassiveDataSource   │
   │   simulator.py       │          │  massive_client.py   │
   │                      │          │                      │
   │ GBMSimulator.step()  │          │ REST poll via        │
   │ every 500ms          │          │ asyncio.to_thread()  │
   └──────────┬───────────┘          │ every 15s            │
              │                      └───────────┬──────────┘
              │                                  │
              └──────────────┬───────────────────┘
                             │
                             ▼
                   ┌─────────────────┐
                   │   PriceCache    │     Thread-safe, in-memory
                   │    cache.py     │     Single point of truth
                   └────────┬────────┘
                            │
              ┌─────────────┼──────────────┐
              │             │              │
              ▼             ▼              ▼
         SSE Stream   Trade Execution  Portfolio
         stream.py    (fill price)     Valuation
```

The key design principle: **data sources are producers, everything else is a consumer.** Sources write to the `PriceCache` on their own schedule. Consumers read from the cache. There is no direct coupling between sources and consumers.

---

## Core Data Model — `PriceUpdate`

File: `backend/app/market/models.py`

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        """Absolute price change from previous update."""

    @property
    def change_percent(self) -> float:
        """Percentage change from previous update."""

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat'."""

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE transmission."""
```

Design choices:
- **Frozen dataclass** — immutable, safe to share across threads/tasks
- **`slots=True`** — memory-efficient, many of these exist simultaneously
- **Properties for derived values** — `change`, `change_percent`, `direction` are computed, not stored
- **`to_dict()`** — serialization for SSE events, avoids coupling to any JSON library

---

## Abstract Interface — `MarketDataSource`

File: `backend/app/market/interface.py`

```python
class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices —
    it reads from the cache.
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates. Starts a background task.
        Must be called exactly once."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources.
        Safe to call multiple times."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active set. Also removes from PriceCache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

The interface is deliberately minimal:
- **Async lifecycle** — `start()` and `stop()` are async because both implementations use `asyncio.create_task()`
- **No price retrieval** — sources push to cache, consumers read from cache
- **Ticker management** — `add_ticker()`/`remove_ticker()` for watchlist sync
- **`get_tickers()` is sync** — returns a list copy, no I/O involved

---

## Price Cache

File: `backend/app/market/cache.py`

```python
class PriceCache:
    """Thread-safe in-memory cache of the latest price for each ticker."""

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price. Computes direction/change from previous. Returns PriceUpdate."""

    def get(self, ticker: str) -> PriceUpdate | None:
        """Get latest price for a single ticker."""

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all current prices (shallow copy)."""

    def get_price(self, ticker: str) -> float | None:
        """Convenience: just the price float."""

    def remove(self, ticker: str) -> None:
        """Remove a ticker from the cache."""

    @property
    def version(self) -> int:
        """Monotonic counter, increments on every update. Used for SSE change detection."""
```

Key features:
- **Thread-safe** — uses `threading.Lock` because the Massive client runs in a thread pool via `asyncio.to_thread()`
- **Version counter** — SSE streamer skips sending when nothing changed, reducing bandwidth
- **Automatic direction** — `update()` computes `up`/`down`/`flat` by comparing to the previous cached price
- **Rounds to 2 decimals** — prices stored as `round(price, 2)` for display consistency

---

## Factory Function

File: `backend/app/market/factory.py`

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Create the appropriate source based on environment variables.

    - MASSIVE_API_KEY set and non-empty → MassiveDataSource (real data)
    - Otherwise → SimulatorDataSource (GBM simulation)

    Returns an unstarted source. Caller must await source.start(tickers).
    """
```

Selection logic:
```python
api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
if api_key:
    return MassiveDataSource(api_key=api_key, price_cache=price_cache)
else:
    return SimulatorDataSource(price_cache=price_cache)
```

---

## Implementation: SimulatorDataSource

File: `backend/app/market/simulator.py`

Wraps the `GBMSimulator` (see [MARKET_SIMULATOR.md](MARKET_SIMULATOR.md)) in an async loop:

- **Update interval**: 500ms (configurable)
- **On `start()`**: Creates `GBMSimulator`, seeds cache with initial prices, starts background task
- **On `add_ticker()`**: Delegates to `GBMSimulator.add_ticker()`, immediately seeds cache
- **On `remove_ticker()`**: Delegates to simulator, removes from cache
- **Loop**: Calls `simulator.step()` → writes all prices to cache → sleeps 500ms

No external dependencies. No API key needed. This is the default.

---

## Implementation: MassiveDataSource

File: `backend/app/market/massive_client.py`

Polls the Massive REST API for real market data:

- **Poll interval**: 15s default (safe for free tier's 5 req/min limit)
- **On `start()`**: Creates `RESTClient`, does an immediate first poll, starts background task
- **Each poll**: Calls `get_snapshot_all()` via `asyncio.to_thread()` (client is synchronous)
- **Extracts**: `snap.last_trade.price` and `snap.last_trade.timestamp` from each snapshot
- **Error handling**: Logs and continues — never crashes the poll loop. Handles missing `lastTrade` fields (free tier) via try/except on `AttributeError`

See [MASSIVE_API.md](MASSIVE_API.md) for full API documentation.

---

## SSE Streaming

File: `backend/app/market/stream.py`

```python
def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Create FastAPI router with GET /api/stream/prices SSE endpoint."""
```

The SSE streamer reads from `PriceCache` every 500ms and pushes to connected clients:

- Uses `PriceCache.version` for change detection — only sends when data has actually changed
- SSE format: `data: {"AAPL": {...}, "GOOGL": {...}, ...}\n\n`
- Sends `retry: 1000\n\n` on connect so the browser auto-reconnects after 1 second
- Detects client disconnect via `request.is_disconnected()`
- Response headers disable caching and nginx buffering

---

## File Structure

```
backend/app/market/
├── __init__.py           # Public exports: PriceCache, PriceUpdate, MarketDataSource, create_market_data_source
├── models.py             # PriceUpdate frozen dataclass
├── interface.py          # MarketDataSource ABC
├── cache.py              # PriceCache (thread-safe in-memory store)
├── factory.py            # create_market_data_source() factory
├── simulator.py          # GBMSimulator + SimulatorDataSource
├── massive_client.py     # MassiveDataSource (REST poller)
├── seed_prices.py        # Seed prices, per-ticker GBM params, correlation groups
└── stream.py             # SSE streaming endpoint factory
```

---

## Application Lifecycle

```python
from app.market import PriceCache, create_market_data_source, create_stream_router

# 1. Startup (FastAPI lifespan)
cache = PriceCache()
source = create_market_data_source(cache)  # Reads MASSIVE_API_KEY
await source.start(["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
                     "NVDA", "META", "JPM", "V", "NFLX"])

# 2. Register SSE endpoint
stream_router = create_stream_router(cache)
app.include_router(stream_router)

# 3. Runtime — watchlist changes
await source.add_ticker("PYPL")
await source.remove_ticker("NFLX")

# 4. Runtime — read prices (from cache, not source)
update = cache.get("AAPL")          # PriceUpdate or None
price = cache.get_price("AAPL")     # float or None
all_prices = cache.get_all()        # dict[str, PriceUpdate]

# 5. Shutdown
await source.stop()
```

---

## Design Decisions & Rationale

| Decision | Why |
|----------|-----|
| Strategy pattern (ABC + 2 implementations) | Downstream code is source-agnostic; easy to test with either implementation |
| PriceCache as single point of truth | Decouples producers from consumers; no direct coupling between source and SSE/portfolio |
| Thread-safe cache (Lock) | Massive client runs in thread pool via `asyncio.to_thread()` |
| Version counter on cache | SSE streamer skips sends when nothing changed — reduces unnecessary bandwidth |
| Factory function, not class | Simple selection logic doesn't need inheritance or configuration objects |
| `PriceUpdate` is frozen | Immutable = safe to share, no defensive copies needed |
| No direct price retrieval on interface | Sources push on their own schedule; pull-based would create coupling and timing issues |

---

## Downstream Integration Points

| Consumer | How it uses prices | Cache method |
|----------|-------------------|--------------|
| SSE streaming | Push all prices to browser every 500ms | `cache.get_all()` + `cache.version` |
| Trade execution | Fill price for market orders | `cache.get_price(ticker)` |
| Portfolio valuation | Current value of all positions | `cache.get_all()` |
| Watchlist API | Return prices alongside watchlist tickers | `cache.get(ticker)` |
| LLM context | Portfolio snapshot for chat prompts | `cache.get_all()` |
