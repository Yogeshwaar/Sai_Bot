# SMC Futures Paper-Trading Strategy Design

**Status:** Approved conversational design

**Date:** 2026-09-29

**Initial instruments:** NIFTY and BANKNIFTY front-month futures

**Initial mode:** Deterministic historical replay followed by live paper trading

## 1. Purpose

Add an independently implemented, fully automatic intraday strategy to the broker-neutral copy-trading platform. The strategy detects deterministic Smart Money Concepts-style price-action setups, emits desired futures positions, and manages structural stop-loss and profit-target exits through the existing risk and execution boundaries.

The first milestone proves the strategy using historical OHLCV replay and the simulated broker. Passing replay acceptance permits live market-data paper trading only. It does not permit real-money trading.

## 2. Licensing and external-reference boundary

The implementation may study general public descriptions of market structure, liquidity sweeps, displacement, fair-value gaps, order blocks, and the architectural patterns exposed by LuxAlgo's public repositories. It must not copy, translate, or derive source code from the LuxAlgo Smart Money Concepts indicator, which is published under CC BY-NC-SA 4.0.

Relevant public repository patterns include:

- PineTS: completed-bar time-series semantics and separation between indicator source, runtime, and data provider.
- Broker SDK: normalized external data, capability discovery, fail-soft reads, explicit trading capability, and conformance vectors.
- Trade Relay: a safety boundary between strategy alerts and broker execution, idempotency, and flight-recorder-style evidence.
- Edge Stats: sample sizes, confidence intervals, and condition-based outcome analysis.
- Vela: future chart rendering only; it is not part of the first strategy milestone.

These repositories are references, not dependencies. The strategy will be native Python within the existing modular monolith unless a later approved design changes the runtime.

## 3. Scope

### 3.1 Included

- NIFTY and BANKNIFTY futures
- Intraday trading only
- 15-minute directional structure and 5-minute entries
- Bullish and bearish setups
- Strict SMC confluence
- Structural stop with ATR buffer
- Fixed 2R profit target
- Risk-based whole-lot sizing
- Deterministic replay, simulated fills, charges, and slippage
- Full decision, signal, order, fill, and exit evidence
- Front-month contract selection and deterministic rollover

### 3.2 Excluded

- Real-money orders
- Options and cash equities
- Overnight positions
- One-minute or 30/15-minute profiles
- Pyramiding, averaging down, trailing stops, and discretionary stop movement
- AI-generated or AI-modified trading decisions
- Direct TradingView or Pine Script execution
- A charting UI beyond data needed for validation reports

## 4. Architecture

The strategy is an isolated domain module that consumes normalized completed candles and emits commands through the existing target application service. It cannot call a broker adapter directly or bypass risk, margin reservation, execution fencing, audit, or reconciliation.

```text
historical futures OHLCV
          |
deterministic replay clock
          |
normalized completed candles
     5-minute + 15-minute
          |
market-structure feature engine
      /                 \
15m directional bias    5m setup features
      \                 /
 strict confluence state machine
          |
0.2% risk position sizing
          |
target application service
          |
risk + margin + fenced execution
          |
deterministic simulated broker
          |
fills + OCO exits + reconciliation
          |
trade journal + replay report
```

Separate strategy instances manage NIFTY and BANKNIFTY. Account-level risk evaluates their combined exposure.

## 5. Component responsibilities

### 5.1 Market-data normalization

The input contract supplies canonical instrument identity, contract expiry, lot size, exchange timestamp, timeframe, open, high, low, close, and volume. Prices use fixed-precision decimals. Candles are ordered by exchange timestamp and uniquely identified by instrument, timeframe, and opening timestamp.

The normalizer rejects malformed OHLC relationships, non-positive prices, duplicate conflicts, unknown sessions, and contract metadata mismatches. Exact duplicate candles are idempotent. Late or missing candles close the strategy to new setups until the stream is repaired and replayed consistently.

### 5.2 Replay clock and aggregation

The replay clock releases events in deterministic timestamp order. Strategy evaluation occurs only after candle completion. Five-minute source candles are aggregated into 15-minute candles on exchange-session boundaries. The same normalized candle interface will be used by a later live paper feed.

No event may expose a future candle, a future high or low, or a partially formed candle. Runs using the same input, configuration, fee model, slippage seed, and simulator seed must produce identical results.

### 5.3 Market-structure engine

The engine produces confirmed swing pivots, BOS, CHoCH, liquidity sweeps, displacement, three-candle fair-value gaps, ATR, and order-block references. Version one uses fair-value-gap retracement for entries; order blocks are recorded for explanation and later research but are not an independent entry trigger.

Every feature includes its source candle identities, confirmation time, direction, price bounds, and configuration version. A feature becomes visible only when every required confirming candle has closed.

### 5.4 Confluence state machine

The state machine progresses through explicit states such as `IDLE`, `BIAS_READY`, `SWEEP_CONFIRMED`, `DISPLACEMENT_CONFIRMED`, `ENTRY_WORKING`, `POSITION_OPEN`, `EXIT_PENDING`, and terminal outcomes. Newer structure can invalidate a pending setup, but never rewrite a past decision.

### 5.5 Position sizing

Maximum planned loss is 0.2% of current confirmed paper-account equity. Quantity is calculated from the entry-to-stop distance, contract multiplier, current lot size, expected entry slippage, expected stop slippage, and configured costs. Quantity always rounds down to a whole lot.

If one lot exceeds the risk allowance, the setup is classified `RISK_ROUNDED_TO_ZERO` and no order is created. The platform risk engine may further reduce or reject the resulting target.

## 6. Deterministic signal rules

### 6.1 Fifteen-minute directional bias

1. A swing pivot requires two closed candles on each side of the candidate candle.
2. A close above the latest confirmed swing high creates a bullish BOS.
3. A close below the latest confirmed swing low creates a bearish BOS.
4. A CHoCH marks a possible reversal but cannot authorize an entry.
5. The new direction becomes tradable only after a BOS confirms it.
6. Before confirmation, or when required structure is unavailable, bias is neutral.

### 6.2 Five-minute long setup

1. The confirmed 15-minute bias is bullish.
2. Price sweeps a confirmed 5-minute swing low: the candle trades below the level and closes back above it.
3. A later bullish displacement candle closes above the latest confirmed internal swing high.
4. The displacement candle body is at least 1.5 times the median body of the preceding 14 completed 5-minute candles.
5. The displacement sequence creates a bullish three-candle fair-value gap: the third candle's low is above the first candle's high.
6. The entry is a limit order at the midpoint of that gap.
7. The unfilled entry expires after six completed 5-minute candles.
8. Before fill, a trade through the planned stop invalidates and cancels the setup.

### 6.3 Five-minute short setup

The short setup is symmetrical: bearish 15-minute bias, sweep and close back below a confirmed 5-minute swing high, bearish displacement through the latest internal swing low, qualifying bearish fair-value gap, and midpoint limit entry.

### 6.4 Stop and target

For a long, the stop is the sweep low minus 0.1 times 14-period 5-minute ATR. For a short, it is the sweep high plus the same buffer. The initial target is exactly twice the actual entry-to-stop distance.

After a fill, the simulator manages stop and target as an OCO pair. There is no break-even move, trailing stop, partial profit-taking, pyramiding, or averaging down in version one.

## 7. Session and trade limits

All times use `Asia/Kolkata` and the authoritative NSE trading calendar.

- Ignore signals before 09:30.
- Permit new entries from 09:30 through 14:45.
- Create no new entry after 14:45.
- Cancel every unfilled strategy entry at 15:00.
- Force-close every remaining position by 15:20.
- Permit at most one open position per instrument.
- Permit at most three completed trades per instrument per trading day.
- Disable further entries for an instrument after two consecutive losing trades that day.
- Retain the platform-wide 3% daily-loss limit as a final account safety ceiling.

A forced session exit is an ordinary auditable strategy exit and must not be represented as a profit-target or stop-loss event.

## 8. Contract selection and rollover

The resolver selects the front-month NIFTY and BANKNIFTY futures contract from effective-dated instrument metadata. It switches to the next contract three trading days before expiry or earlier when a configured, reproducible liquidity rule establishes that the next contract is more liquid.

The initial liquidity rule switches early when the next contract's completed-session volume exceeds the current contract's completed-session volume for two consecutive trading sessions. Only sessions complete before the resolver runs are eligible. If either contract lacks comparable volume data, the deterministic three-trading-day rule applies.

No position crosses a session boundary, so rollover never migrates an open trade. Contract changes reset instrument-local pending setups while preserving daily account safety state.

## 9. Execution and failure behavior

- Entry orders are price-limited and expire through an explicit cancel command.
- Confirmed fills create or resize the protective OCO pair to the filled quantity.
- Partial fills never protect more or less than confirmed open quantity.
- Rejections release strategy-local planned risk and are not blindly retried.
- Ambiguous submission or cancellation outcomes enter platform reconciliation and quarantine.
- Missing, duplicated-conflicting, late, or out-of-order market data pauses new setups.
- Unknown session, expiry, lot size, contract multiplier, or fee configuration fails closed.
- Strategy restart rebuilds state from persisted events and replays from the last committed candle without duplicating targets.
- The simulator models configurable latency, partial fills, rejected orders, price gaps, brokerage, exchange charges, taxes, and slippage.

## 10. Persistence and evidence

Persist normalized candle references or immutable dataset coordinates, structure events, setup transitions, invalidations, sizing inputs, target intents, risk results, orders, fills, OCO transitions, exits, daily limit state, contract decisions, and replay-run metadata.

Every decision records the strategy configuration version, instrument-master version, source candle identities, correlation ID, and deterministic replay run ID. Reports must explain why a setup traded, expired, was invalidated, rounded to zero, or was rejected.

## 11. Verification

### 11.1 Unit and property tests

- Confirmed swing, BOS, CHoCH, sweep, displacement, fair-value-gap, ATR, and order-block calculations
- Bullish and bearish symmetry
- Decimal position sizing and whole-lot rounding
- Session cutoffs, trade count, consecutive losses, and forced exit
- Contract selection and holiday-aware rollover
- Property test that appending future candles cannot change previously emitted signals
- Property test that no input candle is consumed before its close time

### 11.2 Integration and scenario tests

- Complete replay from source candles through target, risk, execution, fills, OCO, and journal
- Entry expiry and pre-fill invalidation
- Gap through stop and target
- Same-candle stop/target ambiguity resolved by an explicit conservative fill policy
- Partial fill, rejection, restart, duplicate delivery, missing candle, and out-of-order candle
- 15:00 cancellation and 15:20 forced square-off
- Front-month rollover and metadata failure

### 11.3 Research method

Evaluation uses walk-forward partitions. Parameters are selected only on earlier data and then frozen for unseen periods. All charges and modeled slippage are included. Results report trade count, expectancy, profit factor, win rate, average win/loss, maximum drawdown, exposure, consecutive losses, and confidence intervals where meaningful.

The strategy is compared with no-trade and simple directional baselines. Results are reported separately for NIFTY, BANKNIFTY, long, short, volatility regime, and replay period so aggregate performance cannot hide a failing segment.

## 12. Acceptance gate

The replay milestone passes only when:

1. At least 200 completed trades exist across both instruments on unseen data.
2. Brokerage, exchange charges, taxes, modeled latency, and slippage are included.
3. Unseen-data expectancy is positive and profit factor is at least 1.20.
4. Maximum strategy drawdown is no greater than 10%.
5. No lookahead, repainting, or premature-candle access test fails.
6. No position survives the 15:20 square-off.
7. The two-consecutive-loss and three-trades-per-instrument limits always prevent further entries.
8. Repeated deterministic runs produce identical decisions and fills.
9. Every target and exit can be traced to its source candles and configuration.
10. A fresh review confirms the implementation is independent and contains no copied LuxAlgo SMC source.

Passing this gate permits a separate live-data paper-trading milestone. It does not authorize live-money trading or imply that future profitability is guaranteed.

## 13. Future evolution

Later designs may add 5-minute/1-minute and 30-minute/15-minute profiles, score-based confluence, order-block entries, partial profit-taking, trailing exits, chart visualization, and additional instruments. Each requires its own walk-forward validation and cannot inherit approval from this version.
