# Market Data Backend — Implementation Design

Complete implementation design for the FinAlly market data subsystem. All code lives in `backend/app/market/`. This document covers every module with working code snippets, explains the design decisions, and shows how to integrate the subsystem into the broader FastAPI application.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [File Structure](#2-file-structure)
3. [Data Model — `models.py`](#3-data-model)
4. [Price Cache — `cache.py`](#4-price-cache)
5. [Abstract Interface — `interface.py`](#5-abstract-interface)
6. [Seed Prices & Ticker Parameters — `seed_prices.py`](#6-seed-prices--ticker-parameters)
7. [GBM Simulator — `simulator.py`](#7-gbm-simulator)
8. [Massive API Client — `massive_client.py`](#8-massive-api-client)
9. [Factory — `factory.py`](#9-factory)
10. [SSE Streaming Endpoint — `stream.py`](#10-sse-streaming-endpoint)
11. [Public API — `__init__.py`](#11-public-api)
12. [FastAPI Lifecycle Integration](#12-fastapi-lifecycle-integration)
13. [Watchlist Coordination](#13-watchlist-coordination)
14. [Frontend SSE Integration](#14-frontend-sse-integration)
15. [Testing Strategy](#15-testing-strategy)
16. [Error Handling & Edge Cases](#16-error-handling--edge-cases)
17. [Configuration Summary](#17-configuration-summary)

---

## 1. Architecture Overview

The market data subsystem follows the **Strategy pattern**: two implementations behind one abstract interface. All downstream code (SSE streaming, portfolio valuation, trade execution) is source-agnostic.

```
MarketDataSource (ABC)
├── SimulatorDataSource  →  GBM simulator (default, no API key required)
└── MassiveDataSource    →  Polygon.io REST poller (when MASSIVE_API_KEY set)
        │
        ▼
   PriceCache (thread-safe, in-memory, single source of truth)
        │
        ├──→ SSE stream endpoint  /api/stream/prices
        ├──→ Portfolio valuation   /api/portfolio
        └──→ Trade execution       /api/portfolio/trade
```

**Key design decisions:**

| Decision | Rationale |
|---|---|
| Abstract interface | Downstream code never depends on the data source — swap simulator for real data by changing one env var |
| PriceCache as single point of truth | Producers write on their schedule; consumers read whenever they need. No direct coupling. |
| Version counter on PriceCache | SSE endpoint only sends a payload when data has changed, avoiding redundant network traffic |
| Thread-safe cache with `Lock` | The Massive client runs blocking API calls via `asyncio.to_thread` — the lock protects against concurrent writes |
| Immediate cache seeding at start | Both sources pre-populate the cache before starting their background loop so the SSE stream has data on the first client connection |

---

## 2. File Structure

```
backend/
  app/
    market/
      __init__.py         # Public API re-exports
      models.py           # PriceUpdate dataclass
      cache.py            # PriceCache (thread-safe in-memory store)
      interface.py        # MarketDataSource ABC
      seed_prices.py      # SEED_PRICES, TICKER_PARAMS, correlation constants
      simulator.py        # GBMSimulator + SimulatorDataSource
      massive_client.py   # MassiveDataSource (Polygon.io REST client)
      factory.py          # create_market_data_source() — env-driven factory
      stream.py           # FastAPI SSE endpoint (create_stream_router)
```

---

## 3. Data Model

**File:** `backend/app/market/models.py`

The `PriceUpdate` dataclass is the **only** data structure that leaves the market data layer. Everything downstream — SSE serialization, portfolio P&L, trade validation — works with `PriceUpdate` objects.

```python
from __future__ import annotations

import time
from dataclasses import dataclass, field


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
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        """Percentage change from previous update."""
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat'."""
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE transmission."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

**Design notes:**
- `frozen=True` — immutable after creation; safe to share across threads without copying
- `slots=True` — faster attribute access and lower memory overhead (Python 3.10+)
- Computed properties (`change`, `change_percent`, `direction`) are derived — no storage overhead
- `to_dict()` produces the exact JSON shape the frontend consumes via SSE

**Example usage:**
```python
update = PriceUpdate(ticker="AAPL", price=191.50, previous_price=190.00)
print(update.direction)       # "up"
print(update.change)          # 1.5
print(update.change_percent)  # 0.7895
print(update.to_dict())
# {"ticker": "AAPL", "price": 191.5, "previous_price": 190.0,
#  "timestamp": 1741000000.0, "change": 1.5, "change_percent": 0.7895, "direction": "up"}
```

---

## 4. Price Cache

**File:** `backend/app/market/cache.py`

The `PriceCache` is the shared in-memory store. Producers (simulator or Massive) write to it; consumers (SSE, portfolio) read from it.

```python
from __future__ import annotations

import time
from threading import Lock

from .models import PriceUpdate


class PriceCache:
    """Thread-safe in-memory cache of the latest price for each ticker.

    Writers: SimulatorDataSource or MassiveDataSource (one at a time).
    Readers: SSE streaming endpoint, portfolio valuation, trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Monotonically increasing; bumped on every update

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price for a ticker. Returns the created PriceUpdate.

        Automatically computes direction and change from the previous price.
        If this is the first update for the ticker, previous_price == price (direction='flat').
        """
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        """Get the latest price for a single ticker, or None if unknown."""
        with self._lock:
            return self._prices.get(ticker)

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all current prices. Returns a shallow copy."""
        with self._lock:
            return dict(self._prices)

    def get_price(self, ticker: str) -> float | None:
        """Convenience: get just the price float, or None."""
        update = self.get(ticker)
        return update.price if update else None

    def remove(self, ticker: str) -> None:
        """Remove a ticker from the cache (e.g., when removed from watchlist)."""
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        """Current version counter. Useful for SSE change detection."""
        return self._version

    def __len__(self) -> int:
        with self._lock:
            return len(self._prices)

    def __contains__(self, ticker: str) -> bool:
        with self._lock:
            return ticker in self._prices
```

**Version counter for SSE change detection:**

The `version` property increments on every `update()` call. The SSE generator compares the current version against the version from the previous loop iteration — it only serializes and sends a payload when data has actually changed:

```python
last_version = -1
while True:
    current_version = price_cache.version
    if current_version != last_version:
        last_version = current_version
        prices = price_cache.get_all()
        # ... send SSE event ...
    await asyncio.sleep(0.5)
```

**Example usage in trade execution:**
```python
# In the trade execution route:
price = price_cache.get_price(ticker)
if price is None:
    raise HTTPException(400, f"No price data for {ticker}")
total_cost = price * quantity
```

---

## 5. Abstract Interface

**File:** `backend/app/market/interface.py`

```python
from __future__ import annotations

from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices —
    it reads from the cache.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])
        # ... app runs ...
        await source.add_ticker("TSLA")
        await source.remove_ticker("GOOGL")
        # ... app shutting down ...
        await source.stop()
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates for the given tickers.

        Starts a background task that periodically writes to the PriceCache.
        Must be called exactly once. Calling start() twice is undefined behavior.
        """

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources.

        Safe to call multiple times. After stop(), the source will not write
        to the cache again.
        """

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present.

        The next update cycle will include this ticker.
        """

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active set. No-op if not present.

        Also removes the ticker from the PriceCache.
        """

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

**Why the interface doesn't return prices:**

The ABC defines how to *control* the data source (start, stop, add/remove tickers), not how to *read* prices. Prices flow through the `PriceCache`. This separation means:
- The SSE stream can read prices at 500ms without coupling to either data source
- Portfolio valuation can call `cache.get_price("AAPL")` without knowing whether it's simulated or real
- Both implementations are interchangeable with zero changes to consuming code

---

## 6. Seed Prices & Ticker Parameters

**File:** `backend/app/market/seed_prices.py`

Constants only — no logic. Imported by the simulator.

```python
# Realistic starting prices for the default watchlist
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

# Per-ticker GBM parameters
# sigma: annualized volatility (higher = more price movement per tick)
# mu: annualized drift / expected return
TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT": {"sigma": 0.20, "mu": 0.05},
    "AMZN": {"sigma": 0.28, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},  # High volatility
    "NVDA": {"sigma": 0.40, "mu": 0.08},  # High volatility, strong drift
    "META": {"sigma": 0.30, "mu": 0.05},
    "JPM": {"sigma": 0.18, "mu": 0.04},  # Low volatility (bank)
    "V": {"sigma": 0.17, "mu": 0.04},    # Low volatility (payments)
    "NFLX": {"sigma": 0.35, "mu": 0.05},
}

# Default parameters for tickers not in the list above (dynamically added)
DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}

# Correlation groups for the simulator's Cholesky decomposition
CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

# Correlation coefficients
INTRA_TECH_CORR = 0.6    # Tech stocks move together
INTRA_FINANCE_CORR = 0.5  # Finance stocks move together
CROSS_GROUP_CORR = 0.3    # Between sectors / unknown tickers
TSLA_CORR = 0.3           # TSLA does its own thing (despite being "tech")
```

**Volatility intuition:**

| Ticker | sigma | Real-world equivalent |
|--------|-------|----------------------|
| V | 0.17 | Stable payments company |
| JPM | 0.18 | Large-cap bank |
| MSFT | 0.20 | Steady tech giant |
| AAPL | 0.22 | Large-cap consumer tech |
| GOOGL | 0.25 | Ad-dependent tech |
| AMZN | 0.28 | E-commerce + cloud mix |
| META | 0.30 | Social media risk premium |
| NFLX | 0.35 | Subscription + content risk |
| NVDA | 0.40 | AI/chip cyclicality |
| TSLA | 0.50 | High-beta momentum stock |

Tickers added dynamically (not in `SEED_PRICES`) start at a random price between $50–$300 and use `DEFAULT_PARAMS` (`sigma=0.25, mu=0.05`).

---

## 7. GBM Simulator

**File:** `backend/app/market/simulator.py`

Two classes: `GBMSimulator` (pure math, no asyncio) and `SimulatorDataSource` (wraps GBMSimulator in an async loop, implements `MarketDataSource`).

### 7.1 GBM Math

Geometric Brownian Motion is the standard model for continuous stock prices (underpins Black-Scholes). At each time step:

```
S(t+dt) = S(t) × exp((μ - σ²/2) × dt + σ × √dt × Z)
```

- `S(t)` — current price
- `μ` — annualized drift (expected return), e.g., 0.05 = 5%/year
- `σ` — annualized volatility, e.g., 0.25 = 25%/year
- `dt` — time step as a fraction of a trading year
- `Z` — standard normal random variable

For 500ms updates over 252 trading days × 6.5 hours/day:
```
dt = 0.5 / (252 × 6.5 × 3600) ≈ 8.48 × 10⁻⁸
```

This tiny `dt` produces sub-cent moves per tick that accumulate naturally over time. GBM is multiplicative (uses `exp()`), so prices can never go negative.

### 7.2 Correlated Moves

Real stocks don't move independently. Tech stocks tend to move together during market-wide events. We use **Cholesky decomposition** to generate correlated random draws:

1. Build an N×N correlation matrix `C` where `C[i,j]` is the pairwise correlation between tickers `i` and `j`
2. Compute the lower Cholesky factor `L = chol(C)` (a lower-triangular matrix)
3. Generate N independent standard normal draws `Z_independent`
4. Transform: `Z_correlated = L @ Z_independent`

The resulting `Z_correlated` draws have the desired correlations. The Cholesky decomposition guarantees the output has the correct covariance structure as long as `C` is positive semi-definite (which it is, given our correlation values are between 0 and 1).

### 7.3 GBMSimulator Implementation

```python
import math
import random
import logging

import numpy as np

from .seed_prices import (
    CORRELATION_GROUPS, CROSS_GROUP_CORR, DEFAULT_PARAMS,
    INTRA_FINANCE_CORR, INTRA_TECH_CORR, SEED_PRICES, TICKER_PARAMS, TSLA_CORR,
)

logger = logging.getLogger(__name__)


class GBMSimulator:
    """Generates correlated GBM price paths for multiple tickers."""

    # 500ms as a fraction of a trading year
    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None:
        self._dt = dt
        self._event_prob = event_probability

        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def step(self) -> dict[str, float]:
        """Advance all tickers by one time step. Returns {ticker: new_price}."""
        n = len(self._tickers)
        if n == 0:
            return {}

        z_independent = np.random.standard_normal(n)

        if self._cholesky is not None:
            z_correlated = self._cholesky @ z_independent
        else:
            z_correlated = z_independent

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            mu = self._params[ticker]["mu"]
            sigma = self._params[ticker]["sigma"]

            # GBM step
            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random event: ~0.1% chance per tick
            # With 10 tickers at 2 ticks/sec → ~1 event every 50 seconds
            if random.random() < self._event_prob:
                shock_magnitude = random.uniform(0.02, 0.05)
                shock_sign = random.choice([-1, 1])
                self._prices[ticker] *= 1 + shock_magnitude * shock_sign
                logger.debug(
                    "Random event on %s: %.1f%% %s",
                    ticker, shock_magnitude * 100,
                    "up" if shock_sign > 0 else "down",
                )

            result[ticker] = round(self._prices[ticker], 2)

        return result

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker. Rebuilds the correlation matrix."""
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Rebuilds the correlation matrix."""
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    def _add_ticker_internal(self, ticker: str) -> None:
        """Add without rebuilding Cholesky (used in batch initialization)."""
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        """Rebuild the Cholesky factor of the correlation matrix.

        Called whenever tickers are added or removed. O(n²) but n < 50.
        """
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return

        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = rho
                corr[j, i] = rho

        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        tech = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]

        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR
```

### 7.4 SimulatorDataSource

Wraps `GBMSimulator` in an async background task and implements `MarketDataSource`:

```python
import asyncio
import logging

from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by the GBM simulator."""

    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
    ) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed cache immediately — SSE has data on the very first client connection
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")
        logger.info("Simulator started with %d tickers", len(tickers))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        logger.info("Simulator stopped")

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

**Example: standalone simulator test**
```python
import asyncio
from app.market import PriceCache, create_market_data_source

async def demo():
    cache = PriceCache()
    source = create_market_data_source(cache)  # Returns SimulatorDataSource (no API key)
    await source.start(["AAPL", "GOOGL", "MSFT"])

    for _ in range(5):
        await asyncio.sleep(0.5)
        aapl = cache.get("AAPL")
        print(f"AAPL: ${aapl.price:.2f} ({aapl.direction})")

    await source.stop()

asyncio.run(demo())
```

---

## 8. Massive API Client

**File:** `backend/app/market/massive_client.py`

The Massive (formerly Polygon.io) client polls the REST API for real market data. It uses the same `MarketDataSource` interface as the simulator.

### 8.1 API Overview

- **Package:** `massive` (install: `uv add massive`)
- **Auth:** `RESTClient(api_key=...)` or reads `MASSIVE_API_KEY` from environment
- **Primary endpoint:** `GET /v2/snapshot/locale/us/markets/stocks/tickers` — all tickers in one call
- **Rate limits:** Free tier = 5 req/min (poll every 15s); Paid = effectively unlimited

### 8.2 MassiveDataSource Implementation

```python
from __future__ import annotations

import asyncio
import logging

from massive import RESTClient
from massive.rest.models import SnapshotMarketType

from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    """MarketDataSource backed by the Massive (Polygon.io) REST API.

    Polls all watched tickers in a single API call every `poll_interval` seconds.

    Rate limits:
      Free tier:  5 req/min → poll every 15s (default)
      Paid tiers: higher → poll every 2–5s
    """

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
    ) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        # Immediate first poll — cache is populated before the loop starts
        await self._poll_once()
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")
        logger.info(
            "Massive poller started: %d tickers, %.1fs interval",
            len(tickers), self._interval,
        )

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    async def _poll_loop(self) -> None:
        """Poll on interval. First poll already happened in start()."""
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        """Execute one poll: fetch snapshots, update cache."""
        if not self._tickers or not self._client:
            return

        try:
            # Massive RESTClient is synchronous — run in a thread to avoid
            # blocking the asyncio event loop.
            snapshots = await asyncio.to_thread(self._fetch_snapshots)

            processed = 0
            for snap in snapshots:
                try:
                    price = snap.last_trade.price
                    # Massive timestamps are Unix milliseconds — convert to seconds
                    timestamp = snap.last_trade.timestamp / 1000.0
                    self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
                    processed += 1
                except (AttributeError, TypeError) as e:
                    logger.warning(
                        "Skipping malformed snapshot for %s: %s",
                        getattr(snap, "ticker", "???"), e,
                    )

            logger.debug(
                "Massive poll: updated %d/%d tickers", processed, len(self._tickers)
            )

        except Exception as e:
            # Don't re-raise — retry on next interval.
            # Common failures: 401 (bad key), 429 (rate limit), network errors.
            logger.error("Massive poll failed: %s", e)

    def _fetch_snapshots(self) -> list:
        """Synchronous Massive API call. Runs in a thread via asyncio.to_thread."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

### 8.3 Response Shape

The Massive snapshot endpoint returns objects with this structure per ticker:

```json
{
  "ticker": "AAPL",
  "last_trade": {
    "price": 191.50,
    "size": 100,
    "exchange": "XNYS",
    "timestamp": 1741000000000
  },
  "day": {
    "open": 189.00,
    "high": 192.00,
    "low": 188.50,
    "close": 191.50,
    "volume": 45000000,
    "previous_close": 190.00,
    "change": 1.50,
    "change_percent": 0.7895
  }
}
```

We use `last_trade.price` as the current price and `last_trade.timestamp` (converted from milliseconds to seconds) as the timestamp. The day-level fields (`day.previous_close`, `day.change_percent`) are available if the frontend needs day change display.

### 8.4 Error Handling for the API Client

| Error | Cause | Behavior |
|-------|-------|----------|
| `401 Unauthorized` | Bad API key | Logs error, skips poll, retries next interval |
| `403 Forbidden` | Feature not on plan | Same |
| `429 Too Many Requests` | Rate limit exceeded | Same — backing off via poll interval |
| `5xx Server Error` | Massive outage | Same — the loop continues |
| Network timeout | Connection issue | Same |
| `AttributeError` on snap | Malformed snapshot | Logs warning, skips that ticker, continues |

The `except Exception` in `_poll_once` is intentional: a failing poll should never crash the entire application. The cache retains the last-known price until the next successful poll.

---

## 9. Factory

**File:** `backend/app/market/factory.py`

```python
from __future__ import annotations

import logging
import os

from .cache import PriceCache
from .interface import MarketDataSource
from .massive_client import MassiveDataSource
from .simulator import SimulatorDataSource

logger = logging.getLogger(__name__)


def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Create the appropriate market data source based on environment variables.

    - MASSIVE_API_KEY set and non-empty → MassiveDataSource (real data)
    - Otherwise → SimulatorDataSource (GBM simulation, default)

    Returns an unstarted source. Caller must await source.start(tickers).
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        logger.info("Market data source: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        logger.info("Market data source: GBM Simulator")
        return SimulatorDataSource(price_cache=price_cache)
```

**Usage:**
```python
# In app startup:
cache = PriceCache()
source = create_market_data_source(cache)
# source is either SimulatorDataSource or MassiveDataSource
# All subsequent calls are identical regardless of which was returned:
await source.start(["AAPL", "GOOGL", "MSFT", ...])
```

---

## 10. SSE Streaming Endpoint

**File:** `backend/app/market/stream.py`

```python
from __future__ import annotations

import asyncio
import json
import logging
from collections.abc import AsyncGenerator

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse

from .cache import PriceCache

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/stream", tags=["streaming"])


def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Create the SSE streaming router with a reference to the price cache.

    Factory pattern: injects PriceCache via closure instead of global state.
    """

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        """SSE endpoint — GET /api/stream/prices

        The client connects with EventSource and receives events:

            data: {"AAPL": {"ticker": "AAPL", "price": 190.50, ...}, ...}

        'retry: 1000' tells the browser to reconnect after 1 second if
        the connection drops. EventSource handles this automatically.
        """
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",  # Disable nginx buffering if proxied
            },
        )

    return router


async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    """Async generator that yields SSE-formatted price events.

    Uses version-based change detection: only serializes and sends
    a payload when the cache has been updated since the last send.
    Stops when the client disconnects.
    """
    yield "retry: 1000\n\n"

    last_version = -1
    client_ip = request.client.host if request.client else "unknown"
    logger.info("SSE client connected: %s", client_ip)

    try:
        while True:
            if await request.is_disconnected():
                logger.info("SSE client disconnected: %s", client_ip)
                break

            current_version = price_cache.version
            if current_version != last_version:
                last_version = current_version
                prices = price_cache.get_all()

                if prices:
                    data = {ticker: update.to_dict() for ticker, update in prices.items()}
                    yield f"data: {json.dumps(data)}\n\n"

            await asyncio.sleep(interval)

    except asyncio.CancelledError:
        logger.info("SSE stream cancelled for: %s", client_ip)
```

**SSE event format:**

Each event is a single `data:` line followed by two newlines (`\n\n`):

```
data: {"AAPL":{"ticker":"AAPL","price":191.50,"previous_price":190.00,"timestamp":1741000000.0,"change":1.5,"change_percent":0.7895,"direction":"up"},"GOOGL":{"ticker":"GOOGL",...}}

```

The `retry: 1000` directive at the start tells the browser to wait 1000ms before reconnecting if the connection drops. The `EventSource` API handles reconnection automatically — the client writes no reconnect logic.

**Why version-based change detection:**

Without it, the server sends a new JSON payload every 500ms regardless of whether prices changed. With 10 tickers, that's ~10KB of JSON per second per client. Version-based detection skips sends when no updates arrived (e.g., during market close if using Massive). For the simulator (always running), this is less impactful but still the correct pattern.

---

## 11. Public API

**File:** `backend/app/market/__init__.py`

```python
"""Market data subsystem for FinAlly.

Public API:
    PriceUpdate              - Immutable price snapshot dataclass
    PriceCache               - Thread-safe in-memory price store
    MarketDataSource         - Abstract interface for data providers
    create_market_data_source - Factory (selects simulator or Massive)
    create_stream_router     - FastAPI router factory for SSE endpoint
"""

from .cache import PriceCache
from .factory import create_market_data_source
from .interface import MarketDataSource
from .models import PriceUpdate
from .stream import create_stream_router

__all__ = [
    "PriceUpdate",
    "PriceCache",
    "MarketDataSource",
    "create_market_data_source",
    "create_stream_router",
]
```

Downstream code only needs this import:

```python
from app.market import PriceCache, PriceUpdate, MarketDataSource, create_market_data_source
```

---

## 12. FastAPI Lifecycle Integration

The market data system starts during FastAPI's lifespan and shuts down cleanly on exit.

```python
# backend/app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

from app.market import PriceCache, create_market_data_source, create_stream_router
from app.db import init_db, get_watchlist_tickers  # DB init + watchlist query


# Shared state — created at startup, used for the lifetime of the process
price_cache: PriceCache | None = None
market_source = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    """FastAPI lifespan context: startup → yield → shutdown."""
    global price_cache, market_source

    # 1. Initialize database (creates schema + seeds default data if needed)
    await init_db()

    # 2. Load initial watchlist from DB
    tickers = await get_watchlist_tickers()  # ["AAPL", "GOOGL", ...]

    # 3. Start market data source
    price_cache = PriceCache()
    market_source = create_market_data_source(price_cache)
    await market_source.start(tickers)

    yield  # Application is running

    # 4. Shutdown: cancel background tasks
    if market_source:
        await market_source.stop()


app = FastAPI(title="FinAlly", lifespan=lifespan)

# Register the SSE streaming router
# (done after price_cache is created — but we use a lambda for late binding)
# Better pattern: register in lifespan after price_cache is ready:
#   app.include_router(create_stream_router(price_cache))
# For simplicity, create_stream_router uses a closure that captures price_cache.

# Serve the Next.js static export
app.mount("/", StaticFiles(directory="static", html=True), name="static")
```

**Dependency injection for API routes:**

To make `price_cache` and `market_source` available to route handlers without globals, use FastAPI's dependency injection:

```python
from fastapi import Depends


def get_price_cache() -> PriceCache:
    """Dependency that returns the global price cache."""
    assert price_cache is not None, "Price cache not initialized"
    return price_cache


def get_market_source() -> MarketDataSource:
    assert market_source is not None, "Market source not initialized"
    return market_source


# Usage in a route:
@app.get("/api/portfolio")
async def get_portfolio(cache: PriceCache = Depends(get_price_cache)):
    prices = cache.get_all()
    # ... compute positions P&L using prices ...
```

**Watchlist routes must also update the market source:**

```python
@app.post("/api/watchlist")
async def add_to_watchlist(
    body: AddTickerRequest,
    source: MarketDataSource = Depends(get_market_source),
    # ... db session ...
):
    ticker = body.ticker.upper().strip()
    # 1. Add to DB watchlist
    await db_add_ticker(ticker)
    # 2. Add to market data source (simulator or Massive will start tracking it)
    await source.add_ticker(ticker)
    return {"ticker": ticker}


@app.delete("/api/watchlist/{ticker}")
async def remove_from_watchlist(
    ticker: str,
    source: MarketDataSource = Depends(get_market_source),
):
    ticker = ticker.upper()
    await db_remove_ticker(ticker)
    await source.remove_ticker(ticker)  # Also removes from PriceCache
    return {"ticker": ticker}
```

---

## 13. Watchlist Coordination

The market data source tracks exactly the tickers in the user's watchlist. The two are kept in sync:

```
DB watchlist table
      │
      │  on startup: load all tickers
      ▼
MarketDataSource.start(tickers)
      │
      │  on POST /api/watchlist
      ▼
MarketDataSource.add_ticker(ticker)  →  PriceCache updated on next tick

      │  on DELETE /api/watchlist/{ticker}
      ▼
MarketDataSource.remove_ticker(ticker)  →  PriceCache entry deleted immediately
```

**Simulator behavior on dynamic watchlist changes:**

- `add_ticker("PYPL")`: GBMSimulator seeds PYPL at a random price ($50–$300), rebuilds the Cholesky correlation matrix to include PYPL at `CROSS_GROUP_CORR` with existing tickers. Next `step()` call includes PYPL.
- `remove_ticker("NFLX")`: GBMSimulator drops NFLX from internal state, rebuilds Cholesky. PriceCache removes NFLX immediately. SSE stream stops sending NFLX on the next event.

**Massive behavior on dynamic changes:**

- `add_ticker("PYPL")`: Added to `self._tickers` list. Appears in the next scheduled poll (up to 15 seconds later on free tier). No immediate cache update.
- `remove_ticker("NFLX")`: Removed from `self._tickers`. PriceCache entry deleted immediately. The next poll fetches snapshots without NFLX.

---

## 14. Frontend SSE Integration

The frontend connects to the SSE endpoint using the native `EventSource` API (no libraries needed):

```typescript
// frontend/src/hooks/usePriceStream.ts
import { useEffect, useRef, useState } from "react";

export interface PriceData {
  ticker: string;
  price: number;
  previous_price: number;
  timestamp: number;
  change: number;
  change_percent: number;
  direction: "up" | "down" | "flat";
}

export type PriceMap = Record<string, PriceData>;

type ConnectionStatus = "connecting" | "connected" | "reconnecting" | "disconnected";

export function usePriceStream() {
  const [prices, setPrices] = useState<PriceMap>({});
  const [status, setStatus] = useState<ConnectionStatus>("connecting");
  const esRef = useRef<EventSource | null>(null);

  useEffect(() => {
    const connect = () => {
      const es = new EventSource("/api/stream/prices");
      esRef.current = es;

      es.onopen = () => setStatus("connected");

      es.onmessage = (event) => {
        try {
          const update: PriceMap = JSON.parse(event.data);
          setPrices((prev) => ({ ...prev, ...update }));
        } catch {
          console.error("Failed to parse SSE event", event.data);
        }
      };

      es.onerror = () => {
        setStatus("reconnecting");
        // EventSource auto-reconnects after the retry: 1000 directive
        // No manual reconnect logic needed
      };
    };

    connect();

    return () => {
      esRef.current?.close();
    };
  }, []);

  return { prices, status };
}
```

**Price flash animation trigger:**

```typescript
// Detect direction changes for CSS flash effect
useEffect(() => {
  Object.entries(prices).forEach(([ticker, data]) => {
    if (data.direction !== "flat") {
      const el = document.getElementById(`price-${ticker}`);
      if (el) {
        el.classList.remove("flash-up", "flash-down");
        // Force reflow to restart animation
        void el.offsetHeight;
        el.classList.add(data.direction === "up" ? "flash-up" : "flash-down");
      }
    }
  });
}, [prices]);
```

```css
/* CSS flash animations */
@keyframes flashUp {
  0%   { background-color: rgba(34, 197, 94, 0.4); }  /* green */
  100% { background-color: transparent; }
}
@keyframes flashDown {
  0%   { background-color: rgba(239, 68, 68, 0.4); }  /* red */
  100% { background-color: transparent; }
}
.flash-up   { animation: flashUp   500ms ease-out forwards; }
.flash-down { animation: flashDown 500ms ease-out forwards; }
```

---

## 15. Testing Strategy

Tests live in `backend/tests/market/`. Run with:

```bash
cd backend
uv run --extra dev pytest tests/market/ -v
uv run --extra dev pytest tests/market/ --cov=app/market --cov-report=term-missing
```

### 15.1 Model Tests (`test_models.py`)

```python
from app.market.models import PriceUpdate

def test_direction_up():
    u = PriceUpdate(ticker="AAPL", price=191.0, previous_price=190.0)
    assert u.direction == "up"
    assert u.change == 1.0
    assert u.change_percent == pytest.approx(0.5263, rel=1e-3)

def test_direction_flat():
    u = PriceUpdate(ticker="AAPL", price=190.0, previous_price=190.0)
    assert u.direction == "flat"
    assert u.change == 0.0

def test_to_dict_keys():
    u = PriceUpdate(ticker="AAPL", price=191.0, previous_price=190.0)
    d = u.to_dict()
    assert set(d.keys()) == {
        "ticker", "price", "previous_price", "timestamp",
        "change", "change_percent", "direction",
    }

def test_frozen():
    u = PriceUpdate(ticker="AAPL", price=191.0, previous_price=190.0)
    with pytest.raises((AttributeError, TypeError)):
        u.price = 200.0  # Must be immutable
```

### 15.2 Cache Tests (`test_cache.py`)

```python
from app.market.cache import PriceCache

def test_first_update_is_flat():
    cache = PriceCache()
    update = cache.update("AAPL", 190.0)
    assert update.direction == "flat"
    assert update.previous_price == update.price

def test_version_increments():
    cache = PriceCache()
    v0 = cache.version
    cache.update("AAPL", 190.0)
    assert cache.version == v0 + 1
    cache.update("AAPL", 191.0)
    assert cache.version == v0 + 2

def test_remove():
    cache = PriceCache()
    cache.update("AAPL", 190.0)
    assert "AAPL" in cache
    cache.remove("AAPL")
    assert "AAPL" not in cache
    assert cache.get("AAPL") is None

def test_thread_safety():
    """Multiple threads writing concurrently should not corrupt state."""
    import threading
    cache = PriceCache()
    errors = []

    def writer(ticker: str):
        try:
            for i in range(100):
                cache.update(ticker, float(100 + i))
        except Exception as e:
            errors.append(e)

    threads = [threading.Thread(target=writer, args=(f"T{i}",)) for i in range(10)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    assert not errors
    assert len(cache) == 10
```

### 15.3 Simulator Tests (`test_simulator.py`)

```python
from app.market.simulator import GBMSimulator

def test_prices_never_negative():
    sim = GBMSimulator(["AAPL", "TSLA", "NVDA"])
    for _ in range(1000):
        prices = sim.step()
        for ticker, price in prices.items():
            assert price > 0, f"{ticker} went non-positive: {price}"

def test_add_remove_ticker():
    sim = GBMSimulator(["AAPL"])
    sim.add_ticker("GOOGL")
    prices = sim.step()
    assert "AAPL" in prices
    assert "GOOGL" in prices

    sim.remove_ticker("AAPL")
    prices = sim.step()
    assert "AAPL" not in prices
    assert "GOOGL" in prices

def test_full_10_ticker_cholesky():
    """Cholesky decomposition must succeed for the full default watchlist."""
    tickers = ["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA", "NVDA", "META", "JPM", "V", "NFLX"]
    sim = GBMSimulator(tickers)
    # If Cholesky fails (non-PSD matrix), __init__ would raise LinAlgError
    prices = sim.step()
    assert len(prices) == 10

def test_seed_prices_used():
    from app.market.seed_prices import SEED_PRICES
    sim = GBMSimulator(["AAPL"])
    assert sim.get_price("AAPL") == pytest.approx(SEED_PRICES["AAPL"], rel=0.01)
```

### 15.4 SimulatorDataSource Integration Tests (`test_simulator_source.py`)

```python
import asyncio
import pytest
from app.market.cache import PriceCache
from app.market.simulator import SimulatorDataSource

@pytest.mark.asyncio
async def test_start_seeds_cache():
    cache = PriceCache()
    source = SimulatorDataSource(cache)
    await source.start(["AAPL", "GOOGL"])
    # Cache should be pre-populated immediately after start
    assert cache.get_price("AAPL") is not None
    assert cache.get_price("GOOGL") is not None
    await source.stop()

@pytest.mark.asyncio
async def test_prices_update_over_time():
    cache = PriceCache()
    source = SimulatorDataSource(cache, update_interval=0.05)
    await source.start(["AAPL"])

    initial = cache.get_price("AAPL")
    await asyncio.sleep(0.2)
    # After a few ticks, version should have incremented
    assert cache.version > 1

    await source.stop()

@pytest.mark.asyncio
async def test_add_remove_ticker():
    cache = PriceCache()
    source = SimulatorDataSource(cache)
    await source.start(["AAPL"])

    await source.add_ticker("TSLA")
    assert cache.get_price("TSLA") is not None

    await source.remove_ticker("AAPL")
    assert cache.get("AAPL") is None

    await source.stop()
```

### 15.5 Massive Client Tests (`test_massive.py`)

```python
import asyncio
from unittest.mock import AsyncMock, MagicMock, patch
import pytest
from app.market.cache import PriceCache
from app.market.massive_client import MassiveDataSource


def make_snapshot(ticker: str, price: float, timestamp_ms: int = 1741000000000):
    snap = MagicMock()
    snap.ticker = ticker
    snap.last_trade.price = price
    snap.last_trade.timestamp = timestamp_ms
    return snap


@pytest.mark.asyncio
async def test_poll_updates_cache():
    cache = PriceCache()
    source = MassiveDataSource(api_key="test", price_cache=cache)

    snapshots = [
        make_snapshot("AAPL", 191.50),
        make_snapshot("GOOGL", 176.00),
    ]

    with patch.object(source, "_fetch_snapshots", return_value=snapshots):
        await source._poll_once()

    assert cache.get_price("AAPL") == pytest.approx(191.50)
    assert cache.get_price("GOOGL") == pytest.approx(176.00)


@pytest.mark.asyncio
async def test_malformed_snapshot_skipped():
    cache = PriceCache()
    source = MassiveDataSource(api_key="test", price_cache=cache)
    source._tickers = ["AAPL", "BAD"]

    bad_snap = MagicMock()
    bad_snap.ticker = "BAD"
    bad_snap.last_trade = None  # Will cause AttributeError

    good_snap = make_snapshot("AAPL", 191.50)

    with patch.object(source, "_fetch_snapshots", return_value=[good_snap, bad_snap]):
        await source._poll_once()  # Should not raise

    assert cache.get_price("AAPL") is not None
    assert cache.get_price("BAD") is None


@pytest.mark.asyncio
async def test_poll_failure_does_not_raise():
    cache = PriceCache()
    source = MassiveDataSource(api_key="test", price_cache=cache)
    source._tickers = ["AAPL"]

    with patch.object(source, "_fetch_snapshots", side_effect=Exception("Network error")):
        await source._poll_once()  # Must not raise

    assert cache.get_price("AAPL") is None  # Cache unchanged
```

### 15.6 Factory Tests (`test_factory.py`)

```python
import os
from unittest.mock import patch
from app.market.factory import create_market_data_source
from app.market.cache import PriceCache
from app.market.simulator import SimulatorDataSource
from app.market.massive_client import MassiveDataSource


def test_no_api_key_returns_simulator():
    cache = PriceCache()
    with patch.dict(os.environ, {}, clear=True):
        os.environ.pop("MASSIVE_API_KEY", None)
        source = create_market_data_source(cache)
    assert isinstance(source, SimulatorDataSource)


def test_api_key_set_returns_massive():
    cache = PriceCache()
    with patch.dict(os.environ, {"MASSIVE_API_KEY": "test-key-123"}):
        source = create_market_data_source(cache)
    assert isinstance(source, MassiveDataSource)


def test_empty_api_key_returns_simulator():
    cache = PriceCache()
    with patch.dict(os.environ, {"MASSIVE_API_KEY": ""}):
        source = create_market_data_source(cache)
    assert isinstance(source, SimulatorDataSource)


def test_whitespace_api_key_returns_simulator():
    cache = PriceCache()
    with patch.dict(os.environ, {"MASSIVE_API_KEY": "   "}):
        source = create_market_data_source(cache)
    assert isinstance(source, SimulatorDataSource)
```

---

## 16. Error Handling & Edge Cases

### Empty Watchlist

Both sources handle an empty ticker list gracefully:

```python
# GBMSimulator.step() — returns {} immediately
if n == 0:
    return {}

# MassiveDataSource._poll_once() — returns immediately
if not self._tickers or not self._client:
    return
```

The SSE stream handles an empty cache by skipping the send:

```python
if prices:
    yield f"data: {json.dumps(data)}\n\n"
```

### Ticker Not in Cache

Trade execution routes must handle the case where a price isn't available yet:

```python
price = cache.get_price(ticker)
if price is None:
    raise HTTPException(
        status_code=400,
        detail=f"No price data available for {ticker}. "
               "Try again in a moment or add it to your watchlist first."
    )
```

### First Connection

Both data sources seed the cache before starting their background loop:

- Simulator: calls `cache.update(ticker, price)` for each ticker in `start()` before `create_task()`
- Massive: calls `await self._poll_once()` in `start()` before `create_task()`

This ensures the SSE stream sends real data on the very first event, with no empty/null state for the frontend to handle.

### Simulator Tick During Ticker Removal

Since `SimulatorDataSource._run_loop` and `add_ticker`/`remove_ticker` are all coroutines on the same event loop, Python's cooperative multitasking prevents true concurrent access to `self._sim`. No additional locking is needed in the simulator.

However, `PriceCache` is accessed from both the asyncio loop (writes) and potentially sync code (reads in other threads), hence the `Lock`.

### Massive Poller During Market Close

Outside US market hours, `last_trade.price` in the snapshot reflects the last traded price (may be from after-hours). This is fine for FinAlly — we display the most recent price available. The cache simply retains the last-known price.

---

## 17. Configuration Summary

| Environment Variable | Default | Effect |
|---|---|---|
| `MASSIVE_API_KEY` | (empty) | If set and non-empty, uses Massive REST API. Otherwise uses GBM simulator. |
| `LLM_MOCK` | `false` | No effect on market data; affects LLM chat endpoint only. |

**Simulator tuning** (no env vars — modify code constants if needed):

| Constant | Location | Default | Effect |
|---|---|---|---|
| `update_interval` | `SimulatorDataSource.__init__` | `0.5` | Seconds between price ticks |
| `event_probability` | `GBMSimulator.__init__` | `0.001` | Probability of a random 2–5% shock per ticker per tick |
| `DEFAULT_DT` | `GBMSimulator` | `~8.48e-8` | Time step (fraction of trading year for 500ms ticks) |
| `INTRA_TECH_CORR` | `seed_prices.py` | `0.6` | Correlation between tech stocks |
| `INTRA_FINANCE_CORR` | `seed_prices.py` | `0.5` | Correlation between finance stocks |
| `CROSS_GROUP_CORR` | `seed_prices.py` | `0.3` | Cross-sector and unknown ticker correlation |

**Massive poller tuning:**

| Parameter | Default | Notes |
|---|---|---|
| `poll_interval` | `15.0` seconds | Free tier safe. Paid tier: reduce to 2–5s. |

---

## Summary

The complete market data backend in `backend/app/market/` provides:

1. **`PriceUpdate`** — immutable dataclass that all downstream code works with
2. **`PriceCache`** — thread-safe shared memory; single source of truth for prices
3. **`MarketDataSource`** — abstract interface; downstream code is source-agnostic
4. **`SimulatorDataSource`** — GBM with correlated moves; no external dependencies; always available
5. **`MassiveDataSource`** — Polygon.io REST client; selected via `MASSIVE_API_KEY`
6. **`create_market_data_source()`** — env-driven factory; one call selects the right implementation
7. **`create_stream_router()`** — FastAPI SSE endpoint; version-based change detection; auto-reconnect

All components are independently testable. The system integrates into FastAPI via the `lifespan` context manager. The frontend connects with native `EventSource` — no additional libraries needed.
