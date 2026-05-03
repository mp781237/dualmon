# dualmon 計算公式審查 Review Packet

> 給 Codex 外審用的完整 context 包。
> 日期：2026-05-03
> 對象：`mp781237/dualmon` repo @ commit on main 2026-05-02
> 來源：Claude Opus 4.7（CLI）執行 Phase 1 自審後產出

---

## 1. 專案目的

`dualmon`（雙動能測試）是個小型 web dashboard，每月跑一次 GitHub Actions，從 yfinance 抓月線資料、算 5 個動能策略訊號，輸出 `signals.json` 給前端 `index.html` 顯示。**這是 Ray 自己用的真實投資決策工具**，公式錯誤會直接影響資產配置。

## 2. 5 個策略白話描述

| 策略 ID | 名稱             | 規則白話                                                                                          |
| :------ | :--------------- | :-----------------------------------------------------------------------------------------------|
| **S1**  | 原版雙動能       | 比 VOO vs VXUS 12m → 贏家 vs BIL 12m → 贏者勝出，否則 BND（Antonacci GEM 變種）                 |
| **S2**  | 拉風增強版       | 同 S1 但用 QQQ 取代 VOO                                                                          |
| **S3**  | 加速雙動能       | VOO vs VSS 比 `accel = avg(1m, 3m, 6m)`，贏者勝出（需 > 0），否則用 TLT 1m 判 BIL 或 TLT 防禦   |
| **S4**  | 騷速雙動能       | 同 S3 但用 QQQ 取代 VOO                                                                          |
| **S5**  | VAA 攻擊型       | 4 攻（VOO/VXUS/VWO/BND）VAA 分數全正 → 取最強；任一為負 → 切到 4 守（LQD/IEF/SHY/BIL）取最強     |

**VAA 分數公式**：`12*r1 + 4*r3 + 2*r6 + r12`（Keller 2017 原文）
**Acceleration 分數公式**：`(r1 + r3 + r6) / 3`（簡單平均，命名上有疑慮）

## 3. 完整程式碼

```python
"""
動能資產配置自動更新腳本 (usdn_updater.py)
"""

import json
import sys
from datetime import datetime, timedelta
from pathlib import Path

import yfinance as yf

try:
    from openpyxl import load_workbook
except ImportError:
    load_workbook = None

SCRIPT_DIR = Path(__file__).parent
XLSX_PATH = SCRIPT_DIR / "usdn.xlsx"
JSON_PATH = SCRIPT_DIR / "signals.json"
ETF_LIST = ["VOO", "VXUS", "BND", "BIL", "VSS", "TLT", "VWO", "LQD", "IEF", "SHY", "QQQ"]
MONTHS_NEEDED = 13


def fetch_monthly_closes(ticker: str, months: int = MONTHS_NEEDED) -> list[tuple]:
    end = datetime.now()
    start = end - timedelta(days=months * 35)

    tk = yf.Ticker(ticker)
    hist = tk.history(start=start.strftime("%Y-%m-%d"),
                      end=end.strftime("%Y-%m-%d"),
                      interval="1mo", auto_adjust=False)

    if hist.empty:
        print(f"  [警告] {ticker} 無法取得資料")
        return []

    results = []
    for idx, row in hist.iterrows():
        dt = idx.to_pydatetime().replace(tzinfo=None)
        dt_first = dt.replace(day=1)
        close = round(float(row["Close"]), 2)
        results.append((dt_first, close))

    results.sort(key=lambda x: x[0], reverse=True)
    return results[:months]


def calc_returns(prices: list[float]) -> dict:
    if len(prices) < 2:
        return {}

    def ret(current, past):
        if past and past != 0:
            return (current - past) / past
        return None

    p = prices
    result = {}
    result["1m"] = ret(p[0], p[1]) if len(p) > 1 else None
    result["3m"] = ret(p[0], p[3]) if len(p) > 3 else None
    result["6m"] = ret(p[0], p[6]) if len(p) > 6 else None
    result["12m"] = ret(p[0], p[12]) if len(p) > 12 else None

    if all(result.get(k) is not None for k in ["1m", "3m", "6m", "12m"]):
        result["vaa"] = 12 * result["1m"] + 4 * result["3m"] + 2 * result["6m"] + result["12m"]
    if all(result.get(k) is not None for k in ["1m", "3m", "6m"]):
        result["accel"] = (result["1m"] + result["3m"] + result["6m"]) / 3

    return result


def compute_signals(all_returns: dict) -> dict:
    def r(etf, key):
        return all_returns.get(etf, {}).get(key)

    def pct(v):
        return f"{v*100:+.2f}%" if v is not None else ""

    strategies = []

    # S1 經典雙動能
    voo_12, vxus_12, bil_12 = r("VOO", "12m"), r("VXUS", "12m"), r("BIL", "12m")
    if None not in (voo_12, vxus_12, bil_12):
        winner = "VOO" if voo_12 >= vxus_12 else "VXUS"
        pick = winner if max(voo_12, vxus_12) > bil_12 else "BND"
        # ...append to strategies

    # S2 拉風增強
    qqq_12 = r("QQQ", "12m")
    if None not in (qqq_12, vxus_12, bil_12):
        winner = "QQQ" if qqq_12 >= vxus_12 else "VXUS"
        pick = winner if max(qqq_12, vxus_12) > bil_12 else "BND"
        # ...

    # S3 加速雙動能
    voo_acc, vss_acc, tlt_1m = r("VOO", "accel"), r("VSS", "accel"), r("TLT", "1m")
    if voo_acc is not None and vss_acc is not None:
        if voo_acc > vss_acc and voo_acc > 0:
            pick, score = "VOO", voo_acc
        elif vss_acc > voo_acc and vss_acc > 0:
            pick, score = "VSS", vss_acc
        elif tlt_1m is not None and tlt_1m < 0:
            pick, score = "BIL", None
        else:
            pick, score = "TLT", None
        # ...

    # S4 騷速雙動能（同 S3 但 QQQ 取代 VOO）
    qqq_acc = r("QQQ", "accel")
    # ...

    # S5 VAA 攻擊型
    atk = {"VOO": r("VOO", "vaa"), "VXUS": r("VXUS", "vaa"),
           "VWO": r("VWO", "vaa"), "BND": r("BND", "vaa")}
    defs = {"LQD": r("LQD", "vaa"), "IEF": r("IEF", "vaa"),
            "SHY": r("SHY", "vaa"), "BIL": r("BIL", "vaa")}
    if all(v is not None for v in atk.values()):
        all_pos = all(v > 0 for v in atk.values())
        if all_pos:
            pick = max(atk, key=atk.get)
        else:
            valid_defs = {k: v for k, v in defs.items() if v is not None}
            pick = max(valid_defs, key=valid_defs.get) if valid_defs else "BIL"
        # ...
```

> 完整原始碼會以 stdin pipe 一併傳給 Codex（餵真本不貼摘要）。

## 4. 最近一次輸出 signals.json（2026-05-02 跑出來）

```json
{
  "updated": "2026-05-02",
  "strategies": [
    {"id": "S1", "pick": "VXUS", "modeLabel": "外國股市"},
    {"id": "S2", "pick": "QQQ",  "modeLabel": "美國股市"},
    {"id": "S3", "pick": "VOO",  "modeLabel": "進攻 +3.57%"},
    {"id": "S4", "pick": "QQQ",  "modeLabel": "進攻 +6.95%"},
    {"id": "S5", "pick": "BIL",  "modeLabel": "防禦模式"}
  ],
  "momentum": {
    "rows": [
      {"ticker": "VOO", "m1": 0.002937, "m3": 0.049886, "m6": 0.05428,  "m12": 0.222903, "score": 0.035701},
      {"ticker": "QQQ", "m1": 0.0096,   "m3": 0.110096, "m6": 0.088656, "m12": 0.298665, "score": 0.06945},
      {"ticker": "VSS", "m1": -0.00234, "m3": -0.014558,"m6": 0.097106, "m12": 0.235179, "score": 0.026736},
      {"ticker": "TLT", "m1": -0.000117,"m3": -0.057366,"m6": -0.050992,"m12": -0.007765,"score": -0.036158},
      {"ticker": "BIL", "m1": -0.002619,"m3": -0.002401,"m6": -0.003489,"m12": -0.003706,"score": -0.002836}
    ]
  },
  "vaa": {
    "attack":  [{"VOO": 0.5662}, {"VXUS": 0.4007}, {"VWO": 0.5013}, {"BND": -0.149}],
    "defense": [{"LQD": -0.1885}, {"IEF": -0.2178}, {"SHY": -0.103}, {"BIL": -0.0517}],
    "pick": "BIL", "pickReason": "攻擊資產有負值，切防禦"
  }
}
```

> 完整 JSON 會以 stdin pipe 一併傳給 Codex。

---

## 5. 我自己看出的可疑點（強制欄位）

詳細版見 `claude_audit.md`，這裡列重點：

### 🔴 CRITICAL

1. **partial 當月 bar 進到 1m 計算**：證據是 5/2 跑出來 VOO m1 = +0.29%、QQQ m1 = +0.96%——這顯然是「2 個交易日的報酬」不是「1 個月報酬」。yfinance `interval="1mo"` 在月初 2 號跑時，2026-05 那一根 partial bar 被當成 `p[0]`。**影響**：accel score 進雜訊（1/3 權重）、VAA 公式 `12*r1` 主導項變雜訊（~63% 權重）、TLT 1m 防禦判斷失真。

2. **`auto_adjust=False`**：line 42 用未調整收盤，**忽略股息與分割**。VXUS 配息 ~3%、BND ~3%、TLT ~3.5% 被系統性低估 12m 報酬，可能讓 S1 比較 VOO vs VXUS 出現偽贏家、絕對動能 filter（vs BIL）被低估過門檻機率。

### 🟠 HIGH

3. **`accel` 命名誤導**：實際公式是 `(r1+r3+r6)/3` 簡單平均，數學上不是 acceleration（變化率）。
4. **TLT 1m 防禦 filter 受 #1 連累**：在月初 2 號 TLT 1m 是 1-2 天小數字，等邊界時 BIL/TLT 切換不穩。

### 🟡 MEDIUM

5. **S3/S4 tie-break**：`voo_acc == vss_acc` 時兩個 `>` 都 false，掉到防禦邏輯（極罕見但邏輯漏洞）。
6. **VAA defense fallback hardcode "BIL"**：line 220 全部 None 時 fallback "BIL"，應該 raise/None。

### 🟢 LOW

7. `compute_signals()` 在 main flow 被呼叫兩次。
8. `update_excel()` column 3/4/5 全填 close_val、column 6/7 填空字串，語意不明。
9. 沒 README。

---

## 6. 想請 Codex 特別檢查的點

1. **驗證 CRITICAL #1**：在月初 2 號跑時，yfinance `interval="1mo"` 是否真的會回傳 partial 當月 bar？我推薦的修法 A（過濾掉 `dt >= today.replace(day=1)`）有更好的選擇嗎（例如 yfinance 有沒有官方建議的「只取完成月」flag）？
2. **驗證 CRITICAL #2**：`auto_adjust=True` 是業界標準嗎？dual momentum / Faber GEM / Antonacci GEM 原文用「total return」是否就等同於 yfinance 的 `auto_adjust=True`？有沒有 corner case 我沒想到（特別股息 reinvestment、ETF 分配清算）？
3. **lookahead bias 檢查**：5 個策略的計算是否完全用「過去資料」決定當期持倉？特別是 S5 的 VAA 公式是否該用「前一個月底」的資料而非「當月初部分資料」？
4. **index off-by-one**：`p[0]` / `p[1]` / `p[3]` / `p[6]` / `p[12]` 切片在 13 個月資料下對不對？如果 `MONTHS_NEEDED` 改成 14（為了過濾當月），切片邏輯要不要跟著動？
5. **VAA 公式驗證**：`12*r1 + 4*r3 + 2*r6 + r12` 是 Keller 2017 paper 標準嗎？2020 年後有沒有更新版本（VAA-G2 之類）？
6. **dual momentum 經典版**：Antonacci GEM 的「絕對動能 filter」是 winner_12m > T-bill_12m 還是 winner_12m > 0？code 用 BIL 12m，這個跟 Antonacci 原文一致嗎？
7. **5 個策略決策樹**：以量化金融工程師的眼光通讀，看有沒有任何 edge case、tie-break、boundary value 我漏掉。
8. **金融正確性**：是否該年化（annualize）？monthly returns 是否該做 log return 而非 simple return？monthly cron 在 1/2 號跑（vs 月底跑）對訊號有沒有時序問題？

每個發現請附：**行號 + 嚴重度（CRITICAL/HIGH/MEDIUM/LOW）+ 為什麼是 bug 或可疑 + 建議修法（含程式碼片段）**。

---

## 7. 環境資訊

- **Python**：3.11（GitHub Actions）／本機 3.13
- **依賴**：`requirements.txt` 只有 `yfinance` 和 `openpyxl`（未鎖版號）
- **yfinance 行為**：截至 2026-05 主流版本，`interval="1mo"` 預期回傳月線 bars，但「是否含當月 partial bar」歷史上版本間有變動
- **Cron 排程**：`0 13 2 * *`（每月 2 號 UTC 13:00）→ 台北時間 21:00
- **資料窗口**：`MONTHS_NEEDED = 13`（為了算 12m return 需要 13 個月資料）
- **資料來源**：Yahoo Finance（yfinance unofficial scraper）
- **下游**：`index.html` 直接 `fetch("./signals.json")` 顯示

---

## 8. 重要對照數字（讓 Codex 可以反推真相）

如果以下兩個 CRITICAL 修了，預期 `signals.json` 會出現以下變化：

| 欄位      | 修前（5/2 partial bar + unadjusted） | 修後（5/1 完成月 + adjusted）預期方向 |
| :-------- | :----------------------------------- | :------------------------------------ |
| VOO m1    | +0.29%（噪音）                       | 約 +4-5%（4 月真實月報酬）             |
| QQQ m1    | +0.96%                               | 約 +4-6%                              |
| VXUS m12  | 沒給但低於真值                       | 提高 ~1.5-2pp（含股息）                |
| BIL m12   | 約 +5%（純殖利率）                    | 接近不變（BIL 配息高度同步）            |
| BND vaa   | -0.149                               | 可能轉正（BND 4 月反彈但被 partial 1m 拖累） |
| S5 mode   | 防禦                                  | **可能切回攻擊**（取決於 BND vaa 有無翻正） |

如果 S5 在「partial bar 含」防禦、修了之後切回攻擊——這就是這個 bug 對投資決策的最直接證據。
