# dualmon 修訂前後 signals.json Diff 摘要

> 本機跑 `python usdn_updater.py` 的結果，2026-05-03

---

## 🎯 最重要的修復成果：S5 從防禦切回攻擊

| 項目 | 修前 (5/2 partial + unadjusted) | 修後 (4/30 完成月 + adjusted) | 變化 |
| :--- | :--- | :--- | :--- |
| **BND vaa** | **-0.149** (假負) | **+0.0672** (微正) | **🔄 翻正** |
| **S5 mode** | 防禦 | **攻擊** | **🔄 切換！** |
| **S5 pick** | BIL（現金） | **VOO（美股）** | **🔄 切換！** |

**意義**：原 code 因為 partial month bug（5/2 那 1-2 天的 BND 跌幅被 12*r1 主導項放大），把 BND 的 vaa 算成 -0.149 → 誤判 4 攻有負 → 切到防禦 → 抱現金。修復後算出 BND vaa 真實值 +0.0672 → 4 攻全正 → 切回攻擊 → VOO。

**白話**：原本你以為市場該避險，其實 4 月美股是大反彈月（VOO 月報酬 +10.55%），原 code 讓你錯過。

---

## 詳細對照表

### 動能分數（5 個 ticker）

| Ticker | m1 修前 | m1 修後 | m12 修前 | m12 修後 | accel 修前 | accel 修後 |
| :--- | ---: | ---: | ---: | ---: | ---: | ---: |
| VOO | +0.29% (噪音) | **+10.55%** | +22.29% | **+31.14%** | +3.57% | **+6.90%** |
| QQQ | +0.96% | **+15.69%** | +29.87% | **+41.12%** | +6.95% | **+9.87%** |
| VSS | -0.23% | **+8.44%** | +23.52% | **+35.90%** | +2.67% | **+8.60%** |
| TLT | -0.01% | -0.84% | -0.78% | +0.04% | -3.62% | -1.51% |
| BIL | -0.26% | +0.29% | -0.37% (假負) | **+3.94%** | -0.28% | +0.97% |

**重點**：
- **m1 全面活化**：之前是「2 天噪音」，現在是「4 月真實月報酬」
- **m12 全面上修**：含股息與分割調整後總報酬更高
- **BIL m12 從 -0.37% 翻正到 +3.94%**：這是最重要的——S1/S2 的 GEM filter 用 BIL 12m 當門檻，原本門檻被低估 4pp，修後變嚴

### VAA 分數（8 檔）

| Ticker | 修前 vaa | 修後 vaa | 變化 |
| :--- | ---: | ---: | :--- |
| VOO | 0.5662 | **1.8631** | ⬆ |
| VXUS | 0.4007 | **1.7122** | ⬆ |
| VWO | 0.5013 | **1.7826** | ⬆ |
| **BND** | **-0.149** | **+0.0672** | **🔄 翻正（觸發 S5 攻擊）** |
| LQD | -0.1885 | +0.0724 | 🔄 翻正 |
| IEF | -0.2178 | +0.0054 | 🔄 翻正 |
| SHY | -0.103 | +0.0867 | 🔄 翻正 |
| BIL | -0.0517 | +0.1434 | 🔄 翻正 |

**重點**：原 code 連 BIL 短券的 vaa 都是負的（不可能！短券永遠正報酬），修後全部歸正。

### 5 個策略 pick 比較

| ID | 修前 pick | 修後 pick | 是否變化 |
| :--- | :--- | :--- | :--- |
| **S1** 原版雙動能 | VXUS | VXUS | 不變（VXUS 12m 仍 > VOO 12m，BIL 門檻變嚴但仍 < VXUS） |
| **S2** 拉風增強版 | QQQ | QQQ | 不變 |
| **S3** 加速雙動能 | VOO | **VSS** | **🔄 改變**（VSS 含 4 月後 6m=13.70%，accel 反超 VOO） |
| **S4** 騷速雙動能 | QQQ | QQQ | 不變 |
| **S5** VAA 攻擊型 | **BIL（防禦）** | **VOO（攻擊）** | **🔄 改變**（最關鍵） |

---

## 程式碼改動摘要

| 改動 | 行為差異 |
| :--- | :--- |
| **M1 partial month** | `interval="1d" + resample("ME")`，end 設本月 1 號 → 永遠用完成月資料 |
| **M2 auto_adjust=True** | 含股息與分割調整 → BIL/SHY 的 12m 不再因除息日被誤算虧損 |
| **M3 VAA defense 4 檔** | 維持現狀（你選 B 保留 BIL） + README 註明 |
| **M4 fail-fast** | 任何 ETF 抓不到、或 VAA defense 全 None → `sys.exit(1)` 中止，不再靜默產出 partial signals |
| **S1 tie-break** | S3/S4 用 `max(candidates.items())`，平手取插入順序首位（VOO/QQQ 先） |
| **S3 accel 加註解** | 變數名保留，加 docstring 說明「不是真加速度，是 1m/3m/6m 平均」 |
| **O1 dedupe** | `compute_signals()` 從 main flow 呼叫一次，傳給 `write_signals_json` 與 `print_signal` |
| **O2 Excel 註解** | 解釋 col 1-7 用途（OHLC placeholder） |
| **O3 README** | 新增，含 5 策略說明、VAA 客製化決定、本機跑指引 |
| **O4 pin yfinance** | `yfinance==1.3.0`、`openpyxl==3.1.5` |

---

## 已驗證

✅ 本機 `python usdn_updater.py` 跑成功（13 個月資料 + Excel 同步 + signals.json 寫入）
✅ 11 個 ETF 全部抓到、無 fetch failure
✅ S5 切換攻擊符合 revision_plan 預測
✅ BIL m12 從假負變正符合 auto_adjust 預測

---

## 還沒做的（待你拍板）

- [ ] commit + push 到 `mp781237/dualmon` main
- [ ] 觀察 GitHub Actions 下次 cron 跑（6/2）是否一切正常
- [ ] 把 `usdn.xlsx` 變化也 commit（pandas 3.x + openpyxl 跑過後 xlsx 內容會變）
