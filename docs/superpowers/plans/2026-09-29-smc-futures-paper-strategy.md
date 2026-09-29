# SMC Futures Paper-Trading Strategy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a deterministic, non-repainting NIFTY/BANKNIFTY futures strategy that detects strict 15-minute/5-minute SMC confluence, sizes each trade to 0.2% equity risk, and validates stop, target, session, and rollover behavior through historical replay and a simulated broker.

**Architecture:** Implement a pure Python strategy core beneath `copytrader.strategy.smc`, isolated from I/O behind candle-source, target-sink, fee-model, and simulator interfaces. A deterministic replay runner supplies completed 5-minute candles, aggregates 15-minute structure, records explainable decisions, and drives an in-memory paper broker; later platform code can implement the same target sink without changing signal logic.

**Tech Stack:** Python 3.12, standard-library `dataclasses`, `Decimal`, `zoneinfo`, pytest, pytest-asyncio, Hypothesis, Ruff, and mypy. FastAPI/PostgreSQL integration remains behind protocols until the secure-foundation milestone exists.

**Spec:** `docs/superpowers/specs/2026-09-29-smc-futures-paper-strategy-design.md`

## Global Constraints

- Use only completed candles; no future candle, partial candle, or lookahead value may affect an earlier decision.
- Implement market-structure concepts independently; do not copy or translate LuxAlgo SMC source.
- Support NIFTY and BANKNIFTY front-month futures only in this milestone.
- Use 15-minute confirmed structure for bias and 5-minute strict confluence for entries.
- Risk exactly at most 0.2% of confirmed paper equity per trade and round down to whole lots.
- Use structural stops with a 0.1 ATR buffer and a fixed 2R target.
- Permit entries only from 09:30 through 14:45 Asia/Kolkata; cancel entries at 15:00 and flatten at 15:20.
- Permit one open position, three completed trades, and at most two consecutive losses per instrument per day.
- Use fixed-precision decimals for prices, equity, fees, slippage, and P&L; reject binary floats.
- Same-candle stop-and-target ambiguity resolves conservatively as stop-first.
- Historical data, rates, fees, slippage, instrument metadata, and strategy settings are versioned inputs to every replay.
- The milestone cannot pass the research acceptance gate until a licensed historical dataset yields at least 200 unseen-data trades with costs included.

## Review Focus

- A missing, duplicate-conflicting, out-of-order, or partial candle must pause new setups instead of silently altering structure; Task 2 pins each case.
- Appending future bars must never change an already emitted feature, setup, or target; Tasks 3, 4, and 9 include prefix-invariance tests.
- Zero/negative stop distance, one-lot risk above allowance, changing lot size, and Decimal boundary rounding must fail closed; Task 5 pins each case.
- A partial entry fill, gap through stop, and candle touching stop and target must leave no unprotected or overstated position; Task 7 pins each case.
- Holiday/expiry boundaries, missing volume, early volume crossover, and session timezone transitions must select one deterministic contract and cutoff; Tasks 6 and 8 pin each case.

---

### Task 1: Strategy package foundation and immutable domain types

**Files:**
- Modify: `pyproject.toml` if it exists; otherwise create it with the Python 3.12/dev-tool settings from `docs/superpowers/plans/2026-09-25-secure-foundation.md`
- Create: `src/copytrader/__init__.py` if absent
- Create: `src/copytrader/strategy/__init__.py`
- Create: `src/copytrader/strategy/smc/__init__.py`
- Create: `src/copytrader/strategy/smc/types.py`
- Create: `src/copytrader/strategy/smc/config.py`
- Create: `tests/unit/strategy/smc/test_types.py`

**Interfaces:**
- Produces: `InstrumentId`, `Timeframe`, `Candle`, `ContractMetadata`, `Direction`, `PriceRange`, `StrategyConfig`
- Produces: `Candle.__post_init__()` validation that rejects floats, naive timestamps, invalid OHLC, non-positive values, and non-aligned timeframe opens
- Consumes: no earlier strategy interfaces

- [ ] **Step 1: Write failing domain-validation tests**

Test `test_valid_candle_is_immutable`, `test_float_price_is_rejected`, `test_naive_timestamp_is_rejected`, `test_invalid_ohlc_is_rejected`, and `test_default_config_matches_approved_design`. Assert `risk_fraction == Decimal("0.002")`, `reward_multiple == Decimal("2")`, `atr_buffer == Decimal("0.1")`, displacement lookback 14, displacement multiple 1.5, entry expiry six 5-minute candles, pivot width two, and the approved session limits.

- [ ] **Step 2: Create the Python 3.12 development environment**

If `.venv` does not exist, create it with the bundled Python 3.12 runtime named in the secure-foundation plan. Install the project in editable mode with its `dev` extras:

```powershell
& 'C:\Users\yoges\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe' -m venv .venv
& '.\.venv\Scripts\python.exe' -m pip install --upgrade pip
& '.\.venv\Scripts\python.exe' -m pip install -e '.[dev]'
```

Expected: editable installation succeeds and `.venv/` is ignored by Git.

- [ ] **Step 3: Run the tests and verify the package is missing**

Run: `& '.\.venv\Scripts\python.exe' -m pytest tests/unit/strategy/smc/test_types.py -q`

Expected: FAIL because `copytrader.strategy.smc.types` does not exist.

- [ ] **Step 4: Implement the immutable types and configuration**

Use frozen, slotted dataclasses and enums. Accept only `Decimal` for numeric market values and timezone-aware `datetime` for exchange timestamps. `Timeframe` supports `FIVE_MINUTES` and `FIFTEEN_MINUTES`.

- [ ] **Step 5: Run focused quality checks**

Run:

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/unit/strategy/smc/test_types.py -q
& '.\.venv\Scripts\python.exe' -m ruff check src/copytrader/strategy tests/unit/strategy
& '.\.venv\Scripts\python.exe' -m mypy src/copytrader/strategy
```

Expected: all pass.

- [ ] **Step 6: Commit**

```powershell
git add pyproject.toml src/copytrader tests/unit/strategy/smc/test_types.py
git commit -m "feat: define SMC strategy domain types"
```

---

### Task 2: Validated candle stream, replay clock, and 15-minute aggregation

**Files:**
- Create: `src/copytrader/strategy/smc/stream.py`
- Create: `src/copytrader/strategy/smc/replay.py`
- Create: `tests/unit/strategy/smc/test_stream.py`
- Create: `tests/unit/strategy/smc/test_replay.py`

**Interfaces:**
- Consumes: `Candle`, `InstrumentId`, `Timeframe`
- Produces: `CandleStream.accept(candle: Candle) -> StreamResult`, returning accepted completed 5-minute candles plus any newly completed 15-minute candle
- Produces: `StreamStatus` with `READY` and `PAUSED_DATA_INTEGRITY`
- Produces: `ReplayClock.run(events: Iterable[Candle], consumer: CandleConsumer) -> ReplaySummary`
- Produces: protocol `CandleConsumer.on_candle(candle: Candle) -> None`

- [ ] **Step 1: Write failing stream-integrity and aggregation tests**

Cover exact duplicate idempotency, conflicting duplicate pause, missing interval pause, out-of-order pause, incomplete-candle rejection, session-boundary alignment, and correct aggregation of three 5-minute candles into one 15-minute candle. Include the Review Focus cases.

- [ ] **Step 2: Run the focused tests and verify failure**

Run: `& '.\.venv\Scripts\python.exe' -m pytest tests/unit/strategy/smc/test_stream.py tests/unit/strategy/smc/test_replay.py -q`

Expected: FAIL because stream and replay modules are missing.

- [ ] **Step 3: Implement deterministic validation, aggregation, and clock ordering**

`StreamResult` tracks the next expected 5-minute opening timestamp per instrument/session. It emits a 15-minute candle only after all three component candles are accepted. `ReplayClock` rejects equal-time ambiguity unless instrument and timeframe provide a stable total order.

- [ ] **Step 4: Verify focused tests and static checks**

Run the Task 2 tests, Ruff, and mypy for the strategy tree. Expected: all pass.

- [ ] **Step 5: Commit**

```powershell
git add src/copytrader/strategy/smc/stream.py src/copytrader/strategy/smc/replay.py tests/unit/strategy/smc
git commit -m "feat: add deterministic candle replay"
```

---

### Task 3: Confirmed market structure without repainting

**Files:**
- Create: `src/copytrader/strategy/smc/structure.py`
- Create: `tests/unit/strategy/smc/test_structure.py`
- Create: `tests/property/strategy/smc/test_structure_properties.py`

**Interfaces:**
- Consumes: completed `Candle` values and pivot width from `StrategyConfig`
- Produces: `StructureEngine.on_candle(candle: Candle) -> tuple[StructureEvent, ...]`
- Produces: `SwingPivot`, `BreakOfStructure`, `ChangeOfCharacter`, `TrendBias`
- Guarantees: a pivot at index `i` is emitted only after candles `i+1` and `i+2` close; emitted events are immutable

- [ ] **Step 1: Write failing hand-calculated pivot/BOS/CHoCH tests**

Use short literal Decimal candle fixtures. Assert exact source candle IDs, confirmation timestamps, broken levels, direction, and bias transitions. CHoCH changes reversal state but only a subsequent same-direction BOS produces tradable bias.

- [ ] **Step 2: Write the failing prefix-invariance property test**

Generate valid candle prefixes and suffixes with Hypothesis. Assert events emitted for the prefix remain byte-for-byte equal after processing any valid suffix.

- [ ] **Step 3: Run the tests and verify failure**

Run the Task 3 unit and property files. Expected: FAIL because `StructureEngine` is missing.

- [ ] **Step 4: Implement confirmed pivots and structure transitions**

Maintain bounded candle/pivot state. Emit each level break once and require a close beyond the level; wick-only violations do not create BOS/CHoCH.

- [ ] **Step 5: Verify and commit**

Run focused tests, Ruff, and mypy; then commit as `feat: detect confirmed market structure`.

---

### Task 4: Five-minute SMC feature engine

**Files:**
- Create: `src/copytrader/strategy/smc/features.py`
- Create: `tests/unit/strategy/smc/test_features.py`
- Create: `tests/property/strategy/smc/test_feature_properties.py`

**Interfaces:**
- Consumes: completed 5-minute `Candle`, confirmed internal pivots from Task 3, and `StrategyConfig`
- Produces: `FeatureEngine.on_candle(candle: Candle) -> FeatureSnapshot`
- Produces: `LiquiditySweep`, `Displacement`, `FairValueGap`, `OrderBlockReference`, and 14-period Wilder `ATR`
- Guarantees: displacement body is at least `1.5 * median(previous 14 completed bodies)` and a valid FVG uses the exact three-candle inequalities from the spec

- [ ] **Step 1: Write failing golden feature tests**

Test bullish and bearish sweep close-back behavior, wick-only internal break rejection, displacement threshold equality, bullish/bearish FVG bounds, ATR warm-up, and order-block explanatory references.

- [ ] **Step 2: Write failing symmetry and prefix-invariance properties**

Mirror prices around a fixed axis and assert bullish features become equivalent bearish features. Append suffix candles and assert prior snapshots do not change.

- [ ] **Step 3: Run and confirm failure**

Expected: missing `FeatureEngine`.

- [ ] **Step 4: Implement the feature engine**

Use Decimal-only calculations. Median body excludes the candidate displacement candle. FVG and sweep events retain source candle IDs and confirmation time.

- [ ] **Step 5: Verify and commit**

Run focused tests and static checks; commit as `feat: detect SMC confluence features`.

---

### Task 5: Strict confluence state machine and risk-based sizing

**Files:**
- Create: `src/copytrader/strategy/smc/setup.py`
- Create: `src/copytrader/strategy/smc/sizing.py`
- Create: `tests/unit/strategy/smc/test_setup.py`
- Create: `tests/unit/strategy/smc/test_sizing.py`

**Interfaces:**
- Consumes: 15-minute `TrendBias`, 5-minute `FeatureSnapshot`, `ContractMetadata`, confirmed equity, and `StrategyConfig`
- Produces: `SetupMachine.on_market(input: MarketInput) -> tuple[StrategyDecision, ...]`
- Produces: states `IDLE`, `BIAS_READY`, `SWEEP_CONFIRMED`, `DISPLACEMENT_CONFIRMED`, `ENTRY_WORKING`, `POSITION_OPEN`, `EXIT_PENDING`
- Produces: `PositionSizer.size(entry: Decimal, stop: Decimal, equity: Decimal, contract: ContractMetadata, costs: CostEstimate) -> SizeDecision`
- Produces: `EntryIntent` containing side, whole-lot quantity, limit, stop, target, expiry timestamp, source features, and configuration version

- [ ] **Step 1: Write failing strict-sequence tests**

Assert no entry for missing bias, CHoCH without confirming BOS, sweep without displacement, displacement without FVG, wrong-direction confluence, or expired/invalidated setup. Assert a valid sequence emits one midpoint-limit `EntryIntent` and expires after six completed 5-minute candles.

- [ ] **Step 2: Write failing sizing boundary tests**

Assert 0.2% risk, contract multiplier and expected costs are included, lots round down, risk below one lot yields `RISK_ROUNDED_TO_ZERO`, and float/zero/negative inputs fail closed. Include changing lot-size boundaries.

- [ ] **Step 3: Run and confirm failure**

Expected: setup and sizing modules are missing.

- [ ] **Step 4: Implement the state machine and sizer**

Compute long stop as sweep low minus `0.1 * ATR`, short stop symmetrically, and target as entry plus/minus `2 * stop_distance`. Persist every transition as a `StrategyDecision`, including rejection reasons.

- [ ] **Step 5: Verify and commit**

Run focused tests and static checks; commit as `feat: create risk-sized SMC entry intents`.

---

### Task 6: Contract resolver and NSE session controls

**Files:**
- Create: `src/copytrader/strategy/smc/contracts.py`
- Create: `src/copytrader/strategy/smc/session.py`
- Create: `tests/unit/strategy/smc/test_contracts.py`
- Create: `tests/unit/strategy/smc/test_session.py`

**Interfaces:**
- Consumes: effective-dated `ContractMetadata`, exchange trading dates, completed-session volume, timestamp, and daily trade outcomes
- Produces: `ContractResolver.resolve(underlying: str, trading_date: date) -> ContractSelection`
- Produces: `SessionGuard.evaluate(instrument: InstrumentId, at: datetime, state: DailyTradeState) -> SessionPermission`
- Produces: actions `ALLOW_ENTRY`, `BLOCK_ENTRY`, `CANCEL_ENTRIES`, and `FLATTEN_POSITION`

- [ ] **Step 1: Write failing rollover tests**

Cover ordinary front month, switch three trading days before expiry, early switch after two consecutive completed sessions where next-contract volume exceeds current volume, missing comparable volume fallback, holiday counting, and metadata failure.

- [ ] **Step 2: Write failing session and daily-limit tests**

Pin 09:30/14:45 entry boundaries, 15:00 cancellation, 15:20 flatten, one-open-position limit, three completed trades, two consecutive losses, new-day reset, and Asia/Kolkata conversion from UTC.

- [ ] **Step 3: Run and confirm failure**

Expected: contract/session modules are missing.

- [ ] **Step 4: Implement resolver and guard**

Use only the supplied exchange calendar; never infer weekdays as trading days. Persist the reason and input versions for each selection or block.

- [ ] **Step 5: Verify and commit**

Run focused tests and static checks; commit as `feat: enforce futures contract and session rules`.

---

### Task 7: Deterministic paper broker with protective OCO execution

**Files:**
- Create: `src/copytrader/simulator/__init__.py`
- Create: `src/copytrader/simulator/models.py`
- Create: `src/copytrader/simulator/broker.py`
- Create: `src/copytrader/simulator/costs.py`
- Create: `tests/unit/simulator/test_broker.py`
- Create: `tests/unit/simulator/test_costs.py`

**Interfaces:**
- Consumes: completed candles, `EntryIntent`, deterministic `SimulationConfig`, and `FeeModel`
- Produces: `PaperBroker.submit(intent: EntryIntent) -> BrokerReceipt`
- Produces: `PaperBroker.on_candle(candle: Candle) -> tuple[BrokerEvent, ...]`
- Produces: accepted, partial-fill, filled, cancelled, rejected, stop-filled, target-filled, and forced-exit events
- Produces: protocol `FeeModel.calculate(fill: Fill) -> Decimal`

- [ ] **Step 1: Write failing order-lifecycle tests**

Test unfilled limit, fill, deterministic partial fill, rejection, cancellation, gap-through stop, target, OCO sibling cancellation, entry expiration, and 15:20 forced exit.

- [ ] **Step 2: Write failing ambiguity and protection tests**

Assert stop-first when one candle touches stop and target, protective quantity equals confirmed partial-fill quantity, and no broker event can create negative remaining quantity or an unprotected open position.

- [ ] **Step 3: Run and confirm failure**

Expected: simulator modules are missing.

- [ ] **Step 4: Implement broker and injected costs**

Use explicit deterministic slippage and latency settings; never call random APIs without a supplied seed. Fee rates are replay inputs so statutory changes do not alter historical run reproducibility.

- [ ] **Step 5: Verify and commit**

Run simulator tests and static checks; commit as `feat: simulate protected futures execution`.

---

### Task 8: Target boundary, orchestration, and restart-safe journal

**Files:**
- Create: `src/copytrader/strategy/smc/targets.py`
- Create: `src/copytrader/strategy/smc/engine.py`
- Create: `src/copytrader/strategy/smc/journal.py`
- Create: `tests/integration/strategy/smc/test_engine.py`
- Create: `tests/integration/strategy/smc/test_restart.py`

**Interfaces:**
- Consumes: Tasks 2-7
- Produces: protocol `TargetSink.apply(target: StrategyTarget) -> TargetReceipt`
- Produces: `InMemoryTargetSink` for replay and a future platform adapter seam
- Produces: `StrategyEngine.on_candle(candle: Candle) -> EngineResult`
- Produces: append-only `DecisionJournal` and `EngineCheckpoint`

- [ ] **Step 1: Write the failing end-to-end engine scenario**

Feed hand-calculated 15-minute/5-minute bars through a bullish setup, entry fill, target fill, journal, and daily state. Assert exact target quantity, stop, target, source candle IDs, correlation ID, P&L, fees, and state transitions.

- [ ] **Step 2: Write failing restart/idempotency tests**

Checkpoint after every boundary around target submission and fill processing. Restore, replay duplicate inputs, and assert no duplicate target, order, fill, or trade result.

- [ ] **Step 3: Run and confirm failure**

Expected: orchestration modules are missing.

- [ ] **Step 4: Implement orchestration and append-only in-memory journal**

Keep storage behind a `JournalStore` protocol. The later PostgreSQL implementation can replace the in-memory store without changing engine logic.

- [ ] **Step 5: Verify and commit**

Run integration tests plus all strategy/simulator unit tests; commit as `feat: orchestrate replay strategy lifecycle`.

---

### Task 9: CSV replay ingestion, reports, and non-repainting gate

**Files:**
- Create: `src/copytrader/research/__init__.py`
- Create: `src/copytrader/research/csv_feed.py`
- Create: `src/copytrader/research/report.py`
- Create: `src/copytrader/research/runner.py`
- Create: `tests/unit/research/test_csv_feed.py`
- Create: `tests/unit/research/test_report.py`
- Create: `tests/integration/research/test_replay_runner.py`
- Create: `tests/property/research/test_prefix_invariance.py`

**Interfaces:**
- Consumes: CSV fields `timestamp,symbol,expiry,open,high,low,close,volume,open_interest,lot_size,tick_size,multiplier` and Tasks 1-8
- Produces: `CsvCandleFeed.read(path: Path) -> Iterator[Candle]`
- Produces: `ReplayRunner.run(request: ReplayRequest) -> ReplayReport`
- Produces: per-instrument/side/regime metrics for trade count, expectancy, profit factor, win rate, average win/loss, maximum drawdown, exposure, consecutive losses, and fee/slippage totals
- Produces: no-trade and simple 15-minute directional baseline comparisons over the same replay window

- [ ] **Step 1: Write failing strict CSV tests**

Test Decimal parsing from strings, timezone requirement, ordering, malformed rows, duplicate conflicts, missing metadata, and content hash recorded in replay inputs.

- [ ] **Step 2: Write failing report tests**

Use a small fixed trade ledger and assert every metric exactly, including zero-loss profit-factor representation, drawdown from the equity high-water mark, per-segment totals, and both baseline comparisons.

- [ ] **Step 3: Write failing whole-engine prefix-invariance test**

Run each generated valid prefix, append arbitrary valid suffix bars, and assert every earlier journal record, target, and fill remains unchanged.

- [ ] **Step 4: Implement feed, runner, and report**

The runner records dataset hash, config version, instrument version, fee model, slippage model, and seed. It emits separate NIFTY/BANKNIFTY and long/short breakdowns.

- [ ] **Step 5: Verify and commit**

Run research tests, full strategy/simulator tests, Ruff, and mypy; commit as `feat: add deterministic SMC replay reports`.

---

### Task 10: Milestone verification and operating documentation

**Files:**
- Create: `scripts/verify_smc_strategy.ps1`
- Create: `docs/runbooks/smc-replay.md`
- Create: `docs/evidence/smc-strategy.md`
- Modify: `README.md`
- Create: `tests/unit/scripts/test_verify_smc_strategy.py`

**Interfaces:**
- Consumes: all earlier tasks
- Produces: one local verification command and an evidence map from every spec acceptance item to a test or replay report

- [ ] **Step 1: Write the failing verification-script parse test**

Assert `scripts/verify_smc_strategy.ps1` exists, parses without PowerShell errors, and contains the exact unit, property, integration, Ruff, and mypy commands.

- [ ] **Step 2: Implement the verification script and runbook**

The runbook documents the CSV schema, exchange timezone, metadata, fee/slippage configuration, deterministic seeds, replay command, output interpretation, and the prohibition on treating replay success as live-money authorization.

- [ ] **Step 3: Map every acceptance criterion to evidence**

`docs/evidence/smc-strategy.md` distinguishes automated implementation gates from the external-data research gate. It must mark the 200-trade unseen-data, profit-factor, expectancy, and drawdown criteria as `PENDING DATASET EVALUATION` until a licensed dataset is supplied and evaluated; it must not fabricate results.

- [ ] **Step 4: Run the complete implementation gate**

Run:

```powershell
& '.\scripts\verify_smc_strategy.ps1'
git status --short
```

Expected: all automated tests, lint, formatting, and types pass; only intended documentation/evidence files remain changed.

- [ ] **Step 5: Commit**

```powershell
git add scripts/verify_smc_strategy.ps1 docs/runbooks/smc-replay.md docs/evidence/smc-strategy.md README.md tests/unit/scripts/test_verify_smc_strategy.py
git commit -m "docs: add SMC replay verification gate"
```

---

## Completion gate

Implementation is complete when Tasks 1-10 pass from a clean checkout and a fresh reviewer confirms spec coverage, independent implementation, deterministic replay, no-lookahead behavior, and conservative execution semantics. Research acceptance remains explicitly incomplete until the user supplies or authorizes access to a licensed historical NIFTY/BANKNIFTY futures dataset and the unseen-data report satisfies the numerical gate in the specification.
