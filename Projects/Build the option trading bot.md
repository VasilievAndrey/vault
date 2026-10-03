# Options Trading Platform: Plan and Microservice Architecture

I'm not a financial advisor, and this is an engineering design, not trading advice. Short-premium strategies like iron condors can lose money quickly in volatile markets, so your phased approach (backtest, then paper trade, then small live) is the right one.

## 1. Key decisions to settle first

**Historical options data is your biggest constraint.** Backtest quality depends almost entirely on it.

- Alpaca's historical options data only goes back to roughly early 2024, which is too short to cover multiple volatility regimes.
- For a serious backtest, plan on a paid vendor such as ThetaData, Polygon, ORATS, CBOE DataShop, or OptionMetrics. You need historical chains with bid/ask quotes, IV, greeks, and open interest, plus earnings dates and dividends.
- Backtest on bid/ask (or mid with a slippage model), never on last-trade prices. Wide spreads on 4-leg orders are where most backtests become fantasies.

**Broker capabilities need checking before you design execution.**

- Alpaca supports multi-leg options orders (up to 4 legs) as a single order, which suits iron condors and double calendars. It requires options approval level 3 and offers paper trading.
- Moomoo's OpenAPI requires the local OpenD gateway process. Please verify its current support for multi-leg or combo orders. If it only supports single-leg orders, you would have to leg in, which creates legging risk, so I'd start with Alpaca as the primary broker.
- Both brokers sit behind one broker-agnostic interface, so strategies never know which broker they trade through.

**Strategy definitions** (defaults to test, all parameterized):

||Iron condor|Double calendar|
|---|---|---|
|Structure|Short put spread plus short call spread, same expiry|Short near-term and long far-term at both a put strike and a call strike|
|Edge|Sell rich IV, collect theta|Long vega, front-month decay, term-structure steepness|
|Max loss|Width minus credit (defined)|Debit paid (defined)|
|Entry|High IV rank, 30-45 DTE, short strikes around 10-20 delta|Low-to-moderate IV with upward-sloping term structure, front expiry about 7-30 DTE|
|Exit|50% of max profit, 21 DTE, or loss at 1.5-2x credit|20-30% of debit as profit, before front expiry, or on a stop|
|Avoid|Earnings or events inside the window|Earnings inside the window, except deliberate earnings plays|

Both strategies have defined risk, which makes the risk-budget sizing clean.

## 2. Architecture overview

```
                 ┌────────────┐      ┌─────────────────────┐
  Vendors/Brokers│ Market Data│─────▶│ Data Lake / TimeSeries│◀───────────┐
                 │ Ingestion  │      └──────────┬──────────┘            │
                 └────────────┘                 │                        │
                                   ┌────────────▼───────────┐    ┌──────┴───────┐
                                   │ Analytics (IV, greeks, │    │  Backtesting │
                                   │ IVR, term structure)   │    │    Engine    │
                                   └────────────┬───────────┘    └──────┬───────┘
 ┌──────────┐   ┌─────────┐   ┌─────────────────▼───┐   ┌───────────────▼──┐
 │ Universe │──▶│ Ranker  │──▶│  Strategy Service   │◀──│ Portfolio & Risk │
 │ Scanner  │   │ (Top 10)│   │ (entry/exit signals)│   │ (budget, sizing) │
 └──────────┘   └─────────┘   └─────────┬───────────┘   └───────▲──────────┘
                                        │ order intents         │
                                ┌───────▼───────┐        ┌──────┴───────┐
                                │ Order Mgmt    │───────▶│ Position &   │
                                │ Service (OMS) │        │ Exit Monitor │
                                └───────┬───────┘        └──────────────┘
                              ┌─────────┴─────────┐
                        ┌─────▼─────┐       ┌─────▼─────┐
                        │  Alpaca   │       │  Moomoo   │
                        │  Gateway  │       │  Gateway  │
                        └───────────┘       └───────────┘
   Cross-cutting: Orchestrator/Scheduler · API Gateway + UI · Notifications · Observability · Config/Secrets
```

**Communication:** an event bus (Kafka, Redpanda, or NATS JetStream) for market data, signals, order events, and fills. gRPC or REST for request/response calls. **Storage:** Postgres for transactional state, TimescaleDB or ClickHouse for time series, and Parquet on object storage for the historical lake. **Language:** Python is the pragmatic choice for most services, with Rust or Go optional later if latency matters (it likely won't for these strategies).

## 3. Microservices and requirements

### 3.1 Market Data Ingestion

- Pull underlying quotes, option chains, and quotes/greeks from the vendor and from Alpaca (and Moomoo if desired).
- Normalize to one schema (OCC symbol, expiry, strike, right, bid, ask, size, IV, greeks, OI, timestamp).
- Handle rate limits, reconnects, and gap detection and backfill.
- Publish to the bus and persist to the time-series store.
- _Non-functional:_ idempotent writes, data-quality checks (crossed quotes, stale quotes), and source tagging.

### 3.2 Reference and Events Service

- Earnings calendar, ex-dividend dates, splits and corporate actions, market holidays, and early closes.
- Provides "event inside window?" queries to the scanner and strategies.
- Handles adjusted contracts (non-standard options after splits or special dividends), which should be excluded by default.

### 3.3 Historical Data Service (the data lake)

- Bulk-load and version historical chains, underlying prices, and events.
- Point-in-time correctness: no survivorship bias (include delisted tickers) and no look-ahead.
- Query API by date range, symbol, and expiry with fast columnar access.
- Data-quality reports (missing days, bad IV).

### 3.4 Analytics Service

- Compute IV (use the vendor's, or your own Black-Scholes/Bjerksund-Stensland), greeks, IV rank/percentile, realized vs. implied volatility, term structure slope, skew, and expected move.
- A shared library used identically by live trading and the backtester, so there are no logic differences between the two.
- Cache results per symbol and timestamp.

### 3.5 Universe Scanner (liquidity)

- Daily and intraday scan of the whole market down to a tradable universe.
- Underlying filters: average daily volume, price range, market cap, no halts.
- Options filters: open interest at target strikes, option volume, bid-ask spread as a percent of mid (for example, under 5-10% for the 4-leg package), and number of listed expiries and strikes.
- Output: a liquidity-scored universe with the reasons for any exclusions (auditable).

### 3.6 Opportunity Ranker (Top 10)

- For each strategy, build candidate structures on each universe symbol, then score them.
- Score components: liquidity, IV rank (condor) or term-structure (calendar), expected-move vs. strike placement, estimated probability of profit, risk/reward, events exclusion, and **fit to risk budget**.
- Budget fit means the max loss per contract set (one condor or calendar) must fit within the per-trade risk allocation. This naturally favors lower-priced underlyings on smaller accounts.
- Diversification constraints: sector caps, correlation limits, and no duplicate exposure.
- Output: a ranked Top 10 per strategy, with every score input stored for explainability.

### 3.7 Strategy Service

- Defines each strategy as a plugin with the same interface: `build_candidates()`, `entry_signal()`, `exit_signal()`, `adjust()` (optional).
- **Entry timing:** the rules engine decides when to enter, for example IV rank thresholds, time-of-day windows (avoiding the open and close for spread quality), a minimum spread-quality check, and trend or volatility filters. Start with simple rules, then test improvements in the backtester rather than assuming them.
- **Exit timing:** profit target, stop loss, DTE threshold, delta breach of short strikes, IV collapse, and event proximity. Exits are evaluated on every position update.
- The same code runs in backtest, paper, and live (this is the single most important design principle).

### 3.8 Portfolio and Risk Service

- Tracks net liquidation value from all brokers and applies the **risk budget as a percent of current portfolio value**, recalculated live. Suggested layers:
    - Total capital at risk across all open positions (for example, 5-10% of NLV).
    - Max loss per trade (for example, 0.5-2%).
    - Per-symbol and per-sector caps.
    - Portfolio greek limits (net delta, vega, and theta/gamma exposure).
- Position sizing: `contracts = floor(per_trade_risk_budget / max_loss_per_contract)`, and reject if the result is zero.
- Pre-trade checks: buying power, margin impact, concentration, and open-order exposure.
- **Kill switches:** daily loss limit, max drawdown halt, and a manual global halt.
- Veto power: no order reaches the OMS without risk approval.

### 3.9 Order Management Service (OMS)

- Order lifecycle state machine: created, risk-approved, submitted, partial, filled, canceled, rejected, expired.
- Smart execution for spreads: start at mid, then walk the price toward the natural price in steps within a max-slippage limit, with a timeout and cancel/replace logic.
- Idempotent order IDs, reconciliation against broker state, and handling of partial fills.
- Leg-by-leg fallback only where a broker lacks combo orders (with explicit legging-risk limits).
- Full audit log of every state change.

### 3.10 Broker Gateway Services (Alpaca and Moomoo adapters)

- One common interface: `place_order`, `cancel`, `get_positions`, `get_account`, `get_orders`, and a streaming order and fill feed.
- Alpaca: REST and websocket, paper and live environments.
- Moomoo: wraps the OpenD gateway (it must run as a managed sidecar, with unlock-trade and reconnect handling).
- Per-broker capability flags (combo orders supported, supported order types, trading hours).
- Environment switch (paper vs. live) enforced at the configuration level, with a hard guard against accidental live trading.

### 3.11 Position and Exit Monitor

- Continuously re-prices open positions, computes P&L and greeks, and evaluates exit rules from the Strategy Service.
- Emits exit intents to the OMS and alerts on assignment risk (short options near ITM close to expiry or ex-dividend) and pin risk.
- Reconciles internal positions with broker positions on a schedule.

### 3.12 Backtesting Engine

- Event-driven replay over the historical lake, using the same Scanner, Ranker, Strategy, and Risk code paths as live trading (injected via a simulated clock and a simulated broker).
- Fill model: mid plus a configurable slippage (percent of spread), per-leg commissions and fees, and bid/ask-based exits.
- Handles early assignment approximations, expiration settlement, and dividends.
- Outputs: equity curve, CAGR, Sharpe/Sortino, max drawdown, win rate, average win/loss, profit factor, expectancy, tail loss, greeks over time, and results by market regime, by symbol, and by DTE/delta bucket.
- Parameter sweeps and walk-forward testing with out-of-sample splits (to guard against overfitting), plus a side-by-side strategy comparison.
- Reproducibility: every run stores its config, data version, and code version.

### 3.13 Paper Trading Mode

- Not a separate service: the live pipeline pointed at the paper broker environments. Required as a gate between backtest and live, to validate fills against your slippage assumptions.

### 3.14 Orchestrator / Scheduler

- Runs the daily workflow: pre-market scan, ranking, intraday entry-window checks, monitoring, end-of-day reconciliation, and report.
- Workflow engine such as Temporal, Airflow, or Prefect, with retries and market-calendar awareness.

### 3.15 API Gateway and Web UI

- Dashboards: universe and Top 10, open positions and greeks, risk budget usage, order blotter, and backtest results and comparison.
- Controls: strategy enable/disable, parameter editing (versioned), halt button, and paper/live toggle with confirmation.
- Authentication and role-based access.

### 3.16 Notification Service

- Alerts via Telegram, Slack, or email for fills, rejects, risk breaches, kill switch events, data outages, and daily summaries.

### 3.17 Platform Services

- **Config and Secrets:** API keys in a vault (never in code) and versioned strategy configs.
- **Observability:** structured logs, metrics, tracing, and health checks. Alert on stale data, since trading on stale quotes is a classic failure.
- **Audit and Compliance log:** immutable record of decisions and the inputs behind them.

## 4. Phased roadmap

1. **Foundations (weeks 1-3):** data vendor choice, historical ingestion, Reference Service, Analytics library (IV, greeks, IVR).
2. **Backtester MVP (weeks 3-7):** simulated broker, fill model, and iron condor first, then the double calendar. Reports and metrics.
3. **Scanner, Ranker, Risk (weeks 6-10):** liquidity universe, Top 10 ranking, budget-based sizing, all running inside the backtester so the whole selection process is tested historically.
4. **Strategy tuning (weeks 9-12):** entry and exit timing studies, walk-forward validation, regime analysis. Decide which strategies, if any, are worth deploying.
5. **Live-data and paper trading (weeks 12-16):** ingestion, OMS, Alpaca gateway, monitor, UI, and alerts, running on paper for at least 1-2 months.
6. **Small live (after that):** minimal risk budget (for example, 0.5-1% total), add the Moomoo gateway, scale up only if live results match paper and backtest within tolerance.

## 5. Biggest risks to design against

- **Overfitting and optimistic fills** in the backtest, mitigated by walk-forward testing, conservative slippage, and a paper-trading gate.
- **Early assignment, pin risk, and dividend risk** on short legs.
- **Gap and volatility-spike losses** (a defined-risk structure caps loss, but it can still hit max loss repeatedly).
- **Stale or bad data** triggering bad trades. Block trading on failed data checks.
- **Duplicate or runaway orders.** Idempotency, order-rate limits, and a kill switch address this.

---

If you'd like, I can turn this into a Word or PDF design document, or go deeper on a specific piece, such as the event schemas between services, the Top 10 scoring formulas, or the backtester's fill model. Which would be most useful next?