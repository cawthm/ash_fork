# Research Summary: Options Flow Signal Analysis

> This document is updated incrementally as phases complete. Each section records methods, results, and decisions.

---

## Phase 0: Data Reconnaissance

**Status**: COMPLETE

### Schema

_Stock trades table_:
- Table name: `stock_trades`
- Key columns: `time` (timestamptz), `ticker` (text), `price` (numeric), `size` (int), `conditions` (text), `exchange` (int)
- Pre-computed features: `gkyz_vol_1min_annual`, `gkyz_vol_5min_annual` (realized vol), `vwap_exp_3min`, `vwap_exp_10min` (exponential VWAP), `dsecs` (numeric)
- Raw metadata: `id`, `participant_timestamp`, `sip_timestamp`, `sequence_number`, `tape`, `trf_id`, `trf_timestamp`, `correction`
- Row count: ~5.19 billion (from pg_stat)
- Date range: 2024-05-01 to 2025-05-24 (SPY/QQQ/NVDA/AMZN); 2024-07-01 to 2025-05-24 (NXPI/LULU)
- Indexes: `idx_stock_trades_ticker_time` (ticker, time DESC) ✅ valid; `idx_stock_trades_time` (time DESC) ✅ valid
- Table size: 869 GB data, 1.1 TB total with indexes

_Options trades table_:
- Table name: `options_data`
- Key columns: `time` (timestamptz), `underlying_ticker` (text), `raw_ticker` (text), `option_type` (text), `strike_price` (numeric), `option_price` (numeric), `option_volume` (numeric), `underlying_price` (numeric)
- Contract info: `exp_date` (date), `time_to_expiry` (numeric), `days_to_expiry` (numeric), `moneyness` (numeric)
- Greeks: `delta`, `gamma`, `theta`, `vega`, `rho` (all numeric)
- Volatility: `implied_vol` (numeric), `gkyz_vol_1min_annual`, `gkyz_vol_5min_annual`
- Alignment metadata: `lag_ms`, `min_distance_ms`, `avg_distance_ms`, `buckets_used`
- Theoretical prices: `theoretical_price_1m`, `theoretical_price_5m`
- **NOTE: No trade direction/side column** — cannot directly sign flow as buy vs. sell
- Row count: ~983 million (from pg_stat)
- Date range: 2024-05-01 to 2025-05-23 (confirmed via physical page reads)
- Indexes: `idx_options_time` (time DESC) ✅ NOW VALID (18 GB); `idx_options_underlying_time` (underlying_ticker, time DESC) ❌ INVALID (failed build, 0 bytes)
- Table size: 257 GB data, 275 GB total
- Contains ~50+ underlying tickers (not just our 6 targets)

### ~~Critical Finding: Options Timestamps Are Processing Time~~ — RESOLVED

**Original diagnosis** (2026-02-19): The `time` column appeared to show batch processing timestamps — no data during RTH, all data in a ~3hr batch window post-close.

**Corrected diagnosis** (2026-02-20): The timestamps are the **correct original SIP trade times**, but shifted by +5hr (CDT summer) or +6hr (CST winter) due to a timezone bug in the R→PostgreSQL loading pipeline.

**Root cause**: The R loader (`/home/cawthm/R_projects/new_db_setup/vectorized_options_processor4.R:712`) correctly computes `time = as.POSIXct(sip_timestamp / 1e9, tz = "UTC")` from the original Polygon SIP timestamps. However, `RPostgreSQL::dbWriteTable()` sends bare timestamp strings without timezone qualifiers. PostgreSQL (server timezone = America/Chicago) interprets these as local CDT/CST time instead of UTC.

**Verification**: Traced specific options trades from raw Polygon CSVs (at `/mnt/external/polygon_data/us_options_opra/trades_v1/`) through to the DB. Example: `O:NVDA250417C00110000` first trade has `sip_timestamp = 1744291800007000000` = 09:30:00.007 ET in raw CSV, but appeared as 14:30:00.007 ET in DB (+5hr CDT shift). Option prices, volumes, and sub-second timing match exactly.

**Fix applied**:
1. Created new table with corrected timestamps: `(time AT TIME ZONE 'America/Chicago') AT TIME ZONE 'UTC'` — handles DST automatically
2. R loader fixed: added `dbExecute(con, "SET timezone = 'UTC'")` after `dbConnect()` in `processor.R`
3. All other columns (underlying_price, lag_ms, Greeks, IV) were always correct — computed from the right `sip_timestamp` in R before the timezone conversion bug

**Timestamps are now millisecond-precise** — they are the original SIP feed timestamps from Polygon data. This is sufficient for all planned bucket sizes (5s–300s).

### Infrastructure

1. **`idx_options_time` (time DESC)** — VALID, rebuilt on corrected table.
2. **`idx_options_implied_vol`** — VALID, rebuilt on corrected table.
3. **`idx_options_underlying_time`** — dropped (was INVALID). Can be rebuilt if per-ticker time queries are too slow.
4. Table swapped via CREATE TABLE AS (983M rows, ~3hr) + index builds + rename.

### Ticker Selection & Density

_Stock trades density (RTH = 9:30-16:00 ET, single day 2025-05-20):_

| Ticker | Type | Stock trades/day (RTH) | Stock trades/sec (RTH) | Options trades/sec (est.) | Stock date range | Viable buckets |
|--------|------|------------------------|------------------------|---------------------------|------------------|----------------|
| SPY    | ETF  | 166,637                | 7.12                   | TBD (index building)      | 2024-05-01 – 2025-05-24 | TBD |
| QQQ    | ETF  | 164,720                | 7.04                   | TBD (index building)      | 2024-05-01 – 2025-05-24 | TBD |
| NVDA   | Mega | 535,909                | 22.90                  | TBD (index building)      | 2024-05-01 – 2025-05-24 | TBD |
| AMZN   | Mega | 126,494                | 5.41                   | TBD (index building)      | 2024-05-01 – 2025-05-24 | TBD |
| NXPI   | Mid  | 6,665                  | 0.28                   | ~0.01 (221 trades/day)    | 2024-07-01 – 2025-05-24 | ❌ likely none |
| LULU   | Mid  | 10,762                 | 0.46                   | ❌ NOT IN OPTIONS DATA    | 2024-07-01 – 2025-05-24 | ❌ none |

_Notes:_
- Stock trades include substantial after-hours volume (73% of SPY daily trades are outside RTH). All density figures above are RTH-only.
- NXPI options density (~0.01/sec = ~221 trades/day) means even at 300s buckets, median options/bucket ≈ 2.8, below the viability threshold of 3.
- LULU has zero rows in `options_data`. Dropped from grid.
- Options density can now be measured with corrected timestamps. Preliminary (NVDA, 2025-04-10): 566K options during RTH = ~24/sec, well above viability threshold for all bucket sizes.

Viability threshold: median ≥ 3 options trades per bucket

### Data Quality Notes

**Stock trades (SPY, 2025-05-20 sample, 227,875 rows):**
- Zero null prices or sizes
- 87 null GKYZ vol values (0.04%) — likely at session boundaries where lookback is incomplete
- Price range: $550.25 – $595.04 (intraday range reasonable for SPY)
- Size range: 1 – 131,308 (large block trades present but expected)

**Options data (from physical page samples, ~29K rows each):**
- SPY sample: 0 null deltas, 166 null implied_vol (0.57%), 0 deltas > |1.01|, 0 negative DTE. IV range 0.046–2.48. Median lag_ms=124, p99=1010ms. 95% near-ATM (moneyness 0.95–1.05). 51% calls / 49% puts.
- NVDA sample: 0 null deltas, 0 null IVs, 0 deltas > |1.01|, 0 negative DTE. Median lag_ms=56, p99=256ms.
- No trade direction/side column — delta flow will use option delta × volume as a directional proxy (signed by option_type: calls positive, puts negative).

**Timezone note:** Both stock_trades and options_data timestamps are now correct `timestamptz` values. Filter RTH using ET-aware queries. Server timezone is `America/Chicago` (CDT=UTC-5 summer, CST=UTC-6 winter).

---

## Phase 1: Unconditional Signal

**Status**: Not started

### Delta Flow → Forward Returns (Correlation Grid)

_Rows = tickers, Columns = bucket sizes. Cells = Pearson r (sign accuracy %)._

| Ticker | 5s | 15s | 30s | 60s | 300s |
|--------|-----|------|------|------|-------|
| SPY    |     |      |      |      |       |
| QQQ    |     |      |      |      |       |
| NVDA   |     |      |      |      |       |
| AMZN   |     |      |      |      |       |
| NXPI   |     |      |      |      |       |
| LULU   |     |      |      |      |       |

_Note: cells left blank where (ticker, bucket_size) pair is not viable._

### Forward Horizon Sensitivity (best ticker/bucket pair)

| Horizon    | Pearson r | p-value | Sign Accuracy | N |
|------------|-----------|---------|---------------|---|
| 1× bucket  |           |         |               |   |
| 2× bucket  |           |         |               |   |
| 5× bucket  |           |         |               |   |
| 10× bucket |           |         |               |   |

### Interpretation

- (is there an unconditional signal? how strong? which horizon?)

### Per-Ticker Progress Log

> Updated after each ticker completes. Detailed results in `results/{TICKER}_phase1.csv`.

**SPY**: _not started_

**QQQ**: _not started_

**🔍 Review gate (after SPY + QQQ)**: _pending_

**NVDA**: _not started_

**AMZN**: _not started_

**NXPI**: _not started_

**LULU**: _not started_

---

## Phase 1b: Secondary Features

**Status**: Not started

| Feature          | Corr w/ fwd_1min | Sign Accuracy |
|------------------|-------------------|---------------|
| net_delta_flow   |                   |               |
| pc_ratio         |                   |               |
| volume_surprise  |                   |               |
| rv_1min          |                   |               |
| atm_iv           |                   |               |

---

## Phase 2: Conditional Structure

**Status**: Not started

### Stratified Correlations

| Condition           | Corr (flow → fwd_1min) | N    |
|---------------------|------------------------|------|
| Open (9:30–9:45)    |                        |      |
| Morning (9:45–10:30)|                        |      |
| Midday (10:30–15:00)|                        |      |
| Late (15:00–15:30)  |                        |      |
| Close (15:30–16:00) |                        |      |
| High RV             |                        |      |
| Low RV              |                        |      |
| High Volume         |                        |      |
| Low Volume          |                        |      |

### Model Comparison

| Model              | Accuracy | AUC  | Log-Loss |
|---------------------|----------|------|----------|
| Logistic Regression |          |      |          |
| XGBoost             |          |      |          |

### XGBoost Feature Importances

1. ???
2. ???
3. ???

### Decision

- Does conditional structure exist? `???`
- Proceed to Phase 3? `???`

---

## Phase 3: Temporal Dependence

**Status**: Not started

_(to be filled if Phase 2 warrants)_

---

## Phase 4: Next-Token Framework

**Status**: Not started

_(to be filled if Phase 3 warrants)_

---

## Running Conclusions

- **Phase 0 complete**: Data reconnaissance done. Timestamps fixed (timezone bug in R→PG loader, not batch processing). All data now has correct millisecond-precision trade times.
- **Both tables are clean**: stock_trades and options_data timestamps are correct. Greeks, IV, underlying_price, lag_ms all verified.
- **Grid reduction confirmed**: LULU (no options data) and NXPI (~0.01 options/sec) are non-viable. Grid reduced to SPY, QQQ, NVDA, AMZN.
- **Ready for Phase 1**: No remaining blockers. Proceed with unconditional signal analysis.
