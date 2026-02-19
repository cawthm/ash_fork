# Research Summary: Options Flow Signal Analysis

> This document is updated incrementally as phases complete. Each section records methods, results, and decisions.

---

## Phase 0: Data Reconnaissance

**Status**: Not started

### Schema

_Stock trades table_:
- Table name: `stock_trades`
- Key columns: `time` (timestamptz), `ticker` (text), `price` (numeric), `size` (int), `conditions` (text), `exchange` (int)
- Pre-computed features: `gkyz_vol_1min_annual`, `gkyz_vol_5min_annual` (realized vol), `vwap_exp_3min`, `vwap_exp_10min` (exponential VWAP), `dsecs` (numeric)
- Raw metadata: `id`, `participant_timestamp`, `sip_timestamp`, `sequence_number`, `tape`, `trf_id`, `trf_timestamp`, `correction`
- Row count: `TBD`
- Date range: `TBD`

_Options trades table_:
- Table name: `options_data`
- Key columns: `time` (timestamptz), `underlying_ticker` (text), `raw_ticker` (text), `option_type` (text), `strike_price` (numeric), `option_price` (numeric), `option_volume` (numeric), `underlying_price` (numeric)
- Contract info: `exp_date` (date), `time_to_expiry` (numeric), `days_to_expiry` (numeric), `moneyness` (numeric)
- Greeks: `delta`, `gamma`, `theta`, `vega`, `rho` (all numeric)
- Volatility: `implied_vol` (numeric), `gkyz_vol_1min_annual`, `gkyz_vol_5min_annual`
- Alignment metadata: `lag_ms`, `min_distance_ms`, `avg_distance_ms`, `buckets_used`
- Theoretical prices: `theoretical_price_1m`, `theoretical_price_5m`
- **NOTE: No trade direction/side column** — cannot directly sign flow as buy vs. sell
- Row count: `TBD`
- Date range: `TBD`

### Ticker Selection & Density

| Ticker | Type | Stock trades/sec | Options trades/sec | Date range | Viable buckets |
|--------|------|------------------|--------------------|------------|----------------|
| SPY    | ETF  |                  |                    |            |                |
| QQQ    | ETF  |                  |                    |            |                |
| NVDA   | Mega |                  |                    |            |                |
| AMZN   | Mega |                  |                    |            |                |
| NXPI   | Mid  |                  |                    |            |                |
| LULU   | Mid  |                  |                    |            |                |

Viability threshold: median ≥ 3 options trades per bucket

### Data Quality Notes

- (nulls, gaps, outliers, etc.)

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

- (updated after each phase)
