# Alpha Radar V16.7.1 — Verified Execution Price Hotfix

## Incident reproduced from 2026-10-07 川湖 2059

Observed UI: Doctor displayed `13,100` as `盤中即時價` at about 13:17.
Public market references for the same session showed:
- previous close: 13,100
- current / limit-down price: 11,790
- change: -10.00%

This exposed a correctness bug in the realtime execution-price pipeline.

## Root cause

There were three independent ways a completed-day close could be mislabeled as live:

1. MIS conversion accepted undocumented `pz` / book-derived values when documented latest matched price `z` was absent.
2. The old MIS fallback could ultimately substitute `y` (previous close) as the current price.
3. `recommendation_engine._components(..., live=True)` and Doctor holding logic could fall back from missing intraday price to `stock.price` (the completed-day close) while still treating the object as live.

In addition, Doctor trusted the normal provider chain. A fast third-party realtime snapshot could therefore be used without an independent official-price verification step.

## V16.7.1 corrections

1. `TWSE/TPEx MIS` is authoritative for any price that can trigger Doctor BUY/SELL/HOLD decisions.
2. New `FinMindRealtimeClient.verified_snapshots()` calls official MIS first.
3. Provider-only realtime is never labeled official. If official verification fails, Doctor does not overlay it as an actionable live price.
4. MIS `z` is the only field treated as latest matched trade price. `y` remains previous close only; undocumented `pz` and book midpoint are not promoted to last trade.
5. `previous_close`, `limit_up`, and `limit_down` are retained as separate metadata.
6. `rerank_snapshot()` rejects a realtime row with no valid live price; it does not fall back to the completed-day close.
7. `recommendation_engine` live mode no longer substitutes `stock.price` when `intraday.price` is missing.
8. Doctor `live=True` now requires a finite intraday price, not merely a non-empty intraday dictionary.
9. The one-minute intraday execution pool is re-quoted through official MIS before buy-opportunity scoring. Full-market discovery can still use the faster provider chain, but trade decisions use verified prices.
10. UI explicitly labels `官方即時價` vs `完整日線基準價` and shows official verification status/time.

## Regression reproduction

Synthetic 川湖 case:
- completed-day close = 13,100
- official MIS latest matched price = 11,790
- model invalidation = 11,853.57

Expected V16.7.1 behavior:
- displayed actionable price = 11,790
- new-buy decision is blocked
- existing-holder decision breaches model stop and returns `SELL`
- price source contains `TWSE/TPEx MIS`

All assertions pass.

## Test results

- Python compileall: PASS
- V16.6 architecture regression: 23 tests PASS
- V16.7 Doctor regression: 8 tests PASS
- V16.7.1 verified-price regression: 5 tests PASS
- Research detail regression: PASS
- Official research-data regression: PASS
- Research UI axis regression: PASS
- Research preload regression: PASS
- App import smoke: PASS
- Decision views smoke: PASS
- App main smoke: PASS

## Model impact

No short/mid/long selection weights were changed. No historical analog formula was changed. No fundamental/flow ranking weight was changed. This release changes the execution-price trust boundary and removes an invalid fallback, so intraday decisions can legitimately differ from V16.7 when the previous version had mislabeled stale/missing quotes.
