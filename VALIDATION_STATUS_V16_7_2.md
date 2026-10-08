# V16.7.2 Validation Status

## PASS
- selection / entry / intraday / holding 分層
- completed-day / intraday price separation
- TWSE/TPEx MIS `z`-only executable price rule
- partial/missing MIS price handling
- Doctor buy/hold no-live-price DATA downgrade
- full-market discovery skips no-trade rows
- 60-second intraday refresh architecture
- research revenue / EPS / institutional detail regressions
- all major app surfaces smoke tested
- compileall / API contracts / UTF-8 / session-state migration

## Selection core
Unchanged from `v16.6.0-dynamic-intraday`.  V16.7.2 does not retune short/mid/long ranking weights.

## Real-market validation
NOT RUN in this container because external DNS/network access to TWSE/TPEx/MIS is unavailable. The validators refuse to substitute synthetic data for real-market performance.

Run the bundled GitHub Actions workflows for networked 2026 walk-forward validation.
