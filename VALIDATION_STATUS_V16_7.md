# V16.7 Validation Status

## 已完成
- 核心 selection / intraday / entry / regime 分層 regression：PASS。
- 新增 doctor buy / holding 分層：PASS。
- 盤中無即時 quote 不產生假 BUY：PASS。
- 持股不受「現在不適合新買」直接誤殺：PASS。
- model stop / user stop / thesis broken hard-exit：PASS。
- direct ticker completed-day cutoff：PASS。
- 研究資料 V16.6.2 regression：PASS。
- Streamlit stub import / views / main：PASS。

## 真實 2026 Doctor walk-forward
狀態：**NOT_RUN（非 FAIL）**。

本容器實際執行：
`python validate_doctor_2026.py --out data/doctor_validation_probe --stocks 20 --points 12 --period 8y`

結果：TWSE 0 檔、TPEX 0 檔，外部資料源不可達，因此 validator 主動停止；沒有用合成行情替代真實績效。

已提供 `.github/workflows/validate_doctor_2026.yml`，可在有網路的 GitHub runner 以 60 檔 × 12 cutoff（可自行調整）重跑。

## 能聲稱的範圍
可以聲稱：程式決策層次、無半根日 K 汙染、hard-stop precedence、UI 可測性與 regression 測試通過。

不能聲稱：V16.7 已證明可提高未來報酬、已證明勝率、已完成真實盤中一分鐘成交回測。盤中歷史 minute/order-book 資料不在 daily OHLCV validator 範圍。
