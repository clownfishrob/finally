# Market Simulator Design

Approach and code structure for simulating realistic stock prices when no Massive API key is configured. This is the default market data source for FinAlly.

**Implementation status:** Complete. Code is in `backend/app/market/simulator.py` and `backend/app/market/seed_prices.py`.

---

## Overview

The simulator generates realistic stock price paths using **Geometric Brownian Motion (GBM)** — the standard model underlying Black-Scholes option pricing. Prices evolve continuously with random noise, can't go negative, and exhibit the lognormal distribution observed in real markets.

Updates run every 500ms, producing a continuous stream of sub-cent price changes that accumulate naturally over time. With 10 tickers updating twice per second, the dashboard feels alive and responsive.

---

## GBM Math

At each time step, a stock price evolves as:

```
S(t+dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)
```

Where:
- **S(t)** — current price
- **mu** — annualized drift (expected return), e.g. 0.05 (5% per year)
- **sigma** — annualized volatility, e.g. 0.22 (22% per year)
- **dt** — time step as a fraction of a trading year
- **Z** — standard normal random variable (drawn from N(0,1))

### Time Step Calculation

For 500ms updates with 252 trading days and 6.5 hours per day:

```
Trading seconds per year = 252 * 6.5 * 3600 = 5,896,800
dt = 0.5 / 5,896,800 ≈ 8.48e-8
```

This tiny `dt` produces sub-cent moves per tick. With `sigma = 0.22` (AAPL), a single tick moves the price by roughly $0.001-$0.01. Over a simulated trading day (~46,800 ticks), this accumulates to realistic intraday ranges.

### Why GBM?

- Prices **never go negative** — the `exp()` function is always positive
- Returns are **lognormally distributed** — matches real-world stock return distributions
- **Simple and fast** — one `exp()` call per ticker per tick
- **Well-understood** — the standard model in quantitative finance

---

## Correlated Moves

Real stocks don't move independently — tech stocks tend to move together, banks correlate with each other, etc. The simulator uses a **Cholesky decomposition** of a correlation matrix to generate correlated random draws.

### How It Works

1. Build a correlation matrix `C` based on sector groupings
2. Compute `L = cholesky(C)` — the lower-triangular Cholesky factor
3. At each step, generate `n` independent standard normals `Z_independent`
4. Transform: `Z_correlated = L @ Z_independent`
5. Use `Z_correlated[i]` as the random draw for ticker `i`

This preserves the marginal distribution of each ticker (still standard normal) while introducing the specified pairwise correlations.

### Correlation Structure

Defined in `backend/app/market/seed_prices.py`:

| Pair | Correlation | Rationale |
|------|-------------|-----------|
| Tech + Tech | 0.6 | AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX move together |
| Finance + Finance | 0.5 | JPM, V move together |
| Cross-sector | 0.3 | Tech vs Finance, or unknown tickers |
| TSLA + anything | 0.3 | TSLA does its own thing (treated as special case despite being in tech group) |

```python
CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR = 0.6
INTRA_FINANCE_CORR = 0.5
CROSS_GROUP_CORR = 0.3
TSLA_CORR = 0.3
```

The correlation matrix is rebuilt whenever tickers are added or removed. This is O(n^2) but n is small (< 50 tickers), so it's negligible.

---

## Random Shock Events

Every step, each ticker has a ~0.1% probability of a random shock — a sudden 2-5% move up or down. This adds visual drama and makes the dashboard more engaging.

```python
if random.random() < 0.001:
    shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
    price *= (1 + shock)
```

**Expected frequency:** With 10 tickers at 2 ticks/second, expect a shock event somewhere roughly every 50 seconds — enough to keep things interesting without being unrealistic.

---

## Seed Prices and Parameters

File: `backend/app/market/seed_prices.py`

### Starting Prices

Realistic prices for the 10 default watchlist tickers:

```python
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00,
    "GOOGL": 175.00,
    "MSFT": 420.00,
    "AMZN": 185.00,
    "TSLA": 250.00,
    "NVDA": 800.00,
    "META": 500.00,
    "JPM": 195.00,
    "V": 280.00,
    "NFLX": 600.00,
}
```

Tickers added dynamically (not in the seed list) start at a random price between $50-$300.

### Per-Ticker Volatility and Drift

Each ticker has its own parameters reflecting real-world behavior:

```python
TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},   # Moderate vol
    "GOOGL": {"sigma": 0.25, "mu": 0.05},   # Moderate vol
    "MSFT":  {"sigma": 0.20, "mu": 0.05},   # Lower vol (mature)
    "AMZN":  {"sigma": 0.28, "mu": 0.05},   # Higher vol
    "TSLA":  {"sigma": 0.50, "mu": 0.03},   # Very high vol, lower drift
    "NVDA":  {"sigma": 0.40, "mu": 0.08},   # High vol, strong drift (AI momentum)
    "META":  {"sigma": 0.30, "mu": 0.05},   # Moderate-high vol
    "JPM":   {"sigma": 0.18, "mu": 0.04},   # Low vol (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},   # Low vol (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},   # Higher vol
}

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}  # For dynamically added tickers
```

---

## Implementation — `GBMSimulator`

File: `backend/app/market/simulator.py`

```python
class GBMSimulator:
    """Geometric Brownian Motion simulator for correlated stock prices."""

    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None:
        # Initialize per-ticker state (prices, params)
        # Build Cholesky decomposition of correlation matrix

    def step(self) -> dict[str, float]:
        """Advance all tickers by one time step. Returns {ticker: new_price}.
        This is the hot path — called every 500ms."""
        # 1. Generate n independent standard normal draws
        # 2. Apply Cholesky to get correlated draws
        # 3. For each ticker: apply GBM formula, check for random event
        # 4. Return {ticker: rounded_price}

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker. Seeds from SEED_PRICES or random $50-$300. Rebuilds Cholesky."""

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Rebuilds Cholesky."""

    def get_price(self, ticker: str) -> float | None:
        """Current price for a ticker, or None if not tracked."""

    def get_tickers(self) -> list[str]:
        """Return the list of currently tracked tickers."""
```

### Step Method — The Hot Path

```python
def step(self) -> dict[str, float]:
    n = len(self._tickers)
    if n == 0:
        return {}

    # Correlated random draws via Cholesky
    z_independent = np.random.standard_normal(n)
    z_correlated = self._cholesky @ z_independent if self._cholesky is not None else z_independent

    result: dict[str, float] = {}
    for i, ticker in enumerate(self._tickers):
        mu = self._params[ticker]["mu"]
        sigma = self._params[ticker]["sigma"]

        # GBM: S(t+dt) = S(t) * exp((mu - 0.5*sigma^2)*dt + sigma*sqrt(dt)*Z)
        drift = (mu - 0.5 * sigma**2) * self._dt
        diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
        self._prices[ticker] *= math.exp(drift + diffusion)

        # Random shock event (~0.1% chance per tick per ticker)
        if random.random() < self._event_prob:
            shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
            self._prices[ticker] *= (1 + shock)

        result[ticker] = round(self._prices[ticker], 2)

    return result
```

---

## Implementation — `SimulatorDataSource`

The `MarketDataSource` wrapper that runs the simulator in an async loop:

```python
class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by the GBM simulator."""

    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5, event_probability: float = 0.001):
        # Store cache reference, interval, and event probability

    async def start(self, tickers: list[str]) -> None:
        # 1. Create GBMSimulator with initial tickers
        # 2. Seed cache with initial prices (so SSE has data immediately)
        # 3. Start background asyncio task

    async def stop(self) -> None:
        # Cancel the background task

    async def add_ticker(self, ticker: str) -> None:
        # Delegate to GBMSimulator.add_ticker()
        # Immediately seed cache with the new ticker's initial price

    async def remove_ticker(self, ticker: str) -> None:
        # Delegate to GBMSimulator.remove_ticker()
        # Remove from PriceCache

    def get_tickers(self) -> list[str]:
        # Delegate to GBMSimulator.get_tickers()

    async def _run_loop(self) -> None:
        # Loop: simulator.step() → write all prices to cache → sleep 500ms
        # Exception handling: log and continue (never crash the loop)
```

---

## Behavior Summary

| Property | Value |
|----------|-------|
| Update frequency | Every 500ms (2 ticks/second) |
| Price distribution | Lognormal (via GBM) |
| Prices go negative? | No (exp() is always positive) |
| Correlation | Cholesky-decomposed sector-based matrix |
| Random events | ~0.1% per tick per ticker (every ~50s across 10 tickers) |
| Event magnitude | 2-5% sudden move (up or down) |
| Dynamic tickers | Supported — Cholesky rebuilt on add/remove |
| Seed prices | Realistic values for 10 default tickers |
| Unknown ticker seed | Random $50-$300 |
| Cholesky rebuild cost | O(n^2), negligible for n < 50 |

---

## Testing

17 unit tests for `GBMSimulator` + 10 integration tests for `SimulatorDataSource` in `backend/tests/market/`:

- Prices remain positive after many steps
- Correlated tickers actually correlate (statistical test)
- Random events occur at expected frequency
- Add/remove ticker works correctly
- Cholesky matrix is rebuilt after ticker changes
- SimulatorDataSource writes to PriceCache correctly
- Background task starts and stops cleanly

Run tests:
```bash
cd backend
uv run --extra dev pytest tests/market/test_simulator.py tests/market/test_simulator_source.py -v
```

---

## Demo

A Rich terminal demo is available:

```bash
cd backend
uv run market_data_demo.py
```

Displays a live-updating dashboard with all 10 tickers, sparklines, color-coded direction arrows, and an event log for notable price moves. Runs 60 seconds or until Ctrl+C.
