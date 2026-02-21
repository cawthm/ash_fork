# Ash Fork — Agent Guide

Research project: does options market flow predict short-term equity returns? See `SPEC.md` for the research question, `IMPLEMENTATION_PLAN.md` for the task checklist (check off tasks as you complete them), and `SUMMARY.md` for the living results document (append findings after each step).

**Current status**: Phase 0 complete. Phase 1 (unconditional signal analysis) is next.

## File Layout

| File | Role |
|------|------|
| `SPEC.md` | Research scope and rationale (read-only reference) |
| `IMPLEMENTATION_PLAN.md` | Phased task checklist — mark tasks done as you go |
| `SUMMARY.md` | Living results — append after every ticker/step |
| `data_path.env` | DB credentials — **do NOT commit** |
| `results/{TICKER}_phase{N}.csv` | Per-ticker output files |
| `db_utils.py`, `feature_builder.py`, etc. | Python modules (created in Phase 1.5) |

## Database

Connection: `localhost:5432/datawarehouse` — credentials in `data_path.env`.

Server timezone is `America/Chicago` (CDT = UTC-5 summer, CST = UTC-6 winter). The DB stores `timestamptz`. Use ET-aware queries for RTH filtering (9:30–16:00 ET). EDT = UTC-4 (Mar–Nov), EST = UTC-5 (Nov–Mar). Do NOT use fixed UTC offsets year-round.

### Tables

**`stock_trades`** — ~5.2B rows, 869GB
- Indexes: `(ticker, time DESC)`, `(time DESC)` — both valid
- Timestamps are correct (original SIP feed times, ms precision)
- Includes after-hours trades (~73% of SPY daily volume is outside RTH). Always filter for RTH when computing density or building features.

**`options_data`** — ~983M rows, 257GB
- Index: `(time DESC)` — valid
- Timestamps are correct (fixed 2026-02-20; were previously shifted by a timezone bug)
- Contains ~50+ underlyings, not just target tickers
- Pre-computed Greeks (delta, gamma, theta, vega, rho), implied_vol, underlying_price, lag_ms
- **No trade direction/side column** — use `delta × volume` signed by option_type (calls positive, puts negative) as directional flow proxy

### Query Rules

- **NEVER** run unbound `COUNT(*)` on stock_trades or options_data — will timeout (>30 min). Use `SELECT n_live_tup FROM pg_stat_user_tables WHERE relname = 'table_name'` for approximate counts.
- **30-second timeout convention**: if a query exceeds 30s, abort and narrow scope (fewer rows, shorter date range, larger bucket). Do not retry the same query.
- Use the `(ticker, time DESC)` index on stock_trades for per-ticker time-range queries — fast.
- Use the `(time DESC)` index on options_data for time-range queries. Per-ticker filtering requires a row-level scan (no composite index). If per-ticker queries are slow, consider building `(underlying_ticker, time DESC)`.

## Data Pipeline

Raw Polygon data lives on disk at `/mnt/external/polygon_data/`:
- Options: `us_options_opra/trades_v1/YYYY/MM/YYYY-MM-DD.csv.gz` (columns: ticker, conditions, correction, exchange, price, sip_timestamp, size)
- Stocks: `us_stocks_sip/trades_v1/YYYY/MM/YYYY-MM-DD.csv.gz`
- Coverage: 2014-06-02 to 2025-05-23 (2,766 options files)

R loading code: `/home/cawthm/R_projects/new_db_setup/processor.R` calls `vectorized_options_processor4.R`. Pipeline: raw CSV → parse option tickers → align with stock VWAP buckets (100ms, 3s lookback) → compute Greeks via RQuantLib → `dbWriteTable` into TimescaleDB.

The `time` column in both tables comes from `sip_timestamp` (nanoseconds since epoch from the SIP feed). All other columns (underlying_price, lag_ms, Greeks, IV) are derived from the correct `sip_timestamp` in R.

## Verification Discipline

**Before concluding that data is broken or missing, cross-reference against the raw Polygon CSVs.** This is the single most important rule. The raw files have the ground-truth `sip_timestamp`. Convert to human-readable (`datetime.fromtimestamp(ts_ns / 1e9, tz=timezone.utc)`) and compare with what the DB shows.

- Always verify with specific traced examples: find the same trade in the raw CSV and in the DB.
- Do not extrapolate from hourly distributions alone — timezone shifts can mimic batch-loading patterns.
- Do not spend more than one session building a workaround without first validating the root cause against raw data.

## Research Workflow

1. **One ticker at a time**: SPY → QQQ → NVDA → AMZN (most liquid first).
2. **Start with 1 trading day** per ticker. Widen to 5 days only after confirming the query completes in <30s.
3. **Write results after every ticker**: `results/{TICKER}_phase{N}.csv` + update `SUMMARY.md`.
4. **Review gates** are built into `IMPLEMENTATION_PLAN.md`. When you hit one, write a clear assessment in `SUMMARY.md` and stop if the gate condition is met.
5. **If a phase yields a clear negative** (no signal), document it and skip subsequent phases.
6. Grid: SPY, QQQ, NVDA, AMZN × bucket sizes 5s, 15s, 30s, 60s, 300s. NXPI and LULU were dropped (insufficient options density / no data).

## Escalation

- If stuck or findings are ambiguous, write a clear assessment in `SUMMARY.md` and stop.
- For decisions that would change the research direction, create a `.blocked` file in `loopdocs/research/` explaining the situation and options.
- When in doubt, ask rather than assume. Incorrect conclusions cost more than a pause.
