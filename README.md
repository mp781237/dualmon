# dualmon — 雙動能策略監控

每月跑一次的小型 dashboard，從 Yahoo Finance 抓 ETF 月線，算 5 個動能配置策略訊號，輸出 `signals.json` 給 `index.html` 顯示。

## 5 個策略

| ID  | 名稱             | 邏輯白話                                                                                          |
| :-- | :--------------- | :------------------------------------------------------------------------------------------------|
| S1  | 原版雙動能       | 比 VOO vs VXUS 12m → 贏家 vs BIL 12m → 贏者勝出，否則 BND（Antonacci GEM 變種）                  |
| S2  | 拉風增強版       | 同 S1 但用 QQQ 取代 VOO                                                                          |
| S3  | 加速雙動能       | VOO vs VSS 比 `accel = avg(1m, 3m, 6m)`，贏者勝出（需 > 0），否則用 TLT 1m 判 BIL 或 TLT 防禦   |
| S4  | 騷速雙動能       | 同 S3 但用 QQQ 取代 VOO                                                                          |
| S5  | VAA 攻擊型       | 4 攻（VOO/VXUS/VWO/BND）13612W 分數全正 → 取最強；任一為負 → 切到 4 守（LQD/IEF/SHY/BIL）取最強  |

**13612W 公式**（Keller & Keuning 2017）：`12*r1 + 4*r3 + 2*r6 + r12`

**accel 公式**（命名為「加速」但實際是平均）：`(r1 + r3 + r6) / 3`

## 本機跑

```bash
pip install -r requirements.txt
python usdn_updater.py
```

輸出 `signals.json`，若 `usdn.xlsx` 存在則同步股價分頁。

## 排程

GitHub Actions：每月 2 號 UTC 13:00（台北 21:00）自動跑、commit、push。
也支援 workflow_dispatch 手動觸發。

## 資料來源

- Yahoo Finance（透過 yfinance 套件，unofficial scraper）
- 抓**日線** `auto_adjust=True`（含股息與分割調整，近似 total return），再 resample 到月底
- `end` 設為本月 1 號（exclusive），永遠只用「已完成的月份」計算，避開 partial 當月 bar

## 已知限制與設計決策

- **資料源風險**：Yahoo 無官方 API、yfinance 可能因網站改版突然失效。`requirements.txt` 已 pin 版本減少 breaking change 風險。
- **VAA defensive universe 客製化**：標準 Keller 2017 VAA-G4 防禦組是 LQD/IEF/SHY 3 檔，本專案多加入 **BIL 作為純現金選項**（4 檔）。這是有意識的偏離原版，因 BIL 流動性最佳、適合作為避險時的現金停泊。
- **不年化 / 不用 log return**：原 VAA 規則用 raw simple total return，13612W 權重就是針對 raw return 設計，不應改 log。
- **訊號鎖定上月底**：cron 在月初跑，但所有資料都是上月底完成資料；交易假設應在當月可交易時點執行。
- **fail-fast**：任一 ETF 抓不到、或 VAA defense 全 None，腳本會以 `sys.exit(1)` / `RuntimeError` 中止，避免靜默產出 partial signals.json 誤導投資決策。

## 檔案結構

```
dualmon/
├── index.html              # 前端 dashboard
├── usdn_updater.py         # 每月跑的計算腳本
├── signals.json            # 計算輸出（給 index.html 讀）
├── notes.json              # 個人筆記/觀察存檔
├── usdn.xlsx               # 可選：股價分頁同步（若存在才寫）
├── requirements.txt        # Python 依賴（已 pin 版本）
├── .github/workflows/
│   └── update.yml          # cron + workflow_dispatch
└── reviews/                # 歷次審稿與修訂計畫存檔
```

## 變更歷史

- **2026-05-03**：partial month bug 修復、auto_adjust=True、fail-fast、tie-break、補 README、pin yfinance 1.3.0。詳見 `reviews/2026-05-03_audit/revision_plan.md`。
