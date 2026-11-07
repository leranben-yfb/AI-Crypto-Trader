# ELMS Crypto Agent — AI Bitcoin Futures Trading System

**Status:** Design specification / not yet implemented  
**Established:** 2026-11-06  
**Repository:** `leranben-yfb/AI-Crypto-Trader`

> This document records the trading-system design discussed on 2026-11-06. The parameters are *proposed starting assumptions*, not demonstrated trading edges or authorization for live leveraged trading. Do not enable live execution until venue, account, API, controls, and simulation gates are verified.

## Mission and relationship to ELMS Trading Agent

Build a standalone, Python-based automated Bitcoin futures trading agent inspired by the existing ELMS IBKR equities bot. Reuse the analytical philosophy, operational discipline, logging, and risk-management patterns, **not** the same brokerage connection or risk state. Keep crypto equity, credentials, trade ledger, kill switch, and strategy lifecycle isolated from the IBKR system.

The initial target is **BTC perpetual-style futures** with the ability to **go long, go short, stay flat, add, trim, and exit**. Primary objectives are risk-adjusted returns, controlled drawdowns, and verifiable performance net of fees, spreads, slippage, funding/financing, and other costs. Never assume a profitable strategy exists before testing.

## Independence from the equities trading system — permanent architecture principle

The [ELMS AI Trading System](https://github.com/leranben-yfb/ai-trading-system) is a **reference implementation and accumulated engineering knowledge base**, not a parent runtime, dependency, mandated template, or source of trading authority for this crypto project. The ELMS Crypto Agent must be designed and evaluated as a standalone Bitcoin-focused system, with its own development roadmap and evidence-based strategy choices.

**Reuse selectively:** Consult the equities repository for macroeconomic ingestion and rate-of-change methodology, growth/inflation Quad-like analysis, Treasury-curve/yield-driver interpretation, cross-asset context, market metrics (including volume, volatility, vol-of-vol and fractal analysis), AI candidate review, structured logging, audit trails, data freshness, decision-generation validation, tests, service supervision, recovery, and fail-closed operational controls. Reuse concepts or independently adapted code only when applicable, correct, tested, and beneficial to crypto.

**Adapt to the BTC market:** Crypto trades around the clock and has different liquidity, volatility, market structure, contract specifications, funding/financing, margin, liquidation exposure, venue behavior, and operational failure modes. Design crypto-native data, signals, trading horizons, position management, and leveraged-futures safety around those facts. Treat macro inputs as context to be validated against BTC market action, not automatic directional commands.

**Do not blindly transplant:** IBKR connectivity, equities account state, market hours, whole-share sizing, the $250 order floor, PDT logic, Hedgeye equities email/portfolio/RTA authority, current-day equities Risk Ranges, position increments, and equities-specific risk thresholds are **not crypto rules**. An approach effective in equities is a hypothesis in crypto until tested independently.

**Strict isolation:** Maintain a separate GitHub repository, broker/exchange API integration, credentials and secrets, configuration, state and database namespace, order/position/fill ledger, portfolio accounting, risk policy, kill switch, process supervision, logs, and deployments. The crypto bot must be able to operate or fail safely without the equities bot running; neither system can submit orders, share risk state, or mutate positions for the other.

**Evidence and change discipline:** Evaluate shared techniques through crypto-specific backtests, point-in-time macro data, walk-forward and paper/shadow testing, realistic fees/slippage/funding/liquidation constraints, and operational failure tests before live use. Prefer an independent crypto implementation when that is cleaner or safer. Avoid unnecessary parity with equities features; new crypto capabilities need not be added to equities.

**Working rule for future development:** Before borrowing a component from `ai-trading-system`, identify (1) the reusable principle, (2) BTC-specific assumptions requiring adaptation, (3) isolation boundaries, and (4) validation proving usefulness. Reference the equities repository's current code and tests, not stale chat summaries or historical release notes.

## Venue and eligibility (OPEN / BLOCKER)

User has a **regular personal Coinbase account with Advanced Trade** and is located in **New York, USA**. Coinbase offers U.S.-regulated perpetual-style futures products; **whether this particular account can access them and whether a retail Advanced Trade API key can submit/modify orders for them has NOT been verified**. Exchange-level FIX APIs and international Deribit/Coinbase derivatives APIs do **not** establish U.S. retail API eligibility. Before implementation:
- Verify NY residency/product eligibility, account approval, applicable contract(s), and actual trade permissions.
- Identify the precise supported *retail* REST/FIX/WebSocket API endpoints for BTC perpetual-style futures and verify authentication, order entry, cancellation, position reads, margin and liquidation fields.
- Confirm API support for protective, reduce-only, partial, and stop orders; tick/lot/contract increments, minimum notional, trading hours, maintenance, fee schedule, and data licensing.
- Confirm futures vs international perpetual differences, contract specifications and financing/funding mechanics; **do not assume standard offshore perpetual mechanics apply to U.S. contracts**.
- No live orders until a test confirms an approved, eligible account and exact compatible API.

No API credentials or secrets should be committed to this public repository. Use environment variables/secret storage, permission minimization, and a default-off live-trading switch.

## Strategy architecture

Separate signal analysis from deterministic portfolio/risk controls and exchange execution.

```text
Market data + macro/liquidity context
            |
   Regime classification (weekly/daily/4H)
            |
Strategy candidates (trend / mean reversion / reversal)
            |
Signal validation + expected edge after costs
            |
Deterministic risk gate (size, margin, liquidation, stops)
            |
Execution adapter -> exchange (after account/API verification)
            |
Reconciliation, position manager, audit logs, reporting
```

**Regime and direction:** Decide independently whether BTC is bullish, bearish, or range-bound. Hedgeye Quad analysis, Treasury yields, dollar and broader liquidity conditions can inform context but **cannot override BTC price/market structure**. Dominant trends receive priority: in bear regimes focus on shorts, but allow smaller, higher-confirmation countertrend longs; inverse in bull regimes. In ranges consider longs at support and shorts at resistance. Staying flat is encouraged when signals conflict.

**Three candidate strategies:**
1. Trend following: trade with dominant direction, ideally entering on pullbacks to support or rallies to resistance.
2. Mean reversion: selective countertrend bounces/fades; smaller risk budgets, tighter invalidation and time limits.
3. Breakout / regime-reversal detection: respond when structure indicates an actual change in direction, not a single noisy candle.

**Feature candidates:** Trend structure, returns, volatility, volatility-of-volatility, volume and volume divergence, momentum, fractal/market structure signals, support/resistance and risk ranges, open interest, futures basis/financing (if applicable), order book liquidity and transaction costs. Any AI confidence/probability must be calibrated out-of-sample; do not equate model confidence with a real edge. The AI proposes trades; rules-based controls determine whether they are permissible.

## Timeframe priorities

| Timeframe | Role |
| --- | --- |
| Weekly | Structural macro backdrop |
| Daily (1D) | Primary bullish/bearish directional bias |
| **4-hour (4H)** | **Main trade thesis, trend and setup** |
| **1-hour (1H)** | **Signal confirmation and position management** |
| 15-minute (15m) | Entries, partial profit-taking, tactical exits |
| 5-minute (5m) | Execution refinement only; never overrule larger trend on noise |

**Primary horizon:** 4H/1H swing trades lasting hours to days. **Secondary horizon:** smaller-risk 1H/15m tactical trades lasting minutes to hours, accounted for separately. These horizons are initial hypotheses for backtesting, not optimized settings.

**Proposed evaluation cadence:**
- Continuous streaming price/order/position monitoring; event-driven protective risk responses.
- Evaluate potential entries/exits every ~1–5 minutes, with completed 15m/1H/4H candle confirmation as appropriate.
- Recompute larger-regime views on closed candles and after significant verified macro/data events.
- Avoid tick-by-tick trade churn and look-ahead bias.

## Trade entry workflow

1. Identify the current market regime and volatility/liquidity conditions.
2. Generate long, short or no-trade candidates from eligible strategies.
3. Check multi-timeframe trend agreement or explicitly classify countertrend trades.
4. Validate a measurable trigger (e.g., rejection of resistance in a downtrend, break/retest, confirmed momentum), an invalidation price, expected payoff vs downside, and likely slippage/fees.
5. Estimate opportunity after costs; reject weak, conflicting, illiquid or uncalibrated signals.
6. Apply risk limits **before** submitting the order.
7. Record the input snapshot, thesis, model version, rationale, risk calculation, and final decision (including rejected trades).

Example only: BTC in a daily/4H downtrend rallies toward resistance; 1H momentum fades and 15m confirms rejection -> short candidate. A bear-market relief long is possible **only** under stronger reversal evidence and smaller risk limits.

## Leveraged futures risk engine — hard constraints

- Proposed initial **risk budget 0.25%–0.50% of account equity per trade**; validate before using.
- Start with minimal effective leverage (possibly unleveraged-equivalent exposure); any suggested 1–2x limit is a **design ceiling for testing, not a directive**. Margin requirements, contract granularity and account constraints govern practical feasibility.
- Calculate notional from `max_risk_dollars / stop_distance_fraction`, capped by leverage, margin, contract size, portfolio risk, and liquidity. Quantize contracts to venue minimums; reject trades that cannot be sized safely.
- Protective stops placed/verified at the venue when supported. Stops are **not guaranteed fills**; model gaps, slippage, fees and emergency failure.
- Require a substantial buffer between stop/invalidation and estimated liquidation level; recalculate after every fill, trim, add and margin change.
- No martingale or averaging down simply to rescue a losing position; any add requires a fresh favorable setup and full portfolio-risk recheck.
- Hard account-level exposure, loss, drawdown, concentration and consecutive-loss/cooldown limits; independent emergency kill switch.
- Fail closed on missing/stale market data, API uncertainty, state mismatch, order rejection, inability to verify protective orders, or degraded exchange connectivity.
- Use reduce-only orders for trims/exits **where available**; otherwise ensure sizing prevents accidental reversal.
- No implied permission to trade live. Manual review gates and simulation first.

## Active position management and “feed the ducks”

**No mandatory all-or-nothing fixed take-profit.** Manage an open trade based on remaining expected edge, trend health, volatility, risk-range location, and transaction costs:
- **HOLD:** underlying trade structure and reward/risk remain favorable.
- **TRIM (“feed the ducks”):** realize some gain as price reaches a favorable target/risk-range edge, becomes extended, or momentum/volume wanes; maintain a core position while larger trend remains valid.
- **ADD:** only into a position when a new favorable price/risk signal justifies it and total exposure stays within deterministic limits.
- **EXIT:** trade thesis invalidated, protective stop triggered, regime materially changed, risk limits breached, funding/holding costs overwhelm expected edge, or opportunity cost/time stop rules fire.
- **EMERGENCY EXIT:** separate risk engine responds immediately to critical conditions regardless of model preference.

Illustrative *nonbinding* long example: enter at BTC $100k, trim 20% of original exposure near $103k, another 25% near $106k, and hold the remaining 55% if the higher-timeframe trend stays strong. Short positions use mirrored logic. Actual trim fractions and trigger thresholds are **to be discovered by testing**, not hardcoded from this illustration.

Dynamic volatility-adjusted trailing stops may protect a portion of open profits but cannot guarantee gains; compare against static exits through testing. After each reduction or increase, reconcile filled contracts and protective orders so remaining risk is correct. Respect contract minimums and commission efficiency; no tiny trims that fees overwhelm.

Track maximum favorable excursion (MFE), maximum adverse excursion (MAE), realized and unrealized P&L, surrendered gains, net returns, and whether partial profit-taking or trailing stops help relative to alternate policies.

## Operational architecture

Modules proposed:
- `connectors/` — Coinbase/authorized venue account, execution and streaming data adapter
- `data/` — market, candle, macro and contract-spec ingestion with quality checks
- `regime/` — weekly/daily/4H market classification
- `signals/` — trend, reversion, breakout/reversal proposals
- `portfolio/` — open position truth, fills, equity, funding/fees, reconciliation
- `risk/` — deterministic sizing, exposure, margin, liquidation, kill switch
- `execution/` — idempotent order lifecycle, reduce-only checks, retry/recovery
- `analytics/` — backtesting, walk-forward testing, reporting, P&L attribution
- `ops/` — structured logs, alerts, health checks, status/dashboard

Never trust a submitted order to have filled. Reconcile exchange orders, positions and balances; recover safely from disconnects and restarts. Distinguish transient network errors from rejected/unknown-order states to prevent duplicate leveraged orders. Define timezone-independent timestamps (UTC), venue trading-calendar exceptions, rate limits, and incident runbooks.

## Minimal desktop GUI — V1 design guidelines

The GUI is a **small operator control panel for the private, self-directed ELMS Crypto trading bot**, not a charting platform, analytical terminal, or substitute for the autonomous engine. It must be simple enough to understand at a glance. The trading engine runs independently of the GUI and continues to respect its own validated risk/execution rules. The commercial, alert-only product is a separate potential offering; do not silently grant it live execution capabilities because this GUI supports them.

### Required V1 controls

- **Start / Stop:** clear running/stopped state. Define Stop as **stop new entries/increases**, not abandoning open leveraged positions. Existing protective orders and risk management must remain intact or follow a verified safe shutdown policy. Show the actual engine state, not just whether the window is open.
- **Maximum trade leverage:** user-selectable only within the eligible venue's verified contract limits and the engine's hard safety ceiling; leverage is not a substitute for risk-per-trade sizing.
- **Minimum and maximum position size:** clearly labeled units and calculation basis (proposed default: position notional as a percent of account equity), with validation that minimum <= maximum and hard margin/liquidation/exposure constraints always prevail.
- **Position risk-increase / risk-decrease increment:** configurable sizing step for authorized adds and trims. Distinguish *position notional increment* from *dollars at risk to a stop*. Do not import the equities bot's fixed 50 bp increments or $250 order floor as default crypto rules.
- **Trade activity log:** timestamp (with timezone), instrument, side, action (ENTRY, EXIT, INCREASE, REDUCE), quantity, fill price, notional, fees/financing where applicable, realized profit/loss and cumulative realized P&L. Derive trade results from reconciled exchange fills; preserve partial-fill and adjustment history. Show failed/rejected actions distinctly from fills.
- **PnL view with selectable periods:** 1D, 1W, 1M, 3M, 1Y, ALL; simple account equity / cumulative P&L line chart and clear net dollar and percentage returns. Make it evident whether P&L is realized, unrealized, or combined; net of exchange fees and relevant financing; avoid hypothetical performance being mistaken for actual history.
- **Connection/health indicator:** concise exchange connectivity, live market-data freshness, bot health, and alert for critical failures.
- **Emergency Close:** clearly separated, confirmation-protected control intended to cancel applicable open orders and attempt to flatten exposure safely. Account for partial fills, disconnections, and exchange failure; never promise guaranteed liquidation or imply an unconfirmed close succeeded.

### Interaction and safety requirements

- **Separate Stop New Trades from Emergency Close.** Provide visible state for STOPPED, RUNNING, DEGRADED, and EMERGENCY / ENTRY HALTED as applicable. License expiration or GUI closure must never silently strand risky positions.
- Settings changes must be validated, persisted safely, auditable, and applied at well-defined boundaries. Invalid settings (e.g., min > max, excessive leverage, granularity below contract minimum) must be rejected with plain-language feedback; existing protective safeguards cannot be weakened by UI values.
- Prefer read-only presentation of trading activity and P&L; do not add discretionary manual order tickets, advanced charts, macro dashboards, or dozens of strategy controls to V1.
- Keep the GUI responsive and decoupled from market-data polling, AI model evaluation, trade decisions, exchange connections, and protective execution. UI crashes must not kill an otherwise healthy risk engine.
- **Implementation preference:** Python **PySide6 (Qt)** for a lightweight Windows desktop control panel, subject to packaging, dependency, distribution/licensing, and maintainability validation. Avoid adding a React/Tauri/FastAPI stack solely to render these basic controls.
- GUI mockups and sample numbers are illustrative; never display fictional P&L as real trading results.

### Future product boundary

The private automated-trading application and any commercial standardized **signals-only** offering are different operating products. A signals-only subscriber GUI must not expose exchange-account connection, order execution, or live-trading buttons. Decide licensing/regulatory architecture separately; do not present an educational disclaimer as a replacement for reviewing the product's actual functionality.

**V1 guiding principle:** **Start/stop, configure risk, see every trade, review P&L, and know whether the system is healthy. Nothing more unless operational safety requires it.**

## Development gates / definition of done

1. **Access/contract validation:** U.S./NY retail account eligibility, exact API permissions, instrument specs, and safe order types documented.
2. **Read-only prototype:** secure connectivity, live/historical BTC candles, futures contract and margin state, auditable data checks.
3. **Research:** backtest multiple regimes with realistic spreads, fees, financing/funding where applicable, slippage, leverage and liquidations; avoid look-ahead and overfitting; evaluate risk-adjusted return vs BTC holding and flat cash.
4. **Walk-forward/out-of-sample:** calibrated predictive usefulness, sensitivity analyses, market shocks, data outages, and realistic execution constraints.
5. **Paper/shadow mode:** shadow trades, order simulator, position and stop management, “feed the ducks,” outage/disconnect and restart tests.
6. **Small, manually approved live pilot (only if eligible):** kill switch tested, strict loss caps, alerts, audits, periodic review.
7. **Scale only on evidence:** require sustained net performance and operational reliability; never increase leverage to compensate for weak signals.

## Open design questions (not settled)

- Confirm Coinbase retail U.S. perpetual-style derivatives **API trade support**, specific BTC contract, NY eligibility and permission status.
- Clarify 24/7 access, daily session/maintenance behavior, margin, fees and actual financing conventions.
- Choose an exact risk-range algorithm and explicit model features/thresholds.
- Decide whether short-term countertrend signals should be permitted simultaneously with swing positions (netting vs hedge-mode limitations).
- Establish portfolio caps, max drawdown/daily loss and cooldown parameters from stress testing.
- Validate partial trim granularity and protective-order management.
- Determine historical data availability and reliable backtesting sources.
- Determine whether the initial implementation should stay fully rules-based for entries or incorporate a calibrated ML score.

## Guiding principle

**AI discovers and ranks opportunities; deterministic controls manage survival and execution.** The system should prefer no trade to a weak trade, harvest profits selectively while keeping healthy core exposure, and remain isolated from the IBKR equities agent.
