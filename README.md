# Crypto_Trading
hackathon project
# TRUMP Flow & Market Inefficiency Detection

A research prototype that tests whether large, publicly observable on-chain capital flows in the $TRUMP ecosystem produce temporary, tradeable dislocations between spot and perpetual futures markets.

Backtesting and paper trading only. No capital is deployed and no live orders are placed.

## Hypothesis

```
Large on-chain capital flow
        ↓
Change in market demand/supply
        ↓
Temporary price/liquidity imbalance
        ↓
Spot / perpetual divergence
        ↓
Convergence
        ↓
Market-neutral trading opportunity
```

When a large flow is detected, the system checks whether the perpetual trades at a premium or discount to spot. If the expected convergence exceeds total modelled cost, it simulates a market-neutral position — long spot, short perpetual, or the reverse — and holds until the basis converges.

## Token

The official token is on Solana at:

```
6p6xgHyF7AeE6TZkSmFsko444wqoP15icUSqi2jfGiPN
```

Many near-identical "TrumpCoin" mints exist with negligible liquidity and near-total insider concentration. This address is defined once in `config.py` and referenced everywhere. Do not hardcode it elsewhere.

## Tech stack

| Layer | Tool | Role |
|---|---|---|
| Language | Python 3.11 | Single runtime for ingestion, analysis and app |
| Environment | uv (or venv + pip) | Shared lockfile, reproducible across four machines |
| Market data | ccxt | Unified historical OHLCV, funding and order books across venues |
| Market data | httpx | Hyperliquid POST /info and anything ccxt does not cover |
| Market data | websockets, asyncio | Live mark price, trades and funding streams |
| On-chain data | Dune Analytics | SQL over indexed Solana transfers for the historical bulk pull |
| On-chain data | Helius | Parsed transfers, enhanced transactions, live websocket subscriptions |
| On-chain data | Bitquery | GraphQL fallback if Dune coverage is thin |
| On-chain data | Solscan | Manual address labelling and verification, UI only |
| Storage | pandas, pyarrow | Dataframes in, Parquet out |
| Storage | DuckDB | Query Parquet directly if frames outgrow memory |
| Analysis | numpy, scipy.stats | Thresholds, control sampling, two-sample tests |
| Charts | plotly | Interactive charts reused in the dashboard |
| Charts | matplotlib | Static exports for slides |
| Strategy | pandas, numpy | Vectorised entry and exit logic, trade ledger |
| Live output | rich | Readable terminal panel for the live demo |
| Presentation | Streamlit | Dashboard over the same dataframes |
| Reliability | tenacity | Retry with backoff on rate-limited bulk pulls |
| Secrets | python-dotenv | API keys out of version control |

`requirements.txt`:

```
ccxt
httpx
websockets
pandas
pyarrow
duckdb
numpy
scipy
plotly
matplotlib
streamlit
rich
tenacity
python-dotenv
dune-client
```

## Repository structure

```
trump-flow/
├── README.md
├── requirements.txt
├── .env.example
├── .gitignore
├── config.py
├── queries/
│   └── trump_transfers.sql
├── ingest/
│   ├── __init__.py
│   ├── market.py
│   ├── chain.py
│   └── labels.py
├── features.py
├── events.py
├── costs.py
├── backtest.py
├── live.py
├── app.py
├── notebooks/
│   ├── 01_market_sanity.ipynb
│   ├── 02_flow_exploration.ipynb
│   └── 03_event_study.ipynb
└── data/
    ├── raw/
    │   ├── market/
    │   └── chain/
    ├── processed/
    │   └── timeline.parquet
    └── output/
        ├── events.csv
        └── trades.csv
```

## What goes in each file

### config.py

Every constant the project depends on, so no magic numbers live in analysis code. A judge asking "what if fees were double" should be answered by editing one line here.

Contains: mint address, venue list and symbols, event window bounds, entry and exit basis thresholds, maximum hold time, fee rates, slippage coefficient, data directory paths.

Imports nothing from the project. Everything else imports it.

### queries/trump_transfers.sql

The Dune query for the historical bulk pull. Kept in the repo rather than only in the Dune UI so the extraction is reproducible and reviewable.

Returns: block time, signature, source address, destination address, amount, mint.

### ingest/labels.py

The labelled address set built by hand in Phase 2 and the classification function that uses it.

Contains: dictionaries of treasury and unlock wallets, known Solana exchange deposit addresses, and major DEX pool addresses. Exposes `classify_transfer(src, dst)` returning one of `to_exchange`, `from_exchange`, `treasury_out`, `pool_add`, `pool_remove`, `wallet_to_wallet`.

This file is where manual research becomes code. Cite a source in a comment for every labelled address.

### ingest/market.py

Fetches and normalises all exchange data.

Reads: Binance and Hyperliquid public endpoints.
Writes: `data/raw/market/*.parquet`.
Output schema: `venue, instrument, ts, open, high, low, close, volume` for prices, plus a separate frame of `venue, ts, funding_rate` and one of `venue, ts, open_interest`.

Handles pagination, rate limits and UTC normalisation. Nothing downstream should ever touch an exchange API directly.

### ingest/chain.py

Fetches SPL transfers for the mint and classifies them.

Reads: Dune export or Helius for history, `ingest/labels.py` for classification.
Writes: `data/raw/chain/transfers.parquet`.
Output schema: `ts, signature, src, dst, amount_tokens, amount_usd, flow_type`.

### features.py

The analytical core, and the most important file in the repo. Pure functions only — no file reads, no API calls, no globals.

Takes aligned dataframes and returns the derived series: basis, annualised funding, rolling net exchange flow, flow as share of circulating float, rolling volume, and OI delta.

Also contains the timeline alignment step: resample to one-minute UTC bars, forward-fill funding between settlements, left-join chain flows onto the bar index.

Writes: `data/processed/timeline.parquet`.

Imported by `events.py`, `backtest.py` and `live.py`. The fact that the live script computes its signals through this same module is the main architectural claim of the project — do not let a second copy of the basis calculation appear anywhere.

### events.py

Turns continuous flow data into a discrete event table, and generates the matched control set.

Reads: `data/processed/timeline.parquet`.
Writes: `data/output/events.csv` with schema `ts, type, size_usd, direction, wallet, source`, and a control file of the same shape with randomly sampled non-event timestamps.

Thresholds come from `config.py` and are frozen before Phase 5 runs.

### costs.py

One function per cost component, each returning a number in USD or basis points.

Contains: `taker_fee(notional, venue)`, `funding_cost(rate, notional, hours_held)`, `slippage(notional, recent_volume)`, `gas_cost(n_transactions)`, and a `total_cost(trade)` that sums them.

No dependencies beyond `config.py`. Deliberately simple so it can be read aloud during a demo.

### backtest.py

Applies the entry and exit rules across the event table and produces the trade ledger.

Reads: `data/output/events.csv`, `data/processed/timeline.parquet`, `costs.py`.
Writes: `data/output/trades.csv` with schema `event_ts, entry_ts, exit_ts, direction, notional, gross_pnl, fees, funding, slippage, net_pnl`, plus an equity curve series.

Also computes the summary block shown in the demo: hit rate, mean hold time, total net PnL, worst trade.

### live.py

A single async script proving the same signal logic runs on live data.

Subscribes to Binance mark-price and trade streams, the Hyperliquid websocket, and Helius transfer subscriptions on the mint. Maintains a rolling in-memory buffer, passes it to `features.py`, and renders live basis, live funding and current signal state to the terminal.

Never places an order. There is no execution path in this codebase.

### app.py

The Streamlit dashboard, read-only over files already produced.

Four views: the event table, the dislocation chart for a selected event with the control overlay, the trade ledger with net PnL, and a live tab showing the feed.

Contains no analysis. If a number appears here it was computed upstream.

### notebooks/

Exploratory work, kept separate from the modules so nothing in the pipeline depends on a notebook. `01` checks market data sanity after Phase 1, `02` explores flow distributions to set thresholds in Phase 4, `03` produces the event-study chart for the pitch in Phase 5.

## Data flow

```
ingest/market.py ─┐
                  ├─→ features.py ─→ events.py ─→ backtest.py ─→ app.py
ingest/chain.py ──┘        │                          ↑
        ↑                  │                      costs.py
   labels.py               │
                           └─────────────→ live.py
```

Each arrow is a file on disk, not an in-memory handoff. Any stage can be rerun independently, which matters when one pipeline breaks at 3am and the other three people need to keep working.

## Setup

```bash
git clone <repo-url>
cd trump-flow
uv venv && source .venv/bin/activate
uv pip install -r requirements.txt
cp .env.example .env    # add HELIUS_API_KEY, DUNE_API_KEY
```

Verify exchange API reachability from your network before committing to a venue:

```bash
curl -s "https://api.binance.com/api/v3/ping"
```

Some exchange APIs geo-block by region. Bybit and Hyperliquid are drop-in fallbacks.

## Build phases

### Phase 0 — Lock the target, venues and rules

Decide what counts as a signal before looking at any results. Write the event window, entry threshold and exit rule into the repo and do not change them after Phase 5 without disclosing it.

Agree the shared dataframe schema in hour one: column names, UTC-only timestamps, one-minute bars.

Stack: Python 3.11, uv or venv + pip, GitHub, `config.py`, python-dotenv

### Phase 1 — Market data

Pull Binance spot 1m klines, perp klines, mark price, premium index and full funding-rate history since the January 2025 listing. Hyperliquid exposes funding history and candle snapshots through a single unauthenticated POST endpoint.

Normalise into long format: `venue, instrument, ts, open, high, low, close, volume`.

Binance serves roughly 30 days of open interest history. Either treat OI as a live-signal input only, or source it from a third-party aggregator. Decide in this phase, not in Phase 5.

Stack: ccxt, httpx, pandas, pyarrow, tenacity

### Phase 2 — On-chain data

Build a labelled address set first: treasury and unlock wallets tied to the insider allocation, known Solana exchange deposit addresses, and the largest DEX pools.

Pull SPL token transfers for the mint across full history. Classify each transfer as to-exchange, from-exchange, treasury-outbound, pool-add, pool-remove or wallet-to-wallet.

Split tooling by job: Dune for the historical bulk pull (one SQL query, CSV export, no pagination code), Helius for the live path in Phase 8. Backfilling eighteen months over raw RPC will consume a full day.

Stack: Dune Analytics, Helius, Bitquery (fallback), Solscan (manual verification), pandas

### Phase 3 — Timeline alignment

Resample to one-minute UTC bars. Forward-fill funding between eight-hour settlements. Left-join chain events onto the bar index.

Derive the features everything downstream reads:

- basis = `(perp - spot) / spot`
- funding, annualised
- rolling net exchange flow
- flow as share of circulating float
- rolling volume

Keep these as pure functions with no I/O in `features.py`.

Stack: pandas, DuckDB (if frames outgrow memory)

### Phase 4 — Event detection

An event is a threshold crossing computable on any historical bar and any live tick, not a narrative about a whale.

Candidate definitions:

- transfer above the 99th percentile of trailing 30-day transfers
- net exchange inflow exceeding a set share of circulating float within one hour
- a scheduled unlock timestamp

Emit one table: `ts, type, size_usd, direction, wallet, source`. Generate a matched control set of random timestamps with no event — without it you cannot claim post-event behaviour is distinctive.

Freeze the threshold before running Phase 5. Tuning it after seeing which events produced good dislocations is the failure mode reviewers probe for.

Stack: pandas, numpy

### Phase 5 — Dislocation measurement

For each event, extract basis, funding and OI over the window, indexed to event time. Average across events and plot against the control average with a confidence band.

Run a two-sample test on peak basis, event versus control. Report sample size honestly.

If there is no separation between event and control groups, that is a real finding. Report it rather than retuning.

Stack: pandas, plotly, matplotlib, scipy.stats

### Phase 6 — Cost model

Model every cost that eats the edge:

- taker fees on both legs
- funding paid or received per 8h period held
- slippage as a function of order size against recent volume
- Solana transaction cost where a DEX leg is involved

Every value is a named constant so a single edit reruns the whole analysis under different assumptions.

Slippage is estimated from volume, not measured from historical order books. State this explicitly.

Stack: plain Python, ccxt (live order book to calibrate the slippage coefficient)

### Phase 7 — Backtest

Entry: at event + k minutes, if basis exceeds threshold and expected convergence beats total modelled cost, open long spot / short perp or the reverse.

Exit: basis inside the exit band, maximum hold reached, or stop triggered.

Output a per-trade ledger — entry, exit, gross, fees, funding, net — plus equity curve, hit rate, mean hold time and worst trade. The ledger is more persuasive than any summary statistic.

Do not adopt Backtrader or Zipline. Setup cost exceeds the benefit at this scope.

Stack: pandas, numpy, vectorbt (optional, only if already familiar)

### Phase 8 — Live data path

Demonstrates that the architecture is not bound to static files. Not a production pipeline.

One async script subscribing to Binance mark-price and trade streams, Hyperliquid websocket, and Helius transfer subscriptions on the mint. Ticks feed into the same `features.py` the backtest uses, printing live basis, live funding and current signal state.

Stack: websockets, asyncio, ccxt.pro (alternative), Helius websockets, rich

### Phase 9 — Presentation

One Streamlit app with four views: event table, dislocation chart for a selected event, trade ledger with net PnL, and a live tab showing the feed ticking.

Run locally rather than deploying. Record a fallback screen capture the night before.

Stack: Streamlit, plotly

## Ownership

| Role | Phases | Primary stack |
|---|---|---|
| Market data | 1, market half of 8 | ccxt, httpx, websockets |
| On-chain data | 2, Solana half of 8 | Dune, Helius, pandas |
| Events and analysis | 4, 5 | pandas, scipy, plotly |
| Strategy and output | 6, 7, 9 | numpy, Streamlit |

All four converge on the Phase 3 schema in hour one so the data tracks can run in parallel without blocking.

## Cut list

Drop from the top if running behind.

| Drop | Because |
|---|---|
| Hyperliquid as second venue | Binance alone is enough to show a basis |
| Open interest as a feature | Basis and funding carry the argument |
| DEX pool liquidity data | Exchange transfer flows are the cleaner signal |
| Live on-chain feed | A live market feed still proves the architecture |
| Streamlit dashboard | A well-organised notebook demos fine |
| Statistical testing | Show the event chart against the control |

## Limitations

- Small sample. Eighteen months of token history with a handful of genuinely large flow events gives limited statistical power.
- No historical order-book depth. Slippage is estimated from volume rather than measured.
- Regime concentration. Launch and scheduled unlocks dominate the event set, so results may not generalise to ordinary market conditions.
- No execution latency modelled. A real deployment would compete with market makers reading the same chain data.
- Backtest and paper trading only.

## License

MIT