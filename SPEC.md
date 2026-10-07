# Stock Market Analysis & Backtesting Platform: Product + Technical Spec

Version 0.1. Offline-first build, AWS/Git-ready.

---

## 0. Recommended stack and project layout

Build offline now, deploy later without rewrites. Choices that keep both paths open:

| Layer | Choice | Why |
|---|---|---|
| Language | TypeScript everywhere | Shared types between UI, API, and engine |
| Frontend | Next.js (App Router) + Tailwind + shadcn/ui | You already have a landing page; reuse its styles |
| Charts | TradingView **Lightweight Charts** (candles) + Recharts/ECharts (equity curves, heatmaps) | Free, fast, license-friendly |
| API | Next.js route handlers or a separate Fastify service (`apps/api`) | Fastify if you want the engine isolated early |
| Engine | Pure TS package (`packages/engine`), no I/O, runs in a worker thread | Deterministic, testable, portable to Lambda/Fargate |
| DB | PostgreSQL (Docker locally → RDS later) | Relational fits strategies/backtests/billing |
| Bars storage | Parquet/CSV files locally → Postgres/Timescale or S3+Parquet later | Cheap, swappable behind the provider interface |
| Queue | BullMQ + Redis (Docker) → SQS/ElastiCache later | Backtests are async jobs |
| Auth | Auth.js (NextAuth) with credentials + Google | Swap to Cognito later if desired |
| Billing | Stripe (test mode works offline via Stripe CLI) | Plans, trials, webhooks |
| Infra later | Terraform; ECS Fargate (web, api, worker), RDS, ElastiCache, S3, CloudFront, Secrets Manager | |
| CI/CD later | GitHub Actions → ECR → ECS | |

Monorepo:

```
/apps/web            Next.js UI
/apps/api            REST/tRPC API (optional split)
/apps/worker         Backtest/simulation job runner
/packages/engine     Backtest engine (pure functions + event loop)
/packages/indicators SMA, EMA, RSI, MACD, ATR, BBands, VWAP, etc.
/packages/market-data  MarketDataProvider interface + providers
/packages/shared     Zod schemas, DTOs, enums
/packages/db         Prisma/Drizzle schema + migrations
/infra               docker-compose.yml now, Terraform later
/docs                this spec, ADRs
```

Local dev: `docker compose up` gives Postgres + Redis; `pnpm dev` runs everything.

---

## 1. Module-by-module spec (12 modules)

### M1. Auth & Onboarding
**Pages:** `/signup`, `/login`, `/forgot-password`, `/verify-email`, `/onboarding`
**Controls:** email/password, Google OAuth, TOTP 2FA (later), "experience level" select, "markets of interest" chips, accept Terms + risk disclaimer checkbox (required).
**Data:** user, email_verified_at, onboarding_state, accepted_terms_version.
**Rules:** rate-limit login; password ≥ 10 chars; session cookie httpOnly; new users land in a seeded sample strategy and sample backtest so the first screen is never empty.

### M2. Dashboard
**Widgets:** recent backtests (name, symbol, return %, Sharpe, date), saved strategies, open paper positions, equity summary of paper account, watchlist mini-quotes, plan usage meter (backtests used / limit), "Start here" checklist (run first backtest, build strategy, try replay).
**Controls:** quick-run bar (symbol + template + date range → Run), "New strategy" button.
**Data points:** latest N backtests, paper account equity, watchlist last/Δ%, usage counters.

### M3. Charts
**Page:** `/charts/:symbol`
**Controls:**
- Symbol search (autocomplete), timeframe (1m, 5m, 15m, 1h, 1D, 1W), range presets (1D, 5D, 1M, 6M, 1Y, 5Y, Max), chart type (candle, Heikin-Ashi, line, bar).
- Indicator panel: add/remove overlays (SMA, EMA, BBands, VWAP, Supertrend) and panes (RSI, MACD, Stoch, ATR, Volume), each with editable params and color.
- Drawing tools: trendline, horizontal line, rectangle, fib retracement, text (stored per user per symbol).
- Overlays: **earnings dates** (E markers with EPS surprise tooltip), dividends, splits, backtest trade markers (toggle).
- Compare symbol (normalized % overlay), log/linear scale, crosshair OHLCV legend, save layout, fullscreen.
**Data:** OHLCV bars, corporate actions, earnings events, user layouts.

### M4. Strategy Builder
**Page:** `/strategies`, `/strategies/:id/edit`
**Modes:** (a) Visual rule builder, (b) Template picker, (c) Code/JSON view (power users, later).
**Sections:**
1. **Instrument:** equity/ETF now; options legs later. Position side: long, short, both.
2. **Entry rules:** condition groups (AND/OR). Each condition = `[left operand] [operator] [right operand]`.
   Operands: price (open/high/low/close), indicator(params), constant, previous-bar value (`[n]`), time of day, day-of-week, earnings-in-N-days.
   Operators: `>`, `<`, `>=`, `<=`, `crosses above`, `crosses below`, `between`.
3. **Exit rules:** signal exit (same condition builder), stop loss (% / ATR× / fixed $), take profit (% / ATR× / R-multiple), trailing stop (% / ATR), time stop (N bars / end of day), break-even move.
4. **Position sizing:** fixed shares, fixed $, % of equity, risk % per trade (uses stop distance), volatility-targeted.
5. **Filters:** trading window (time of day), days of week, max trades/day, max concurrent positions, skip earnings week, minimum price/volume.
6. **Costs:** commission model (per share / per trade / % / none), slippage (bps / ticks / % of ATR), SEC/TAF fee toggle.
**Controls:** name, description, tags, duplicate, version history (each save creates a version), validate button (shows errors inline), "Run backtest" button, share-as-read-only link (later).
**Templates (seed):** SMA crossover, RSI mean reversion, Bollinger bounce, Opening Range Breakout, Donchian breakout, MACD trend, Gap fill, Buy & Hold benchmark.
**Data:** strategy → strategy_version with `definition` JSON (see §3).

### M5. Single-Symbol Backtester
**Page:** `/backtests/new`, `/backtests/:id`
**Inputs:** strategy (+ version), symbol, timeframe, start/end date, starting capital, benchmark (SPY default), costs override, data adjustment (split+dividend adjusted default), "run in background" automatic.
**Result page tabs:**
- **Summary:** headline cards: Net P&L, Return %, CAGR, Max drawdown, Sharpe, Sortino, Calmar, Win rate, Profit factor, Expectancy (R), # trades, Avg hold, Exposure %, vs Benchmark.
- **Equity & Drawdown:** equity curve with benchmark overlay, underwater plot, log/linear toggle.
- **Trades:** sortable table (entry/exit time, side, qty, entry/exit price, P&L $, P&L %, R-multiple, MAE, MFE, bars held, exit reason). Click a row → chart jumps to trade with markers.
- **Analytics:** monthly returns heatmap, return distribution histogram, P&L by weekday/hour, win/loss streaks, rolling Sharpe.
- **Chart:** price chart with entries/exits and indicators.
- **Settings & Log:** frozen copy of strategy definition, parameters, engine version, data version, warnings (e.g., "gaps in data on 3 days").
**Actions:** export trades CSV, export report PDF (later), save, rename, delete, "Optimize" (parameter sweep; Phase 3), "Walk-forward" (Phase 3), "Send to simulator".
**Limits:** per-plan max date range, max runs/month, intraday history length.

### M6. Basket Backtester
**Page:** `/baskets`, `/baskets/:id`, `/backtests/basket/new`
**Basket builder:** name, symbols list (paste/CSV import/search), preset universes (S&P 100, Nasdaq 100, sector ETFs, Dow 30), weights (equal, custom %, inverse volatility, market-cap), max 50 symbols (plan-gated).
**Run inputs:** strategy, basket, date range, capital, **allocation mode** (equal split per symbol; shared pool with max concurrent positions; risk-parity), rebalance frequency (none/monthly/quarterly), correlation cap (later).
**Results:**
- Portfolio equity curve + per-symbol contribution (stacked area).
- Table per symbol: trades, win rate, return, max DD, contribution %.
- Correlation matrix heatmap of per-symbol strategy returns.
- Diversification metrics: portfolio Sharpe vs average single-symbol Sharpe, DD reduction, concentration (HHI).
- "Drop worst N" / "Only top N" what-if buttons (clearly labeled as in-sample).
**Engine note:** shared-capital portfolio requires a portfolio-level event loop (§4.7).

### M7. Replay Simulator (Paper Trading on Historical Data)
**Page:** `/simulator`, `/simulator/:sessionId`
**Session setup:** symbol(s), start date/time (or "random day"), timeframe, starting cash, speed.
**Playback controls:** play/pause, step 1 bar, step N bars, speed (1×, 2×, 5×, 10×, max), jump to next day/open/close, restart.
**Trading panel:** Buy/Sell market, limit, stop, stop-limit, bracket (SL/TP), quantity, % of cash, flatten, reverse. Order ticket validates buying power.
**Panels:** chart (no future bars rendered), position table (qty, avg, unrealized P&L), open orders (cancel/modify), fills log, account stats (cash, equity, day P&L), notes box.
**Rules:** hide future data strictly (engine API only exposes bars ≤ cursor); fills simulated with same slippage model as backtester.
**End of session:** summary + added to journal; "Compare to strategy" overlays what a strategy would have done.
**Later:** live forward-testing mode (same UI, real-time bars, §6).

### M8. Option Chain
**Page:** `/options/:symbol`
**Controls:** underlying search, expiry tabs/selector, strikes range (ATM ±N), calls/puts/both layout (straddle view), moneyness highlighting, filter (min OI, min volume, delta range), columns picker.
**Columns:** bid, ask, last, mid, volume, OI, IV, delta, gamma, theta, vega, rho, break-even, % OTM, probability ITM.
**Extras:** IV smile/term-structure chart, expected move (ATM straddle), max pain, put/call ratio, "add leg to strategy" click-to-build.
**Data:** chain snapshot per (underlying, as-of). Simulated chain via Black-Scholes from historical vol offline; real data later via provider.
**Note:** historical option backtesting requires historical chains (paid data; Phase 4).

### M9. Option Strategy / Payoff & Margin Calculator
**Page:** `/options/builder`
**Controls:** leg table (buy/sell, call/put, strike, expiry, qty), templates (covered call, vertical spreads, iron condor, straddle, strangle, butterfly, calendar, cash-secured put).
**Outputs:** payoff diagram at expiry + T+0 curve, slider for underlying price/date/IV, Greeks totals, max profit/loss, break-evens, probability of profit, **margin/buying-power estimate** (Reg-T rules for defined-risk spreads, naked options approximations; portfolio margin out of scope), credit/debit, commissions.
**Disclaimer:** margin is an estimate; broker's requirement governs.

### M10. Portfolio & Trade Journal
**Page:** `/portfolio`, `/journal`
**Portfolio:** accounts (paper, manual, later broker-synced), positions, allocation donut, sector exposure, realized/unrealized P&L, equity history.
**Journal:** entry per trade (auto-created from simulator/forward test or manual): symbol, setup tag, thesis, screenshot upload, emotion/rating, mistakes tag, outcome.
**Analytics:** P&L by tag, by weekday/hour, expectancy by setup, discipline score, calendar heatmap, streaks.
**Controls:** filters, CSV import (broker statements), export.

### M11. Billing & Subscription
**Page:** `/pricing`, `/settings/billing`
**Plans (suggested):**

| | Free | Pro | Elite |
|---|---|---|---|
| Backtests / month | 20 | 500 | Unlimited |
| History depth | 2y daily | 10y daily + 2y intraday | Max |
| Basket size | 5 | 25 | 100 |
| Saved strategies | 3 | 50 | Unlimited |
| Simulator | 3 sessions/mo | Unlimited | Unlimited |
| Options | View only | Chain + calculator | + option backtests |
| Forward testing | – | 1 strategy | 10 |

**Controls:** monthly/annual toggle (annual discount), upgrade/downgrade, payment method (Stripe Checkout + Customer Portal), invoices list, cancel (end of period), promo code, usage meters, trial (14 days, no card).
**Rules:** entitlements derived from plan; enforced server-side in a single `checkEntitlement(user, feature)` function; Stripe webhooks are the source of truth (`checkout.session.completed`, `customer.subscription.updated/deleted`, `invoice.payment_failed`).

### M12. Settings, Admin & Platform
**User settings:** profile, timezone, default benchmark, default costs, theme, API keys (later), notification prefs, delete account/export data.
**Admin (internal):** user search, plan override, job queue view, failed backtests, data-health dashboard (missing bars, stale symbols), feature flags, audit log.
**Platform:** notifications (email on long backtest done), help/docs, changelog, legal pages (Terms, Privacy, risk disclosure).

---

## 2. Five end-to-end workflows

### W1. First backtest (new user, under 2 minutes)
1. Landing page → **Start free** → `/signup` → enter email/password → **Create account**.
2. Verify email link → `/onboarding`: pick experience level, accept risk disclaimer → **Continue**.
3. Dashboard shows checklist; click **Run your first backtest** (quick-run bar prefilled: `SPY`, template "SMA 50/200 crossover", last 5 years, $100,000).
4. Click **Run**. UI posts `POST /backtests`, receives `{id, status:"queued"}`, navigates to `/backtests/:id`.
5. Page shows progress bar (polling or SSE: queued → running 40% → complete).
6. Result Summary tab renders. Tooltip hints on each metric.
7. Click **Trades** tab, click a row, chart tab highlights that trade.
8. Banner: "Like it? Save as a strategy and tweak the rules" → **Open in builder** (leads to W2).

### W2. Build a custom strategy
1. Sidebar → **Strategies** → **New strategy** → choose **Blank** (or a template).
2. Name it "RSI Pullback in Uptrend". Set instrument type: Equity, side: Long.
3. **Entry** section → **Add condition group (AND)**:
   a. `Close > SMA(200)`; b. `RSI(14) crosses below 30`.
4. **Exit** → **Add signal exit**: `RSI(14) crosses above 55`. Add **Stop loss**: 2 × ATR(14). Add **Time stop**: 15 bars.
5. **Sizing** → Risk % per trade → 1%.
6. **Filters** → trading days Mon–Fri; skip earnings week = on.
7. **Costs** → commission $0, slippage 2 bps.
8. Click **Validate** → inline checks pass (or errors list with jump links).
9. Click **Save** (creates version v1). Click **Run backtest** → modal: symbol `AAPL`, 2015–2025, $50k → **Run**.
10. Review results; click **Edit strategy**, change RSI threshold to 25, save (v2), re-run; **Compare runs** to see v1 vs v2 side by side.

### W3. Replay paper trading
1. Sidebar → **Simulator** → **New session**.
2. Choose symbol `TSLA`, timeframe 5m, "Random day from last 2 years" (or pick date), cash $25,000, speed 5×.
3. **Start** → chart loads up to 09:30 bar, future hidden.
4. Click **Play**. Price replays. At a setup, press **Pause**.
5. Order ticket: **Buy**, type **Bracket**, qty 50 via "% of cash 20%", stop −1%, target +2% → **Submit**. Order fills next bar open + slippage; position row appears.
6. Step forward with **Next bar** or play on. Either bracket leg triggers, or click **Flatten**.
7. Write a note in the notes panel; tag the trade "ORB failed".
8. At end of day (or **End session**) → summary modal: P&L, trades, R-multiples → **Save to journal**.
9. Optional: **Compare to strategy** → pick strategy → overlay markers where it would have traded.

### W4. Basket diversification
1. Sidebar → **Baskets** → **New basket**. Name "Mega-cap Tech + Defensive".
2. Add symbols: type/paste `AAPL, MSFT, NVDA, GOOGL, AMZN, JNJ, PG, KO, XLU, XLP` → **Add**. Weights: Equal.
3. **Save basket**. Click **Run basket backtest**.
4. Select strategy "SMA 50/200", date range 2015–2025, capital $100k, allocation "Shared pool, max 5 concurrent", rebalance none → **Run**.
5. Results: portfolio equity curve vs SPY; per-symbol contribution table.
6. Open **Correlation** tab: spot highly correlated pairs (e.g., AAPL/MSFT).
7. Click **Diversification** card: compare portfolio Sharpe/DD vs average single-symbol.
8. Click **What-if: drop worst 2** → re-run → compare; a warning label states this is in-sample and prone to overfit.
9. **Save** results; export CSV.

### W5. Subscription upgrade
1. User on Free hits a limit (e.g., tries a 6th symbol in a basket) → modal "Basket limit reached on Free" → **See plans**.
2. `/pricing` → toggle **Annual** → click **Upgrade to Pro**.
3. `POST /billing/checkout` returns Stripe Checkout URL → redirected → enters card → **Subscribe**.
4. Stripe redirects to `/billing/success?session_id=…`; page shows "Finalizing…" and polls `/me` until plan = Pro (webhook updates DB).
5. Entitlements refresh; toast "You're on Pro". Previously blocked action becomes available; user returns to the basket automatically.
6. Later: `/settings/billing` → **Manage subscription** (Stripe Portal) to change plan, update card, view invoices, cancel.
7. Failure path: `invoice.payment_failed` → banner "Update payment method", 7-day grace, then downgrade to Free (data kept, over-limit items read-only).

---

## 3. Data model

Postgres. `id` = UUID/ULID, all tables have `created_at`, `updated_at`.

**users**: id, email (unique), password_hash, name, timezone, email_verified_at, onboarding_state, accepted_terms_version, role (`user|admin`), stripe_customer_id, status.

**plans**: id, code (`free|pro|elite`), name, price_monthly_cents, price_annual_cents, stripe_price_id_monthly, stripe_price_id_annual, entitlements JSONB (e.g. `{"backtests_per_month":500,"max_basket_size":25,"max_history_years":10,"intraday":true,"options":"chain"}`).

**subscriptions**: id, user_id, plan_id, status (`trialing|active|past_due|canceled`), stripe_subscription_id, current_period_start/end, cancel_at_period_end, trial_end.

**usage_counters**: user_id, period (YYYY-MM), metric (`backtests`, `sim_sessions`), count. Unique (user_id, period, metric).

**strategies**: id, user_id, name, description, tags[], current_version_id, archived_at.

**strategy_versions**: id, strategy_id, version_no, definition JSONB, schema_version, created_at.
`definition` shape (abridged):
```json
{
  "instrument": {"type": "equity", "side": "long"},
  "entry": {"op": "AND", "conditions": [
    {"left": {"ind": "close"}, "cmp": ">", "right": {"ind": "sma", "params": {"period": 200}}},
    {"left": {"ind": "rsi", "params": {"period": 14}}, "cmp": "crosses_below", "right": {"const": 30}}
  ]},
  "exit": {
    "signal": {"op": "AND", "conditions": []},
    "stop": {"type": "atr", "mult": 2, "period": 14},
    "target": {"type": "r_multiple", "value": 3},
    "trailing": null,
    "time_stop_bars": 15
  },
  "sizing": {"type": "risk_pct", "value": 1.0},
  "filters": {"days": [1,2,3,4,5], "window": ["09:45","15:45"], "skip_earnings_days": 5, "max_concurrent": 1},
  "costs": {"commission": {"type": "per_share", "value": 0.0}, "slippage": {"type": "bps", "value": 2}}
}
```

**baskets**: id, user_id, name, description.
**basket_items**: basket_id, symbol, weight (nullable), position.

**backtests**: id, user_id, strategy_version_id, kind (`single|basket`), basket_id (nullable), symbols[], timeframe, start_date, end_date, initial_capital, benchmark_symbol, params_override JSONB, status (`queued|running|complete|failed|canceled`), progress, engine_version, data_version, error, started_at, finished_at.

**backtest_results**: backtest_id (PK), metrics JSONB, equity_curve_ref (JSON in DB if small, else S3 key), drawdown_ref, monthly_returns JSONB, per_symbol JSONB, warnings JSONB.

**backtest_trades**: id, backtest_id, symbol, side, entry_time, entry_price, exit_time, exit_price, qty, pnl, pnl_pct, r_multiple, mae, mfe, bars_held, exit_reason, fees, slippage_cost. Index (backtest_id, entry_time).

**sim_sessions**: id, user_id, symbols[], timeframe, start_ts, cursor_ts, speed, starting_cash, status, rng_seed, ended_at.
**sim_orders**: id, session_id, symbol, side, type, qty, limit_price, stop_price, parent_order_id (brackets), status, created_ts (sim time), filled_ts, fill_price.
**sim_fills / paper_trades**: id, session_id (or account_id), order_id, symbol, side, qty, price, fees, ts.

**accounts**: id, user_id, kind (`paper|manual|broker`), name, base_currency, broker_connection_id (nullable).
**positions**: account_id, symbol, qty, avg_price, updated_at.
**journal_entries**: id, user_id, trade_ref (nullable FK to paper/backtest trade), symbol, setup_tag, thesis, notes, rating, mistakes[], screenshot_url, opened_at, closed_at, pnl.

**watchlists / watchlist_items**: standard.
**chart_layouts**: user_id, symbol (nullable), json (indicators, drawings).
**forward_tests** (Phase 3): id, user_id, strategy_version_id, symbols, status, started_at, last_bar_ts, account_id.
**audit_log**: actor_id, action, entity, entity_id, meta JSONB.

Market data (separate schema or files): `symbols`, `bars_1d`, `bars_intraday` (partitioned by month), `corporate_actions` (splits, dividends), `earnings_events` (symbol, report_ts, eps_est, eps_actual, session `bmo|amc`), `option_chains` (later).

---

## 4. Backtest engine design

Package: `packages/engine`. Pure, deterministic: same inputs → same outputs. No DB or network calls inside; the worker loads data and passes it in.

### 4.1 Inputs/outputs
```ts
runBacktest(input: {
  strategy: StrategyDefinition,
  universe: SymbolBars[],          // loaded via MarketDataProvider
  config: { start, end, capital, benchmark?, seed },
  calendar: TradingCalendar,
}): { trades: Trade[], equity: EquityPoint[], metrics: Metrics, warnings: string[] }
```

### 4.2 Preprocessing
1. Align bars to the exchange calendar; flag gaps/halts.
2. Apply split/dividend adjustment consistently (store raw and adjusted; engine uses adjusted for signals, raw price × adj factor for fills to avoid distortion; keep one documented policy).
3. Precompute indicators over the full series (vectorized) into arrays indexed by bar. **Every indicator at index `i` uses only bars ≤ i** (unit-tested for look-ahead).
4. Compile strategy rules into closures: `entrySignal(i) → boolean`, `exitSignal(i) → boolean`.

### 4.3 Event loop (single symbol)
```
state = { cash, position: null, pendingOrders: [], equity: [] }
for i in warmupEnd .. N-1:
    bar = bars[i]

    # 1. Fill orders queued on the previous bar (signal at close[i-1] → fill at open[i])
    for order in pendingOrders: tryFill(order, bar)    # applies slippage + fees

    # 2. Intrabar exits for open position using this bar's high/low
    if position:
        evaluateStops(position, bar)    # stop, target, trailing; see 4.4

    # 3. Mark to market at close
    equity[i] = cash + position.qty * bar.close

    # 4. Evaluate signals at bar close (uses data ≤ i only)
    if position and (exitSignal(i) or timeStop(position, i) or sessionEnd(i)):
        queue(exit order for open of i+1)
    elif !position and filtersPass(i) and entrySignal(i):
        qty = sizing(equity[i], bar.close, stopDistance(i))
        queue(entry order for open of i+1)

    # 5. Update trailing stops, break-even, MAE/MFE trackers
close any open position at final bar (reason: "end_of_data")
```
**Default execution model:** signal on bar close, fill on next bar open ("no peeking"). Option: `fillAt: "close"` for daily strategies (flagged as optimistic).

### 4.4 Intrabar stop/target resolution
When a bar's range touches both stop and target, order is ambiguous. Policy (configurable):
- Default **pessimistic**: stop assumed hit first for longs (target first for shorts' adverse case equivalently).
- Optional: if lower-timeframe bars available, replay them to resolve exactly.
- Gap handling: if `open` is beyond the stop, fill at `open` (not at stop price).

### 4.5 Slippage and costs
- `bps`: `fill = price × (1 ± bps/10000)` (buys worse up, sells worse down).
- `ticks`: ±N × 0.01.
- `atr_pct`: `± k × ATR(14)[i-1]`.
- Optional volume constraint: cap fill qty at X% of bar volume; partial fills carry over.
- Commissions per share/trade/%, plus regulatory fees on sells (toggle). Store slippage and fees per trade for reporting.

### 4.6 Position sizing
`risk_pct`: `qty = floor(equity × risk% / (entry − stop))`, capped by buying power (cash, or 2× for margin mode) and optional max position %. Reject orders with qty < 1.

### 4.7 Basket/portfolio loop
Merge all symbols onto a unified timeline. At each timestamp:
1. Process fills/exits for every symbol with a bar.
2. Collect entry signals → rank candidates (tie-break: configured score, e.g. lowest volatility, or alphabetical for determinism).
3. Admit entries while `openPositions < maxConcurrent` and capital/allocation rules allow.
4. Portfolio equity = cash + Σ mark-to-market. Rebalance on schedule if enabled.
Per-symbol attribution: track P&L per symbol; contribution = symbol P&L / total P&L.

### 4.8 Metrics (computed from equity series and trades)
- Returns: total, CAGR = (end/start)^(1/years) − 1.
- Daily returns r_t from equity (resample to daily for intraday runs). Sharpe = mean(r − rf)/std(r) × √252; Sortino uses downside deviation; Calmar = CAGR / |MaxDD|.
- Max drawdown = min over t of (equity_t / running_max − 1); also duration and recovery.
- Trade stats: win rate, avg win/loss, payoff ratio, profit factor = Σwins / |Σlosses|, expectancy, avg R, MAE/MFE, longest streaks, exposure % = bars in market / total bars.
- Benchmark: buy-and-hold of benchmark over same range; alpha/beta via regression of daily returns; correlation.
- Monthly returns grid, rolling 6-month Sharpe, return histogram.

### 4.9 Replay simulator reuse
The simulator uses the **same** fill/slippage/cost modules, driven by a user-controlled cursor instead of a loop. `SimSession.advance(n)` moves the cursor and runs steps 1–3 of the loop; user orders are injected into `pendingOrders`. Only bars ≤ cursor are ever returned by the API.

### 4.10 Execution architecture
API validates + checks entitlements → inserts `backtests` row (queued) → enqueues job → worker loads data via provider → runs engine in worker thread (so CPU doesn't block) → writes results/trades → updates status; progress emitted every N bars to Redis → SSE to client. Timeouts, cancellation flag checked per chunk, idempotent job keys.

### 4.11 Testing
Golden tests (known strategies vs hand-computed results), property tests (equity never NaN; cash + positions reconcile), no-look-ahead test (truncate future data and confirm identical past decisions), regression snapshots per `engine_version`. Cross-check against a reference (e.g., vectorbt/backtrader in Python) on a few strategies.

### 4.12 Honesty features
Always show: "Hypothetical results. Past performance doesn't predict future results." Surface survivorship bias warning if the universe uses today's constituents; flag optimistic settings (fill at close, zero slippage); show in-sample/out-of-sample split option.

---

## 5. Market data layer

All data access goes through one interface so you can start with simulated/CSV data and swap in vendors with no changes to engine or UI.

```ts
// packages/market-data/src/types.ts
export type Timeframe = '1m'|'5m'|'15m'|'30m'|'1h'|'1d'|'1w';

export interface Bar {
  ts: number;       // epoch ms, bar START in UTC
  o: number; h: number; l: number; c: number;
  v: number;
  vwap?: number; trades?: number;
}

export interface BarQuery {
  symbol: string;
  timeframe: Timeframe;
  from: number; to: number;           // epoch ms
  adjusted: 'none' | 'splits' | 'all';
  limit?: number;
}

export interface Quote { symbol: string; ts: number; bid?: number; ask?: number; last: number; volume?: number; }
export interface SymbolInfo { symbol: string; name: string; exchange: string; type: 'stock'|'etf'|'index'; currency: string; tradable: boolean; }
export interface CorporateAction { symbol: string; type: 'split'|'dividend'; exDate: string; ratio?: number; amount?: number; }
export interface EarningsEvent { symbol: string; reportTs: number; session: 'bmo'|'amc'|'unknown'; epsEst?: number; epsActual?: number; }

export interface OptionContract {
  symbol: string; underlying: string; expiry: string; strike: number; right: 'C'|'P';
  bid?: number; ask?: number; last?: number; volume?: number; openInterest?: number;
  iv?: number; delta?: number; gamma?: number; theta?: number; vega?: number; rho?: number;
}
export interface OptionChain { underlying: string; asOf: number; spot: number; expiries: string[]; contracts: OptionContract[]; }

export interface ProviderCapabilities {
  intraday: boolean; realtime: boolean; options: boolean; historicalOptions: boolean;
  earnings: boolean; maxHistoryYears: number; rateLimitPerMin?: number;
}

export interface MarketDataProvider {
  readonly name: string;
  capabilities(): ProviderCapabilities;

  searchSymbols(query: string, limit?: number): Promise<SymbolInfo[]>;
  getSymbol(symbol: string): Promise<SymbolInfo | null>;

  getBars(q: BarQuery): Promise<Bar[]>;                       // historical
  streamBars?(symbol: string, tf: Timeframe, onBar: (b: Bar, final: boolean) => void): Unsubscribe;
  getQuote(symbol: string): Promise<Quote>;
  getQuotes(symbols: string[]): Promise<Quote[]>;

  getCorporateActions(symbol: string, from: string, to: string): Promise<CorporateAction[]>;
  getEarnings(symbol: string, from: string, to: string): Promise<EarningsEvent[]>;

  getOptionChain?(underlying: string, expiry?: string, asOf?: number): Promise<OptionChain>;
  getMarketCalendar(from: string, to: string): Promise<{ date: string; open: number; close: number; early?: boolean }[]>;
}
type Unsubscribe = () => void;
```

### Implementations
| Provider | Use | Notes |
|---|---|---|
| `SimulatedProvider` | Offline dev/tests | Geometric Brownian motion + regime switching + gaps, seeded RNG so tests are reproducible; synthetic earnings dates; Black-Scholes chains |
| `CsvProvider` / `ParquetProvider` | Offline with real historical files (e.g., exported from a free source for dev) | Reads from `/data/bars/{symbol}/{tf}.parquet` |
| `PolygonProvider` | Production historical + realtime + options | Maps aggregates API → `Bar`; handle pagination, rate limits |
| `TradierProvider` | Realtime quotes, option chains, later broker execution | Sandbox for dev |
| `CachedProvider` (decorator) | Wraps any provider | Read-through cache: memory → Postgres/Parquet → upstream; cache keys `(symbol, tf, adjusted, range)` |
| `FailoverProvider` | Primary + fallback | Circuit breaker on errors |

Selected by env: `MARKET_DATA_PROVIDER=simulated|csv|polygon|tradier`, composed in a factory:
`new CachedProvider(new FailoverProvider([polygon, tradier]))`.

### Rules
- Timestamps are UTC epoch ms, bar-start convention; exchange calendar in `America/New_York`.
- Adjustment is explicit in the query; engine records `data_version` (provider + fetch date + adjustment) in every backtest.
- Providers never throw vendor-specific errors upward; map to `RateLimited`, `NotFound`, `Unavailable`.
- **Licensing:** vendor terms often restrict redistribution; display data to your logged-in users only, check real-time/exchange fees before launch.
- Contract tests: one shared test suite that every provider must pass (ordering, no duplicates, bar alignment, adjustments).
- Option-chain interface is optional (`?`) so a bars-only provider remains valid.

---

## 6. Phased roadmap

### Phase 0: Foundations (1–2 weeks)
Monorepo, docker-compose, DB schema + migrations, Auth, design system from your landing page, `SimulatedProvider`, CI (lint, test, typecheck).
**Exit:** sign up/login works; chart renders simulated SPY.

### Phase 1: Core MVP (4–6 weeks): "Backtest in 2 minutes"
- M1 Auth/onboarding, M2 minimal dashboard, M3 basic charts (candles + 5 indicators), M4 strategy builder (entry/exit conditions, stop/target, sizing, templates), M5 single backtester with full metrics + trades table, queue + worker, CSV export.
- Real daily data via CSV/Parquet or Polygon free tier.
**Exit:** user builds RSI strategy and sees a trustworthy report. Engine golden tests pass.

### Phase 2: Differentiators (4–6 weeks)
- M6 baskets, M7 replay simulator, M10 journal (simulator trades), M11 billing (Stripe, 3 plans, entitlements), earnings overlays, comparison of runs.
- Intraday timeframes (paid data).
**Exit:** first paying users; launch publicly (private beta before).

### Phase 3: Depth (6–8 weeks)
- M8/M9 option chain + payoff + margin calculator, parameter optimization with heatmaps, **walk-forward analysis**, Monte Carlo trade reshuffling, alerts, **forward testing** (paper-run strategies on live bars with daily email), strategy sharing/marketplace-lite.

### Phase 4: Execution & scale (ongoing)
- **US broker execution** via Alpaca/Tradier/IBKR APIs (OAuth link, order routing, position sync, kill switch, per-order confirmations). Legal review needed before this: broker-dealer/investment-adviser regulatory exposure (don't give personalized advice; users place their own orders).
- Historical option backtesting (needs OPRA historical data, expensive), portfolio margin sim, mobile app, public API, team workspaces.

### Differentiation ideas (beyond the ones you listed)
1. **US broker execution:** one-click "deploy to paper/live" from a backtested strategy.
2. **Forward testing:** live out-of-sample validation with a "backtest vs live drift" chart. Strong anti-overfitting story.
3. **Margin calculator:** Reg-T and defined-risk spreads with buying-power impact.
4. **Earnings overlays and filters:** avoid/target earnings, post-earnings drift templates, IV crush stats.
5. **Overfitting guardrails:** automatic in-sample/out-of-sample split, walk-forward, parameter-stability heatmaps, "deflated Sharpe" warning when many variants were tried.
6. **Realistic cost transparency:** slippage sensitivity chart (re-run at 0/2/5/10 bps instantly).
7. **Natural-language strategy builder:** "buy when RSI < 30 above 200-day MA, stop 2 ATR" → rule JSON (LLM parsing, user confirms visual rules).
8. **AI report explainer:** plain-English critique of a backtest (concentration, regime dependence, bad streaks).
9. **Regime analysis:** performance split by bull/bear/high-VIX periods.
10. **Trade-journal insights:** discipline score, "your best setup" suggestions.
11. **Strategy versioning + diff**, shareable read-only report links, embeddable result cards for social.
12. **Data-quality badges** on every result (survivorship, adjustments, gaps).
13. **Importable community templates**, tax-aware (wash-sale, short-/long-term) P&L view.
14. **Screener → basket → backtest** pipeline in a few clicks.

---

## 7. Moving to AWS + Git later

**Git now:** trunk-based, protected `main`, conventional commits, `.env.example` only (no secrets), ADRs in `/docs/adr`, Prettier/ESLint/Husky, GitHub Actions: lint → typecheck → unit → engine golden tests → build.

**Environment parity:** all config via env vars (12-factor); `docker compose` mirrors prod services; DB migrations run by CI/CD, never by hand.

**AWS target (start small):**
- Route 53 + CloudFront + ACM → ALB → ECS Fargate services: `web`, `api`, `worker`.
- RDS Postgres (Multi-AZ later), ElastiCache Redis (or SQS), S3 (equity curves, exports, Parquet data, screenshots), Secrets Manager, CloudWatch logs/alarms, WAF.
- Terraform modules per env (`dev`, `staging`, `prod`), GitHub OIDC to AWS (no static keys).
- Stripe webhook endpoint on `api` with signature verification; Sentry for errors.
- Cost-conscious start: single small Fargate tasks, `db.t4g.micro`, scale workers on queue depth.

**Making it "official":** incorporate/entity, Terms, Privacy Policy, clear "not investment advice / hypothetical performance" disclosures, data-vendor commercial licenses (redistribution + display), Stripe business account, tax handling (Stripe Tax), cookie/GDPR consent if serving EU, and a securities-law review before any broker execution or signals features.

---

## 8. Suggested build order for your first 10 working days

1. Day 1: monorepo + docker-compose + DB schema + auth.
2. Day 2: `MarketDataProvider` interface + `SimulatedProvider` + `CsvProvider`.
3. Day 3: Chart page with Lightweight Charts.
4. Day 4: `packages/indicators` with tests (SMA, EMA, RSI, ATR, MACD, BBands).
5. Days 5–6: engine event loop (single symbol, long only, market fills, stops) + golden tests.
6. Day 7: metrics module + results JSON.
7. Day 8: job queue + worker + progress API.
8. Day 9: strategy JSON schema (Zod) + rule compiler.
9. Day 10: results UI (summary, equity, trades) → first end-to-end backtest (W1).

---

*Disclaimer: this document is a product/engineering plan, not legal or financial advice. Consult counsel before launching anything involving trading signals, brokerage execution, or paid market data redistribution.*
