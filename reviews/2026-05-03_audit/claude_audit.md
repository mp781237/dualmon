# dualmon `usdn_updater.py` Claude 自審報告

> 日期：2026-05-03
> 對象：`mp781237/dualmon` commit @main（2026-05-02 跑過一次 cron）
> 審查者：Claude Opus 4.7（CLI）
> 目標：在送 Codex 第二意見前，先把 Claude 看出的可疑點列清楚

---

## 嚴重度分級

- **🔴 CRITICAL** — 公式錯誤或實際資料證實的 bug，會影響投資決策
- **🟠 HIGH** — 邏輯有疑慮、邊界條件處理不當，特定情境會誤判
- **🟡 MEDIUM** — 命名/可讀性問題，不影響輸出但會誤導維護者
- **🟢 LOW** — 風格、效能、文件

---

## 🔴 CRITICAL #1 — 月線資料含「未完成的當月」，1m 報酬變成「幾天報酬」

**位置**：`fetch_monthly_closes()` lines 36-56；`calc_returns()` lines 92-95

**證據**（最有力的單一證據）：`signals.json` 2026-05-02 跑出：
- VOO m1 = +0.29%（一個月怎麼可能只有 0.29%？大盤靜止？）
- QQQ m1 = +0.96%
- VSS m1 = -0.23%

對照 m3 = 4.99% / 11.01% — 顯然 m1 不是真正一個月的報酬。**這證實 yfinance 月線回傳了 2026-05 那一根還沒完成的 bar（只有 5/1, 5/2 兩個交易日）**，然後 `p[0]` 就是這根 partial bar、`p[1]` 才是 4 月底完整收盤。

實際算式變成：
- 「1m return」= May 5/2 close / Apr 月底 close - 1 ≈ **1-2 個交易日**的報酬
- 「3m return」= May 5/2 close / Feb 月底 close - 1 ≈ **3 個月又 2 天**
- 「12m return」= May 5/2 close / 去年 5 月底 close - 1 ≈ **12 個月又 2 天**

m3/m6/m12 偏差較小（多 1-2 天無傷大雅），但 **m1 完全失真**。

**影響範圍**：
- S3、S4「加速雙動能」直接用 `accel = avg(1m, 3m, 6m)`，1m 進雜訊（雖只佔 1/3）
- VAA 公式 `12*1m + 4*3m + 2*6m + 1*12m`，1m 權重最高（佔 ~63%），**partial bar 1m 直接主導 VAA 分數**——這個影響非常大
- 看現在 S5 切到防禦模式，原因是 BND vaa = -0.149。BND 月底前的真實 1m 報酬我們不知道，但 partial 1m 進去後可能讓 BND 偏負

**建議修法**（三選一，推薦 A）：

A. **明確去掉當月 partial bar**（最簡單最安全）：
```python
# 在 fetch_monthly_closes 結尾加
today = datetime.now().replace(tzinfo=None)
results = [(d, c) for d, c in results if d.replace(day=1) < today.replace(day=1)]
```
這樣不管 yfinance 行為怎樣，永遠只用「已收盤完成」的月份。配 `MONTHS_NEEDED = 13` 得改成 `14`（多抓一根備用）。

B. **改用日線資料自己取月底**：用 `interval="1d"` 抓 14 個月日線，groupby month 取最後一個交易日的 close。最精準，控制力最強，但 code 改動較大。

C. **end_date 設為上月最後一天**：`end = datetime(today.year, today.month, 1) - timedelta(days=1)`。簡單，但仍依賴 yfinance 行為。

---

## 🔴 CRITICAL #2 — `auto_adjust=False` 用未調整收盤，忽略股息與分割

**位置**：`fetch_monthly_closes()` line 42

```python
hist = tk.history(..., auto_adjust=False)
```

**問題**：`auto_adjust=False` 取得的是「Close」原始收盤價，**沒有調整股息與股票分割**。雙動能本質上比較「總報酬」（含再投資股息），不調整會：

- **系統性低估配息較高的 ETF**（VXUS ~3% / 年、BND ~3% / 年、TLT ~3.5% / 年配息）
- **完全錯漏分割**（VOO/QQQ 偶發拆股，會產生假的暴跌訊號）

對投資決策的具體影響：
- S1 比 VOO vs VXUS 的 12m return：VOO 配息 ~1.3%、VXUS 配息 ~3%，**VXUS 被系統性低估 ~1.7pp**。今天 VOO=22.29%、VXUS 假設 18%（若加回股息應該是 ~21%），可能讓不該贏的 VOO 贏
- 絕對動能 vs BIL：BIL 是現金等值（殖利率約等於 yield），其他股票被低估股息後，**過門檻機率被低估**——更容易誤判進防禦模式

**建議修法**：
```python
hist = tk.history(..., auto_adjust=True)  # 改 True
```

`auto_adjust=True` 後 `Close` 欄位即為調整後價格（含股息與分割回填）。**這是金融產業計算動能的標準做法**。

注意：改完後 m1/m3/m6/m12 的數值會略升（多了股息成分），歷史 backtest 結果可能改變排序（但這才是正確的）。

---

## 🟠 HIGH #3 — `accel` 命名為「加速」但實際是「平均動能」

**位置**：`calc_returns()` line 100

```python
result["accel"] = (result["1m"] + result["3m"] + result["6m"]) / 3
```

數學上「加速度」是動能的**變化率**，例如 `(recent_3m - prev_3m)`、或 `(1m_return - 3m_avg_per_month)`。
這裡的 `accel` 其實是「短中期動能簡單平均」，跟 acceleration 的物理意義無關。

`subtitle: "分數 = avg(1m, 3m, 6m)"`（在 `compute_signals` 第 264 行）已誠實標註是平均，但策略名稱叫「加速雙動能」、變數叫 `accel` 會誤導使用者以為這真的測加速度。

**建議修法**（兩擇一）：

A. **改名為 `momentum_avg` 或 `mom_score`**（誠實命名）：
```python
result["mom_score"] = (result["1m"] + result["3m"] + result["6m"]) / 3
```
策略也對應改名「短中期動能均值」。

B. **真的算加速度**（保留現有命名，但改公式）：
例如 `accel = result["1m"] - (result["3m"] / 3)`，比較最近 1 個月的月化報酬 vs 過去 3 個月的月化平均，正值代表動能「加速中」。

**Ray 偏好我猜是 A**——維持既有 5 個策略的選股結果不變，只改命名/註解，避免動到實際決策邏輯。

---

## 🟠 HIGH #4 — TLT 1-month 防禦判斷受 partial month bug 連累

**位置**：`compute_signals()` line 164、line 188

```python
elif tlt_1m is not None and tlt_1m < 0:
    pick, score = "BIL", None
else:
    pick, score = "TLT", None
```

S3/S4 在「股票兩邊都沒打贏」時用 TLT 1-month 是否為負來決定 BIL（現金）還是 TLT（長債避險）。但 **`tlt_1m` 就是 CRITICAL #1 那條 partial bar 的 1m 報酬**——如果現在是月初 2 號，TLT 1m 可能只是 1-2 天的小漲跌，對「長債最近是否被搶」判讀失真。

**附帶**：當 5/2 跑時 TLT 1m = -0.012%（接近 0），剛好決定 BIL vs TLT。這種等邊界判斷在資料雜訊下不穩定。

**建議修法**：CRITICAL #1 修了之後這條自動好轉。額外可考慮把 `tlt_1m < 0` 改成 `tlt_3m < 0`（更穩定），或加入 buffer（例如 `tlt_1m < -0.01` 才轉防禦）避免邊界抖動。

---

## 🟡 MEDIUM #5 — S3/S4 完全並列時掉到 defense

**位置**：`compute_signals()` lines 160-167（S3）、lines 184-191（S4）

```python
if voo_acc > vss_acc and voo_acc > 0:
    pick = "VOO"
elif vss_acc > voo_acc and vss_acc > 0:
    pick = "VSS"
elif tlt_1m is not None and tlt_1m < 0:
    pick = "BIL"
else:
    pick = "TLT"
```

如果 `voo_acc == vss_acc`（精確相等到 6 位小數，極罕見），兩個 `>` 條件都 false，掉到防禦邏輯。即使兩者都 > 0、應該選股票，卻被誤判為「沒打贏」進防禦。

**機率**：浮點數精確相等很少見，但加 tie-break 不費成本。

**建議修法**：第二個分支改 `>=`，明確讓 VSS 在平手時贏（因為 VSS 是「小型新興」，通常風險較高，平手選 VOO 更保守也合理 — 那就讓 VOO 贏）：

```python
if voo_acc >= vss_acc and voo_acc > 0:
    pick = "VOO"
elif vss_acc > voo_acc and vss_acc > 0:
    pick = "VSS"
elif ...
```

---

## 🟡 MEDIUM #6 — VAA defense 預設 fallback 是 BIL（hardcode）

**位置**：`compute_signals()` line 220

```python
pick = max(valid_defs, key=valid_defs.get) if valid_defs else "BIL"
```

如果 `valid_defs` 全部是 None（極端情境，例如 yfinance 全失敗），fallback hardcode 為 "BIL"。這個 fallback 不會在正常情境觸發（4 個 ETF 全抓不到才會），但寫死字面值少了個 sanity check。

**建議修法**：fallback 時應該 raise error 或用 `pick = None`，讓上層知道資料異常，而不是裝沒事丟個 BIL 出去。`signals.json` 應該反應「無資料」而不是「假裝防禦中」。

---

## 🟢 LOW #7 — `compute_signals()` 被呼叫兩次

**位置**：`main()` 中 `write_signals_json` 內部呼叫一次（line 272），`print_signal` 內又呼叫一次（line 286）。

無 bug，但浪費。改成：
```python
data = compute_signals(all_returns)
write_signals_json(data)  # accept already-computed
print_signal(data)
```

---

## 🟢 LOW #8 — `update_excel()` 把 column 3/4/5 都填同一個 close_val

**位置**：`update_excel()` lines 70-74

```python
for c in (3, 4, 5, 6, 7):
    if c in (3, 4, 5):
        ws.cell(row=r, column=c, value=close_val)
    else:
        ws.cell(row=r, column=c, value="")
```

語意不明——column 3/4/5 在 Excel 裡分別是什麼？看起來像 Open/High/Low 全填 Close，可能是 placeholder 給下游公式用。

**建議**：加一行註解說明這三欄的用途（或者乾脆刪掉、只保留 column 1 日期、column 2 close）。

---

## 🟢 LOW #9 — 沒有 README

repo 目前沒有 README.md。新人接手要靠讀 source 才知道這幹嘛。`index.html` 有 UI 但不解釋 5 個策略的數學定義。

**建議**：補一個簡短 README 列：(1) 5 個策略邏輯白話 (2) cron 排程 (3) 本機怎麼跑 (4) 資料來源 (5) 已知限制。

---

## 給 Codex 的待驗證清單

1. **CRITICAL #1**：請驗證 yfinance `interval="1mo"` 在月初 2 號跑時是否真的會回傳 partial 當月 bar？修法 A 是否有更好的選擇？
2. **CRITICAL #2**：`auto_adjust=True` 是 yfinance 動能策略的最佳實作嗎？有沒有 corner case 我沒想到？
3. **HIGH #4**：TLT 1m 是否該改成 3m / 6m / 用其他指標（殖利率曲線、credit spread）？
4. 我**漏看**的：請從金融工程角度檢查 5 個策略決策樹有沒有 lookahead bias、index off-by-one、tie-break、邊界條件問題。
5. VAA 公式 `12*r1 + 4*r3 + 2*r6 + r12` 是否是 Keller 2017 原文標準？或有更新版本？
6. dual momentum (Antonacci GEM) 經典版的「絕對動能 filter」用 T-bill 12m 還是「risk-free rate」？code 用 BIL 12m 是否就是這個意思？

---

## 推薦修法優先序

1. **先修 CRITICAL #1 + #2**（partial month + auto_adjust）。這兩個會直接影響 5 月開始每個月的訊號。
2. **再修 HIGH #3 + #4**（命名 + TLT 防禦）。
3. **再修 MEDIUM #5 + #6**（tie-break + fallback）。
4. **LOW 全部隨手做**（重複呼叫、Excel 欄位、README）。

預估改動 LOC：CRITICAL 兩條約 5-10 行；HIGH 兩條約 10-15 行；MEDIUM 兩條 5 行；LOW 三條 30-50 行。整體不到 100 行。
