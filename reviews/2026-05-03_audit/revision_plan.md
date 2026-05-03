# dualmon 修訂計畫（Claude + Codex 整合）

> 日期：2026-05-03
> 來源：`claude_audit.md` + `codex_review.md` 兩份審稿合併、去重、排序
> 用途：給 Ray 拍板 → 我照清單改 `usdn_updater.py`

---

## TL;DR

兩 AI 共識：**至少 4 條必修（含 Codex 抓到 Claude 漏看的兩條高風險）+ 3 條建議修 + 4 條可選**。

**Codex 抓到 Claude 漏看的兩個高風險**：
1. **BIL 加入 VAA defensive universe 改了原策略** — Keller 2017 標準 VAA-G4 defensive 是 LQD/IEF/SHY 3 檔，code 用 4 檔（多了 BIL）
2. **BIL 月配息 + `auto_adjust=False` → 假負報酬** — `signals.json` BIL m12 = -0.37% 是除息日累積跌幅，真實 total return 約 +5%。這直接破壞 S1/S2 的 GEM filter（winner_12m vs BIL_12m 的 reference 失真）

**Codex 給的更佳修法**（比 Claude 推薦的 A 更好）：用日線資料 `resample("ME")` 取月底完成 close，順便解決 partial month + auto_adjust + repair 等多個問題。

---

## 必修（修了直接影響投資決策）

### M1. partial 當月 bar + 用日線重採樣月底（CRITICAL）

**對應**：Claude CRITICAL #1、Codex 主要發現 1
**位置**：`fetch_monthly_closes()` lines 35-56

**問題**：5/2 跑出 VOO m1 = +0.29% 是「2 個交易日噪音」不是月報酬。VAA `12*r1` 主導項被嚴重放大。

**修法**（採用 Codex 建議，用日線 resample，比 Claude 修法 A 更穩）：

```python
def fetch_monthly_closes(ticker: str, months: int = MONTHS_NEEDED) -> list[tuple]:
    # end 設為本月 1 號（exclusive）→ 只到上月底完成資料
    end = datetime.now().replace(day=1, hour=0, minute=0, second=0, microsecond=0)
    start = end - timedelta(days=(months + 2) * 35)  # 多抓 2 個月當緩衝

    tk = yf.Ticker(ticker)
    hist = tk.history(
        start=start.strftime("%Y-%m-%d"),
        end=end.strftime("%Y-%m-%d"),
        interval="1d",                    # 改日線
        auto_adjust=True,                 # ← M2 一起修
        actions=True,
        repair=True,                      # ← yfinance 自動修錯誤資料
    )

    if hist.empty:
        raise RuntimeError(f"{ticker} no data")  # ← M4 一起修：fail fast

    if hist.index.tz is not None:
        hist.index = hist.index.tz_localize(None)

    # resample 到月底，取最後一個交易日 close
    closes = hist["Close"].resample("ME").last().dropna().tail(months)
    return [(d.to_pydatetime().replace(day=1), float(c))
            for d, c in closes.sort_index(ascending=False).items()]
```

**順便解決**：M2 (auto_adjust)、M4 (fail fast)、M5 (不要過早 round)。

**驗證**：跑完看 `signals.json`，VOO m1 應該變成 4 月真實月報酬（約 +4-5%）；BIL m12 從 -0.37% 變正（約 +5%）。

### M2. `auto_adjust=True`（CRITICAL）

**對應**：Claude CRITICAL #2、Codex 主要發現 2
**位置**：line 42

**問題**：未調整收盤忽略股息與分割。BIL/SHY 月配息 ETF 除息日的價跌會被當虧損 → BIL m12 變負 → S1/S2 GEM filter 用 BIL 12m 當門檻時，門檻被嚴重低估。

**修法**：M1 的 code 已含 `auto_adjust=True`。同時把 `Close` 從未調整改成已調整（adjusted close ≈ total return 近似）。

**邊界條件**（Codex 提醒）：不要在 `fetch_monthly_closes` 裡 `round(close, 2)`，會損失精度，等到輸出 `signals.json` 時再 round。M1 的 code 已用 `float(c)` 不 round。

### M3. VAA defensive universe 改回標準 3 檔，或文件化客製化決定（HIGH）

**對應**：Codex 找到、Claude 漏看
**位置**：lines 208-209 / 219-220

**問題**：
- 標準 Keller 2017 VAA-G4 defensive **= LQD / IEF / SHY**（3 檔）
- 現 code defensive = **LQD / IEF / SHY / BIL**（4 檔，多了 BIL）

加入 BIL 改變了 VAA-G4 行為。BIL（短期國庫券，殖利率 ≈ 純現金）vs SHY（短期公債 1-3y）的 vaa 分數會有差，加入 BIL 等於把「最保守的現金」也納入競賽。

**Ray 必須拍板的決策**：

| 選項 | 行為 | 適合誰 |
| :--- | :--- | :--- |
| **A. 改回標準 3 檔** | defs = {LQD, IEF, SHY}，刪除 BIL | 想嚴格遵循 Keller 2017 paper |
| **B. 保留 4 檔，加文件** | 維持現狀，README 註明「客製化版本，加入 BIL 作為純現金選項」 | 你**有意識**地希望加入 cash 選項，且原本就知道這偏離原版 |
| **C. SHY 計分 + BIL 下單** | defs 仍 {LQD, IEF, SHY}，但若 SHY 贏，最後 pick 改寫成 BIL（因為 BIL 流動性更好/手續費更低） | 用 SHY 算分但實際買 BIL |

**我推薦 A**（改回標準 3 檔），理由：
1. 你寫的「VAA 攻擊型」策略名暗示遵循 Keller 標準
2. 看 signals.json 現狀 BIL 因 M2 bug 是 vaa = -0.0517 而 SHY = -0.103，BIL 「贏」是因為 BIL 配息少受 unadjusted bug 影響——一旦 M2 修了，BIL 跟 SHY 的 vaa 會接近，加入 BIL 的意義變小
3. 標準版易維護、易跟其他 VAA 文獻對照

但這是 **Ray 的策略偏好題**，不是 bug，**請拍板**。

### M4. 資料抓取失敗 → fail fast 而非靜默 partial signals（HIGH）

**對應**：Claude MEDIUM #6（強化）+ Codex MEDIUM 8（防禦 fallback）+ 主要發現 1（fetch raise）
**位置**：lines 44-46（`fetch_monthly_closes` 的 hist.empty）+ line 220（VAA defense fallback）

**問題**：
- `fetch_monthly_closes` 抓不到 ticker 時印 warning + return `[]` → 該 ticker 從 `all_returns` 缺席 → 下游 `compute_signals` 用 `r(...)` 取 None → 該策略被靜默跳過
- VAA defense 全 None 時 fallback hardcode `"BIL"` → 假裝防禦中

兩個都是「靜默壞掉」型 bug。在投資工具裡這比「擲出 error 讓 cron job 顯示紅燈」更危險。

**修法**：

```python
# fetch_monthly_closes
if hist.empty:
    raise RuntimeError(f"{ticker}: yfinance returned empty")

# 在 main() 的 try/except 也對應改：
try:
    data = fetch_monthly_closes(etf)
except Exception as e:
    print(f"失敗 ({e})")
    raise  # ← 之前是 continue，改成 raise 讓 cron job 失敗顯示

# VAA defense fallback
valid_defs = {k: v for k, v in defs.items() if v is not None}
if not valid_defs:
    raise RuntimeError("VAA defense universe missing data")
pick = max(valid_defs, key=valid_defs.get)
```

**取捨**：是否所有 ticker 全要齊？S5 VAA 用 8 個 ticker，如果 LQD 抓不到，是要全 strategy fail 還是只 skip S5？

我建議：**S5 缺資料 → S5 不寫入 strategies + vaa_block = None + UI 顯示「資料不全」**。但 ETF list 中任何一個全失敗應該全程 fail（因為這代表 yfinance 整體有問題）。

---

## 建議修（邏輯小漏洞，影響邊界情境）

### S1. S3/S4 tie-break（MEDIUM）

**對應**：Claude MEDIUM #5、Codex 主要發現 5
**位置**：lines 159-167（S3）、lines 184-191（S4）

**修法**（Codex 給的簡潔版）：

```python
# S3
voo_acc, vss_acc, tlt_1m = r("VOO", "accel"), r("VSS", "accel"), r("TLT", "1m")
if voo_acc is not None and vss_acc is not None:
    candidates = {"VOO": voo_acc, "VSS": vss_acc}
    winner, score = max(candidates.items(), key=lambda kv: kv[1])
    if score > 0:
        pick = winner
    elif tlt_1m is not None and tlt_1m < 0:
        pick, score = "BIL", None
    else:
        pick, score = "TLT", None
    # ...
```

平手時 `max` 取字典插入順序的第一個（VOO），合理。

### S2. TLT 防禦判斷加 buffer 或改用 3m（MEDIUM）

**對應**：Claude HIGH #4
**位置**：lines 164、188

**問題**：M1 修了之後 TLT 1m 已是真實月報酬，但等邊界（如 -0.001%）仍可能抖動。

**修法**：兩擇一
- A. 加 buffer：`tlt_1m < -0.01`（必須跌超過 1% 才轉 BIL）
- B. 改 3m：`tlt_3m < 0`（更穩）

我推薦 A 維持「短期動能」精神但加保險。

### S3. `accel` 命名 → 改 `mom_score` 或加註解（LOW，從 Claude HIGH 降級）

**對應**：Claude HIGH #3、Codex 沒特別提
**位置**：line 100

**取捨**：Codex 沒列為優先項，可能因為這只是命名問題、不影響輸出。但 Ray 自己看 code 會被誤導。

**修法**（最低成本）：保留變數名 `accel`，但**加註解 + 對前端 subtitle 也對齊**：
```python
# 注意：這不是真正的「加速度」(d momentum / dt)，是 1m/3m/6m 動能簡單平均
result["accel"] = (result["1m"] + result["3m"] + result["6m"]) / 3
```
`compute_signals()` line 264 已經有 `subtitle: "分數 = avg(1m, 3m, 6m)"`，誠實標示，無需更動。

---

## 可選（純改善，不影響行為）

### O1. 重複呼叫 `compute_signals()` 改成呼叫一次

```python
# main()
data = compute_signals(all_returns)
write_signals_json_data(data)  # 改成接受已算好的
print_signal_data(data)
```

### O2. `update_excel()` column 3/4/5 加註解或簡化

```python
# 目前把 3/4/5 全填 close_val 是 placeholder for OHLC 公式參考
# 如不需要可改為只寫 column 1, 2
```

### O3. 補 README

最簡版列：
```markdown
# dualmon — 雙動能策略監控

每月 GitHub Actions 從 yfinance 抓 ETF 月線，算 5 個動能策略訊號。

## 5 個策略
- S1 原版雙動能（Antonacci GEM 變種，VOO/VXUS/BND）
- S2 拉風增強版（QQQ 取代 VOO）
- S3 加速雙動能（VOO/VSS 比 1/3/6m 平均，TLT 為防禦）
- S4 騷速雙動能（QQQ 取代 VOO）
- S5 VAA 攻擊型（Keller 2017，4 攻 vs 3 守）

## 本機跑
pip install yfinance openpyxl
python usdn_updater.py

## 排程
每月 2 號 UTC 13:00（台北 21:00）by GitHub Actions

## 已知限制
- 用 adjusted close 近似 total return
- VAA defensive universe 是標準 LQD/IEF/SHY 3 檔
```

### O4. pin yfinance 版本

`requirements.txt` 改成：
```
yfinance==0.2.55  # 或當下最新穩定版
openpyxl>=3.1
```

避免 yfinance 之後 breaking change（過去半年改過很多次 API）。

---

## 修完預期 signals.json 變化（驗證對照表）

修了 M1 + M2 後預期：

| 欄位 | 修前（5/2 partial + unadjusted） | 修後預期 |
| :--- | :--- | :--- |
| **VOO m1** | +0.29% | **約 +4-5%（4 月實際月報酬）** |
| **QQQ m1** | +0.96% | **約 +4-6%** |
| **VSS m1** | -0.23% | **接近 4 月實際** |
| **TLT m1** | -0.01% | **可能小幅正/負（月報酬）** |
| **BIL m12** | -0.37%（除息累積跌） | **約 +5%（純殖利率含配息）** |
| **VXUS m12** | 沒給但低於真值 | **+1.5-2pp（含股息）** |
| **BND vaa** | -0.149 | **可能轉正** |
| **S1 pick** | VXUS | **可能改變**（VOO/VXUS 重排序 + 對 BIL 門檻變嚴） |
| **S5 mode** | 防禦 | **可能切回攻擊**（BND vaa 翻正後 4 攻全正） |

修完 M3（VAA defense 改 3 檔）：S5 防禦時 `pick` 從 BIL 變 SHY/IEF/LQD 之一。

---

## 實作順序與時程

| 步驟 | 動作 | 預估時間 | 需 Ray 確認 |
| :--- | :--- | :--- | :--- |
| 1 | M1 + M2 + M5（一起改 `fetch_monthly_closes`） | 10 分鐘 | ❌ |
| 2 | M3（VAA defensive 3 檔 / 4 檔 / 客製化） | 5 分鐘 | ✅ **必問** |
| 3 | M4（fail fast）| 5 分鐘 | ❌ |
| 4 | S1（tie-break） | 3 分鐘 | ❌ |
| 5 | S2（TLT buffer 或改 3m）— 二擇一 | 3 分鐘 | ✅ |
| 6 | S3（accel 加註解） | 1 分鐘 | ❌ |
| 7 | 本機跑 `python usdn_updater.py`，diff signals.json | 5 分鐘 | ❌ |
| 8 | 給 Ray 看 diff，確認方向 | — | ✅ |
| 9 | O1-O4（可選改善） | 15 分鐘 | ✅（要做哪幾條） |
| 10 | git commit + push | 2 分鐘 | ✅（push 要再問） |

**全部跑完 ~50 分鐘**。

---

## 待 Ray 拍板的決策（共 3 個）

1. **M3 VAA defensive universe**：改回標準 3 檔（A）／保留 4 檔加文件（B）／SHY 計分 BIL 下單（C）？
2. **S2 TLT 防禦**：加 buffer（A）／改 3m（B）／不修？
3. **O1-O4 可選項**：要全做、選做、或全不做？
