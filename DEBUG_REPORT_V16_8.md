# Alpha Radar V16.8 — Panic Reversal Debug Report

## Goal

Add a disciplined contrarian execution path for broad US/Taiwan selloffs without turning "down a lot" into an automatic buy signal and without contaminating the frozen short/mid/long ranking model.

## Architecture preserved

- `return_first_model.py`: unchanged
- `policy_engine.py`: unchanged
- short/mid/long `SELECTION_WEIGHTS`: unchanged
- MIS verified executable-price rule: unchanged
- completed-day / intraday separation: unchanged

New/changed downstream files:

- `global_context.py` — US/global overnight stress and cache
- `strategy_config.py` — global/panic execution thresholds only
- `intraday_engine.py` — rebound-from-low features and panic-resilience discovery
- `recommendation_engine.py` — strict `PANIC_REVERSAL` route and global risk sizing
- `doctor_engine.py` — single-stock buy diagnosis supports the new route
- `app.py` — transparent global-factor display and panic-route labels

## Safety invariants

A panic reversal cannot qualify merely because the stock fell.  It is rejected when:

- the market is not in a broad shock;
- price is at/below the model stop;
- price is near limit-down;
- there is no rebound from the intraday low;
- relative performance vs the Taiwan benchmark is weak;
- expected return / reward-risk / evidence gates fail.

Global context changes risk budget, not stock-selection quality.

## Offline QA executed

`python run_full_qa.py` passed after the final code change.

Legacy groups passed:

- architecture regression (23 cases)
- Doctor regression (8 cases)
- verified-price regression (5 cases)
- V16.7.2 full-QA regression (10 cases)
- research data/detail/UI regressions
- app import / decision views / all-view / app-main smokes

V16.8 panic tests passed:

1. crash context activates panic mode while normal context does not;
2. a high-quality stock down in a broad selloff can qualify after relative resilience + rebound;
3. a falling knife with no rebound is rejected;
4. a price below the structural stop is rejected;
5. no broad shock => no panic-reversal route;
6. global shock reduces ordinary entry size but leaves daily selection score unchanged;
7. full-market discovery computes a separate panic-resilience candidate score.

## Important limitation

The packaging container does not provide reliable live external connectivity.  Therefore the global yfinance fetch and TWSE/TPEx MIS live end-to-end path are coded defensively but not claimed as live-network validated here.  When global data is unavailable, the app returns `UNKNOWN` and keeps the Taiwan core model running; it does not invent neutral market data.
