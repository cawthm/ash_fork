# Signal Research: Options Flow → Short-Term Returns

## Objective

Determine whether options market activity (specifically net delta flow and related metrics) contains predictive information for short-term equity returns (seconds to minutes), and if so, characterize the nature of that signal (unconditional vs. conditional, linear vs. nonlinear, stationary vs. regime-dependent).

## Motivation

Options markets aggregate informed and hedging activity. Market makers delta-hedge, creating mechanical price impact. Informed traders may prefer options for leverage and limited downside. If either channel is detectable in the data, net options flow should lead stock returns at short horizons.

The key unknowns:
- **Signal existence**: Is there any unconditional correlation between options flow and forward returns?
- **Signal structure**: If yes, is it linear (regression captures it) or conditional (requires interaction terms, tree models, or sequence models)?
- **Signal decay**: How quickly does predictive power decay with lag? Seconds? Minutes?
- **Signal stability**: Is it consistent across regimes, times of day, and volatility environments?

## Data

- **Stock trades**: Millisecond-resolution trade data (price, size, exchange, conditions) stored in TimescaleDB
- **Options trades**: Millisecond-resolution with pre-computed Greeks (delta, gamma, vega, theta, IV) aligned to underlying stock prices

## Scope

### In Scope
- Grid search across tickers (SPY, QQQ, NVDA, AMZN, NXPI, LULU) and bucket sizes (5s, 15s, 30s, 60s, 300s)
- Univariate and bivariate signal analysis (flow vs. returns)
- Feature construction: call/put delta flow, put-call ratio, volume surprises, IV changes, realized volatility
- Baseline models: correlation analysis, logistic regression, XGBoost
- Time-of-day and volatility regime stratification
- Forward return horizons scaled to bucket size (1×, 2×, 5×, 10× bucket)
- Viability filtering: only (ticker, bucket_size) pairs with sufficient options density proceed past Phase 0

### Out of Scope (for now)
- Sequence models / next-token prediction (Phase 4 — only if justified)
- Multi-asset cross-signal analysis
- Live execution, slippage modeling, transaction cost analysis
- Order book / quote data (unless already in DB)

## Phased Approach

| Phase | Goal | Success Criterion |
|-------|------|-------------------|
| 0 | Data reconnaissance | Understand schema, coverage, gaps, grain |
| 1 | Unconditional signal | Measure raw correlation, establish baseline accuracy |
| 2 | Conditional structure | Identify interaction effects, regime dependence |
| 3 | Sequence dependence | Test whether temporal patterns in flow add value |
| 4 | Next-token framework | Only if Phases 2-3 confirm complex temporal structure |

## Key Assumptions

- Greeks in the options table are reasonably accurate (pre-computed, aligned to stock prices at trade time)
- Delta-hedging flow is a meaningful proxy for directional pressure
- The signal, if it exists, is small (think r = 0.02–0.05, accuracy 51–53%)
- We are looking for statistical robustness over large samples, not anecdotal patterns

## Decision Points

- **After Phase 0**: Do we have enough data density (trades/sec) in both tables for the chosen ticker(s)?
- **After Phase 1**: Is unconditional |r| > 0.01? If not, move directly to conditional analysis or reconsider.
- **After Phase 2**: Does XGBoost meaningfully beat logistic regression? If not, the signal is ~linear and we stop.
- **After Phase 3**: Do lagged flow patterns matter? If not, no need for sequence models.
