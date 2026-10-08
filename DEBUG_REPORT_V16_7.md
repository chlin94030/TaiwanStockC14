# Alpha Radar V16.7 — Doctor Decision Debug Report

## 使用者回報

個股診斷頁在「已有持股」模式同時顯示成本、原防守價、獲利保護價三個 `0.00`。手機版標籤對比也不足，使用者很難判斷三個欄位用途，更重要的是原邏輯必須依賴使用者自己提供 stop / trail 才能產生有意義的持股結論。

## 根因

1. 舊 doctor 把「想買」與「已持有」塞在同一個表單。
2. 持股判斷只呼叫 `holding_review()`；若沒有原始防守價或獲利保護價，就只能回 `ORIGINAL_THESIS_UNKNOWN_MANUAL_REVIEW`，不能完成使用者要的續抱／減碼／賣出判斷。
3. 新買進 Entry Score 與持股管理沒有正式分層；雖然核心架構已分開，但 doctor UI 沒有把這個分工呈現出來。
4. direct diagnose 對不在 snapshot 的股票會重新抓 daily history；若盤中 provider 含今日未完成 daily candle，原實作沒有強制截到 snapshot `price_date`，存在半根日 K 混入個股診斷的風險。
5. 自訂淺色背景沒有全面覆寫 Streamlit widget label 顏色，深色系統／瀏覽器下標籤可讀性差。

## V16.7 修正

### 1. 個股頁分成兩個真正不同的決策問題
- `考慮買入`：盤中走 realtime actionable buy；盤後／盤前走 next-session plan。
- `已有持股`：使用新的 holding decision，不用新買進 Entry Score 決定是否賣出。

### 2. 移除三個不明 `0.00`
- 成本改為空白文字欄，標示「持股成本（元／股，建議填寫）」。
- 原始停損與獲利保護價改為選填且預設空白。
- 留白時顯示模型自己計算的結構防守價；成本未填時不偽造損益。

### 3. 新增 `doctor_engine.py`
持股分數只使用持股適合的訊息：
- 結構防守 35%
- 趨勢 25%
- 歷史剩餘報酬 25%
- 標的品質 15%

硬性優先條件：
- 使用者確認原買進理由已失效；
- 跌破使用者原始停損；
- 跌破使用者獲利保護價；
- 跌破模型結構防守價。

其餘才依持股分數判斷續抱／減碼。歷史案例 confidence 或樣本不足時，forecast 回到中性，不用弱證據強迫賣出。

### 4. 盤中／盤後自動切換
- 盤中：單股即時報價可用時，以 live quote overlay；即時來源失敗時降級為完整日線觀察，絕不顯示「現在可買」。
- 盤後／盤前／休市：使用 completed-day 模型，買進頁顯示下一交易日規劃。

### 5. Direct diagnosis 日 K 汙染修正
若 dashboard 已有 `price_date`，任何 direct ticker 皆強制截到該日期；沒有 snapshot 時，PREOPEN/INTRADAY 僅使用今日以前的完整 daily session。

## 已執行測試

- `python -m compileall -q .` → PASS
- V16.6/V16.7 architecture regression 23 項 → PASS
- Doctor regression 8 項 → PASS
- Research detail / official history parser / research UI regression → PASS
- App import smoke → PASS
- Decision views smoke → PASS
- App main smoke → PASS

Doctor regression 特別覆蓋：
1. 盤後決策不冒充即時成交。
2. 盤中沒有即時 quote 時不准產生 BUY。
3. 新買進 timing 很差，不得自動把健康的既有持股判成 SELL。
4. 跌破模型防守價必須 SELL。
5. 使用者原始 stop / thesis broken 具有 hard override。
6. 獲利保護參考只有達到足夠浮盈才出現。
7. UI 不再出現三個預設 0.00。
8. direct diagnosis 必須有 completed-day cutoff guard。

## 真實歷史驗效

新增 `validate_doctor_2026.py`：
- 預設真實台股 60 檔；
- 2026 年 12 個共同 cutoff；
- 每個 cutoff 只看當時以前資料；
- POSTCLOSE Doctor 的 `READY` 訊號以次一交易日開盤價作為假設進場；
- 追蹤 10 / 40 / 120 交易日報酬、正報酬率、MAE、MFE、同日相對 lift。

本封裝容器實際執行 `--stocks 20 --points 12` 時，TWSE/TPEX 外部網路無法取得股票母體，因此回 `NOT_RUN`；沒有以 synthetic data 冒充真實績效。GitHub workflow `.github/workflows/validate_doctor_2026.yml` 已加入，可在有網路 runner 執行。

## 判定

V16.7 修正的是「決策正確性與 UI 可測性」，核心 `ARCHITECTURE_VERSION` 仍維持 `v16.6.0-dynamic-intraday`；沒有改短／中／長線 selection weights，也沒有用未完成的回測結果調參。程式結構與資料時間座標目前通過回歸；未來報酬能力仍需真實 2026 walk-forward 結果才能下結論。
