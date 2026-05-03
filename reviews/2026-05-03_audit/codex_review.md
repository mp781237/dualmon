Reading additional input from stdin...
OpenAI Codex v0.124.0 (research preview)
--------
workdir: C:\Users\ray\iCloudDrive\Raymond-Agent\100_Todo\projects\2026-05-03_dualmon\reviews\2026-05-03_audit
model: gpt-5.5
provider: openai
approval: never
sandbox: read-only
reasoning effort: xhigh
reasoning summaries: none
session id: 019debcf-f2fd-7db1-a6f8-7b385cc8920d
--------
user
你是一位資深量化金融工程師。下面我會餵你三個檔案：(1) review_packet.md（含專案說明、5 個策略白話、Claude 自審看出的可疑點、想請你檢查的 8 個問題），(2) usdn_updater.py 完整原始碼（主要審查對象），(3) signals.json 最近一次輸出範例。

請根據 review_packet.md 第 6 節「想請 Codex 特別檢查的點」逐項回答，並另外從你的金融工程師視角找出 Claude 沒看到的問題。每個發現附：行號 + 嚴重度（CRITICAL/HIGH/MEDIUM/LOW）+ 為什麼是 bug 或可疑 + 建議修法（含程式碼片段）。

特別關鍵的議題：
- partial 當月 bar 進到計算（CRITICAL #1，Claude 用 signals.json 的 m1 數字推論）：請驗證 yfinance interval=1mo 行為，並評估 Claude 的修法 A 是否 OK。
- auto_adjust=False（CRITICAL #2）：dual momentum / Faber GEM / Antonacci GEM 是否該用 total return（含股息）？
- VAA 公式 12*r1+4*r3+2*r6+r12 是否就是 Keller 2017 的標準公式？
- Antonacci GEM 的絕對動能 filter 是「winner_12m > T-bill_12m」還是「winner_12m > 0」？
- 5 個策略決策樹的 lookahead bias、index off-by-one、tie-break、邊界條件
- 是否該年化、log return、月底 vs 月初時序

最後給一個總體建議：哪幾項必修、哪幾項 nice-to-have、有沒有任何 Claude 自審完全漏掉的高風險問題。請用繁體中文回覆，控制在 2500 字內。

<stdin>
=== review_packet.md ===
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

=== usdn_updater.py ===
"""
動能資產配置自動更新腳本 (usdn_updater.py)
─────────────────────────────────────────
使用 yfinance 抓取 ETF 月度收盤價，計算五大策略訊號，並輸出：
  1. signals.json — 供 動能資產配置.html 儀表板讀取
  2. usdn.xlsx    — 若存在則同步股價分頁（可選）

使用方式：
    pip install yfinance openpyxl
    python usdn_updater.py

股價來源：yfinance close (non-adjusted)
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

# ── 設定 ──────────────────────────────────
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


def update_excel(wb, etf: str, data: list[tuple]):
    if etf not in wb.sheetnames:
        print(f"  [略過] 工作表 '{etf}' 不存在")
        return

    ws = wb[etf]

    for i, (date_val, close_val) in enumerate(data):
        r = i + 2
        ws.cell(row=r, column=1, value=date_val)
        ws.cell(row=r, column=2, value=close_val)
        for c in (3, 4, 5, 6, 7):
            if c in (3, 4, 5):
                ws.cell(row=r, column=c, value=close_val)
            else:
                ws.cell(row=r, column=c, value="")

    for r in range(len(data) + 2, ws.max_row + 1):
        for c in range(1, 8):
            ws.cell(row=r, column=c, value=None)


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


def _classify_pick(pick: str) -> tuple[str, str]:
    """Return (mode_class, default_label) for a ticker."""
    if pick in ("VOO", "QQQ"):
        return ("attack", "美國股市")
    if pick in ("VXUS", "VSS", "VWO"):
        return ("intl", "外國股市")
    if pick in ("BND", "TLT", "LQD", "IEF"):
        return ("bond", "債券")
    if pick in ("BIL", "SHY"):
        return ("defense", "現金")
    return ("attack", "")


def _round(v, d=6):
    return round(v, d) if v is not None else None


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
        mode, default_label = _classify_pick(pick)
        label = "債券避險" if pick == "BND" else default_label
        strategies.append({
            "id": "S1", "name": "原版雙動能",
            "pick": pick, "assets": ["VOO", "VXUS", "BND", "BIL"],
            "mode": mode, "modeLabel": label,
        })

    # S2 拉風增強
    qqq_12 = r("QQQ", "12m")
    if None not in (qqq_12, vxus_12, bil_12):
        winner = "QQQ" if qqq_12 >= vxus_12 else "VXUS"
        pick = winner if max(qqq_12, vxus_12) > bil_12 else "BND"
        mode, default_label = _classify_pick(pick)
        label = "債券避險" if pick == "BND" else default_label
        strategies.append({
            "id": "S2", "name": "原版雙動能 · 拉風增強版",
            "pick": pick, "assets": ["QQQ", "VXUS", "BND", "BIL"],
            "mode": mode, "modeLabel": label,
        })

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
        mode, _ = _classify_pick(pick)
        if pick in ("VOO", "VSS"):
            label = f"進攻 {pct(score)}"
        elif pick == "TLT":
            label = "債券避險"
        else:
            label = "現金"
        strategies.append({
            "id": "S3", "name": "加速雙動能",
            "pick": pick, "assets": ["VOO", "VSS", "TLT", "BIL"],
            "mode": mode, "modeLabel": label,
        })

    # S4 騷速雙動能
    qqq_acc = r("QQQ", "accel")
    if qqq_acc is not None and vss_acc is not None:
        if qqq_acc > vss_acc and qqq_acc > 0:
            pick, score = "QQQ", qqq_acc
        elif vss_acc > qqq_acc and vss_acc > 0:
            pick, score = "VSS", vss_acc
        elif tlt_1m is not None and tlt_1m < 0:
            pick, score = "BIL", None
        else:
            pick, score = "TLT", None
        mode, _ = _classify_pick(pick)
        if pick in ("QQQ", "VSS"):
            label = f"進攻 {pct(score)}"
        elif pick == "TLT":
            label = "債券避險"
        else:
            label = "現金"
        strategies.append({
            "id": "S4", "name": "騷速雙動能 · 含 QQQ",
            "pick": pick, "assets": ["QQQ", "VSS", "TLT", "BIL"],
            "mode": mode, "modeLabel": label,
        })

    # S5 VAA 攻擊型
    atk = {"VOO": r("VOO", "vaa"), "VXUS": r("VXUS", "vaa"),
           "VWO": r("VWO", "vaa"), "BND": r("BND", "vaa")}
    defs = {"LQD": r("LQD", "vaa"), "IEF": r("IEF", "vaa"),
            "SHY": r("SHY", "vaa"), "BIL": r("BIL", "vaa")}
    vaa_block = None
    if all(v is not None for v in atk.values()):
        all_pos = all(v > 0 for v in atk.values())
        if all_pos:
            pick = max(atk, key=atk.get)
            pick_label = "攻擊模式 · 最強標的"
            pick_reason = "ALL 攻擊資產得分 > 0"
            mode_label = "攻擊模式"
        else:
            valid_defs = {k: v for k, v in defs.items() if v is not None}
            pick = max(valid_defs, key=valid_defs.get) if valid_defs else "BIL"
            pick_label = "防禦模式 · 最強標的"
            pick_reason = "攻擊資產有負值，切防禦"
            mode_label = "防禦模式"
        mode, _ = _classify_pick(pick)
        strategies.append({
            "id": "S5", "name": "VAA 攻擊型",
            "pick": pick, "assets": list(atk.keys()),
            "mode": mode, "modeLabel": mode_label,
        })
        vaa_block = {
            "attack": [
                {"ticker": k, "score": _round(v, 4), "winner": all_pos and k == pick}
                for k, v in atk.items()
            ],
            "defense": [
                {"ticker": k, "score": _round(v, 4), "winner": (not all_pos) and k == pick}
                for k, v in defs.items()
            ],
            "pickLabel": pick_label,
            "pickReason": pick_reason,
            "pick": pick,
        }

    # 動能分數表（S3/S4 共用）
    accel_tickers = ["VOO", "QQQ", "VSS", "TLT", "BIL"]
    attack_accel = {t: r(t, "accel") for t in ("VOO", "QQQ", "VSS")}
    valid_attack = {k: v for k, v in attack_accel.items() if v is not None and v > 0}
    winner_ticker = max(valid_attack, key=valid_attack.get) if valid_attack else None
    momentum_rows = [{
        "ticker": t,
        "m1": _round(r(t, "1m")),
        "m3": _round(r(t, "3m")),
        "m6": _round(r(t, "6m")),
        "m12": _round(r(t, "12m")),
        "score": _round(r(t, "accel")),
        "winner": t == winner_ticker,
    } for t in accel_tickers]

    return {
        "updated": datetime.now().strftime("%Y-%m-%d"),
        "strategies": strategies,
        "momentum": {
            "title": "加速雙動能 · 騷速雙動能",
            "subtitle": "分數 = avg(1m, 3m, 6m)",
            "rows": momentum_rows,
        },
        "vaa": vaa_block,
    }


def write_signals_json(all_returns: dict) -> dict:
    data = compute_signals(all_returns)
    with open(JSON_PATH, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)
    print(f"已儲存訊號至 {JSON_PATH}")
    return data


def print_signal(all_returns: dict):
    def r(etf, key):
        return all_returns.get(etf, {}).get(key)

    def fmt_pct(v):
        return f"{v*100:+.2f}%" if v is not None else "N/A"

    data = compute_signals(all_returns)

    print("\n" + "=" * 60)
    print("  動能資產配置 — 當月策略訊號")
    print("=" * 60)

    for s in data["strategies"]:
        print(f"\n  {s['id']} {s['name']}:  {s['pick']}  ({s['modeLabel']})")

    print("\n  動能分數（加速分數）:")
    for row in data["momentum"]["rows"]:
        print(f"     {row['ticker']:4s}  "
              f"1m={fmt_pct(row['m1']):>9s}  "
              f"3m={fmt_pct(row['m3']):>9s}  "
              f"6m={fmt_pct(row['m6']):>9s}  "
              f"accel={fmt_pct(row['score']):>9s}"
              f"{'  ★' if row['winner'] else ''}")

    if data["vaa"]:
        print("\n  VAA 攻擊資產分數:")
        for item in data["vaa"]["attack"]:
            mark = " ★" if item["winner"] else ""
            print(f"     {item['ticker']:4s}  {item['score']:+.4f}{mark}")

    print("\n" + "=" * 60)


def main():
    print("動能資產配置更新程式 v2.1")
    print(f"JSON 輸出: {JSON_PATH}")
    print(f"XLSX 目標: {XLSX_PATH} {'(存在)' if XLSX_PATH.exists() else '(不存在 — 略過)'}")
    print(f"更新時間: {datetime.now().strftime('%Y-%m-%d %H:%M')}")
    print("-" * 40)

    all_data = {}
    all_returns = {}

    for etf in ETF_LIST:
        print(f"  抓取 {etf}...", end=" ", flush=True)
        try:
            data = fetch_monthly_closes(etf)
        except Exception as e:
            print(f"失敗 ({e})")
            continue
        if data:
            all_data[etf] = data
            prices = [d[1] for d in data]
            all_returns[etf] = calc_returns(prices)
            print(f"OK ({len(data)} 個月)")
        else:
            print("失敗")

    if not all_returns:
        print("\n[錯誤] 無任何 ETF 資料，退出")
        sys.exit(1)

    # 1. 寫 JSON（主要輸出）
    print()
    write_signals_json(all_returns)

    # 2. 同步 Excel（可選）
    if XLSX_PATH.exists() and load_workbook is not None:
        print("\n更新 Excel 檔案...")
        wb = load_workbook(str(XLSX_PATH))
        for etf, data in all_data.items():
            update_excel(wb, etf, data)
            print(f"  {etf} 已更新 ({len(data)} 行)")
        wb.save(str(XLSX_PATH))
        print(f"已儲存至 {XLSX_PATH}")

    # 3. 印訊號
    print_signal(all_returns)

    print("\n完成！")


if __name__ == "__main__":
    main()

=== signals.json (latest output 2026-05-02) ===
{
  "updated": "2026-05-02",
  "strategies": [
    {
      "id": "S1",
      "name": "原版雙動能",
      "pick": "VXUS",
      "assets": [
        "VOO",
        "VXUS",
        "BND",
        "BIL"
      ],
      "mode": "intl",
      "modeLabel": "外國股市"
    },
    {
      "id": "S2",
      "name": "原版雙動能 · 拉風增強版",
      "pick": "QQQ",
      "assets": [
        "QQQ",
        "VXUS",
        "BND",
        "BIL"
      ],
      "mode": "attack",
      "modeLabel": "美國股市"
    },
    {
      "id": "S3",
      "name": "加速雙動能",
      "pick": "VOO",
      "assets": [
        "VOO",
        "VSS",
        "TLT",
        "BIL"
      ],
      "mode": "attack",
      "modeLabel": "進攻 +3.57%"
    },
    {
      "id": "S4",
      "name": "騷速雙動能 · 含 QQQ",
      "pick": "QQQ",
      "assets": [
        "QQQ",
        "VSS",
        "TLT",
        "BIL"
      ],
      "mode": "attack",
      "modeLabel": "進攻 +6.95%"
    },
    {
      "id": "S5",
      "name": "VAA 攻擊型",
      "pick": "BIL",
      "assets": [
        "VOO",
        "VXUS",
        "VWO",
        "BND"
      ],
      "mode": "defense",
      "modeLabel": "防禦模式"
    }
  ],
  "momentum": {
    "title": "加速雙動能 · 騷速雙動能",
    "subtitle": "分數 = avg(1m, 3m, 6m)",
    "rows": [
      {
        "ticker": "VOO",
        "m1": 0.002937,
        "m3": 0.049886,
        "m6": 0.05428,
        "m12": 0.222903,
        "score": 0.035701,
        "winner": false
      },
      {
        "ticker": "QQQ",
        "m1": 0.0096,
        "m3": 0.110096,
        "m6": 0.088656,
        "m12": 0.298665,
        "score": 0.06945,
        "winner": true
      },
      {
        "ticker": "VSS",
        "m1": -0.00234,
        "m3": -0.014558,
        "m6": 0.097106,
        "m12": 0.235179,
        "score": 0.026736,
        "winner": false
      },
      {
        "ticker": "TLT",
        "m1": -0.000117,
        "m3": -0.057366,
        "m6": -0.050992,
        "m12": -0.007765,
        "score": -0.036158,
        "winner": false
      },
      {
        "ticker": "BIL",
        "m1": -0.002619,
        "m3": -0.002401,
        "m6": -0.003489,
        "m12": -0.003706,
        "score": -0.002836,
        "winner": false
      }
    ]
  },
  "vaa": {
    "attack": [
      {
        "ticker": "VOO",
        "score": 0.5662,
        "winner": false
      },
      {
        "ticker": "VXUS",
        "score": 0.4007,
        "winner": false
      },
      {
        "ticker": "VWO",
        "score": 0.5013,
        "winner": false
      },
      {
        "ticker": "BND",
        "score": -0.149,
        "winner": false
      }
    ],
    "defense": [
      {
        "ticker": "LQD",
        "score": -0.1885,
        "winner": false
      },
      {
        "ticker": "IEF",
        "score": -0.2178,
        "winner": false
      },
      {
        "ticker": "SHY",
        "score": -0.103,
        "winner": false
      },
      {
        "ticker": "BIL",
        "score": -0.0517,
        "winner": true
      }
    ],
    "pickLabel": "防禦模式 · 最強標的",
    "pickReason": "攻擊資產有負值，切防禦",
    "pick": "BIL"
  }
}
</stdin>
codex
我會先核對本地檔案行號，再用公開來源確認 yfinance、VAA、GEM 幾個關鍵行為；最後把發現壓縮成可執行的修法清單。
2026-05-03T03:09:46.993608Z ERROR codex_core::tools::router: error=`"C:\\WINDOWS\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -Command '$i=1; Get-Content -Path .'"\\usdn_updater.py | ForEach-Object { '{0,4}: {1}' -f "'$i, $_; $i++ }'` rejected: blocked by policy
exec
"C:\\WINDOWS\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -Command 'Get-ChildItem -Force' in C:\Users\ray\iCloudDrive\Raymond-Agent\100_Todo\projects\2026-05-03_dualmon\reviews\2026-05-03_audit
exec
"C:\\WINDOWS\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -Command '$i=1; Get-Content -Path .'"\\usdn_updater.py | ForEach-Object { '{0,4}: {1}' -f "'$i, $_; $i++ }' in C:\Users\ray\iCloudDrive\Raymond-Agent\100_Todo\projects\2026-05-03_dualmon\reviews\2026-05-03_audit
 declined in 0ms:
`"C:\\WINDOWS\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -Command '$i=1; Get-Content -Path .'"\\usdn_updater.py | ForEach-Object { '{0,4}: {1}' -f "'$i, $_; $i++ }'` rejected: blocked by policy
 succeeded in 956ms:


    目錄: C:\Users\ray\iCloudDrive\Raymond-Agent\100_Todo\projects\2026-05-03_dualmon\reviews\2026-05-03_audit


Mode                 LastWriteTime         Length Name                                                                 
----                 -------------         ------ ----                                                                 
-a---l        2026/5/3  上午 11:02          10607 claude_audit.md                                                      
-a----        2026/5/3  上午 11:09              0 codex_review.md                                                      
-a---l        2026/5/3  上午 11:04          12389 review_packet.md                                                     


無法設定屬性。此語言模式的核心類型只支援屬性設定。
位於 線路:1 字元:1
+ [Console]::OutputEncoding=[System.Text.Encoding]::UTF8;
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (:) []，RuntimeException
    + FullyQualifiedErrorId : PropertySetterNotSupportedInConstrainedLanguage
 

codex
本地資料夾沒有獨立的 `usdn_updater.py`，只有 `review_packet.md` 與 Claude 審查稿；我會以你貼上的完整原始碼為準標行號，並用 `review_packet.md` 輔助定位。
2026-05-03T03:09:56.423344Z ERROR codex_core::tools::router: error=`"C:\\WINDOWS\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -Command "Select-String -Path .\\review_packet.md -Pattern 'auto_adjust=False|def fetch_monthly_closes|result\\[\"1m\"\\]|winner =|tlt_1m|valid_defs|datetime.now|MONTHS_NEEDED|def compute_signals' | ForEach-Object { \""'$($_.LineNumber): $($_.Line.Trim())" }'` rejected: blocked by policy
exec
"C:\\WINDOWS\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -Command "Select-String -Path .\\review_packet.md -Pattern 'auto_adjust=False|def fetch_monthly_closes|result\\[\"1m\"\\]|winner =|tlt_1m|valid_defs|datetime.now|MONTHS_NEEDED|def compute_signals' | ForEach-Object { \""'$($_.LineNumber): $($_.Line.Trim())" }' in C:\Users\ray\iCloudDrive\Raymond-Agent\100_Todo\projects\2026-05-03_dualmon\reviews\2026-05-03_audit
 declined in 0ms:
`"C:\\WINDOWS\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -Command "Select-String -Path .\\review_packet.md -Pattern 'auto_adjust=False|def fetch_monthly_closes|result\\[\"1m\"\\]|winner =|tlt_1m|valid_defs|datetime.now|MONTHS_NEEDED|def compute_signals' | ForEach-Object { \""'$($_.LineNumber): $($_.Line.Trim())" }'` rejected: blocked by policy
exec
"C:\\WINDOWS\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -Command "Select-String -Path .\\review_packet.md -Pattern auto_adjust" in C:\Users\ray\iCloudDrive\Raymond-Agent\100_Todo\projects\2026-05-03_dualmon\reviews\2026-05-03_audit
 succeeded in 608ms:

review_packet.md:60:                      interval="1mo", auto_adjust=False)
review_packet.md:199:2. **`auto_adjust=False`**：line 42 用未調整收盤，**忽略股息與分割**。VXUS 配息 ~3%、BND ~3%、TLT ~3.
5% 被系統性低估 12m 報酬，可能讓 S1 比較 VOO vs VXUS 出現偽贏家、絕對動能 filter（vs BIL）被低估過門檻機率。
review_packet.md:222:2. **驗證 CRITICAL #2**：`auto_adjust=True` 是業界標準嗎？dual momentum / Faber GEM / Antonacci GE
M 原文用「total return」是否就等同於 yfinance 的 `auto_adjust=True`？有沒有 corner case 我沒想到（特別股息 reinvestment、E
TF 分配清算）？


無法設定屬性。此語言模式的核心類型只支援屬性設定。
位於 線路:1 字元:1
+ [Console]::OutputEncoding=[System.Text.Encoding]::UTF8;
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (:) []，RuntimeException
    + FullyQualifiedErrorId : PropertySetterNotSupportedInConstrainedLanguage
 

web search: 
web search: yfinance history interval 1mo current month partial bar auto_adjust docs
web search: 
web search: site:ranaroussi.github.io/yfinance advanced price repair 1mo interval current month auto_adjust history
web search: 
web search: site:github.com/ranaroussi/yfinance/issues 1mo interval current month partial
web search: 
web search: https://github.com/ranaroussi/yfinance/blob/master/yfinance/utils.py
web search: 
web search: https://raw.githubusercontent.com/ranaroussi/yfinance/main/yfinance/utils.py
web search: 
web search: 'fix_Yahoo_returning_live_separate' in https://raw.githubusercontent.com/ranaroussi/yfinance/main/yfinance/utils.py
web search: 
web search: Keller Keuning Vigilant Asset Allocation 2017 12 4 2 1 formula PDF
web search: 
web search: https://papers.ssrn.com/sol3/Delivery.cfm/SSRN_ID3002624_code1935527.pdf?abstractid=3002624
web search: 
web search: https://papers.ssrn.com/sol3/Delivery.cfm/SSRN_ID3002624_code1935527.pdf?abstractid=3002624&mirid=1&type=2
web search: 
web search: Allocate Smartly Vigilant Asset Allocation 12 * p0 / p1 4 p0 / p3 2 p0 / p6 p0 / p12
web search: 
web search: Meb Faber Global Tactical Asset Allocation total return monthly end of month paper
codex
來源核對結果：yfinance 文件明列 `end` 是 exclusive、`auto_adjust` 預設為 True，但沒有「只取完成月」參數；yfinance 原始碼也有處理 Yahoo live weekly/monthly partial row 的邏輯。GEM 與 VAA 規則也已用作者/SSRN與策略整理來源交叉確認。
codex
結論：Claude 的 CRITICAL #1/#2 都成立。yfinance 文件列 `end` 是 exclusive、`auto_adjust` 預設 True，但沒有「只取完成月」flag；yfinance 原始碼也有修 Yahoo live weekly/monthly partial row 的邏輯。GEM 官網說絕對動能是超過 T-bill 且績效用含息 total return；VAA 13612W 公式也符合 Keller/Keuning 實作整理。來源：yfinance docs、yfinance utils、OptimalMomentum GEM、AllocateSmartly VAA、Faber GTAA。

**主要發現**
1. 行 35-55 / CRITICAL：partial 當月 bar 進入 `p[0]`。5/2 的 VOO/QQQ m1 是 5 月 1 日附近雜訊，不是 4 月完整月報酬；VAA 的 `12*r1` 會被嚴重放大。Claude 修法 A「濾掉 `dt >= 當月1日`」方向 OK，但要多抓月份；更佳是用日線 adjusted close 重採樣完成月。
```python
def fetch_monthly_closes(ticker, months=13):
    end = datetime.now().replace(day=1, hour=0, minute=0, second=0, microsecond=0)
    start = end - timedelta(days=(months + 2) * 35)
    hist = yf.Ticker(ticker).history(
        start=start.strftime("%Y-%m-%d"),
        end=end.strftime("%Y-%m-%d"),   # exclusive: 只到上月底
        interval="1d", auto_adjust=True, actions=True, repair=True,
    )
    if hist.empty:
        raise RuntimeError(f"{ticker} no data")
    hist.index = hist.index.tz_localize(None) if hist.index.tz else hist.index
    closes = hist["Close"].resample("ME").last().dropna().tail(months)
    return [(d.to_pydatetime().replace(day=1), float(c))
            for d, c in closes.sort_index(ascending=False).items()]
```

2. 行 41/51 / CRITICAL：`auto_adjust=False` 且用 `Close`。Dual momentum、Antonacci GEM、Faber 類 GTAA 都應用 total return；yfinance `auto_adjust=True` 的 Close 是含股息/拆分調整近似。BIL/SHY 這種月配息 ETF 用未調整價會把除息價跌當虧損，signals 裡 BIL m12 為負就是警訊。修法同上，且不要在行 51 先 round，輸出時再 round。

3. 行 89-95 / HIGH：lookahead/off-by-one 本身在「完整月 p0」下正確：13 個完整月底價可算 1/3/6/12m；若保留 partial 就錯。`MONTHS_NEEDED` 可維持 13，但抓取層要多抓後再 tail 13。

4. 行 128-143 / MEDIUM：S1/S2 GEM filter 用 `winner_12m > BIL_12m` 比 `>0` 更貼近 Antonacci「超過 T-bill」。但 BIL 必須是 total return；若想做簡化版才用 `>0`，文件要寫明。

5. 行 157-190 / MEDIUM：S3/S4 tie 時會掉到防禦，邏輯不合理。修：
```python
winner, score = max({"VOO": voo_acc, "VSS": vss_acc}.items(), key=lambda kv: kv[1])
pick = winner if score > 0 else ("BIL" if tlt_1m is not None and tlt_1m < 0 else "TLT")
```

6. 行 199-214 / MEDIUM：VAA 公式 `12*r1+4*r3+2*r6+r12` 是標準 13612W；有些實作除以 4，不影響排名/正負。2020 後 DAA/BAA 是新策略，不是 VAA-G4 的替代更新。

7. 行 213-214 / HIGH：標準 VAA-G4 defensive 是 LQD/IEF/SHY；加入 BIL 會改策略。若只是把 SHY 視為 cash，應保留 SHY 計分、下單時映射到 BIL。

8. 行 214 / MEDIUM：defense 全 None 時 fallback `"BIL"` 會靜默錯配。
```python
if not valid_defs:
    raise RuntimeError("Missing VAA defense returns")
```

**金融設定**
不該年化，也不該改 log return；原規則用 raw simple total returns，VAA 權重就是針對 raw return。月初 cron 可以，但訊號必須鎖定「上月底完成資料」，交易假設應是下一個可交易時點。

必修：完成月過濾、`auto_adjust=True`/不先 round、VAA defensive universe、missing data fail-fast。Nice-to-have：tie-break 文件化、pin yfinance 版本、加 fixture regression test。Claude 漏掉的高風險點是「BIL 加入 VAA defensive 改策略」與「BIL 未調整價因月配息產生假負報酬」。
tokens used
92,186
結論：Claude 的 CRITICAL #1/#2 都成立。yfinance 文件列 `end` 是 exclusive、`auto_adjust` 預設 True，但沒有「只取完成月」flag；yfinance 原始碼也有修 Yahoo live weekly/monthly partial row 的邏輯。GEM 官網說絕對動能是超過 T-bill 且績效用含息 total return；VAA 13612W 公式也符合 Keller/Keuning 實作整理。來源：yfinance docs、yfinance utils、OptimalMomentum GEM、AllocateSmartly VAA、Faber GTAA。

**主要發現**
1. 行 35-55 / CRITICAL：partial 當月 bar 進入 `p[0]`。5/2 的 VOO/QQQ m1 是 5 月 1 日附近雜訊，不是 4 月完整月報酬；VAA 的 `12*r1` 會被嚴重放大。Claude 修法 A「濾掉 `dt >= 當月1日`」方向 OK，但要多抓月份；更佳是用日線 adjusted close 重採樣完成月。
```python
def fetch_monthly_closes(ticker, months=13):
    end = datetime.now().replace(day=1, hour=0, minute=0, second=0, microsecond=0)
    start = end - timedelta(days=(months + 2) * 35)
    hist = yf.Ticker(ticker).history(
        start=start.strftime("%Y-%m-%d"),
        end=end.strftime("%Y-%m-%d"),   # exclusive: 只到上月底
        interval="1d", auto_adjust=True, actions=True, repair=True,
    )
    if hist.empty:
        raise RuntimeError(f"{ticker} no data")
    hist.index = hist.index.tz_localize(None) if hist.index.tz else hist.index
    closes = hist["Close"].resample("ME").last().dropna().tail(months)
    return [(d.to_pydatetime().replace(day=1), float(c))
            for d, c in closes.sort_index(ascending=False).items()]
```

2. 行 41/51 / CRITICAL：`auto_adjust=False` 且用 `Close`。Dual momentum、Antonacci GEM、Faber 類 GTAA 都應用 total return；yfinance `auto_adjust=True` 的 Close 是含股息/拆分調整近似。BIL/SHY 這種月配息 ETF 用未調整價會把除息價跌當虧損，signals 裡 BIL m12 為負就是警訊。修法同上，且不要在行 51 先 round，輸出時再 round。

3. 行 89-95 / HIGH：lookahead/off-by-one 本身在「完整月 p0」下正確：13 個完整月底價可算 1/3/6/12m；若保留 partial 就錯。`MONTHS_NEEDED` 可維持 13，但抓取層要多抓後再 tail 13。

4. 行 128-143 / MEDIUM：S1/S2 GEM filter 用 `winner_12m > BIL_12m` 比 `>0` 更貼近 Antonacci「超過 T-bill」。但 BIL 必須是 total return；若想做簡化版才用 `>0`，文件要寫明。

5. 行 157-190 / MEDIUM：S3/S4 tie 時會掉到防禦，邏輯不合理。修：
```python
winner, score = max({"VOO": voo_acc, "VSS": vss_acc}.items(), key=lambda kv: kv[1])
pick = winner if score > 0 else ("BIL" if tlt_1m is not None and tlt_1m < 0 else "TLT")
```

6. 行 199-214 / MEDIUM：VAA 公式 `12*r1+4*r3+2*r6+r12` 是標準 13612W；有些實作除以 4，不影響排名/正負。2020 後 DAA/BAA 是新策略，不是 VAA-G4 的替代更新。

7. 行 213-214 / HIGH：標準 VAA-G4 defensive 是 LQD/IEF/SHY；加入 BIL 會改策略。若只是把 SHY 視為 cash，應保留 SHY 計分、下單時映射到 BIL。

8. 行 214 / MEDIUM：defense 全 None 時 fallback `"BIL"` 會靜默錯配。
```python
if not valid_defs:
    raise RuntimeError("Missing VAA defense returns")
```

**金融設定**
不該年化，也不該改 log return；原規則用 raw simple total returns，VAA 權重就是針對 raw return。月初 cron 可以，但訊號必須鎖定「上月底完成資料」，交易假設應是下一個可交易時點。

必修：完成月過濾、`auto_adjust=True`/不先 round、VAA defensive universe、missing data fail-fast。Nice-to-have：tie-break 文件化、pin yfinance 版本、加 fixture regression test。Claude 漏掉的高風險點是「BIL 加入 VAA defensive 改策略」與「BIL 未調整價因月配息產生假負報酬」。
