# Implementation Plan

## Batch Execution Strategy

> **Core principle**: one ticker at a time, bounded date ranges, inspectable checkpoints.

- Queries pull **one ticker** at a time, never the full grid in one shot.
- Default date range per batch: **1 trading day**. Widen to 5 days only after confirming query completes in <30s.
- After each ticker completes a phase, append results to `SUMMARY.md` and to a per-ticker results file (`results/{TICKER}_phase{N}.csv`) so progress is visible.
- Ticker execution order: **SPY → QQQ → NVDA → AMZN** (most liquid first; NXPI dropped for insufficient options density, LULU dropped for no options data).
- After SPY and QQQ complete Phase 1, **pause and review**: if both show r ≈ 0 and accuracy ≈ 50%, the signal may not exist and we should discuss before burning time on the remaining four.
- If any query exceeds 30s, abort, narrow the date range or bucket size, and note the constraint.
- When/if signal is confirmed and we move to model building (Phase 2+), we can widen date ranges aggressively since the feature matrix construction will be scripted and tested.

## Phase 0: Data Reconnaissance

- [x] **0.1** Connect to TimescaleDB and confirm access (use creds from `.env`)
- [x] **0.2** List all tables, inspect schemas for stock trades and options trades tables
- [x] **0.3** Identify key columns: timestamp, symbol, price, size, delta, gamma, IV, trade direction/side
- [x] **0.4** Check date range coverage per ticker (stock_trades: used index-based LIMIT 1 queries; full COUNT(*) infeasible on 5.2B rows — used pg_stat approx)
- [x] **0.5** Same for options_data — date range confirmed via physical page reads (2024-05-01 to 2025-05-23). Per-ticker density blocked by timestamp issue (see 0.11).
- [x] **0.6** Compute data density for each ticker: stock trades/sec done (RTH, 2025-05-20). Options density now measurable with corrected timestamps. NVDA ~24/sec RTH. NXPI ~0.01/sec, LULU not in options_data.
- [x] **0.7** Determine viable bucket sizes: SPY/QQQ/NVDA/AMZN all viable for 5s–300s buckets (sufficient density). NXPI/LULU dropped.
- [x] **0.8** Check for nulls, outliers, and data gaps — stock trades clean (0 null prices/sizes, 0.04% null GKYZ); options samples clean (0 null deltas, <0.6% null IVs, all deltas ≤ |1|, reasonable lag_ms)
- [x] **0.9** Document schema, data profile in `SUMMARY.md` — updated with critical timestamp finding.
- [x] **0.10** (unplanned) Discovered and cleaned up: invalid options_data index, 36 stale queries from prior sessions. New composite index failed; `idx_options_time` now valid.
- [x] **0.11** (unplanned) ~~**CRITICAL**: Discovered that options_data `time` column is batch processing time~~ **RESOLVED**: The `time` column is the correct trade timestamp but with a timezone bug. R's `dbWriteTable` sent UTC timestamps as bare strings; PostgreSQL (server tz=America/Chicago) misinterpreted them as local time, shifting +5hr (CDT) or +6hr (CST). Fix: `(time AT TIME ZONE 'America/Chicago') AT TIME ZONE 'UTC'`. Verified against raw Polygon CSVs to millisecond precision. R loader fixed with `SET timezone = 'UTC'` on connection.
- [x] **0.12** (unplanned) Applied fix to all 983M rows via CREATE TABLE AS + swap. All other columns (underlying_price, lag_ms, Greeks, IV) were always correct.

## ~~Phase 0.5: Options Timestamp Reconstruction~~ — SKIPPED

> Phase 0.5 is no longer needed. The timestamp issue was a timezone bug in the R→PostgreSQL loading pipeline, not missing trade times. Fixed in-place. Timestamps are now correct to millisecond precision (original SIP timestamps from Polygon data).

## Phase 1: Unconditional Signal Analysis

> Execute per-ticker in order: SPY → QQQ → NVDA → AMZN.
> Each ticker: start with 1 trading day, widen only if queries are fast.
> Write results to `results/{TICKER}_phase1.csv` and update `SUMMARY.md` after each ticker.

- [ ] **1.1** Create `results/` directory
- [ ] **1.2** **SPY**: For each viable bucket size, construct bucketed delta flow and forward returns for 1 day. Compute correlations and sign accuracy. Save to `results/SPY_phase1.csv`. Update `SUMMARY.md`.
- [ ] **1.3** **QQQ**: Same as 1.2. Save to `results/QQQ_phase1.csv`. Update `SUMMARY.md`.
- [ ] **1.4** **REVIEW GATE**: Inspect SPY and QQQ results. If both show |r| < 0.005 and sign accuracy < 50.2% across all bucket sizes, flag for human review before continuing. Otherwise proceed.
- [ ] **1.5** **NVDA**: Same. Save to `results/NVDA_phase1.csv`. Update `SUMMARY.md`.
- [ ] **1.6** **AMZN**: Same. Save to `results/AMZN_phase1.csv`. Update `SUMMARY.md`.
- [ ] **1.7** Compile cross-ticker summary: which (ticker, bucket_size) pairs show signal, if any. Identify patterns (ETFs vs. singles, short vs. long buckets). Update `SUMMARY.md`.

## Phase 1b: Secondary Features

- [ ] **1b.1** Construct put-call volume ratio per 1-min bucket
- [ ] **1b.2** Construct volume surprise: current volume / trailing 10-min avg volume
- [ ] **1b.3** Construct 1-min realized volatility from stock trades
- [ ] **1b.4** Extract ATM implied volatility (nearest-strike, nearest-expiry option IV) per bucket
- [ ] **1b.5** Correlate each feature individually with forward returns
- [ ] **1b.6** Record results in `SUMMARY.md`

## Phase 1.5: Analysis Infrastructure

> Before modeling, build the scripts and environment needed to go from DB queries to fitted models.

- [ ] **1.5.1** Create a Python virtual environment and install dependencies: `psycopg2`, `pandas`, `numpy`, `scipy`, `scikit-learn`, `xgboost`. Record versions in a `requirements.txt`.
- [ ] **1.5.2** Write `db_utils.py`: a helper module that reads `.env`, connects to TimescaleDB, and exposes a `query_to_df(sql) -> pd.DataFrame` function with timeout handling.
- [ ] **1.5.3** Write `feature_builder.py`: given a **single ticker**, **single date** (or small date range), and bucket size, queries both tables and returns a single DataFrame with one row per bucket. Columns: `bucket_start`, `fwd_return_1x`, `fwd_return_2x`, ..., `call_delta_flow`, `put_delta_flow`, `net_delta_flow` (call − put), `pc_ratio`, `volume_surprise`, `rv`, `atm_iv`. This is the canonical feature matrix used by all downstream analysis. Must complete in <30s for one ticker-day.
- [ ] **1.5.4** Write `signal_tests.py`: given a feature DataFrame, computes correlations, sign accuracy, and p-values for each feature × horizon pair. Outputs a results dict or CSV.
- [ ] **1.5.5** Write `model_runner.py`: given a feature DataFrame and a target column, fits logistic regression and XGBoost with walk-forward cross-validation (train on days 1..N, test on day N+1). Reports accuracy, AUC, log-loss per fold and overall.
- [ ] **1.5.6** Validate the full pipeline end-to-end on one (ticker, bucket_size, single day) before running the grid.

## Phase 2: Conditional Structure

- [ ] **2.1** Using `feature_builder.py`, build feature matrices for all viable (ticker, bucket_size) pairs across available date range
- [ ] **2.2** Using `signal_tests.py`, stratify Phase 1 correlations by time-of-day (9:30–9:45, 9:45–10:30, 10:30–15:00, 15:00–15:30, 15:30–16:00)
- [ ] **2.3** Stratify by realized volatility regime (above/below median RV over trailing hour)
- [ ] **2.4** Stratify by volume regime (above/below median volume)
- [ ] **2.5** Using `model_runner.py`, run logistic regression on each viable (ticker, bucket_size) pair
- [ ] **2.6** Using `model_runner.py`, run XGBoost on same
- [ ] **2.7** Compare accuracy, AUC, and log-loss between logistic and XGBoost across the grid
- [ ] **2.8** Extract XGBoost feature importances and top interaction effects
- [ ] **2.9** Decision: does XGBoost meaningfully beat logistic? Record in `SUMMARY.md`

## Phase 3: Temporal Dependence (only if Phase 2 shows conditional structure)

- [ ] **3.1** Compute autocorrelation of net_delta_flow at lags 1–20 buckets
- [ ] **3.2** Test whether a sequence of flow buckets (e.g., last 5) predicts better than current bucket alone
- [ ] **3.3** Simple LSTM or 1D-CNN on flow sequences → forward return
- [ ] **3.4** Compare to XGBoost with lagged features (flow_t-1, flow_t-2, ..., flow_t-5)
- [ ] **3.5** Decision: do lagged features add meaningful lift? Record in `SUMMARY.md`

## Phase 4: Next-Token Framework (only if Phase 3 confirms temporal value)

- [ ] **4.1** Define token schema: what goes in each discrete time-slice
- [ ] **4.2** Define vocabulary / encoding scheme (continuous→discrete)
- [ ] **4.3** Build tokenizer and dataset pipeline
- [ ] **4.4** Train small transformer on next-token prediction
- [ ] **4.5** Evaluate: does generative model beat discriminative baselines?
- [ ] **4.6** Final write-up in `SUMMARY.md`

---

## Notes for Headless Execution

- **One ticker at a time, always.** Never query multiple tickers in a single SQL statement.
- **Start with 1 trading day per ticker.** Only widen after confirming the query runs in <30s.
- **Write results after every ticker**, not after every phase. `results/{TICKER}_phase{N}.csv` plus `SUMMARY.md` update.
- Phase 0 is complete. Options timestamps are now correct (timezone bug fixed, ms-precision SIP times).
- Phases 1–1b are pure SQL queries via MCP or `db_utils.py` — no modeling code needed yet.
- Phase 1.5 builds the Python infrastructure. Validate end-to-end on one (ticker, bucket, single day) before running the grid.
- Phases 2+ use the scripts from 1.5. Do not write one-off analysis code; extend the existing modules.
- If a query takes >30s, abort and reduce scope (fewer rows, shorter date range, larger bucket)
- If a phase yields a clear negative (no signal), document it and skip subsequent phases
- **Review gates** are built into the plan. When you hit one, write a clear assessment in `SUMMARY.md` and stop if the gate condition is met.

### File inventory

| File | Purpose |
|------|---------|
| `data_path.env` | DB creds, table names, grid parameters |
| `SPEC.md` | Scope and rationale (read-only reference) |
| `IMPLEMENTATION_PLAN.md` | This file — check off tasks as completed |
| `SUMMARY.md` | Living results document — update after each step |
| `requirements.txt` | Python dependencies (created in 1.5.1) |
| `db_utils.py` | DB connection helper (created in 1.5.2) |
| `feature_builder.py` | Feature matrix construction (created in 1.5.3) |
| `signal_tests.py` | Correlation / sign-accuracy tests (created in 1.5.4) |
| `model_runner.py` | Logistic + XGBoost fitting (created in 1.5.5) |
| `results/` | Per-ticker CSV outputs: `{TICKER}_phase{N}.csv` (created in 1.1) |
