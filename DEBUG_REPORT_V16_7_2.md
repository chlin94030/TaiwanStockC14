# Alpha Radar V16.7.2 — Full QA / Verified Price Debug Report

## 目的

V16.7.2 是針對 V16.7.1 的完整 QA 強化版。短／中／長線選股核心與權重不變；本版只處理執行價可信度、盤中缺價降級、個股診斷狀態一致性，以及所有主要 UI surface 的回歸測試。

## 本次再次檢查時找到的 4 個邊界問題

1. **MIS 批次成功 ≠ 每一檔都有可執行最新成交價**
   - 舊版只要 MIS 回傳任何 rows，batch status 就可能為 verified。
   - 若特定股票只有昨收 `y`、最新成交 `z` 為 `-`，Doctor 仍可能看到 batch-level verified。
   - V16.7.2 改為：requested ticker 必須有 `z > 0` 才算 executable；若 0 檔有有效成交價，整批 execution status 直接 unavailable。

2. **盤中已有持股缺即時價時，舊版仍可能用完整日線收盤做 HOLD/SELL**
   - 買入路徑原本已會降級成 DATA，但持股路徑仍會以 completed-day price 做判斷。
   - V16.7.2 改為：盤中沒有官方驗證成交價時，持股決策只回 `DATA`，不給續抱／減碼／賣出的即時動作；完整日線價格只標為參考。

3. **live row UI 還留有 completed-day fallback**
   - `_row(..., live=True)` 原本仍有 `live_price or stock.price`。
   - V16.7.2 完全移除；live surface 沒有 live price 就顯示缺值，不允許昨收冒充盤中價。

4. **Doctor hotfix 後 session 可能殘留舊診斷結果**
   - 核心 selection version 不變是刻意的，避免重跑 1000 檔模型；但這也可能讓舊 `alpha_doctor` 留在同一 Streamlit session。
   - 新增 `alpha_doctor_version = DOCTOR_VERSION`，只清 Doctor state，不清完整市場 snapshot。

## 價格可信度規則（V16.7.2）

- `MIS z`：唯一可作為官方最新成交價的欄位。
- `MIS y`：只能當昨收，永遠不能 fallback 成 live price。
- broad discovery 可以使用 provider quote，但只負責「發現」；不能直接觸發 BUY/SELL/HOLD。
- 盤中可執行買入與 Doctor 買／抱／賣，必須先通過 TWSE/TPEx MIS 官方成交價驗證。
- partial batch 允許有效股票繼續判斷，但缺價股票會被排除。
- 盤中缺價時，買入與持股都回 DATA，不以昨收替代。

## 完整回歸測試

執行 `python run_full_qa.py`：PASS。

涵蓋：
- Python compileall
- 23 項 architecture regression
- 8 項 Doctor regression
- 5 項 V16.7.1 verified-price regression
- 10 項 V16.7.2 full-QA regression
- V16.6.2 research data / preload / UI axis regression
- app import smoke
- intraday + next-session decision smoke
- 總覽／短中長／盤中買點／明日關注／Doctor 買入／Doctor 持股 surface smoke
- app main smoke

## V16.7.2 新增的關鍵 regression

- MIS 只有昨收、無 `z` → execution unavailable。
- 同批 2 檔只有 1 檔有 `z` → 只保留有效股票，status 顯示 partial coverage。
- `evaluate_buy_opportunity(live=True)` 無 live price → DATA，不能 fallback completed close。
- Doctor 買入盤中缺價 → DATA。
- Doctor 持股盤中缺價 → DATA，不得用昨收輸出 HOLD/SELL。
- full-market discovery 遇到無最新成交價 row → 略過。
- Doctor batch-level verified 但特定 ticker 無價 → 強制降級 `NO_EXECUTABLE_PRICE_FOR_TICKER`。
- UI source code 不再存在 `live_price or stock.price`。
- Doctor version 更新只清診斷 state，不迫使完整市場重算。
- selection core 權重與 V16.6 frozen core 完全不變。

## 效能

`benchmark_v16_6.py`（1500 檔 synthetic quote、detail 220、refresh 60s）：

- dynamic pool median: 0.5776s
- p95: 0.6442s
- max: 0.6656s

此為純 CPU synthetic benchmark，不含網路下載時間。

## 真實網路／歷史驗效狀態

本容器 DNS 仍無法連 `mis.twse.com.tw`。直接 MIS request 回 `ConnectionError / NameResolution`。

`validate_doctor_2026.py --stocks 5 --points 10` 實際嘗試後正確回 `NOT_RUN`，原因是上市／上櫃股票母體均為 0；沒有用 synthetic data 冒充真實績效。

因此本報告可以確認：**程式邏輯、價格來源防呆、各 UI surface 與離線整合測試正常**；不能聲稱已在本容器完成真實 Streamlit Cloud + TWSE/TPEx/MIS end-to-end，也不能聲稱已證明未來投資報酬。

GitHub 內仍保留：
- `Validate 2026 walk-forward`
- `Validate V16.7 doctor 2026`
- 新增 `QA V16.7.2`

部署後可分別驗證真實歷史績效與每次程式更新的 offline regression。
