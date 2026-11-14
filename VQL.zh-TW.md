# Valynox Quant Language（VQL）說明

- VQL 是 AlphaPrime Stocks 的選股策略語言：一門語言寫分數公式、選股範圍、回測規則、篩選與檢驗。作者通常是 AI——人講白話，AI 經 MCP 寫成 VQL。
- 同一份內容另有英文版 `VQL.en.md`、給 AI 讀的純文字版 `vql.txt`，以及白話 → 寫法的範例庫 `EXAMPLES.zh-TW.md`（機器讀的 `examples.json`）。
- 這份只寫現在能用的寫法；文中的例子都過了 plan（只驗寫法、不開跑）。
- 在 MCP 裡：分數公式放 `stock_backtest` 的 `formulas`，選股範圍放 `universe`，出場放 `exit_when`／`keep_when`／`grid`，其餘回測與篩選參數（`portfolio`、`screen_gate`、`checks`、`check_params`…）放 `advanced`；查寫法用 `stock_formula_help`。

目錄：1 VQL 是什麼｜2 文法｜3 值與單位｜4 全部算子｜5 保留函數的原子定義式｜6 相似度｜7 資料欄｜8 選股範圍｜9 回測｜10 搜尋｜11 篩選指標與檢驗｜12 自訂函數｜13 上限｜14 錯誤碼與怎麼改｜15 還不能寫的｜16 附錄：完整範例

---

## 1. VQL 是什麼

### 1.1 設計初衷

Valynox Quant Language（VQL）是 AlphaPrime Stocks 的策略語言。作者是 AI，全部透過 MCP 寫；人講白話，AI 把白話寫成 VQL。

設計初衷：**要寫得出各式各樣、連很奇怪的選股策略**。所以語言只給最基本的原子與算子，所有策略都由它們組成。由此而來的三條規矩：

1. 算子保持純粹。不收 RSI、MACD、布林通道這類技術指標型算子；要用就拿原子組成自訂函數（第 12 章有 RSI 的寫法）。
2. 沒有「節點數規則」。公式多大都可以寫，只有一個技術上限（第 13 章）；每支策略附複雜度數據。
3. 類股、題材不用人工清單。由相關性自訂籃子（第 6 章）。

### 1.2 四件事，一門語言

| 事 | 寫在哪裡 | 值的型別 | 產出 |
|---|---|---|---|
| 搜尋 | 分數公式：`stock_backtest` 的 `formulas=[…]`、`trees_from`（自己給、自訂函數格點、系統隨機生成；第 10 章）、自訂函數的 body | i64 定點（第 3 章） | 每天每檔一個分數，越高排越前；一次試很多條 |
| 回測 | `stock_backtest` 的 `portfolio={…}`（第 9 章） | i64 定點 | 怎麼從排名變成持股：換股時點、檔數、權重、出場 |
| 篩選 | 門檻 `screen_gate`、排序 `grid_by`、`apx.metric.v1`（第 11 章） | 實數（double） | 一支策略留不留 |
| 檢驗 | `checks`、`apx.check.v1`（第 11 章） | 整數（i64） | 換條件重跑後的指標與通過條件 |

四件事共用同一份文法、同一套運算子與優先序。差別在值的型別與能用的名字：搜尋與回測用面板的欄與第 4 章的算子；篩選與檢驗用策略層的原子（`r`、`m`、`mean`、`sd`…）。選股範圍（第 8 章）是一條結果為 BOOL 的生成算式。

### 1.3 三層名字

| 層 | 長相 | 誰定義 | 例 |
|---|---|---|---|
| 原子 | 沒有點的名字 | 引擎；新增要寫引擎程式 | `LAG`、`WSUM`、`GT`、`LOGR`、`SCANMAX` |
| 保留函數 | 沒有點的名字，或全名 `apx.NAME` | 官方；由原子組成 | `RET`（＝`apx.RET`）、`RVOL`、`CS_PCT` |
| 自訂函數 | `user.<作者>.NAME` | 使用者，經 `my_stock_strategies_change action=install` 存檔 | `user.local.RSI` |

- 保留函數的原子定義式列在第 5 章：每一個保留函數都拆得回原子，定義式與用名字寫逐位元相同。
- 資料不是計算：由外部取得的資訊（每個市場各一套），也就是第 7 章的欄位（欄鍵 `LEAF` 就是欄位本身）、第 L 層歸屬的編號 `PARENT_ID(L)`（4.8 節），加上第 7.4 節由資料欄相除得到的 5 個財報比值（衍生資料，例 `FN_EP`）。
- 自訂函數展開到原子後跟直接寫原子是同一棵樹，結果相同。
- 另有官方的類別名詞 `cls.NAME()`（第 8 章），只用在條件。

---

## 2. 文法

### 2.1 節點

```
節點 = 欄鍵[:單位]  |  數字[:單位]  |  名字(參數…, 輸入…)  |  $參數名
```

| 項 | 規則 | 例 |
|---|---|---|
| 欄鍵 | 面板的欄名，不分大小寫；可帶 `:單位` 核對 | `close`、`fund.mktcap`、`fund.mktcap:AMOUNT` |
| 數字 | 十進位，可帶小數與 `e` 指數；可帶 `:單位` | `20`、`0.5:RATIO`、`-0.08`、`5e9`、`1.5e-3` |
| 呼叫 | **參數在前、輸入在後**；零輸入照帶括號 | `ROLL_MEAN(20, close)`、`MONTH()`、`PARENT_ID(1)` |
| 兩個參數的算子 | 兩個數字分開寫 | `RET_SKIP(252, 21, close)`、`RWIN(4, 1, revenue)` |
| `$參數名` | 只在自訂函數的 body 裡（第 12 章） | `WSUM($m, x)` |
| 字串 | `"…"`，只在 `in {…}` 裡或代碼欄 `==`／`!=` 的右邊 | `symbol == "2330"` |
| 空白 | 空白、換行都可以，不影響意思 | |

- 參數一律是整數。參數個數＝引數個數－輸入個數。
- 算子最多三個輸入：三輸入的 `STATESCAN`（第 4.13 節）、`GRESID`（第 4.8 節）直接寫；`SELECT` 寫成 `c ? a : b`；四輸入的 `REC2` 公式文字不收。
- 巢狀深度上限 64 層括號。

### 2.2 運算子寫法與函數寫法是同一棵樹

| 運算子寫法 | 等於 | 備註 |
|---|---|---|
| `a + b` | `ADD(a, b)` | 兩邊同單位 |
| `a - b` | `SUB(a, b)` | 兩邊同單位 |
| `a * b` | `MUL(a, b)` | 單位規則見 3.3 |
| `a / b` | `RATIO(a, b)` | b = 0 → 無效 |
| `a % b` | `MOD(a, b)`＝`SUB(a, MUL(b, FLOOR(RATIO(a, b))))` | 餘數和 b 同號；b = 0 → 無效 |
| `a > b` | `GT(a, b)` | 結果 BOOL |
| `a < b` | `LT(a, b)` | |
| `a >= b` | `NOT(LT(a, b))` | |
| `a <= b` | `NOT(GT(a, b))` | |
| `a == b` | `NOT(OR(GT(a, b), LT(a, b)))` | 代碼欄的 `==` 見 2.4 |
| `a != b` | `OR(GT(a, b), LT(a, b))` | |
| `a && b` | `MUL(a, b)` | 兩邊都要是 BOOL |
| `a \|\| b` | `OR(a, b)` | 兩邊都要是 BOOL |
| `!a` | `NOT(a)` | a 要是 BOOL |
| `-a` | `NEG(a)` | `-` 後面直接接數字是負常數 |
| `c ? a : b` | `FILL(MASK(c, a), MASK(NOT(c), b))` | c 要是 BOOL；c 無效 → 無效 |
| `x in {v1, v2, …}` | 集合成員（一個節點） | 結果 BOOL；x 無效 → 無效 |

**優先序由低到高**（同一層左結合）：

| 層 | 運算子 |
|---|---|
| 1（最低） | `? :` |
| 2 | `\|\|` |
| 3 | `&&` |
| 4 | `==`　`!=` |
| 5 | `>`　`<`　`>=`　`<=`　`in {…}` |
| 6 | `+`　`-` |
| 7 | `*`　`/`　`%` |
| 8（最高） | 一元 `-`　`!` |

括號分組。例：`close / LAG(20, close) > 1.1 && volume > ROLL_MEAN(20, volume)` 讀成 `((close / LAG(20, close)) > 1.1) && (volume > ROLL_MEAN(20, volume))`。

注意兩點：

- `&&`、`||` 沒有短路：任一邊無效，結果就無效（`OR` 也一樣）。
- `?:` 的中段裡，`:` 後面接的是已知單位名才當單位後綴，否則是 `?:` 的冒號。`c ? 0.5:RATIO : 1:RATIO` 與 `c ? 0.5 : 1` 都對。

### 2.3 常數與單位

| 寫法 | 讀成 |
|---|---|
| `0.5:RATIO`、`20:DAYS`、`1:COUNT`、`0:PRICE` | 明寫單位的常數 |
| `close > 100` | 沒寫單位的數字照另一邊的單位讀（這裡是 `100:PRICE`） |
| `WREG(20, 0, close, 100)`、`CLIP(close, 10, 100)` | 同一組輸入（加減、比大小、`MAX2`／`MIN2`、`FILL`、`?:` 的兩支、`WREG` 的 y 與 x、保留函數同一列的 X 輸入）是同一種量：沒寫單位的常數跟有單位的那個（這裡都是 PRICE）。原子跟保留函數同一條 |
| `FILL(MASK(r, close), MASK(NOT(r), 0))`、`close + LAG(1, 5)` | 常數包在輸出跟輸入同單位的原子裡（`MASK` 的值、`LAG`、`NEG`、`ROLL_*`、`SCANSUM` 的值…）也一樣跟（這裡 `0`、`5` 是 PRICE） |
| `close / LAG(20, close) > 1.1` | 左邊是 RATIO，`1.1` 讀成 `1.1:RATIO` |
| `MUL(x, 2)`、`RATIO(x, 3)` | 乘除的常數沒有另一邊可照時讀成 RATIO（不帶單位的倍數） |
| `fund.mktcap > 5e9` | 大常數自動處理：放不進 32 位的常數收 7 位有效數字 × 10 的 0～12 次方 |

- 界線：包在原子裡的常數只跟數值單位、不跟 BOOL（`FILL(c, MASK(d, 0))` 要寫 `0:BOOL`；直接寫的 `FILL(c, 0)` 照舊跟 BOOL）；權重（`GWMEAN`、`GWMEAN_BY` 的 w）、`GRESID` 與 `CS_NEUTRAL_SIZE` 的 x 和 z 各自一組，沒寫單位照預設 RATIO。
- 常數的小數位超過該單位的定點精度（1e6 定點是小數 6 位；整數單位不收小數）→ 拒收。
- 公式不能只是常數，至少要有一個欄。

### 2.4 代碼欄、集合、字串

`industry`、`market`、`common`、`symbol` 是 CODE 單位的代碼欄：只能 `==`、`!=`、`in {…}` 或當群鍵，不能做數學。

| 寫法 | 意思 |
|---|---|
| `market in {TW, TWO}` | 上市或上櫃（交易所名是常數） |
| `symbol in {"2330", "2317"}` | 這兩檔（代號用雙引號） |
| `symbol == "2330"` | 這一檔 |
| `industry in {24, 28}` | 產業代碼（代碼表用 `stock_formula_help topic=universe` 查） |
| `MONTH() in {3, 5, 8, 11}` | 數值欄也能用 `in`，值照 x 的單位讀 |

- 條件（選股範圍、`hold`、`rebalance_when`、`exposure`）與分數公式、自訂函數都收。
- 分數公式裡的 `x in {…}`、`symbol == "2330"` 是一欄 BOOL，在輪開始時算好接在面板後面，是結果庫身分的一部分。可當 `MASK`、`?:` 的條件或乘上去。
- 分數公式裡的產業群寫 `PARENT_ID(1)`（群鍵），不是 `industry`。

### 2.5 別名

| 別名 | 等於 |
|---|---|
| `DIV` | `RATIO` |
| `WMAX` | `ROLL_MAX` |
| `WMIN` | `ROLL_MIN` |
| `apx.NAME` | `NAME`（保留函數的全名：`apx.RET`＝`RET`） |
| `GSUM(k, x)`、`GCOUNT`、`GRANK`、`GMAX`、`GMIN`、`GSTD` 的第一個引數不是「沒帶單位的整數」時 | 群鍵版 `GSUM_BY(k, x)`…`GMIN_BY(k, x)`、`GSTD_BY(k, x)`（第 4.6 節） |
| `ROLL_SUM(N, x)` | `WSUM(N, x)` |
| `LE(a, b)`、`GE(a, b)`、`EQ(a, b)`、`NE(a, b)` | `a <= b`、`a >= b`、`a == b`、`a != b` |
| `AND(a, b, …)` | `a && b && …` |
| `IF(c, a, b)`、`IIF(c, a, b)`、`WHERE(c, a, b)` | `c ? a : b` |
| `MIN(a, b, …)`、`MAX(a, b, …)` | `MIN2`／`MAX2` 連著套：逐格取小／取大。N 天裡的最小最大是 `ROLL_MIN(N, x)`／`ROLL_MAX(N, x)`，同一天全市場的是 `GMIN(0, x)`／`GMAX(0, x)` |
| `GPCT(g, x)`、`apx.GPCT(g, x)` | 群內百分位（第 5.2 節） |

條件（BOOL）放進視窗、或跟沒寫單位的 0／1 比，照下表展開成同一棵樹：

| 寫法 | 等於 | 意思 |
|---|---|---|
| `ROLL_MAX(N, c)` | `WSUM(N, c) > 0` | N 天裡有一天成立 |
| `ROLL_MIN(N, c)` | `!(WSUM(N, c) < N)` | N 天每天成立 |
| `ROLL_MEAN(N, c)` | `RATIO(WSUM(N, c), N:COUNT)` | N 天裡成立的比例 |
| `c > 0`、`c == 1`、`c != 0`、`c >= 1` | `c` | |
| `c == 0`、`c < 1`、`c != 1`、`c <= 0` | `!c` | |
| `FILL(c, 0)` | `FILL(c, 0:BOOL)` | 沒寫單位的補值跟另一邊的單位（條件就是 BOOL） |

- 視窗內有一天無效 → 無效（同 `WSUM`）。
- 恆真恆假的比法（`c >= 0`、`c > 0.5`）不收，回單位不合。
- 這一節的寫法印出來都是正名（2.6）：`ROLL_SUM(20, x)` 印成 `WSUM(20, x)`。

### 2.6 印出來能讀回去

列表、報告、搜尋紀錄印的算式是正規文字，原樣貼回 `formulas` 會編出同一棵樹：

| 印法 | 例 |
|---|---|
| 一律函數寫法、單一空白 | `GT(RATIO(close, LAG(20, close)), 1.1:RATIO)` |
| 常數一律帶單位 | `1.1:RATIO`、`20:DAYS` |
| 兩個參數拆開印 | `RET_SKIP(252, 21, close)` |
| 集合 | `(MONTH() in {3, 5, 8, 11})` |
| 群鍵版 | `GSUM_BY(PARENT_ID(1), volume)` |
| 讀成整數元的金額欄 | `fund.mktcap:AMOUNT` |
| 衍生欄（第 6 章）印成整個呼叫 | `CORR_GROUP(252, 126, 1, 3000, 1, RATIO(close, LAG(1, close)))` |

### 2.7 寫錯時的回應：錯在哪＋可照貼的改法

錯誤訊息寫成「錯在哪；改法：<可照貼的 VQL>」。MCP 的診斷拆成 `msg`（錯在哪）、`hint`（改成什麼）、`write`（那段 VQL，英文介面一樣）。

別家語言的名字不收成別名（各家的引數順序與口徑不同，照收會變成「會過但意思不同」）；回錯並附照這條的引數寫好的寫法，視窗與輸入的順序會對調：

| 寫了 | 改法 | 例 |
|---|---|---|
| `MA`、`SMA`、`AVG`、`MEAN`、`ROLL_AVG`、`TS_MEAN` | `ROLL_MEAN(N, x)` | `MA(close, 60)` → `ROLL_MEAN(60, close)` |
| `SUM`、`MOVING_SUM`、`TS_SUM` | `WSUM(N, x)` | `SUM(volume, 5)` → `WSUM(5, volume)` |
| `STD`、`STDEV`、`ROLL_STDEV`、`TS_STD` | `ROLL_STD(N, x)` | |
| `TS_MAX`、`TS_MIN`、`TS_RANK` | `ROLL_MAX`、`ROLL_MIN`、`ROLL_RANK` | |
| `PCT_CHANGE`、`ROC` | `RET(N, x)` | |
| `SHIFT`、`REF`、`DELAY` | `LAG(N, x)` | `SHIFT(close, 5)` → `LAG(5, close)` |
| `RANK`、`PCT_RANK`、`CS_RANK` | `CS_PCT(x)` | |
| 小寫的名字（篩選那一套：`roll_mean(close, 20)`、`lag(close, 5)`、`min(a, b)`） | 大寫、參數在前 | `roll_mean(close, 20)` → `ROLL_MEAN(20, close)` |

其他附改法的錯：

| 錯 | 改法 |
|---|---|
| `RET`、`DIST_HIGH`、`DIST_LOW`、`VOL_RATIO`、`FN_YOY` 的 N 超過 252（≤ 1,260） | 同一個量的長視窗寫法：`RET(300, close)` → `SUB(RATIO(close, LAG(300, close)), 1)`；`DIST_HIGH(500, close)` → `SUB(RATIO(close, ROLL_MAX(500, close)), 1)`；`VOL_RATIO(300, vol)` → `RATIO(vol, LAG(1, ROLL_MEAN(300, vol)))`；`FN_YOY(504, rev)` → `LOGR(rev, LAG(504, rev))` |
| 群內名次 `GRANK(g, x)` 跟比例加減或比大小 | `GPCT(g, x)` |
| `EV_DAYS_SINCE`、`EV_POST_N` 的輸入給了數值欄 | `EV_CHANGE(欄)`（事件＝這一欄的值變動的那天） |
| `FILL(c, 0:RATIO)`（c 是條件） | `FILL(c, 0:BOOL)` |
| BOOL 常數不是 0／1（`2:BOOL`） | `1:BOOL`（要數量寫 `2:COUNT`） |
| `LE`、`AND`、`IF`、`MIN`、`MAX`、`GPCT` 引數個數不對 | 訊息說要幾個，附寫法 |
| `user.*` 自訂函數還沒安裝 | 提示附安裝的整個定義檔形狀；只用一次的話把算式直接寫進公式 |
| 自己用 `CORR`＋`MASK(symbol == …)` 拼相關性，範圍是空的 | `CORR_SEED({"2330", "2454"}, RATIO(close, LAG(1, close))) >= 0.5`（第 6 章） |
| 檢驗參數超出範圍或寫錯鍵 | 提示附可照貼的 `check_params={…}`（夾到邊界的值、預設、每個參數的範圍） |
| `portfolio` 的鍵寫成別的系統的名字（`stop_loss`、`take_profit`、`top_n`、`rebalance`、`weight_cap`…） | 說這件事在這裡的參數與例子（例 `"exit_below": "pos.entry * 0.9"`）；打錯字給最接近的鍵 |

條件跟數字比大小、次數加比例這類照貼會過但意思多半不對的，不給改法，只說單位不合。

### 2.8 三個槽的詞彙

| 槽 | 詞彙 | 例 | 引數順序 |
|---|---|---|---|
| 分數公式、條件（選股範圍、`hold`、`exposure`、`weight`、出場） | 大寫原子與保留函數 | `ROLL_MEAN(20, close)`、`LAG(5, x)` | 參數在前、輸入在後 |
| 篩選（門檻 `screen_gate`、排序 `grid_by`、指標定義式；第 11 章） | 大寫函數，跟分數公式同名 | `ROLL_MEAN(63, r)`、`SKIP(252, r)`、`BY_SUM(year, r)`、`MIN(x)`（聚合） | 參數在前、輸入在後（同分數公式） |
| 檢驗定義式（第 11 章） | 大寫函數，跟分數公式同名 | `AMIN(x)`（聚合）、`MIN2(a, b)`（兩個數）、`COUNT`、`SUM`、`ALL`、`ANY`、`KEEP_BP` | — |

- 三個槽同一套名字：`LAG(k, x)` 在篩選與分數公式都是往回位移；兩個數取小寫 `MIN2(a, b)`、一組數的最小寫 `MIN(x)`（＝`AMIN(x)`）。
- 篩選與檢驗兩個槽的函數名不分大小寫，正規寫法是大寫（11.1、11.6）。篩選的小寫舊名照收：`roll_mean(r, 63)` 這種輸入在前的寫法跟 `ROLL_MEAN(63, r)` 是同一棵樹；整數常數寫在哪一邊都收。篩選舊的 `lag(x, k)`（去掉前 k 格）改名 `SKIP(k, x)`，舊算式自動改寫、身分照改寫後（跟寫 `SKIP` 同一個 `content_hash`）。寫了只有分數公式才有的名字（`WSUM`、`RET`、`CS_PCT`…），回錯說那是分數公式與條件用的名字，有對應的另給這個槽的寫法（`WSUM` → `ROLL_SUM(n, x)`）。
- 分數公式槽寫了篩選的小寫名字：回錯並給大寫、參數在前的寫法（2.7）。

---

## 3. 值與單位

### 3.1 定點

生成、回測、選股範圍的值都是 **64 位整數定點**，不用浮點。所以同一條公式每次算出來逐位元相同。

- 每個原子各往下取整（floor）一次；除法往負無窮取整。
- 某一步超出 i64、除以 0 → 那一格無效。
- 一格有兩種狀態：有效（帶值）或無效。

### 3.2 十一種單位

| 單位 | 存法 | 意思 | 例 |
|---|---|---|---|
| `PRICE` | ×1e6 | 價格（元） | `close`、`raw.limit_up` |
| `QTY` | ×1e6 | 數量（股） | `volume`、`shares`、`foreign_net` |
| `MONEY` | ×1e6 | 金額（元）；上限約 9.22 兆元 | `revenue`、`eps`（每股值也是 MONEY） |
| `RATIO` | ×1e6 | 比值、不帶單位的數 | `px.turnover_rate`、`RET(20, close)` |
| `PCT` | ×1e6 | 百分點 | |
| `INDEX` | ×1e6 | 指數點數 | `idx.taiex` |
| `DAYS` | 整數 | 交易日數 | `EV_DAYS_SINCE(…)`、`WARG(…)` |
| `COUNT` | 整數 | 次數、個數、序號 | `MONTH()`、`COUNT(…)`、`GRANK(…)` |
| `BOOL` | 整數 0／1 | 真假 | `close > open` |
| `CODE` | 整數 | 代碼（只能比相等、`in`、當群鍵） | `industry`、`symbol`、`PARENT_ID(1)` |
| `AMOUNT` | 整數（元） | 大額金額，整數元 | `fund.mktcap:AMOUNT` |

「數值單位」（下面寫 X）＝BOOL、CODE 以外的九種。

BOOL 常數只收 0、1（BOOL 只存成立／不成立）：`2:BOOL`、`FILL(c, 2)`（c 是條件）拒收，改法 `1:BOOL`；要數量寫 `2:COUNT`。

**AMOUNT 是什麼**：MONEY 的 1e6 定點只放得下約 ±9.22 兆元。以元計的總額欄（市值、總資產）超過時用 AMOUNT（整數元）。

- 欄一律標 MONEY；AMOUNT 只是公式裡的單位。
- 欄裡有任何一格放不進 MONEY → 沒寫單位的欄名自動讀成 AMOUNT；也可以明寫 `欄名:AMOUNT`。
- 兩個輸入一邊 MONEY、一邊 AMOUNT 時，MONEY 那邊自動換成它的 AMOUNT 版（整數元的總額欄、常數）。每股值（`eps`）不能跟 AMOUNT 混用。

### 3.3 單位代數：哪些能加、能乘、能比

| 運算 | 收的輸入 → 輸出 | 擋住的例子 |
|---|---|---|
| `ADD`、`SUB`、`MAX2`、`MIN2` | (X, X) 同單位 → X；每股金額（MONEY）與 PRICE 互相加減、取大取小 → 跟第一個輸入 | `close + volume`（PRICE ＋ QTY） |
| `GT`、`LT`（與 `>= <= == !=`） | (X, X) 同單位 → BOOL；每股金額（MONEY）與 PRICE 互比 → BOOL | `close > volume` |
| `MUL` | (RATIO 或 BOOL, X) → X；(X, RATIO 或 BOOL) → X；(BOOL, BOOL) → BOOL；(COUNT 或 DAYS, X) → X；(X, COUNT 或 DAYS) → X | `close * volume`（PRICE × QTY；成交金額直接用欄 `px.turnover`） |
| `RATIO`（`/`）、`MULDIV` | (X, X) 同單位 → RATIO；(X, RATIO) → X；(X, COUNT 或 DAYS) → X；(MONEY, QTY) → MONEY（每股金額，＝`FN_PS` 但股數 ≤ 0 不擋）；(MONEY, PRICE)、(PRICE, MONEY) → RATIO | `revenue > close`（公司總額跟價差一個股數：每股值寫 `FN_PS(revenue, shares)`，成交均價寫 `VWAP(N)`） |
| `LOGR` | (X, X) 同單位 → RATIO | |
| `LOG`、百分位、Z 分數、報酬類 | X → RATIO | |
| `SQRT` | RATIO → RATIO；PRICE／QTY／PCT／MONEY／INDEX → RATIO；COUNT／DAYS／AMOUNT 也收（文字層展開）→ RATIO | |
| `FLOOR`、`ROUND`、`CEIL` | 1e6 定點的 X → X；整數單位就是自己 | |
| `EXP`、`SIN`、`COS`、`ATAN` | RATIO → RATIO | `EXP(close)` |
| `NEG`、`ABS`、`DIFF`、視窗最大最小平均 | X → X | |
| `LAG` | 任何非 CODE（含 BOOL）→ 同單位 | |
| `NOT` | BOOL → BOOL | |
| `OR` | (BOOL, BOOL) → BOOL | |
| `FILL` | (X, X) 同單位（含 BOOL，不含 CODE）→ 同單位 | |
| `MASK` | (BOOL, 任何非 CODE) → 第二個輸入的單位 | |
| `VALID`、`EV_CHANGE`、`in {…}` | 任何（含 CODE）→ BOOL | |

兩邊都是整數單位相乘時：COUNT × COUNT → COUNT、COUNT × DAYS → DAYS、DAYS × DAYS → DAYS。同單位相除一律是 RATIO（COUNT ÷ COUNT 是比值）。

不同單位要相比，先換成不帶單位的數：`RATIO(close, LAG(20, close))`、`CS_PCT(x)`、`LOG(x)`。

各算子自己的單位規則列在第 4 章每一列。

### 3.4 無效值

無效值不參與任何運算：**任一輸入無效 → 輸出無效**。會產生無效的情況：

| 來源 | 例 |
|---|---|
| 面板那一格沒有值 | 上市前、停牌日的 `close`；還沒公布的財報 |
| 視窗不滿、暖機期 | `ROLL_MEAN(20, close)` 的前 19 天 |
| 視窗內有任何一格無效 | 停牌那天會讓跨到它的視窗無效 |
| 除以 0、對數的輸入 ≤ 0、根號的輸入 < 0 | `RATIO(a, 0)`、`LOG(0)` |
| 某一步超出 i64 | `GSUM(0, equity)` 全市場的和放不進 i64；均值類不在此列（見下） |
| 跨檔運算的非成員 | 不在選股範圍的股票的 `CS_PCT(x)` |

均值類（`ROLL_MEAN`、`ROLL_STD`、`XL_PARENT`、`GMEAN_BY`、`GSTD`、`GSTD_BY`、`CS_REL_PARENT`、`CS_Z`、`CS_PEER_Z`、`CS_WINZ`）內部的和用 128 位精確算再除，均值一定在 i64 內，所以不會因為和超出 i64 而無效；`WSUM`、`GSUM` 輸出的是和本身，和放不進 i64 照樣無效。加權均值 `GWMEAN_BY`、`GWMEAN`（與 `CS_NEUTRAL_IND`）先把權重與 x 右移到兩個和放得進 i64 再除，成員一定有值。

顯式處理無效值的四個原子：

| 原子 | 行為 | 例 |
|---|---|---|
| `VALID(x)` | x 有效 → 1，否則 0；永遠有效 | `WSUM(250, VALID(close)) >= 250` |
| `FILL(x, c)` | x 有效用 x，否則用 c | `FILL(foreign_net, 0:QTY)` |
| `MASK(c, x)` | c 有效且為真 → x，否則無效 | `MASK(VOL_RATIO(20, volume) >= 3, RET(1, close))` |
| `EV_CHANGE(x)` | 值變那天 1、否則 0；永遠有效 | `EV_CHANGE(revenue_m)` |

無效值在各處的下場：

| 位置 | 無效的格 |
|---|---|
| 分數公式 | 那一檔那一天不進排名 |
| 選股範圍（VQL 寫法） | 不在範圍內（含暖機期） |
| `hold` | 不買、不持有 |
| `weight` | 不買 |
| `rebalance_when` | 當天沒有任何一檔有效 → 當不換股 |
| `exposure` | 當天沒有任何一檔有效 → 沿用前一天；最前面沒有前一天時當滿倉 |

`MASK` 與相乘的差別：`MUL(條件, 分數)` 在條件為假時給 0，0 還是會進排名；`MASK(條件, 分數)` 在條件為假時無效，不進排名。篩選用 `MASK`。

`!(條件)` 與 `!FILL(條件, 0:BOOL)` 的差別：條件無效時前者無效（被排除），後者為真（留下）。

---

## 4. 全部算子參考表

### 4.1 怎麼讀這些表

| 欄 | 內容 |
|---|---|
| 簽名 | 參數（整數）在前、輸入在後。`N`＝視窗或天數、`L`＝層級、`x a b`＝輸入 |
| 層 | 原＝原子（最小、引擎直接實作）；保＝保留函數（由原子組成；每一個的原子定義式在第 5 章）；資＝資料（外部取得的東西本身，不是計算） |
| 參數 | 參數的合法範圍（程式常數） |
| 單位 | 輸入 → 輸出。X＝任一數值單位；「同」＝輸出跟輸入同單位 |
| 無效與暖機 | 除了「任一輸入無效 → 無效」之外的規則；暖機＝這個節點讓前幾天無效 |
| 例子 | 一條完整、編得過的分數公式 |

共通規則：

- 任一輸入無效 → 輸出無效。視窗類算子：視窗內任何一格無效 → 無效（另有規則的在該列寫明）。
- 視窗 N 含當天：`[t−N+1, t]`。`LAG(N, x)` 是 N 個交易日前。
- 暖機相加：外層的暖機＝子節點暖機＋自己的。
- 報酬類（`RVOL`、`SKEW`、`KURT`、`ACORR`、`BETA`、`RESID_MOM`）內部用對數報酬 `ln(x[t]/x[t−1])`，價 ≤ 0 → 無效。成交量、分數這類任何序列本身的偏態、峰度、自相關用 `WSKEW`、`WKURT`、`WACORR`（不取報酬、不必大於 0）。
- 層級 L：0＝全部（選股範圍內）、1＝產業、2＝市場、3＝國家、4＝全球。
- 跨檔與分群的成員＝當天在選股範圍內、而且輸入有效的股票；非成員的輸出無效。
- 輸出只有少數幾個值的節點（桶號、旗標、群的和與均值——同群每檔同值）當整條分數會大量並列；它們是零件，拿去當條件、群鍵，或跟逐檔的值相除、相減。

### 4.2 葉、常數

| 簽名 | 層 | 參數 | 單位 | 無效與暖機 | 意思 | 例子 |
|---|---|---|---|---|---|---|
| 欄鍵（`LEAF`） | 資 | — | 欄的單位 | 那一格沒值 → 無效 | 面板中該欄當天的值 | `NEG(close)` |
| 數字（`CONST`） | 原 | 定點後放得進 32 位 | 宣告的單位 | 永遠有效 | 常數 | `MASK(close > 10, RET(20, close))` |
| 大數字（`CONSTE`） | 原 | 7 位有效數字 × 10^0～12 | 宣告的單位 | 永遠有效 | 放不進 32 位的常數；直接寫數字，自動選用 | `MASK(fund.mktcap > 5e9, RET(20, close))` |

`LEAF`、`CONST`、`CONSTE` 這三個名字不能當函數寫，直接寫欄名或數字。

### 4.3 逐格算術

| 簽名 | 層 | 參數 | 單位 | 無效與暖機 | 意思 | 例子 |
|---|---|---|---|---|---|---|
| `ADD(a, b)` | 原 | — | (X, X) 同單位 → 同 | 溢位 → 無效 | a + b | `ADD(CS_PCT(RET(20, close)), CS_PCT(RET(60, close)))` |
| `SUB(a, b)` | 原 | — | (X, X) 同單位 → 同 | 溢位 → 無效 | a − b | `SUB(RET(60, close), RET(60, idx.taiex_tr))` |
| `MUL(a, b)` | 原 | — | 見 3.3 | 積放不進 i64 → 無效 | a × b；有尺度時 ÷1e6 再取整 | `MUL(RET(20, close), VOL_RATIO(20, volume))` |
| `RATIO(a, b)`（`DIV`） | 原 | — | 見 3.3 | b = 0 → 無效 | a ÷ b，往下取整 | `RATIO(low, close)` |
| `MULDIV(K, a, b)` | 原 | K 1..1,000,000,000 | 同 `RATIO(a, b)` | b = 0 或結果放不進 i64 → 無效 | K × a ÷ b，乘積 128 位、只取整一次；K × a 再大也不會溢位（`RATIO(MUL(K:COUNT, a), b)` 在 K × a 放不進 i64 時無效，其餘格兩者相同） | `MULDIV(4, net_income, equity)` |
| `NEG(a)` | 原 | — | X → 同 | i64 最小值 → 無效 | −a | `NEG(RVOL(60, close))` |
| `ABS(a)` | 原 | — | X → 同 | | 絕對值 | `ABS(SUB(CS_PEER_RANK(1, close), CS_PEER_RANK(1, fund.mktcap)))` |
| `SQRT(a)` | 原 | — | RATIO → RATIO；PRICE／QTY／PCT／MONEY／INDEX → RATIO；COUNT／DAYS／AMOUNT → RATIO（文字層展開） | a < 0 → 無效 | ⌊√a⌋（定點）；輸出是不帶單位的數 | `SQRT(fund.mktcap)` |
| `LOG(a)` | 原 | — | X → RATIO | a ≤ 0 → 無效 | 自然對數 | `LOG(MUL(MUL(RET(20, close), RET(20, close)), VOL_RATIO(20, volume)))` |
| `EXP(a)` | 原 | — | RATIO → RATIO | 超出 i64 → 無效 | e 的 a 次方 | `EXP(NEG(RVOL(60, close)))` |
| `LOGR(a, b)` | 原 | — | (X, X) 同單位 → RATIO | 任一 ≤ 0 → 無效 | ln(a ÷ b)，只取整一次（`LOG(RATIO(a, b))` 取整兩次，會差最低位） | `LOGR(close, LAG(1, close))` |
| `FLOOR(a)` | 原 | — | 1e6 定點的 X → 同；整數單位就是自己 | 結果放不進 i64 → 無效 | 取到整數、往負無窮（12.7 → 12、−12.3 → −13） | `MASK(FLOOR(raw.close % 10) == 8, RET(20, close))` |
| `MAX2(a, b)` | 保 | — | (X, X) 同單位 → 同（不收 BOOL） | | 逐格較大者 | `MAX2(DIFF(1, close), 0:PRICE)` |
| `MIN2(a, b)` | 保 | — | (X, X) 同單位 → 同（不收 BOOL） | | 逐格較小者 | `MIN2(RET(20, close), RET(60, close))` |
| `SIN(a)` | 原 | — | RATIO → RATIO | | sin a（弧度） | `SIN(RET(20, close))` |
| `COS(a)` | 原 | — | RATIO → RATIO | | cos a | `COS(RET(20, close))` |
| `ATAN(a)` | 原 | — | RATIO → RATIO | | arctan a，值域 (−π/2, π/2) | `ATAN(RET(20, close))` |

取整的三個保留函數 `MOD`、`ROUND`、`CEIL` 在 5.2。

### 4.4 比較、邏輯、無效值

| 簽名 | 層 | 參數 | 單位 | 無效與暖機 | 意思 | 例子 |
|---|---|---|---|---|---|---|
| `GT(a, b)` | 原 | — | (X, X) 同單位 → BOOL | | a > b → 1，否則 0 | `MASK(GT(close, open), RET(5, close))` |
| `LT(a, b)` | 原 | — | (X, X) 同單位 → BOOL | | a < b → 1，否則 0 | `MASK(LT(close, ROLL_MEAN(20, close)), NEG(RET(5, close)))` |
| `NOT(a)` | 原 | — | BOOL → BOOL | | 非 | `MASK(NOT(close < ROLL_MEAN(60, close)), RET(20, close))` |
| `OR(a, b)` | 保 | — | (BOOL, BOOL) → BOOL | 任一無效 → 無效（沒有短路） | 或。「且」是 `MUL(a, b)` | `MASK(OR(RET(5, close) > 0.1, VOL_RATIO(20, volume) >= 3), RET(5, close))` |
| `VALID(x)` | 原 | — | 任何（含 CODE）→ BOOL | 永遠有效 | x 有效 → 1，否則 0 | `MASK(WSUM(250, VALID(close)) >= 250, RET(60, close))` |
| `FILL(x, c)` | 原 | — | (X, X) 同單位（含 BOOL，不含 CODE）→ 同 | x、c 都無效才無效 | x 有效用 x，否則用 c | `ADD(FILL(foreign_net, 0:QTY), FILL(trust_net, 0:QTY))` |
| `MASK(c, x)` | 原 | — | (BOOL, 任何非 CODE) → x 的單位 | c 無效或為假 → 無效 | c 為真才有 x | `MASK(VOL_RATIO(20, volume) >= 3, RET(1, close))` |
| `x in {v, …}`（`IN_SET`） | 原 | — | 任何（含 CODE）→ BOOL | x 無效 → 無效 | x 等於集合裡任一個值 | `MONTH() in {3, 5, 8, 11} ? CS_PCT(FN_EGROW(252, eps)) : CS_PCT(RET(60, close))` |

`IN_SET` 這個名字不能當函數寫，一律寫 `x in {…}`。

### 4.5 時序視窗

| 簽名 | 層 | 參數 | 單位 | 無效與暖機 | 意思 | 例子 |
|---|---|---|---|---|---|---|
| `LAG(N, x)` | 原 | N 1..1,260 | 任何非 CODE → 同 | 暖機 N | N 日前的值 | `RATIO(close, LAG(20, close))` |
| `DIFF(N, x)` | 保 | N 1..1,260 | X → 同 | 暖機 N | x[t] − x[t−N] | `DIFF(60, foreign_ratio)` |
| `WSUM(N, x)` | 原 | N 1..1,260 | X → 同；BOOL → COUNT | 和放不進 i64 → 無效；暖機 N−1 | 近 N 日的和 | `WSUM(5, foreign_net)` |
| `ROLL_MEAN(N, x)` | 原 | N 2..1,260 | X → 同 | 暖機 N−1 | 近 N 日平均（往下取整） | `RATIO(close, ROLL_MEAN(60, close))` |
| `ROLL_MAX(N, x)`（`WMAX`） | 原 | N 2..1,260 | X → 同 | 暖機 N−1 | 近 N 日最大值 | `RATIO(close, ROLL_MAX(250, close))` |
| `ROLL_MIN(N, x)`（`WMIN`） | 原 | N 2..1,260 | X → 同 | 暖機 N−1 | 近 N 日最小值 | `RATIO(close, ROLL_MIN(250, close))` |
| `ROLL_STD(N, x)` | 原 | N 2..1,260 | X → 同 | 暖機 N−1 | 近 N 日母體標準差 | `ROLL_STD(60, CS_PCT(fund.mktcap))` |
| `ROLL_RANK(N, x)` | 保 | N 2..1,260 | X → RATIO 0..1 | 暖機 N−1 | x[t] 在近 N 日裡的百分位 | `ROLL_RANK(60, open)` |
| `COUNT(N, b)` | 保 | N 1..1,260 | BOOL → COUNT | 暖機 N−1 | 近 N 日為真的天數 | `MASK(COUNT(3, volume > LAG(1, volume) && close > LAG(1, close)) >= 3, RET(3, close))` |
| `FIRST_IN_WINDOW(N, b)` | 保 | N 2..1,260 | BOOL → BOOL | 暖機 N−1 | 今天為真、而且前 N−1 天都為假 | `MASK(FIRST_IN_WINDOW(11, trust_net > 0), RATIO(trust_net, volume))` |
| `WRANK(N, x)` | 原 | N 2..1,260 | X → COUNT | 暖機 N−1 | x[t] 在近 N 日裡的中位名次 2·比它小的個數＋同值個數−1 | `WRANK(60, close)` |
| `WARG(N, side, x)` | 原 | N 2..252；side 0 最高／1 最低 | X → DAYS | 暖機 N−1 | 近 N 日最高（或最低）那天距今幾天；同值取最近 | `NEG(WARG(60, 0, close))` |
| `WQUANT(N, q, x)` | 原 | N 2..252；q 0..100 | X → 同 | 暖機 N−1 | 近 N 日由小到大排序後第 ⌊q·(N−1)/100⌋ 個（q = 50 是下中位數） | `RATIO(close, WQUANT(60, 50, close))` |
| `WMAD(N, x, c)` | 原 | N 2..252 | (X, X) 同單位 → 同 | 暖機 N−1 | 近 N 日 x 對今天的 c 的絕對差之和 | `RATIO(WMAD(20, close, ROLL_MEAN(20, close)), close)` |
| `WDROP(N, x)` | 原 | N 2..1,260 | X → 同 | 相減溢位 → 無效；暖機 N−1 | 近 N 日內先高後低的最大落差（≥ 0） | `NEG(WDROP(250, LOG(close)))` |
| `EMA(N, x)` | 原 | N 2..84 | X → 同 | 暖機 3N−1 | 指數平均 α = 2/(N+1)，每格從 3N 格前起算 | `RATIO(close, EMA(20, close))` |
| `RET(N, x)` | 保 | N 1..1,260 | X → RATIO | x[t−N] ≤ 0 → 無效；暖機 N | x[t] ÷ x[t−N] − 1 | `RET(60, close)` |
| `RET_SKIP(N, M, x)` | 保 | N 2..1,260；1 ≤ M < N | X → RATIO | x[t−N] ≤ 0 → 無效；暖機 N | x[t−M] ÷ x[t−N] − 1（跳過最近 M 日） | `CS_PCT(RET_SKIP(252, 21, close))` |
| `RVOL(N, x)` | 保 | N 2..252 | X → RATIO | 暖機 N | N 個對數報酬的母體標準差 | `NEG(RVOL(60, close))` |
| `PARK(N, a, b)` | 原 | N 2..252 | (X, X) 同單位 → RATIO | 暖機 N−1 | Parkinson 波動：a＝最高價、b＝最低價 | `NEG(PARK(20, high, low))` |
| `GK(N, a, b)` | 原 | N 2..252 | (RATIO, RATIO) → RATIO | 暖機 N−1 | Garman-Klass 波動：a＝高÷低、b＝收÷開 | `NEG(GK(20, RATIO(high, low), RATIO(close, open)))` |
| `SKEW(N, x)` | 保 | N 3..252 | X → RATIO | 暖機 N | N 個對數報酬的偏態（＝`WSKEW(N, LOGR(x, LAG(1, x)))`） | `SKEW(60, close)` |
| `KURT(N, x)` | 保 | N 4..252 | X → RATIO | 暖機 N | N 個對數報酬的超額峰度（＝`WKURT(N, LOGR(x, LAG(1, x)))`） | `NEG(KURT(60, close))` |
| `ACORR(N, k, x)` | 保 | N 3..252；1 ≤ k ≤ N−2 | X → RATIO | 暖機 N | N 個對數報酬的落後 k 期自相關（＝`WACORR(N, k, LOGR(x, LAG(1, x)))`） | `ACORR(60, 1, close)` |
| `WSKEW(N, x)` | 原 | N 3..252 | X → RATIO | 視窗內任一格無效、x 沒有變化 → 無效；暖機 N−1 | 近 N 日 x 本身的偏態（母體動差；任何序列，不必大於 0） | `WSKEW(60, volume)` |
| `WKURT(N, x)` | 原 | N 4..252 | X → RATIO | 同 `WSKEW` | 近 N 日 x 本身的超額峰度 | `NEG(WKURT(60, RET(1, close)))` |
| `WACORR(N, k, x)` | 原 | N 3..252；1 ≤ k ≤ N−2 | X → RATIO | 同 `WSKEW` | 近 N 日 x 本身的落後 k 期自相關（樣本 ACF） | `WACORR(60, 1, volume)` |
| `SLOPE(N, x)` | 原 | N 2..252 | X → 同 | 暖機 N−1 | 近 N 日對時間的線性回歸斜率（每日） | `RATIO(SLOPE(60, close), close)` |
| `R2(N, x)` | 原 | N 3..252 | X → RATIO 0..1 | 暖機 N−1 | 同一條回歸的 R² | `R2(60, close)` |
| `WREG(N, mode, y, x)` | 原 | N 3..1,260；mode 0 斜率、1 截距、2 今天的殘差、3 R²、4 斜率 t 值 | (X, X) 同單位 → mode 0、3、4 是 RATIO，1、2 同 y | 暖機 N−1；視窗內任一格無效、x 沒有變化 → 無效 | y 對 x 近 N 日的一元迴歸（最小平方；精確和，只取整一次）；寫法用名字 `REG_SLOPE`…`REG_T`（5.2） | `REG_SLOPE(120, RET(1, close), RET(1, idx.taiex))` |
| `INV_VOL_KEY(N, x)` | 原 | N 2..1,260 | RATIO → COUNT（16,384..1,073,741,824） | 暖機 N−1；視窗內有效的格不到一半 → 無效 | x 近 N 日波動的倒數，當權重鍵（越穩越大）：視窗內有效的 k 格平方和 Q，vol ＝ ⌊√⌊Q ÷ k⌋ ÷ 10⌋（10 萬分之一，夾在 1..65,535），值 ＝ ⌊2³⁰ ÷ vol⌋。`weighting="inv_vol"`（`apx.INV_VOL_KEY()`）的波動就是 `INV_VOL_KEY(20, RET(1, close))` | `apx.SHARE(apx.PICK(10, false), INV_VOL_KEY(60, RET(1, close)))` |
| `MAXDD(N, x)` | 原 | N 2..252 | X → RATIO | 暖機 N−1 | 近 N 日最大回撤（正數；舊定義） | `NEG(MAXDD(250, close))` |
| `DIST_HIGH(N, x)` | 保 | N 2..1,260 | X → RATIO（≤ 0） | 暖機 N−1 | x[t] ÷ 近 N 日最高 − 1 | `DIST_HIGH(250, close)` |
| `DIST_LOW(N, x)` | 保 | N 2..1,260 | X → RATIO（≥ 0） | 暖機 N−1 | x[t] ÷ 近 N 日最低 − 1 | `DIST_LOW(250, close)` |
| `UP_STREAK(N, x)` | 保 | N 1..252 | X → COUNT | 暖機 N | 到今天為止連續上漲的天數，封頂 N | `UP_STREAK(20, close)` |
| `DOWN_STREAK(N, x)` | 保 | N 1..252 | X → COUNT | 暖機 N | 連續下跌的天數，封頂 N | `NEG(DOWN_STREAK(20, close))` |
| `CORR(N, a, b)` | 原 | N 3..252 | (X, Y) 任意數值 → RATIO | 暖機 N−1 | 近 N 日 a 與 b 的 Pearson 相關 | `NEG(CORR(60, close, volume))` |
| `VOL_RATIO(N, x)` | 保 | N 1..252 | X → RATIO | 暖機 N | x[t] ÷ 前 N 日（不含今天）的平均 | `VOL_RATIO(20, volume)` |
| `AMIHUD(N, a, b)` | 原 | N 1..252 | (X, MONEY) → RATIO | 暖機 N | 近 N 日 \|a 的日報酬\| × 1e9 ÷ 成交金額 b 的平均（越大越不流動） | `AMIHUD(20, close, px.turnover)` |
| `BETA(N, a, b)` | 原 | N 3..252 | (X, Y) 任意數值 → RATIO | 暖機 N | a 對 b 的對數報酬的共變異 ÷ b 的變異 | `NEG(BETA(120, close, idx.taiex))` |
| `RESID_MOM(N, a, b)` | 原 | N 3..252 | (X, Y) 任意數值 → RATIO | 暖機 N | 扣掉 β × b 的報酬後，a 的 N 日累計殘差 | `RESID_MOM(120, close, idx.taiex)` |

### 4.6 遞推與重置

| 簽名 | 層 | 參數 | 單位 | 無效與暖機 | 意思 | 例子 |
|---|---|---|---|---|---|---|
| `SCANSUM(r, x)` | 原 | — | (BOOL, X) → 同 x | 第一次 r 為真之前無效；r 或 x 無效 → 當天無效、斷到下一次 r 為真 | r 為真那天從 x 重新起算，之後逐日累加 x | `SCANSUM(foreign_net <= 0, MUL(foreign_net > 0, 1:COUNT))` |
| `SCANMAX(r, x)` | 原 | — | (BOOL, X) → 同 x | 同 `SCANSUM` | 上次 r 為真以來（含當天）x 的最大值 | `RATIO(close, SCANMAX(EV_CHANGE(revenue_m) && revenue_m >= ROLL_MAX(250, revenue_m), close))` |
| `SCANMIN(r, x)` | 原 | — | (BOOL, X) → 同 x | 同 `SCANSUM` | 上次 r 為真以來（含當天）x 的最小值 | `RATIO(close, SCANMIN(EV_CHANGE(revenue_m), close))` |
| `LATCH(r, x)` | 原 | — | (BOOL, X) → 同 x | 第一次取樣前無效；取樣那天 x 無效 → 沿用到下次取樣前都無效 | r 為真那天記下 x，之後沿用 | `MASK(LAG(5, EV_CHANGE(revenue)), RATIO(close, LATCH(EV_CHANGE(revenue), LAG(1, close))))` |
| `SMOOTH(K, a, x)` | 原 | K 0..756 | (RATIO, X) → 同 x | x 無效 → 無效並重新起算；起點後 K 步內無效；暖機 K | 指數平滑 y ← y + ⌊a·(x − y)⌋，起點 y = x | `RATIO(close, SMOOTH(60, 0.1:RATIO, close))` |
| `RWIN(k, mode, x)` | 原 | k 1..12；mode 0 第 k 期前的值／1 近 k 期和／2 平均／3 最小／4 最大／5 標準差／7 與第 k 期前的差 | X → 同 | 期數不夠 → 無效 | 依公布次序的視窗：一「期」＝一次值變（季報一季、月營收一個月） | `RATIO(RWIN(4, 1, fund.div_paid), fund.mktcap)` |
| `RWIN_PCT(k, x)` | 原 | k 1..12 | X → RATIO | 第 k 期前的值 = 0 → 無效 | (本期 − 第 k 期前) ÷ \|第 k 期前\|；月營收年增 k = 12 | `RWIN_PCT(12, revenue_m)` |
| `EWIN(k, mode, b, x)` | 原 | k 1..20；mode 同 `RWIN` | (BOOL, X) → 同 x | 期數不夠、當天 x 或 b 無效 → 無效 | 依事件次序的視窗：一「期」＝ b 為真（且 x 有效）的一天；`RWIN(k, m, x)` ＝ `EWIN(k, m, EV_CHANGE(x), x)` | `EWIN(4, 2, EV_CHANGE(revenue_m), RET(1, close))`（最近 4 次月營收公布當天的平均漲跌） |

- 重置類的常見組合：連續天數 `SCANSUM(不成立, MUL(成立, 1:COUNT))`；事件以來最高 `SCANMAX(事件, x)`；事件當天的值 `LATCH(事件, x)`。
- `RWIN` 的暖機以「期」計，換不成固定天數；面板前段期數不夠的格無效。
- 近四季合計（TTM）＝`RWIN(4, 1, x)`；去年同月＝`RWIN(12, 0, revenue_m)`；上一季＝`RWIN(1, 0, x)`；比上一期的變化＝`RWIN(1, 7, x)`。
- 「期」不是值變、而是某個條件成立的日子，用 `EWIN`：最近 3 次爆量那天的收盤平均 `EWIN(3, 2, volume > 2 * ROLL_MEAN(20, volume), close)`；條件 c 最近一次成立那天的收盤是 `LATCH(c, close)`，再前一次是 `EWIN(1, 0, c, close)`。
- 例：最近 4 次季報公布當天的平均反應＝`EWIN(4, 2, EV_CHANGE(revenue), RET(1, close))`——`EV_CHANGE(revenue)` 是季報公布日（季營收換值那天），每次公布當天的 1 日報酬算一期，取最近 4 期平均；公布不滿 4 次的股票無效。改成 `EWIN(4, 3, …)` 是 4 次裡最差的一次，`EWIN(1, 0, …)` 是上一次的反應。
- 兩條序列的迴歸用下表五個名字，都寫 `(N, y, x)`：y 對 x，y、x 同單位（報酬對報酬、價對價；對時間的斜率用 `SLOPE`）。它們是保留函數（5.2），展開成同一個原子 `WREG(N, mode, y, x)`，名字決定讀哪一個輸出（mode）：

  | 名字 | mode | 輸出 | 單位 | 意思 |
  |---|---|---|---|---|
  | `REG_SLOPE` | 0 | 斜率 | RATIO | x 動 1、y 平均動多少（對大盤報酬就是敏感度） |
  | `REG_INTERCEPT` | 1 | 截距 | 同 y | x 為 0 時 y 的平均 |
  | `REG_RESID` | 2 | 今天的殘差 | 同 y | 今天的 y 減掉迴歸線在今天 x 的值（對大盤報酬就是扣掉大盤之後多出來的） |
  | `REG_R2` | 3 | R² | RATIO 0..1 | 迴歸線解釋了多少 y 的變化 |
  | `REG_T` | 4 | 斜率的 t 值 | RATIO | 斜率除以它的標準誤；絕對值大＝斜率顯著 |

### 4.7 跨檔（同一天、選股範圍內）

| 簽名 | 層 | 參數 | 單位 | 無效與暖機 | 意思 | 例子 |
|---|---|---|---|---|---|---|
| `CS_PCT(x)` | 保 | — | X → RATIO 0..1 | 成員少於 2 檔 → 無效 | 當天在選股範圍內的百分位 | `CS_PCT(RET(60, close))` |
| `CS_Z(x)` | 原 | — | X → RATIO | 標準差 0 → 無效 | (x − 範圍內均值) ÷ 範圍內標準差（＝`GZ(0, x)`） | `CS_Z(RET(60, close))` |
| `CS_WINZ(k, x)` | 保 | k 5..100 | X → RATIO | | `CS_Z` 夾在 ±k/10 個標準差 | `CS_WINZ(30, RET(60, close))` |
| `CS_QCUT(N, x)` | 保 | N 2..100 | X → COUNT 0..N−1 | | 把 `CS_PCT` 切成 N 桶，回桶號 | `GRANK(CS_QCUT(5, fund.mktcap), RET(20, close))` |
| `CS_RANK_CHG(N, x)` | 保 | N 1..250 | X → RATIO | 暖機 N | 今天的 `CS_PCT` − N 日前的 `CS_PCT` | `CS_RANK_CHG(20, RET(60, close))` |
| `CS_RANK_MEAN(N, x)` | 保 | N 2..250 | X → RATIO | 今天要有效、近 N 日有效日 ≥ ⌈N/2⌉；暖機 N−1 | 近 N 日 `CS_PCT` 的平均 | `CS_RANK_MEAN(20, RET(5, close))` |
| `CS_RANKMIX(L, flags, a, b)` | 保 | L 0..4；flags 0..3（1 取反 a、2 取反 b、3 兩邊都取反） | (X, Y) 任意數值 → RATIO 0..2 | 兩邊都要是成員；成員少於 2 檔 → 無效 | a、b 各在群 L 內排百分位再相加（等權兩因子） | `CS_RANKMIX(0, 2, FN_EP(eps, close), RVOL(60, close))` |
| `CS_REL_RET(N, L, x)` | 保 | N 1..252；L 0..4 | X → RATIO | 暖機 N | N 日報酬 − 同群報酬均值 | `CS_REL_RET(20, 1, close)` |
| `CS_NEUTRAL_IND(L, x, w)` | 保 | L 1..4 | (X, 任意數值) → 同 x | w > 0 才是成員；x − 均值超出 i64 → 無效 | x − 同群成員的 w 加權均值＝`SUB(x, GWMEAN(L, w, x))`，逐位元相同（第 5.1 節） | `CS_NEUTRAL_IND(1, RET(20, close), fund.mktcap)` |
| `CS_NEUTRAL_SIZE(x, z)` | 保 | — | (X, 任意數值) → 同 x | | x 對 [1, z] 在範圍內的回歸殘差（＝`GRESID(0, x, z)`，全市場一群） | `CS_NEUTRAL_SIZE(RET(20, close), LOG(fund.mktcap))` |
| `CS_NEUTRAL_COUNTRY(x, w)` | 保 | — | (X, 任意數值) → 同 x | | 同 `CS_NEUTRAL_IND`，群固定是國家（多市場面板用） | `CS_NEUTRAL_COUNTRY(RET(20, close), fund.mktcap)` |
| `CS_NEUTRAL_MODEL(p, x, q)` | 原 | p＝層級 1..4 ＋ K × 256，K ≤ 16 | (X, 任意數值) → 同 x | | 同群去均值後，再減對 K 個曝險欄的投影；q 要是風險模型寫回的欄 | — |

`CS_NEUTRAL_MODEL` 要面板接了風險模型才有那幾欄，正式面板目前沒有，所以沒有例子。

### 4.8 分群

群有三種給法，同一套成員規則：

| 給法 | 寫法 | 群怎麼定 |
|---|---|---|
| 層級 | `GSUM(L, x)`：第一個引數是**沒帶單位的整數** | L 0 全部、1 產業、2 市場、3 國家、4 全球；沒有第 L 層歸屬的股票無效 |
| 群鍵 | `GSUM(k, x)`：第一個引數是其他任何東西（欄、算式、帶單位的常數 `0:COUNT`）；印出來是 `GSUM_BY(k, x)` | 同一天群鍵相同的股票一群；群鍵單位 CODE／COUNT／BOOL／DAYS |
| 全面板成員 | `GSUM_ALL(L, x)` | 群照層級 L，但成員**不看選股範圍** |

| 簽名 | 層 | 參數 | 單位 | 無效與暖機 | 意思 | 例子 |
|---|---|---|---|---|---|---|
| `GSUM(L, x)` | 原 | L 0..4 | X → 同；BOOL → COUNT | 和放不進 i64 → 整群無效 | 同群成員的和，每個成員都拿到 | `RATIO(volume, GSUM(1, volume))` |
| `GCOUNT(L, x)` | 原 | L 0..4 | 任何（含 BOOL、CODE）→ COUNT | | 同群成員數 | `RATIO(GRANK(1, RET(20, close)), GCOUNT(1, RET(20, close)))` |
| `GRANK(L, x)` | 原 | L 0..4 | X → COUNT | 成員少於 2 檔 → 無效 | 同群裡的中位名次 2·比它小的個數＋同值個數−1 | `GRANK(1, RET(20, close))` |
| `GMAX(L, x)` | 原 | L 0..4 | X → 同（不收 BOOL） | 成員 1 檔也算 | 同群最大值 | `RATIO(RET(20, close), GMAX(1, RET(20, close)))` |
| `GMIN(L, x)` | 原 | L 0..4 | X → 同（不收 BOOL） | 成員 1 檔也算 | 同群最小值 | `MASK(close <= GMIN(1, close), NEG(close))` |
| `GSTD(L, x)` | 原 | L 0..4 | X → 同（不收 BOOL） | 成員少於 2 檔 → 無效；全群同值 → 0（有效） | 同群母體標準差：均值 ⌊Σx ÷ n⌋，(x − 均值) 先右移到平方和放得進 64 位，開根號再左移回來（`ROLL_STD` 同一套） | `RATIO(SUB(RET(20, close), XL_PARENT(1, RET(20, close))), GSTD(1, RET(20, close)))` |
| `GSUM_BY(k, x)` | 原 | — | (群鍵, X) → 同 x；(群鍵, BOOL) → COUNT | 群鍵或 x 無效 → 不進任何群 | 群鍵相同的成員的和 | `RATIO(volume, GSUM(close > open, volume))` |
| `GCOUNT_BY(k, x)` | 原 | — | (群鍵, 任何) → COUNT | | 群鍵相同的成員數 | `RATIO(GRANK(CS_QCUT(10, fund.mktcap), RET(20, close)), GCOUNT(CS_QCUT(10, fund.mktcap), RET(20, close)))` |
| `GRANK_BY(k, x)` | 原 | — | (群鍵, X) → COUNT | 成員少於 2 檔 → 無效 | 群鍵相同的成員裡的中位名次 | `GRANK(PARENT_ID(1), RET(20, close))` |
| `GMAX_BY(k, x)` | 原 | — | (群鍵, X) → 同 x | | 群鍵相同的成員裡最大的 x | `RATIO(RET(20, close), GMAX(CS_QCUT(5, fund.mktcap), RET(20, close)))` |
| `GMIN_BY(k, x)` | 原 | — | (群鍵, X) → 同 x | | 群鍵相同的成員裡最小的 x | `SUB(RET(20, close), GMIN(CS_QCUT(5, fund.mktcap), RET(20, close)))` |
| `GSTD_BY(k, x)` | 原 | — | (群鍵, X) → 同 x | 成員少於 2 檔 → 無效 | 群鍵相同的成員的母體標準差（同 `GSTD`） | `GSTD(CS_QCUT(5, fund.mktcap), RET(20, close))` |
| `GMEAN_BY(k, x)` | 原 | — | (群鍵, X) → 同 x | 成員 1 檔也算 | 群鍵相同的成員的均值（128 位和再除，和超出 i64 也有值）。k 同 `GSUM` 的寫法：沒寫單位的整數＝層級（0 全市場、1..4），其餘＝群鍵。`XL_PARENT(L, x)`、`GMEAN(L, x)` 跟 `GMEAN_BY(L, x)` 同一棵樹 | `SUB(RET(20, close), GMEAN_BY(1, RET(20, close)))` |
| `GWMEAN_BY(k, w, x)` | 原 | — | (群鍵, 任意數值, X) → 同 x | 成員要 w 有效且 > 0；成員 1 檔也算 | 群鍵相同的成員裡 x 的 w 加權均值：權重留 20 位（sw＝max(0, bitlen(max w) − 20)）、x 右移 sx＝max(0, bitlen(max|x|) + 20 + bitlen(n) − 62) 位讓兩個和放得進 i64，值＝⌊Σ(w≫sw)(x≫sx) ÷ Σ(w≫sw)⌋·2^sx，成員一定有值。k 同 `GSUM` 的寫法：沒寫單位的整數＝層級（0 全市場、1..4），其餘＝群鍵。`GWMEAN(L, w, x)`、`GWMEAN_BY(PARENT_ID(L), w, x)`、`GWMEAN_BY(L, w, x)` 同一棵樹 | `GWMEAN_BY(CS_QCUT(5, fund.mktcap), fund.mktcap, RET(20, close))` |
| `ORD(k, x)` | 原 | — | (群鍵, X) → COUNT | 非成員無效（成員同 `GRANK_BY`）；群只有一檔＝0 | 群鍵相同的成員裡 x 的**精確名次**：0 起、0＝最大，每群剛好 0..n−1。同值照固定打散（跟選股名次同一套，不靠代號先後）；k 同 `GSUM` 的寫法：沒寫單位的整數＝層級（`0` 全市場、`1` 產業…），其餘＝群鍵 | `ORD(1, x) == 0`＝各產業第一名；前 N 名用保留函數 `GTOP(N, first, x)`、`CS_TOPK(k, x)`（5.2） |
| `GQUANT(q, first, x)` | 原 | q 0..100；`first` 同 `GSUM`：沒帶單位的整數＝層級 0..4，其他＝群鍵 | (群鍵, X) → 同 x（不收 BOOL） | 成員 1 檔＝那一檔 | 同群成員的 x 由小到大第 ⌊q·(n−1)/100⌋ 個（同 `WQUANT`）；50＝中位數、0＝`GMIN`、100＝`GMAX` | `close > GQUANT(50, 1, close)` |
| `GRESID(k, x, z)` | 原 | — | (群鍵, X, 任意數值) → 同 x | 非成員無效（成員：k、x、z 都有效）；群只有一檔＝0；x − 群均值放不進 i64 → 整群無效 | 同群成員的 x 對 [1, z] 的最小平方殘差（均值的和 128 位，只取整一次）。k 同 `GSUM` 的寫法：沒寫單位的整數＝層級（`0` 全市場、`1` 產業…），其餘＝群鍵 | `GRESID(1, RET(20, close), LOG(fund.mktcap))`＝產業內去掉規模影響 |
| `PARENT_ID(L)` | 資 | L 1..4 | → CODE | 沒有第 L 層歸屬 → 無效 | 第 L 層歸屬的編號，只能當群鍵 | `RATIO(volume, GSUM(PARENT_ID(1), volume))` |
| `GSUM_ALL(L, x)` | 原 | L 0..4 | X → 同；BOOL → COUNT | 當天該群沒有成員 → 整群無效；和放不進 i64 → 整群無效 | 成員＝當天 x 有效的每一檔（不看選股範圍）；結果寫到群內每一格，含 x 無效、不在範圍、停牌的格。L = 0 時每檔同值 | `SUB(RET(20, close), RATIO(GSUM_ALL(0, RET(20, close)), GCOUNT_ALL(0, RET(20, close))))` |
| `GCOUNT_ALL(L, x)` | 原 | L 0..4 | 任何 → COUNT | 沒有成員 → 無效（不給 0） | 同 `GSUM_ALL` 的成員數 | `MASK(RATIO(GSUM_ALL(1, close > ROLL_MEAN(60, close)), GCOUNT_ALL(1, close)) > 0.5, RET(20, close))` |
| `CS_PEER_RANK(L, x)` | 保 | L 1..4 | X → RATIO 0..1 | 成員少於 2 檔 → 無效 | 同群裡的百分位 | `CS_PEER_RANK(1, RET(20, close))` |
| `CS_PEER_Z(L, x)` | 原 | L 1..4 | X → RATIO | | 同群裡的 Z 分數（＝`GZ(L, x)`） | `CS_PEER_Z(1, FN_EP(eps, close))` |
| `CS_REL_PARENT(L, x)` | 保 | L 1..4 | X → 同 | | x − 同群成員均值 | `CS_REL_PARENT(1, RET(20, close))` |
| `XL_PARENT(L, x)` | 原 | L 1..4 | X → 同 | | 同群成員均值，群裡每檔拿同一個值；＝`GMEAN_BY(PARENT_ID(L), x)`，兩種寫法同一棵樹 | `RATIO(volume, XL_PARENT(1, volume))` |
| `GWMEAN(L, w, x)` | 保 | L 1..4 | (任意數值, X) → 同 x | 成員要 w > 0 | 同群成員裡 x 的 w 加權均值（產業市值加權報酬）；＝`GWMEAN_BY(PARENT_ID(L), w, x)`，同一棵樹；第一個引數寫 0 或群鍵就是 `GWMEAN_BY` | `GWMEAN(1, fund.mktcap, RET(20, close))` |

常用組合：

| 要什麼 | 寫法 |
|---|---|
| 群內百分位 | `GPCT(k, x)`（＝`RATIO(GRANK(k, x), ADD(SUB(GCOUNT(k, x), 1:COUNT), SUB(GCOUNT(k, x), 1:COUNT)))`，第 5.2 節） |
| 群內均值 | `GMEAN(k, x)`（k 寫層級或群鍵；同 `GMEAN_BY(k, x)`） |
| 群內加權均值（例：市值加權） | `GWMEAN(k, w, x)`（k 寫層級或群鍵；同 `GWMEAN_BY(k, w, x)`） |
| 扣掉同群加權均值（產業中性化） | `CS_NEUTRAL_IND(L, x, w)`＝`SUB(x, GWMEAN(L, w, x))` |
| 群內 Z 分數 | `GZ(k, x)`（第 5.2 節；層級 0＝`CS_Z(x)`、1..4＝`CS_PEER_Z(L, x)`） |
| 比同群平均高多少 | `SUB(x, 群內均值)` |
| 群內中位數／分位數 | `GQUANT(50, k, x)`；前 25% 的門檻 `GQUANT(75, k, x)` |
| 群內去掉另一個量的影響（迴歸殘差） | `GRESID(k, x, z)`；全市場一群＝`CS_NEUTRAL_SIZE(x, z)` |
| 全部當一群 | `GSUM(0, x)` 或群鍵寫常數 `GSUM(0:COUNT, x)` |
| 群鍵舉例 | `PARENT_ID(1)`（產業）、`CS_QCUT(5, ROLL_MEAN(20, volume))`（均量五組）、`close > open`（紅黑 K）、`MONTH()`、`CORR_GROUP(…)`（第 6 章） |

- CODE（代碼欄、`PARENT_ID`）只能當群鍵，不能做數學，也不能當整條公式。
- 條件裡的產業群鍵可以直接寫 `industry`；分數公式裡寫 `PARENT_ID(1)`。
- 例：股價高於同產業中位數＝`close > GQUANT(50, 1, close)`（`1` 是產業層；寫 `PARENT_ID(1)` 一樣）。比中位數高多少（不被極端值拉動，比「減平均」穩）＝`RET(20, close) - GQUANT(50, 1, RET(20, close))`；市值十組裡各組的中位數 `GQUANT(50, CS_QCUT(10, fund.mktcap), x)`；全市場的中位數 `GQUANT(50, 0, x)`。
- 群內百分位每一群的第一名都是 1，當整條分數會大量並列；排名分數用「比同群平均高多少」。一定要用群內百分位（例 `GPCT(g, x)`、產業內百分位 `CS_PEER_RANK(1, x)`）時一起給 `portfolio.tie_by` 寫被排名的那個值（例 `"tie_by": "x"`），並列時照它分先後。

### 4.9 事件

| 簽名 | 層 | 參數 | 單位 | 無效與暖機 | 意思 | 例子 |
|---|---|---|---|---|---|---|
| `EV_CHANGE(x)` | 原 | — | 任何 → BOOL | 永遠有效；暖機 1 | 值變日：x 今天有效，而且是第一天、或昨天無效、或跟昨天不同。向前填的財報／月營收欄的公布日就是值變日 | `MASK(EV_CHANGE(revenue_m), RET(1, close))` |
| `EV_PULSE(c)` | 保 | — | 欄 → BOOL | 永遠有效；c 要直接是欄 | 該欄當天有值 → 1（事件欄：除息日） | `MASK(EV_PULSE(dps), RET(5, close))` |
| `EV_POST_N(N, b)` | 保 | N 1..250 | BOOL → BOOL | 暖機 N | N 日前 b 為真（事件後第 N 天） | `MASK(EV_POST_N(5, EV_CHANGE(revenue_m)), RET(5, close))` |
| `EV_DAYS_SINCE(N, b)` | 保 | N 1..250 | BOOL → DAYS | N 日內沒有事件 → 無效；暖機 N | 距上次 b 為真幾天（當天 0） | `NEG(EV_DAYS_SINCE(60, EV_CHANGE(revenue_m)))` |
| `EV_EARN_DRIFT(N, a, b)` | 保 | N 1..252 | (X, Y) 任意數值 → RATIO | N 日內沒有公布 → 無效；暖機 N | a＝價、b＝財報欄：b 最近一次公布（值變）的前一天到今天，a 的報酬 | `EV_EARN_DRIFT(60, close, eps)` |
| `EV_IDX_CHG(N, b)` | 保 | N 1..252 | BOOL → RATIO −1..1 | 暖機 N | b＝成員旗標：近 N 日內最後一次由假變真給正、由真變假給負，越近絕對值越大；沒變動 0 | `EV_IDX_CHG(60, close > ROLL_MEAN(200, close))` |

### 4.10 財報

| 簽名 | 層 | 參數 | 單位 | 無效與暖機 | 意思 | 例子 |
|---|---|---|---|---|---|---|
| `FN_YOY(N, x)` | 保 | N 1..1,260 | X → RATIO | 任一 ≤ 0 → 無效；暖機 N | ln(x[t] ÷ x[t−N])：對數成長（0.1 約 10%） | `FN_YOY(252, revenue_m)` |
| `FN_EGROW(N, x)` | 保 | N 1..252 | X → RATIO | 基期 0 → 無效；暖機 N | (x[t] − x[t−N]) ÷ \|x[t−N]\|：基期為負也算得出來 | `CS_PCT(FN_EGROW(252, eps))` |
| `FN_GM_CHG(N, a, b)` | 保 | N 1..252 | (MONEY, MONEY) → RATIO | b ≤ 0 → 無效；暖機 N | a＝毛利、b＝營收：毛利率 − N 日前的毛利率 | `FN_GM_CHG(252, fund.gross, revenue)` |
| `FN_DY(N, a, b)` | 保 | N 1..252 | (MONEY, PRICE) → RATIO | b ≤ 0 → 無效；a 要是只在除息日有值的每股股利欄；暖機 N−1 | 近 N 日股利和 ÷ 價＝殖利率 | `FN_DY(250, dps, raw.close)` |

每股值與總額不能混：`FN_EP`、`FN_BP`、`FN_DY` 的第一個輸入要是每股值（`eps`、`dps`、`FN_PS(…)`）；兩個金額輸入一個每股值、一個總額也會被擋。總額要比市值寫 `RATIO(總額, fund.mktcap)`。

由資料欄相除得到的財報比值 `FN_PS`、`FN_EP`、`FN_BP`、`FN_ROE`、`FN_ACCRUAL` 歸資料，在第 7.4 節。這裡留下的四個都帶視窗（N 期前、近 N 日加總），是由原子組成的計算。

### 4.11 日曆

| 簽名 | 層 | 參數 | 單位 | 無效與暖機 | 意思 | 例子 |
|---|---|---|---|---|---|---|
| `MONTH()` | 原 | — | → COUNT 1..12 | 面板沒有日曆 → 無效 | 月份 | `MASK(MONTH() == 1, NEG(RET(20, close)))` |
| `WEEKDAY()` | 原 | — | → COUNT 0..6 | 同上 | 星期，0＝週一、4＝週五 | `MASK(WEEKDAY() == 0, NEG(RET(1, close)))` |
| `MDAY()` | 原 | — | → COUNT 1..31 | 同上 | 幾號 | `MASK(MDAY() <= 10, RET(5, close))` |
| `YEAR()` | 原 | — | → COUNT | 同上 | 公曆年份 | `MASK(YEAR() >= 2020, RET(60, close))` |
| `EPOCH_DAY()` | 原 | — | → DAYS | 同上 | 1970-01-01 起第幾天（日曆日，休市日也算）；週序＝`FLOOR((EPOCH_DAY() + 3) / 7)`（週一起算） | `EPOCH_DAY() - LAG(1, EPOCH_DAY()) >= 4`（隔了長假的第一個交易日） |
| `DAY_INDEX()` | 原 | — | → DAYS | 永遠有效 | 面板第幾天（第一天 0） | `SCANSUM(DAY_INDEX() < 1, volume)` |

日曆月與上市天數的保留函數（`TDAY_OF_MONTH`、`AT_MONTH_START`、`LISTED_TDAYS`、`LISTED_DAYS`）在 5.2。

可重現的雜訊（檢驗「分數加雜訊還選得出來嗎」用）：

| 簽名 | 層 | 參數 | 單位 | 無效與暖機 | 意思 | 例子 |
|---|---|---|---|---|---|---|
| `NOISE(k)` | 原 | k 0..65535 | → RATIO −1..1 | 永遠有效 | 均勻雜訊，只由（檢驗的種子、k、日、檔）決定：同一份檢驗重跑逐位相同、換 k 就是另一組；不在檢驗裡時種子＝0 | `ADD(CS_PCT(RET(60, close)), MUL(0.05, NOISE(1)))` |

### 4.12 演算法型（62 個）

視窗內有界的演算法特徵，四族：LZ（Lempel-Ziv 複雜度，19 個）、RT（創紀錄檢定，21 個）、ENT（夏農熵，3 個）、VG（可視圖，19 個）。全部是原子。

共通規則：

| 項 | 規則 |
|---|---|
| 簽名 | `NAME(W, a)`、`NAME(W, a, b)`；讀門檻的：`NAME(W, level, a)`、`NAME(W, level, a, b)` |
| W | 視窗 2..256 |
| level | 門檻，單位百分之一（`80` ＝ 0.8、`150` ＝ 1.5）；0..32,767。不讀門檻的算子不能寫 level |
| 輸入 | 任意數值單位；兩個輸入可以不同單位 |
| 輸出 | 「定點」→ RATIO；「整數」→ COUNT；「±1」→ COUNT（−1／0／1）；「0/1」→ BOOL |
| 無效與暖機 | 視窗內全有效才有效。暖機：LZ、ENT、VG 是 W；RT 是 W−1 |

輸出是 BOOL 的不能當整條分數公式，放進 `MASK(…, 分數)` 或條件。輸出是「±1」的只有三個值，當整條分數會大量並列；拿去乘別的分數或當條件。

| 名字 | 輸入 | 門檻 | 輸出 | 意思 |
|---|---|---|---|---|
| `LZ_C_DIFF` | a | — | 定點 | 近 W 日 a 的漲／平／跌序列有多雜亂（≈1 像隨機、越低越有規律） |
| `LZ_C_SIGN` | a | — | 定點 | a 的正／零／負序列有多雜亂（例 a＝收−開：紅黑 K 的規律） |
| `LZ_C_MED` | a | — | 定點 | a 高於／低於視窗中位數的序列有多雜亂 |
| `LZ_C_JOINT` | a, b | — | 定點 | a 的漲跌與 b 的高低合成一條序列後有多雜亂 |
| `LZ_N_DIFF` | a | — | 整數 | a 漲跌序列切出的不重複片段數（越少越有規律） |
| `LZ_N_SIGN` | a | — | 整數 | a 正負序列切出的不重複片段數 |
| `LZ_N_MED` | a | — | 整數 | a 高低（相對中位數）序列切出的不重複片段數 |
| `LZ_N_JOINT` | a, b | — | 整數 | a 漲跌與 b 高低合成序列切出的不重複片段數 |
| `LZ_GAP` | a, b | — | 定點 | b 高低序列的雜亂度 − a 漲跌序列的雜亂度 |
| `LZ_ORDERED_DIFF` | a | 讀 | ±1 | a 的漲跌雜亂度低於門檻時給近 W 日的淨方向，否則 0 |
| `LZ_ORDERED_SIGN` | a | 讀 | ±1 | a 的正負雜亂度低於門檻時給多數的正負，否則 0 |
| `LZ_ORDERED_JOINT` | a, b | 讀 | ±1 | 合成序列雜亂度低於門檻時給 a 的淨方向，否則 0 |
| `LZ_ORDERED_X` | a, b | 讀 | ±1 | b 的漲跌雜亂度低於門檻時給 a 的淨方向，否則 0 |
| `LZ_LEADS_MED` | a, b | 讀 | 0/1 | b 的高低有規律、a 的漲跌仍雜亂時為 1 |
| `LZ_LEADS_DIFF` | a, b | 讀 | 0/1 | b 的漲跌有規律、a 的漲跌仍雜亂時為 1（例 b＝量） |
| `LZ_QUIET` | a, b | 讀 | 0/1 | a 的漲跌與 b 的高低都低於門檻時為 1 |
| `LZ_ECHO_DIFF` | a | — | ±1 | a 漲跌序列最後一段是長的重複片段時，給 a 今天的漲跌方向，否則 0 |
| `LZ_ECHO_MED` | a, b | — | ±1 | b 高低序列最後一段是長的重複片段時，給 a 今天的漲跌方向，否則 0 |
| `LZ_ECHO_DOUBLE` | a, b | — | ±1 | a 與 b 的最後一段都是長的重複片段時，給 a 今天的漲跌方向，否則 0 |
| `RT_ZF` | a | — | 定點 | 由舊到新數，創新高次數 − 創新低次數的 z 值 |
| `RT_ZB` | a | — | 定點 | 從今天往回數，創新高 − 創新低的 z 值 |
| `RT_ZS` | a | — | 定點 | 創新高＋創新低總次數的 z 值 |
| `RT_HIGH_SHARE` | a | — | 定點 | 紀錄裡新高所占的比例 0..1 |
| `RT_RECENT_SHARE` | a | — | 定點 | 從今天往回數的紀錄裡新低所占的比例 0..1 |
| `RT_DIVERGENCE` | a | — | 定點 | 由舊到新的 z 值 − 往回數的 z 值 |
| `RT_TREND` | a | 讀 | ±1 | 由舊到新的 z 值超過門檻 +1、低於負門檻 −1，否則 0 |
| `RT_RECENT` | a | 讀 | ±1 | 往回數的 z 值超過門檻 +1、低於負門檻 −1，否則 0 |
| `RT_QUIET` | a | — | 0/1 | 創紀錄的次數比隨機少（總次數 z < 0）時為 1 |
| `RT_TURN` | a | 讀 | ±1 | 由舊到新與往回數的方向相反時給往回數的方向，否則 0 |
| `RT_BURST` | a | 讀 | 0/1 | 創紀錄總次數的 z 值超過門檻（紀錄密集）時為 1 |
| `RT_HIGH_SIDE` | a | — | ±1 | 紀錄夠多時，新高占八成以上 +1、兩成以下 −1，否則 0 |
| `RT_RECENT_SIDE` | a | — | ±1 | 往回數的紀錄夠多時，依新低占比給 +1／−1，否則 0 |
| `RT_AGREE` | a | 讀 | ±1 | 由舊到新與往回數的方向一致時給該方向，否則 0 |
| `RT_LATE` | a | — | ±1 | 最後一個紀錄落在最新四分之一的日子時給它的方向（新高 +1、新低 −1），否則 0 |
| `RT_EARLY` | a | — | 0/1 | 有紀錄、而且最後一個紀錄落在較舊的一半時為 1 |
| `RT_BURST_SIDE` | a | 讀 | ±1 | 紀錄密集時新高或新低哪邊多 +1／−1，否則 0 |
| `RT_STAIR` | a | — | ±1 | +1：往回數沒有新高紀錄、而由舊到新的新高紀錄 ≥ 2 個（階梯式上漲）；−1：新低的鏡像；否則 0 |
| `RT_NET_F` | a | — | 整數 | 由舊到新創新高次數 − 創新低次數 |
| `RT_NET_B` | a | — | 整數 | 從今天往回數創新低次數 − 創新高次數 |
| `RT_COUNT_F` | a | — | 整數 | 創新高＋創新低的總次數 |
| `ENT_DIFF3` | a | — | 定點 | a 漲／平／跌三種的分布有多平均（夏農熵，bit；越高越隨機） |
| `ENT_SIGN3` | a | — | 定點 | a 正／零／負三種的分布有多平均 |
| `ENT_JOINT6` | a, b | — | 定點 | a 的漲跌與 b 的高低合成六種狀態的分布有多平均 |
| `VG_IRREV` | a | — | 定點 | 序列正著看與倒著看差多少（可視圖不可逆性） |
| `VG_IRREV_SKEW` | a | — | 定點 | 上漲段與下跌段的不可逆性差 |
| `VG_DEGREE` | a | — | 定點 | 可視圖的平均連線數 |
| `VG_DEGREE_BIAS` | a | — | 定點 | 往前與往後連線數的偏差 |
| `VG_LAMBDA` | a | — | 定點 | 連線數分布的衰減率（bit） |
| `VG_REACH_DIFF` | a | — | 整數 | 今天往前看得到的點數 − 往後看得到的點數 |
| `VG_SPAN_MEAN` | a | — | 定點 | 每條連線平均跨幾天 |
| `VG_REGIME` | a | — | ±1 | 平均連線數比隨機序列的期望高 5% 以上 +1、低 5% 以上 −1，否則 0 |
| `VG_REACH` | a | — | ±1 | 今天在高點圖連到的點數明顯多於期望 +1；在低點圖同式成立 −1；否則 0 |
| `VG_HUB` | a | — | ±1 | 只有高點圖有樞紐（少數點連到大半的點）+1、只有低點圖有 −1，否則 0 |
| `VG_FULL_REACH` | a | — | ±1 | 今天連得到視窗最舊的一天：高點圖 +1、低點圖 −1，否則 0 |
| `VG_IRREV_SIDE` | a | — | ±1 | 不可逆性夠大時給方向 +1／−1，否則 0 |
| `VG_STACK` | a | — | ±1 | 高點圖的堆疊層數比低點圖多 2 層以上 +1，反過來 −1，否則 0 |
| `VG_SPAN` | a | — | ±1 | 高點圖的連線跨距明顯長於低點圖 +1，反過來 −1，否則 0 |
| `VG_LINK_BREAK` | a | — | 0/1 | 高點圖與低點圖上，今天都只連到昨天時為 1 |
| `VG_LONE` | a | — | ±1 | 今天在低點圖只連到昨天、在高點圖連 3 點以上 +1；鏡像 −1；否則 0 |
| `VG_EXTREME` | a | — | ±1 | 今天是視窗裡的最高 +1 或最低 −1，否則 0 |
| `VG_TILT` | a | — | ±1 | 往前與往後連線數差的方向 |
| `VG_L1` | a | — | 整數 | 正看與倒看的連線數分布差距總和 |

例子：

| 白話 | 寫法 |
|---|---|
| 近 60 日漲跌越有規律越前 | `NEG(LZ_C_DIFF(60, close))` |
| 漲跌有規律時才照 20 日漲幅排（方向相反的取負） | `MUL(LZ_ORDERED_DIFF(60, 80, close), RET(20, close))` |
| 量先有規律、價還雜亂的才給分 | `MASK(LZ_LEADS_DIFF(60, 80, close, volume), RET(20, close))` |
| 近 120 日常創新高 | `RT_ZF(120, close)` |
| 價量合成狀態越集中越前 | `NEG(ENT_JOINT6(60, close, volume))` |

### 4.13 三、四輸入的算子

| 名字 | 輸入 | 意思 | 現況 |
|---|---|---|---|
| `SELECT(c, a, b)` | (BOOL, X, X) → X | c 無效 → 無效；c 為真 → a，否則 b | 保留函數（第 5.1 節）；公式文字不收，寫 `c ? a : b`（＝`FILL(MASK(c, a), MASK(NOT(c), b))`） |
| `REC2(K, x, c0, c1, c2)` | x：X；係數 RATIO → X；K 0..756 | 二階線性遞迴 y[t] ＝ ⌊(c1·y[t−1] ＋ c2·y[t−2] ＋ c0·x[t]) ÷ 1e6⌋；起點兩天 y = x；起點後 K 步內無效 | 公式文字不收 |
| `GRESID(k, x, z)` | (群鍵, X, 任意數值) → X | 群內迴歸殘差（每天、同群；第 4.8 節） | 可寫 |
| `STATESCAN(x, up, lo)` | 三者同一個數值單位 → X | 條件狀態掃描（每檔沿日子走）：起算當天值＝up、狀態上行；上行時 x 小於前值就切下行、值換成 lo，否則值＝max(前值, up)；下行對稱（x 大於前值就切上行、值換成 up，否則值＝min(前值, lo)）。三個輸入任一無效 → 當天無效、下一個三者都有效的日子重新起算 | 可寫 |

`STATESCAN` 例：棘輪式移動停損線 `STATESCAN(close, ROLL_MAX(20, close) * 0.9, ROLL_MIN(20, close) * 1.1)`——上行時停損只往上收、收盤跌破就翻到上方；`close > STATESCAN(close, ROLL_MAX(20, close) * 0.9, ROLL_MIN(20, close) * 1.1)` ＝目前在上行段。

公式樹的一個節點只有兩個子節點：三輸入的算子在樹裡多一個內部節點放第二、三個輸入（不算進複雜度的節點數、看不到）。四輸入的 `REC2` 照舊回 `APX_UNSUPPORTED`。

---

## 5. 保留函數的原子定義式

分界只有兩種：**原子**是最小的運算，引擎直接實作；**保留函數**由原子組成，一定拆得回原子。保留函數分兩種：

- **第 4 章標「保」的**：名字在算子表裡。5.1 列出每一個的原子定義式，跟用名字寫逐位元相同。
- **文字層展開的**（5.2）：讀公式時直接代換成原子，沒有自己的算子編號。展開後的節點數算進公式大小。

### 5.1 標「保」的保留函數與原子定義式（41 個）

保留函數與右邊的原子式逐位元相同（值、有效位、單位）。「前提」欄空白＝對所有輸入；有寫的＝在那個範圍內逐位元相同。

| 保留函數 | 原子定義式 | 前提 |
|---|---|---|
| `DIFF(N, x)` | `SUB(x, LAG(N, x))` | |
| `OR(a, b)` | `NOT(MUL(NOT(a), NOT(b)))` | |
| `MAX2(a, b)` | `FILL(MASK(GT(a, b), a), MASK(VALID(a), b))` | |
| `MIN2(a, b)` | `FILL(MASK(LT(a, b), a), MASK(VALID(a), b))` | |
| `SELECT(c, a, b)`（寫成 `c ? a : b`） | `FILL(MASK(c, a), MASK(NOT(c), b))` | |
| `ROLL_RANK(N, x)` | `RATIO(WRANK(N, x), 2*(N-1):COUNT)` | |
| `CS_PCT(x)` | `RATIO(GRANK(0, x), MUL(2:RATIO, SUB(GCOUNT(0, x), 1:COUNT)))` | |
| `CS_PEER_RANK(L, x)` | `RATIO(GRANK(L, x), MUL(2:RATIO, SUB(GCOUNT(L, x), 1:COUNT)))` | |
| `CS_REL_PARENT(L, x)` | `SUB(x, XL_PARENT(L, x))` | |
| `CS_RANK_CHG(N, x)` | `SUB(CS_PCT(x), LAG(N, CS_PCT(x)))` | |
| `CS_QCUT(N, x)` | `MIN2(MUL(CS_PCT(x), N:COUNT), N-1:COUNT)` | |
| `CS_WINZ(k, x)` | `MAX2(MIN2(CS_Z(x), k/10:RATIO), -k/10:RATIO)` | |
| `CS_RANKMIX(0, 0, a, b)` | `ADD(CS_PCT(a), CS_PCT(b))` | |
| `CS_RANKMIX(0, 1, a, b)` | `ADD(SUB(1:RATIO, CS_PCT(a)), CS_PCT(b))` | |
| `CS_RANKMIX(0, 2, a, b)` | `ADD(CS_PCT(a), SUB(1:RATIO, CS_PCT(b)))` | |
| `CS_RANKMIX(0, 3, a, b)` | `ADD(SUB(1:RATIO, CS_PCT(a)), SUB(1:RATIO, CS_PCT(b)))` | |
| `CS_RANKMIX(L, flags, a, b)` | 同上四式，`CS_PCT` 換成 `CS_PEER_RANK(L, …)` | L 1..4 |
| `RET(N, x)` | `MASK(GT(LAG(N, x), 0), RATIO(SUB(x, LAG(N, x)), LAG(N, x)))` | |
| `RET_SKIP(N, M, x)` | `MASK(GT(LAG(N, x), 0), RATIO(SUB(LAG(M, x), LAG(N, x)), LAG(N, x)))` | |
| `DIST_HIGH(N, x)` | `MASK(GT(ROLL_MAX(N, x), 0), RATIO(SUB(x, ROLL_MAX(N, x)), ROLL_MAX(N, x)))` | |
| `DIST_LOW(N, x)` | `MASK(GT(ROLL_MIN(N, x), 0), RATIO(SUB(x, ROLL_MIN(N, x)), ROLL_MIN(N, x)))` | |
| `FN_YOY(N, x)` | `LOGR(x, LAG(N, x))` | |
| `FIRST_IN_WINDOW(N, b)` | `MUL(b, LT(WSUM(N-1, LAG(1, b)), 1:COUNT))` | b 只有 0／1（比較、`NOT`、`OR`、`VALID`、`EV_PULSE`、`EV_CHANGE` 與它們的 `LAG`、`MASK`、`FILL`、BOOL 相乘） |
| `COUNT(N, b)` | `WSUM(N, b)` | |
| `EV_POST_N(N, b)` | `LAG(N, b)` | |
| `EV_PULSE(c)` | `VALID(c)` | c 是欄 |
| `FN_DY(N, a, b)` | `MASK(GT(b, 0), RATIO(WSUM(N, FILL(a, 0:MONEY)), b))` | a ≥ 0（股利） |
| `FN_EGROW(N, x)` | `RATIO(SUB(x, LAG(N, x)), ABS(LAG(N, x)))` |  |
| `FN_GM_CHG(N, a, b)` | `DIFF(N, MASK(GT(b, 0), RATIO(a, b)))` |  |
| `CS_REL_RET(N, L, x)` | r＝`RET(N, x)`：`MASK(GT(GCOUNT(L, r), 1:COUNT), SUB(r, RATIO(GSUM(L, r), GCOUNT(L, r))))` |  |
| `CS_RANK_MEAN(N, x)` | p＝`CS_PCT(x)`：`MASK(MUL(VALID(p), NOT(LT(MUL(2:COUNT, WSUM(N, VALID(p))), N:COUNT))), RATIO(WSUM(N, FILL(p, 0:RATIO)), WSUM(N, VALID(p))))` |  |
| `RVOL(N, x)` | `ROLL_STD(N, LOGR(x, LAG(1, x)))` |  |
| `SKEW(N, x)` | `WSKEW(N, LOGR(x, LAG(1, x)))` |  |
| `KURT(N, x)` | `WKURT(N, LOGR(x, LAG(1, x)))` |  |
| `ACORR(N, k, x)` | `WACORR(N, k, LOGR(x, LAG(1, x)))` |  |
| `EV_DAYS_SINCE(N, b)` | s＝`SUB(SCANSUM(FILL(b, 0:BOOL), 1:DAYS), 1:DAYS)`：`MASK(NOT(GT(s, N:DAYS)), s)` |  |
| `UP_STREAK(N, x)` | c＝`FILL(GT(x, LAG(1, x)), 0:BOOL)`：`MASK(MUL(VALID(x), VALID(LAG(N, VALID(x)))), MIN2(SCANSUM(NOT(c), MUL(c, 1:COUNT)), N:COUNT))` |  |
| `DOWN_STREAK(N, x)` | 同 `UP_STREAK`，`GT` 換成 `LT` |  |
| `EV_EARN_DRIFT(N, a, b)` | e＝`EV_CHANGE(b)`、base＝`LATCH(e, LAG(1, a))`、s＝`SUB(SCANSUM(e, 1:DAYS), 1:DAYS)`：`MASK(MUL(GT(base, 0), LT(s, N:DAYS)), RATIO(SUB(a, base), base))` |  |
| `EV_IDX_CHG(N, b)` | e＝`EV_CHANGE(b)`、s＝`SUB(SCANSUM(e, 1:DAYS), 1:DAYS)`、m＝`RATIO(SUB(N:DAYS, s), N:DAYS)`：`MASK(VALID(WSUM(N+1, b)), FILL(MASK(LT(s, N:DAYS), FILL(MASK(b, m), MASK(NOT(b), NEG(m)))), 0:RATIO))` | b 只有 0／1 |
| `CS_NEUTRAL_COUNTRY(a, b)` | `CS_NEUTRAL_IND(3, a, b)` |  |
| `GWMEAN(L, w, x)` | `GWMEAN_BY(PARENT_ID(L), w, x)` |  |
| `CS_NEUTRAL_IND(L, a, b)` | `SUB(a, GWMEAN(L, b, a))` |  |
| `CS_NEUTRAL_SIZE(a, b)` | `GRESID(0:COUNT, a, b)` |  |
| `VOL_RATIO(N, x)` | s＝`WSUM(N, LAG(1, x))`：`MASK(GT(s, 0), MULDIV(N, x, s))` | x ≥ 0（量、成交額） |

- 定義式裡的 `2*(N-1)`、`N-1`、`N+1`、`k/10` 是代入參數後的數字（例 `ROLL_RANK(60, x)` 的除數是 `118:COUNT`）。
- 長式子前面的「s＝…」「c＝…」只是把重複的子式寫一次；展開後就是同一棵原子樹。
- `RET`、`RET_SKIP`、`DIST_HIGH`、`DIST_LOW`、`FN_YOY` 的 N 可以寫到 1,260：N 超過 252 時，讀公式那一刻就照上表的定義式展開成原子（`LAG`、`ROLL_MAX`／`ROLL_MIN`、`LOGR` 本來就到 1,260），節點數照展開後算；N 不論多少都是同一個定義、同一個結果。
- `FN_DY` 的除法用到第 3 章的單位規則：每股金額 ÷ 價 → 比值。保留函數另外檢查「第一個輸入要是每股值」，原子式不檢查。
- 第 7.4 節的財報比值（資料）也各有一條逐位元相同的原子式，列在那一節。

### 5.2 文字層展開的保留函數（35 個）

寫 `NAME` 或 `apx.NAME` 都收。展開成右邊的原子。

| 保留函數 | 展開式 | 輸出 | 意思 | 例子 |
|---|---|---|---|---|
| `GPCT(g, x)` | `RATIO(GRANK(g, x), ADD(SUB(GCOUNT(g, x), 1:COUNT), SUB(GCOUNT(g, x), 1:COUNT)))` | RATIO 0..1 | x 在同一群裡的百分位（最小 0、最大 1、同值取中間；群裡不到 2 檔 → 無效）。g 寫層級數字（0 全市場、1..4）或群鍵（任何算式，同一天值相同的算同一群）；算法同 `CS_PCT`、`CS_PEER_RANK`（`GPCT(0, x)`＝`CS_PCT(x)`、`GPCT(1, x)`＝`CS_PEER_RANK(1, x)`）。每一群的第一名都是 1：當整條分數、要選的檔數比群數少時給 `portfolio.tie_by` | `GPCT(CS_QCUT(10, close), RET(20, close))` |
| `MOD(a, n)`（＝`a % n`） | `SUB(a, MUL(n, FLOOR(RATIO(a, n))))` | 同 a | a 除以 n 的餘數，和 n 同號；n = 0 → 無效 | `MASK(sym.code_num % 10 == MDAY() % 10, RET(20, close))` |
| `ROUND(x)` | `FLOOR(ADD(x, 0.5))` | 同 x | 四捨五入（.5 往正無窮）；整數單位就是 x | `ROUND(RATIO(volume, 1000))` |
| `CEIL(x)` | `NEG(FLOOR(NEG(x)))` | 同 x | 無條件進位；整數單位就是 x | `CEIL(RATIO(volume, 1000))` |
| `WEEK_START()` | `GT(FLOOR(RATIO(ADD(EPOCH_DAY(), 3:DAYS), 7:COUNT)), LAG(1, FLOOR(RATIO(ADD(EPOCH_DAY(), 3:DAYS), 7:COUNT))))` | BOOL | 這週第一個交易日（週一休市就是週二，整週休市照日曆算）；週序跟前一個交易日不同才真；面板第一天無效 | `MASK(WEEK_START(), CS_PCT(NEG(RET(5, close))))` |
| `TDAY_OF_MONTH()` | `SCANSUM(MDAY() < LAG(1, MDAY()), 1:COUNT)` | COUNT | 當月第幾個交易日（月初第一個交易日＝1）；面板第一個月不完整，那個月無效 | `TDAY_OF_MONTH() <= 5 ? CS_PCT(NEG(RET(5, close))) : CS_PCT(RET(20, close))` |
| `AT_MONTH_START(x)` | `LATCH(MDAY() < LAG(1, MDAY()), x)` | 同 x | x 在當月第一個交易日的值，整個月沿用（那天 x 無效＝整個月無效） | `RATIO(AT_MONTH_START(LAG(1, close)), AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, close))))) - 1` |
| `LISTED_DAYS()` | `SCANSUM(DAY_INDEX() < 1, MUL(VALID(close), 1:COUNT))` | COUNT | 有收盤價以來的交易日數：面板第一天起累計、含當天；停牌那天不加、復牌不重置 | `SCANMAX(LISTED_DAYS() == 1 && VALID(close), FILL(close, 0:PRICE))` |
| `LISTED_TDAYS()` | `SUB(cal.tday_idx, list.tday_idx)` | COUNT | 上市（櫃）以來的交易日數：從真的上市日算、含停牌日；上市那天＝0；2000-01-04 以前上市的從 2000-01-04 算；上市前無效。面板要有 `cal.tday_idx`、`list.tday_idx` 兩欄 | `MASK(LISTED_TDAYS() >= 500, RET(60, close))` |
| `MKT_MEAN(x)` | `RATIO(GSUM_ALL(0, x), GCOUNT_ALL(0, x))` | 同 x | x 的全市場等權均值（往下取整），每檔同值。成員＝當天 x 有效的每一檔，不看選股範圍、不篩普通股；沒有一檔有效 → 無效 | `RET(20, close) - MKT_MEAN(RET(20, close))` |
| `EW_INDEX(p)` | `EXP(SCANSUM(DAY_INDEX() < 1, FILL(LOG(1 + MKT_MEAN(MASK(common == 1, RATIO(p, LAG(1, p)) - 1))), 0)))` | RATIO | 等權大盤：普通股每天等權再平衡，面板第一天＝1，每檔同值；當天或前一天 p 無效的檔不進當天均值；p 寫 `close`。面板判斷不了普通股 → 錯誤 | `EW_INDEX(close) > ROLL_MEAN(250, EW_INDEX(close)) ? CS_PCT(RET(60, close)) : CS_PCT(NEG(RET(60, close)))` |
| `STREAK(b)` | `SCANSUM(NOT(b), MUL(b, 1:COUNT))` | COUNT | b 連續為真的天數（今天為假＝0）；b 無效那天無效，之後要等到 b 為假才重新起算；序列開頭到第一次 b 為假之前無效 | `STREAK(foreign_net > 0) >= 3` |
| `SINCE(b)` | `SCANSUM(b, 1:DAYS) - 1:DAYS` | DAYS | 距上次 b 為真的交易日數（為真那天＝0）；b 無效那天無效，之後要等到 b 再為真；第一次為真之前無效 | `SINCE(EV_CHANGE(revenue_m) && revenue_m >= ROLL_MAX(250, revenue_m)) <= 20` |
| `STATE(on, off)` | `LATCH(on \|\| off, MUL(on, 1:COUNT)) > 0:COUNT` | BOOL | on 為真那天起為真、off 為真那天起為假，其他天沿用；同一天兩個都真＝真；on、off 任一無效那天沿用前一天；第一次 on 或 off 為真之前無效 | `STATE(close > ROLL_MEAN(250, close), close < ROLL_MEAN(60, close))` |
| `GMEAN(first, x)` | `GMEAN_BY(first, x)`；x 是 COUNT：`GMEAN_BY(first, RATIO(x, 1:COUNT))` | 同 x | 同群成員 x 的平均（往下取整，和超出 i64 也有值），寫給每個成員；成員＝當天選股範圍內、x 有效（群鍵版另要群鍵有效）；first 同 GSUM：沒寫單位的整數＝層級（0 全市場），其餘＝群鍵；x 是 COUNT 時輸出 RATIO；非成員無效。`GMEAN(L, x)` 跟 `XL_PARENT(L, x)` 同一棵樹；x 是 COUNT 時兩者身分不同（單位本來就不同：`GMEAN` 出 RATIO、`XL_PARENT` 出 COUNT） | `GMEAN(CS_QCUT(10, fund.mktcap), RET(20, close))` |
| `GDEMEAN(first, x)` | `MASK(GCOUNT(first, x) > 1:COUNT, x - GMEAN(first, x))` | 同 x | x 減同群平均（GMEAN）；群只有自己一檔 → 無效；非成員無效。x 要 1e6 定點的單位（PRICE、RATIO、MONEY…） | `GDEMEAN(CORR_GROUP(RATIO(close, LAG(1, close))), RET(20, close))` |
| `GTOP(N, first, x)` | `ORD(first, x) < N` | BOOL | x 在同一群裡是前 N 大（精確名次：每群剛好 N 檔，群不到 N 檔就全部）；first 寫層級數字（0 全市場、1 產業…）或群鍵；同值照固定打散，跟選股名次同一套；非成員無效 | `MASK(GTOP(3, 1, fund.mktcap), RET(20, close))`（每個產業市值前 3 大） |
| `GZ(first, x)` | `RATIO(SUB(x, GMEAN_BY(first, x)), GSTD(first, x))` | RATIO | x 在同一群裡的 Z 分數＝(x − 群內平均) ÷ 群內母體標準差；first 同 `GSTD`：沒寫單位的整數＝層級（`GZ(0, x)`＝`CS_Z(x)`、`GZ(L, x)`＝`CS_PEER_Z(L, x)`，逐位元相同），其餘＝群鍵；群不到 2 檔、標準差 0、x − 平均超出範圍 → 無效；非成員無效 | `GZ(CS_QCUT(10, fund.mktcap), RET(20, close))` |
| `CS_TOPK(k, x)` | `ORD(0, x) < k` | BOOL | x 當天在選股範圍裡是前 k 大（剛好 k 檔）；同值照固定打散；x 無效或不在範圍 → 無效 | `MASK(CS_TOPK(50, fund.mktcap), RET(20, close))` |
| `INV_VOL_KEY()` | `INV_VOL_KEY(20, RET(1, close))` | COUNT | 依波動倒數的權重鍵：近 20 日報酬波動的倒數（越穩越大）；`weighting="inv_vol"` 用的就是這個波動。換視窗或序列寫原子 `INV_VOL_KEY(N, x)`（4.6） | `MASK(CS_PCT(INV_VOL_KEY()) > 0.5, RET(20, close))` |
| `MKT_SHARE(b)` | `MKT_MEAN(MASK(common == 1, MUL(b, 1:RATIO)))` | RATIO | 全市場普通股裡 b 為真的比例 0..1（往下取整到 1e-6），每檔同值；分母＝當天 b 有效的普通股，不看選股範圍；沒有一檔有效 → 無效；面板判斷不了普通股 → 錯誤 | `MKT_SHARE(close > ROLL_MEAN(60, close)) < 0.3 ? 0.5 : 1` |
| `MKT_COUNT(b)` | `GSUM_ALL(0, MASK(common == 1, b))` | COUNT | 全市場普通股裡 b 為真的檔數，每檔同值；只數 b 有效的普通股，不看選股範圍；沒有一檔有效 → 無效；面板判斷不了普通股 → 錯誤 | `MKT_COUNT(raw.close >= raw.limit_up)` |
| `BASKET_MEAN(sel, x)` | `MKT_MEAN(MASK(sel, x))` | 同 x | sel 為真那幾檔的 x 等權平均（往下取整），寫給每一檔；不看選股範圍；當天 sel 那幾檔的 x 都無效 → 無效 | `CORR(250, RET(1, close), BASKET_MEAN(symbol == "2330", RET(1, close)))` |
| `CLIP(x, lo, hi)` | `MAX2(MIN2(x, hi), lo)` | 同 x | 把 x 夾在 lo..hi（lo > hi 時＝lo）；任一無效 → 無效；三個同單位 | `CLIP(RET(1, close), -0.1, 0.1)` |
| `BETWEEN(x, lo, hi)` | `x >= lo && x <= hi` | BOOL | lo ≤ x ≤ hi（含兩端）；任一無效 → 無效；三個同單位 | `BETWEEN(CS_PCT(fund.mktcap), 0.3, 0.7)` |
| `SIGN(x)` | `MUL(x > 0, 1:COUNT) - MUL(x < 0, 1:COUNT)` | COUNT | x 的正負號 −1／0／1；x 無效 → 無效 | `GSUM(0, SIGN(RET(1, close)))` |
| `CROSS_UP(a, b)` | `a > b && !(LAG(1, a) > LAG(1, b))` | BOOL | a 由不高於 b 變成高於 b 的那天（今天 a > b、昨天 a ≤ b）；今天或昨天任一無效 → 無效；兩個同單位 | `CROSS_UP(close, ROLL_MEAN(20, close))` |
| `CROSS_DOWN(a, b)` | `a < b && !(LAG(1, a) < LAG(1, b))` | BOOL | a 由不低於 b 變成低於 b 的那天（今天 a < b、昨天 a ≥ b）；今天或昨天任一無效 → 無效；兩個同單位 | `CROSS_DOWN(foreign_net, 0)` |
| `MONTH_START()` | `MDAY() < LAG(1, MDAY())` | BOOL | 當月第一個交易日＝1，其他天 0；面板第一天無效 | `MONTH_START()` |
| `QUARTER_START()` | `MONTH_START() && (MONTH() - 1) % 3 == 0` | BOOL | 1、4、7、10 月第一個交易日＝1，其他天 0；面板第一天無效 | `QUARTER_START()` |
| `YEAR_START()` | `MONTH() < LAG(1, MONTH())` | BOOL | 每年第一個交易日＝1，其他天 0；面板第一天無效 | `YEAR_START()` |
| `EVERY(n, k)` | `MOD(DAY_INDEX(), n) == k` | BOOL | 每 n 個交易日一次、錯開 k 天：面板第一天起算的第 d 天（0 起）d 除以 n 餘 k 那天＝1，其他天 0；每檔同值；n 1..1,260、k 0..n−1 | `EVERY(21, 5)` |
| `PREV_MONTH_RET(p)` | `RATIO(AT_MONTH_START(LAG(1, p)), AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, p))))) - 1` | RATIO | 上一個日曆月的報酬＝上月最後一個交易日的 p ÷ 前月最後一個交易日的 p − 1，整個月同值；那兩天任一個 p 無效（或分母 0）→ 整個月無效 | `CS_PCT(PREV_MONTH_RET(close))` |
| `ROLL_MEAN_AVAIL(N, x)` | `RATIO(WSUM($N, FILL(x, 0)), WSUM($N, VALID(x)))` | 同 x | 近 N 天裡 x 有效那幾天的平均（往下取整；無效的天不算進分母）；面板前 N − 1 天無效；視窗內全無效 → 無效；N 1..1,260 | `ROLL_MEAN_AVAIL(20, foreign_net)` |
| `VWAP(N)` | `RATIO(WSUM($N, px.turnover), WSUM($N, volume))` | MONEY | 近 N 天成交均價＝成交金額合計 ÷ 成交股數合計（元／股，原始價口徑：跟 raw.close 比）；視窗內任一格無效、股數合計 0、成交金額合計超出定點範圍 → 無效；N 1..1,260。面板要有 px.turnover、volume | `raw.close > VWAP(20)` |
| `REG_SLOPE(N, y, x)` | `WREG(N, 0, y, x)` | RATIO | y 對 x 近 N 日一元迴歸的斜率（x 動 1、y 平均動多少）；N 3..1,260，y、x 同單位；視窗內任一格無效或 x 沒變化 → 無效 | `REG_SLOPE(60, RET(1, close), MKT_MEAN(RET(1, close))) < 0.8` |
| `REG_INTERCEPT(N, y, x)` | `WREG(N, 1, y, x)` | 同 y | 截距（x 為 0 時 y 的平均）；其餘同 `REG_SLOPE` | `CS_PCT(REG_INTERCEPT(60, RET(1, close), MKT_MEAN(RET(1, close))))` |
| `REG_RESID(N, y, x)` | `WREG(N, 2, y, x)` | 同 y | 今天的殘差（今天的 y 減迴歸線在今天 x 的值）；其餘同 `REG_SLOPE` | `CS_PCT(REG_RESID(60, RET(1, close), MKT_MEAN(RET(1, close))))` |
| `REG_R2(N, y, x)` | `WREG(N, 3, y, x)` | RATIO 0..1 | R²（迴歸線解釋了多少 y 的變化）；其餘同 `REG_SLOPE` | `REG_R2(120, RET(1, close), MKT_MEAN(RET(1, close))) < 0.2` |
| `REG_T(N, y, x)` | `WREG(N, 4, y, x)` | RATIO | 斜率的 t 值（斜率 ÷ 標準誤）；其餘同 `REG_SLOPE` | `REG_T(60, RET(1, close), MKT_MEAN(RET(1, close))) > 2` |

- 「月初」的判法：`MDAY() < LAG(1, MDAY())`（今天的日期數字比前一個交易日小）。前提是相鄰兩個交易日相差不到 28 個日曆日。
- 上個日曆月的報酬＝`PREV_MONTH_RET(close)`；上市滿 2 年＝`LISTED_TDAYS() >= 500`；上市以來最高價＝`LISTED_DAYS` 那個例子。
- `MKT_MEAN`、`EW_INDEX`、`MKT_SHARE`、`MKT_COUNT`、`BASKET_MEAN`、`MONTH_START`、`QUARTER_START`、`YEAR_START`、`EVERY` 的輸出每檔同值，可以直接放 `exposure`、`rebalance_when`（第 9 章）。`EVERY(n, k)` 從面板第一天起算、不看暖機；portfolio 的 `apx.EVERY(n)`（一個引數）仍是「暖機後每 n 天」＝`rebalance_days`。
- 輸入寫 X 的同一列是同一種量：沒寫單位的常數跟著有單位的那個（`CLIP(close, 10, 100)` 的 10、100 是 PRICE；跟原子同一條規則，見 2.3）。`GMEAN`、`GDEMEAN` 的第一個引數同 `GSUM`：沒寫單位的整數＝層級（0 全市場、1..4），其餘＝群鍵。輸入可以宣告成「X，COUNT 先換 RATIO」（目前 `GMEAN` 的 x）：給 COUNT 時先換成 `RATIO(x, 1:COUNT)` 再代入，展開式照實印出來。
- `EW_INDEX` 實測（台股 4,127 日）：對雙精度參考最大相對誤差 6.07e-3、任意 250 日內 3.86e-4。
- `SQRT` 收整數單位也是文字層展開：COUNT／DAYS 的 `SQRT(n)`＝`SQRT(RATIO(n, 1:同單位))`；AMOUNT 的 `SQRT(a)`＝`MUL(1000, SQRT(RATIO(a, 1000000:AMOUNT)))`（小數三位）。

---

## 6. 相似度

類股、題材不用人工清單：**由相關性自訂籃子**。「相近」看的是哪一條序列由寫的人決定——報酬、成交量變化、熱度、月營收年增、法人買賣超，任何數值算式都行。

兩個函數都是**衍生欄**：主機在輪開始時算好、接在面板後面的一欄。兩者都只用當期之前的資料。

### 6.1 `CORR_GROUP`：把表現相近的股票分成群

```
CORR_GROUP([lookback[, every[, step[, corr_bp[, quant]]]]], <算式>)
```

| 參數 | 預設 | 意思 |
|---|---|---|
| lookback | 252 | 每一期用當期第一天之前幾個交易日 |
| every | 126 | 每幾個交易日重分一次群 |
| step | 1 | 每幾日取一個點（月資料用 21） |
| corr_bp | 3000 | 相關門檻，萬分之一為單位（3000＝0.3） |
| quant | 1 | 1＝先換成視窗內名次再算相關（任何序列）；0＝報酬式（輸入要是日報酬） |
| `<算式>` | — | 任何數值算式，只用面板的欄；不能巢狀 `CORR_GROUP`／`CORR_SEED` |

- 參數寫在算式前面，可以從後面省略：`CORR_GROUP(RATIO(close, LAG(1, close)))` 全用預設。
- 輸出單位 CODE：**只能當群鍵**（第 4.8 節的群鍵版算子）。
- 群的定義：兩兩算相關，相關 ≥ 門檻的併成一群，群內任兩檔都過門檻。
- 點數＝lookback ÷ step，要在 20..1,024（quant 0 時 20..500）。
- 自己一群的股票：`GRANK` 無效、`GCOUNT`＝1。
- 一個結果庫的 `CORR_GROUP` 清單固定，是庫身分的一部分。不指定庫名會照清單自動分庫；指定的庫清單不同會回建議的庫名。

`<算式>` 怎麼選：

| 客戶說的「相近」 | 算式 |
|---|---|
| 股價一起漲跌 | `RATIO(close, LAG(1, close))` 或 `LOGR(close, LAG(1, close))` |
| 成交量一起放大縮小 | `DIFF(1, LOG(volume))` |
| 一起熱起來 | `RATIO(volume, ROLL_MEAN(20, volume))`，配 `CORR_GROUP(60, 20, 1, 3000, 1, …)` |
| 月營收年增同步 | `RWIN_PCT(12, revenue_m)`，配 `CORR_GROUP(1260, 126, 21, 3000, 1, …)` |
| 外資一起買賣 | `foreign_net` 或 `RATIO(foreign_net, volume)` |

排名分數用「比同群平均高多少」（題庫 R02）：

```
MASK(VALID(GRANK(CORR_GROUP(LOGR(close, LAG(1, close))), RET(60, close))),
     SUB(RET(60, close),
         RATIO(GSUM(CORR_GROUP(LOGR(close, LAG(1, close))), RET(60, close)),
               GSUM(CORR_GROUP(LOGR(close, LAG(1, close))), MASK(VALID(RET(60, close)), 1:RATIO)))))
```

外層的 `MASK(VALID(GRANK(…)), …)` 把沒有同群夥伴的股票拿掉。

### 6.2 種子籃子：跟這幾檔同一群的股票

條件裡（選股範圍、`portfolio.hold`、`portfolio.weight`）寫：

```
GSUM(CORR_GROUP(<算式>), symbol in {"2330", "2382", "3231"}) > 0
```

意思：和任一檔種子同一群的股票（含種子自己）。`GSUM(k, BOOL)` 是同群裡條件成立的檔數。

| 要什麼 | 寫法 |
|---|---|
| 排除這個籃子（沒有群的股票留下） | `!FILL(GSUM(CORR_GROUP(RATIO(close, LAG(1, close))), symbol in {"2881", "2882", "2891"}) > 0, 0:BOOL)` |
| 兩個籃子合併 | 種子寫進同一個 `in {…}`；不同序列寫 `FILL(A, 0:BOOL) \|\| FILL(B, 0:BOOL)` |
| 不用種子：有符合條件的股票的整群 | `GSUM(k, <每檔每天的 BOOL>) > 0` |
| 最近很熱的股票和一起熱起來的同一群 | `GSUM(CORR_GROUP(60, 20, 1, 3000, 1, RATIO(volume, ROLL_MEAN(20, volume))), CS_PCT(ROLL_MEAN(20, RATIO(volume, ROLL_MEAN(120, volume)))) >= 0.98) > 0` |

- `GMAX`／`GMIN` 不收 BOOL，所以寫 `GSUM(…) > 0`。
- 寫 `!(…)` 會連沒有群的股票一起排除；要留下寫 `!FILL(…, 0:BOOL)`。
- 群每 every 個交易日重分一次，所以籃子成員每期會換。plan 回每期符合的檔數（平均、最小、最大）、相鄰兩期變動率、最近一期的成員。
- 條件裡的 `CORR_GROUP` 不進結果庫的清單、不分庫。
- 存起來重用：`my_stock_strategies_change action=install` 存成自訂函數（`out_unit` BOOL、不帶 `inputs`、body＝那一條），之後條件裡寫 `user.<你>.NAME()`。這種函數只用在條件。

### 6.3 `CORR_SEED`：每檔對種子的相關

```
CORR_SEED({種子…}[, lookback[, every[, step[, quant[, how]]]]], <算式>)
```

| 參數 | 預設 | 意思 |
|---|---|---|
| 種子 | — | 1..64 檔，代號用雙引號：`{"2330", "2317"}`；順序與重複不影響 |
| lookback | 252 | 每一期用當期第一天之前幾個交易日 |
| every | 21 | 每幾個交易日重算一次（≥ 1） |
| step | 1 | 每幾日取一個點（≥ 1） |
| quant | 1 | 1＝視窗內名次；0＝報酬式 |
| how | 0 | 種子多檔時：0＝把種子合成一條再算、1＝取最大、2＝平均、3＝最小 |
| `<算式>` | — | 任何數值算式 |

- 輸出單位 RATIO，值在 −1..1：是一欄數值，可以當整條公式、乘進別的分數、比門檻。
- 種子自己＝1。資料不足的股票、視窗不滿或沒有可用種子的那一期沒有值。
- 點數＝lookback ÷ step，要在 20..1,024（quant 0 時 20..500）。
- 兩條不相關的序列算出來大約在 ±2 ÷ √點數之內：20 點 ±0.45、40 點 ±0.32、60 點 ±0.26、252 點 ±0.13。
- 公式裡的 `CORR_SEED` 跟 `CORR_GROUP` 一樣算進結果庫的身分；條件與權重裡的不進清單、不分庫。

| 白話 | 寫法 |
|---|---|
| 依跟 2330 的相關排名 | `CORR_SEED({"2330"}, RATIO(close, LAG(1, close)))` |
| 最近跟這三檔一起熱起來的 | `CORR_SEED({"2330", "2382", "3231"}, 40, 5, DIFF(1, LOG(volume)))` |
| 法人買賣超同步 | `CORR_SEED({"2330"}, 60, 5, ADD(ADD(foreign_net, trust_net), dealer_net))` |
| 相關乘上動能 | `MUL(CORR_SEED({"2330"}, RATIO(close, LAG(1, close))), RET(20, close))` |
| 相關夠高的才選（選股範圍或 `hold`） | `CORR_SEED({"2330"}, RATIO(close, LAG(1, close))) >= 0.5` |
| 同上、不買種子自己 | `CORR_SEED({"2330"}, RATIO(close, LAG(1, close))) >= 0.5 && symbol != "2330"` |
| 相關越高買越多（`weight`；≤ 0 的不買） | `CORR_SEED({"2330"}, RATIO(close, LAG(1, close)))` |

### 6.4 檢驗用的同類股

同類股替換檢驗（第 11 章 `apx.PEER_SWAP`）另有自己的配對：把選中的每一檔換成跟它最相近的那一檔重算。那把尺的輸入同樣可以是任何序列。

---

## 7. 資料欄

### 7.1 欄鍵怎麼命名

欄鍵就是公式裡的葉子。常用的短名沒有前綴，其餘照來源分前綴。

| 前綴 | 是什麼 | 例 |
|---|---|---|
| （沒有） | 價量、常用財報與籌碼欄 | `open high low close volume`、`eps revenue net_income equity assets shares`、`revenue_m`、`dps`、`foreign_net trust_net dealer_net`、`margin_bal short_bal` |
| `raw.` | 交易所原始價量與漲跌停價（未還原） | `raw.close`、`raw.limit_up`、`raw.limit_down`、`raw.ref_price` |
| `px.` | 由價量算出的欄 | `px.turnover`（成交金額）、`px.turnover_rate`（週轉率） |
| `fund.` | 財報細項與市值 | `fund.mktcap`、`fund.gross`、`fund.cfo`、`fund.debt` |
| `idx.` | 市場層：指數，每檔同值 | `idx.taiex`、`idx.taiex_tr`、`idx.tpex`、`idx.tpex_tr` |
| `etf.` | 市場層：ETF 收盤，每檔同值 | `etf.0050`、`etf.0050_adj` |
| `list.` | 上市資訊 | `list.date`、`list.tday_idx` |
| `cal.` | 日曆，每檔同值 | `cal.tday_idx` |
| `sym.` | 代號的數值 | `sym.code_num`（2330 → 2330） |
| `news.` | 重大訊息 | `news.major_cnt` |
| `tdcc_` | 集保股權分散（週） | `tdcc_people`、`tdcc_big1000_pct` |
| `dt_`、`sbl_` | 當沖、借券 | `dt_vol`、`sbl_vol` |

另有不在面板上的四個代碼欄：`industry`、`market`、`common`、`symbol`（第 2.4 節）。

### 7.2 怎麼查有哪些欄

欄表隨面板變，這裡不抄整張。

| 要查什麼 | 怎麼查 |
|---|---|
| 這張面板有哪些欄、單位 | `stock_formula_help topic=column`（可帶 `panel=`） |
| 用白話找欄 | `stock_formula_help topic=column query=<白話>`（例 `query=融資`） |
| 一欄的定義、算法、公布規則 | `stock_formula_help topic=column name=<欄鍵>` |
| 有哪些面板 | `stock_formula_help topic=column` |
| 一檔一欄的歷史值 | `stock_formula_help topic=column`（`changes_only=true` 只列值變的日子） |
| 產業代碼、交易所名、類別名詞 | `stock_formula_help topic=universe` |

公式用到的欄不在指定的面板上時，plan 會自動換到全部都有的那張面板。

### 7.3 公布時點：不偷看未來

面板在切出來的時候就把每一格對齊到「那一天收盤後看得到的值」。公式只看格子，不用自己處理公布日。

| 欄的種類 | 哪一天起看得到 | 沒有新值的日子 |
|---|---|---|
| 價量（`close`、`volume`…） | 當天收盤後 | 沒有交易的日子（上市前、休市、停牌）無效 |
| 季財報（`revenue`、`eps`、`fund.*`…） | 有實際申報日：申報日的下一個交易日；沒有申報日：法定期限（年報 3/31、Q1 5/15、Q2 8/14、Q3 11/14）的下一個交易日 | 沿用上一期，從公布日起最長 155 天，超過無效 |
| 月營收 `revenue_m` | 照每筆的公布日換值（每月 10 日前公布） | 沿用到下一筆 |
| 法人買賣超（`foreign_net`…） | 當天收盤後 | 不沿用；2020 年以前沒有進出的日子是缺值，之後是 0 |
| 融資券（`margin_bal`…） | 當天收盤後 | 沿用上一筆；沒有信用交易資格的股票沒有值 |
| 集保（`tdcc_*`） | 基準日的下一個交易日 | 沿用到下一期 |
| 每股現金股利 `dps` | 除息日的下一個交易日 | 事件欄：只有那一天有值，其他天無效 |
| 市場層（`idx.*`、`etf.*`） | 當天收盤後，每檔同值 | 休市日無效 |
| 交易所原價（`raw.*`） | **晚一列**：面板第 t 天的 `raw.close` 是前一個交易日的收盤 | |

要點：

- `close`、`open`、`high`、`low` 是還原價（除權息與分割都還原到最新）。門檻也是對還原價比，不是當天看盤價。要當天的實際價位用 `raw.*`。
- 向前填的欄（財報、月營收）的公布日＝值變日 `EV_CHANGE(欄)`。照「期」算的視窗用 `RWIN`（第 4.6 節），不用天數猜。
- 事件欄（`dps`）加總用 `FN_DY` 或 `WSUM`；向前填的欄拿去加總 N 天會把同一筆算很多次。
- 市場層欄每檔同值，隨機搜尋不會自動抽成葉子，公式寫明欄名才用。
- `eps` 是近四季平均的單季每股盈餘，所以 `FN_EP(eps, close)`＝近四季本益比的倒數。

### 7.4 衍生資料：由資料欄相除得到的財報比值

這 5 個是資料：由資料欄相除得到的財報比值，跟欄位一樣是資料。寫法照舊是呼叫（`FN_EP(每股盈餘, 股價)`），名字、參數、單位、結果都跟以前一樣；名詞清單裡標 `data`。

| 簽名 | 參數 | 單位 | 無效與暖機 | 白話 | 例子 |
|---|---|---|---|---|---|
| `FN_EP(a, b)` | — | (MONEY, PRICE) → RATIO | b ≤ 0 → 無效；a 要是每股值 | 盈餘殖利率，本益比的倒數：4 × 每股盈餘 ÷ 價（季換年） | `FN_EP(eps, close)` |
| `FN_BP(a, b)` | — | (MONEY, PRICE) → RATIO | a ≤ 0 或 b ≤ 0 → 無效；a 要是每股值 | 股價淨值比的倒數：每股淨值 ÷ 價 | `FN_BP(FN_PS(equity, shares), close)` |
| `FN_PS(a, b)` | — | (MONEY, QTY) → MONEY | b ≤ 0 → 無效 | 每股值：總額 ÷ 股數 | `FN_BP(FN_PS(equity, shares), close)` |
| `FN_ROE(a, b)` | — | (MONEY, MONEY) 或 (AMOUNT, AMOUNT) → RATIO | b ≤ 0 → 無效 | 股東權益報酬率：4 × 淨利 ÷ 權益（季換年） | `FN_ROE(net_income, equity)` |
| `FN_ACCRUAL(N, a, b)` | N 1..252 | (MONEY, MONEY) → RATIO | b ≤ 0 → 無效；暖機 N | 應計比率：a＝應計、b＝資產，4 × a ÷ 今天與 N 日前資產的平均 | `NEG(FN_ACCRUAL(252, SUB(net_income, fund.cfo), assets))` |

- 每股值與總額不能混：`FN_EP`、`FN_BP` 的第一個輸入要是每股值（`eps`、`FN_PS(…)`）；兩個金額輸入一個每股值、一個總額也會被擋。總額要比市值寫 `RATIO(總額, fund.mktcap)`。
- 除法用到第 3 章的兩條單位規則：金額 ÷ 股數 → 每股金額（MONEY）、每股金額 ÷ 價 → 比值。

逐位元相同的原子式（值、有效位、單位；跟第 5 章同一套逐條驗證）。原子式不檢查「第一個輸入要是每股值」。`FN_ACCRUAL` 是熔成一次取整的統計量，沒有原子式。

| 名字 | 原子式 |
|---|---|
| `FN_EP(a, b)` | `MASK(GT(b, 0), MULDIV(4, a, b))` |
| `FN_BP(a, b)` | `MASK(MUL(GT(a, 0), GT(b, 0)), RATIO(a, b))` |
| `FN_PS(a, b)` | `MASK(GT(b, 0), RATIO(a, b))` |
| `FN_ROE(a, b)` | `MASK(GT(b, 0), MULDIV(4, a, b))` |

---

## 8. 選股範圍

### 8.1 條件怎麼寫

選股範圍是 `stock_backtest` 的 `universe=…`：一條逐日逐檔的 BOOL 算式，為真的格留下。寫法就是第 2～4 章的同一門語言。

| 白話 | 寫法 |
|---|---|
| 過去 250 個交易日每天都有收盤價 | `WSUM(250, VALID(close)) >= 250` |
| 上市滿兩年 | `cal.tday_idx - list.tday_idx >= 500`（或 `LISTED_TDAYS() >= 500`） |
| 只看上櫃、排除上市不到 60 個交易日的 | `market == TWO && cal.tday_idx - list.tday_idx >= 60` |
| 20 日均量超過 500 張、而且天天有成交 | `COUNT(20, volume > 0) >= 20 && ROLL_MEAN(20, volume) > 500000` |
| 中型股：市值第 30～70 百分位 | `CS_PCT(fund.mktcap) >= 0.3 && CS_PCT(fund.mktcap) <= 0.7` |
| 市值前 30%，但不要台積電、鴻海 | `CS_PCT(fund.mktcap) >= 0.7 && !(symbol in {"2330", "2317"})` |
| 只在這幾檔裡挑 | `symbol in {"2330", "2454", "2317"}` |
| 排除金融與營建 | `!(industry in {17, 14})` 或 `!cls.FIN() && !cls.REALTY()` |
| 近一年最大跌幅小於 40% | `WDROP(250, LOG(close)) < 0.5108` |
| 剛站上年線（20 天內才站上） | `close > ROLL_MEAN(250, close) && COUNT(20, close > ROLL_MEAN(250, close)) < 20` |
| 融資使用率低於 30%、券資比高於 10% | `RATIO(margin_bal, margin_limit) < 0.3 && RATIO(short_bal, margin_bal) > 0.1` |
| 排除 60 天內收過漲停或跌停的 | `COUNT(60, raw.close >= raw.limit_up \|\| raw.close <= raw.limit_down) == 0` |
| 排除全額交割、處置、注意股 | `!full_delivery && !disposition && !attention` |
| 跟這幾檔表現相近的 | `GSUM(CORR_GROUP(RATIO(close, LAG(1, close))), symbol in {"2330", "2382", "3231"}) > 0` |
| 股號尾數跟今天日期尾數相同 | `sym.code_num % 10 == MDAY() % 10` |

規則：

- 算式無效的格（含視窗不滿、暖機期）不在範圍內。
- 跨檔運算（`CS_PCT`、`GSUM`…）的成員＝**篩選前**的範圍，算一次、不迭代。`CS_PCT(fund.mktcap) >= 0.7` 是全市場的前 30%。
- 一個結果庫綁一個選股範圍。換範圍就換庫。
- 條件下平均每天不到 1 檔 → 錯誤 `E_EMPTY_UNIVERSE`。
- 逐檔列代號最多 20 檔（整條條件合計）。超過就是在列一整類，改用類別名詞或種子籃子。
- plan 回 `effective.universe_stats`：每日平均檔數、最後一天檔數、樣本代號。只想試算：`stock_formula_help topic=universe universe="…"`。

### 8.2 預設＝全市場普通股

`universe` 不給＝全市場普通股。

- `common_only`（預設 true）：先排除確定不是普通股的（特別股、ETF、受益憑證、權證、存託憑證），再套 `universe`。要含 ETF 等：`common_only=false`。
- 代碼欄 `common`：1＝普通股、0＝不是、無效＝判斷不了。
- 客戶沒講的條件不加。流動性門檻、上市天數都不是預設。

### 8.3 類別名詞

官方名詞 `cls.NAME()`：一類股票，任何市場同一種寫法。定義＝這個市場原始分類欄的條件（台股是產業代碼）。只用在條件。

| 名詞 | 意思 | 台股定義 |
|---|---|---|
| `cls.FIN()` | 金融 | `industry in {17}` |
| `cls.SEMI()` | 半導體 | `industry in {24}` |
| `cls.TECH()` | 科技、電子（含半導體） | `industry in {24, 25, 26, 28, 29, 30, 31, 36}` |
| `cls.COMM()` | 通訊 | `industry in {27}` |
| `cls.HEALTH()` | 醫療、生技 | `industry in {22}` |
| `cls.STAPLES()` | 民生必需 | `industry in {2, 18, 33}` |
| `cls.DISCR()` | 非必需消費 | `industry in {4, 11, 12, 16, 32, 37, 38}` |
| `cls.INDU()` | 工業 | `industry in {5, 6, 15, 35}` |
| `cls.MATER()` | 原物料 | `industry in {1, 3, 8, 9, 10, 21}` |
| `cls.ENERGY()`、`cls.UTIL()` | 能源、公用事業（台股同一個定義） | `industry in {23}` |
| `cls.REALTY()`、`cls.CONSTR()` | 不動產、營建 | `industry in {14}` |
| `cls.SHIPPING()` | 航運 | `industry in {15}` |
| `cls.STEEL()` | 鋼鐵 | `industry in {10}` |
| `cls.PLASTIC()` | 塑膠 | `industry in {3}` |
| `cls.ECOMP()` | 電子零組件 | `industry in {28}` |
| `cls.PC()` | 電腦及週邊 | `industry in {25}` |
| `cls.OPTO()` | 光電 | `industry in {26}` |
| `cls.BIO()` | 生技 | `industry in {22}` |
| `cls.GREEN()` | 綠能環保 | `industry in {35}` |

- 排除寫 `!cls.FIN()`；只看寫 `cls.SEMI()`。類別可以重疊（`TECH` 含 `SEMI`）。
- 類別名詞帶版本，策略存檔記下用的版本；定義改了，重跑才會變。
- 客戶講的是題材或「像某幾檔」、沒有對應的類別：用種子籃子（第 6.2 節）。

### 8.4 `hold` 對 `universe`

同一條條件放在兩個地方，意思不同：

| 放哪裡 | 排名在哪裡排 | 不合條件的股票 |
|---|---|---|
| `universe="close > ROLL_MEAN(60, close)"` | 只在範圍內比高低 | 不參與排名，也不會被買 |
| `portfolio={"hold":"close > ROLL_MEAN(60, close)"}` | 照全市場排 | 不買；持有中的到換股日賣掉 |

只在某群股票裡比高低用 `universe`；排名照全體、只買只持有合條件的用 `hold`。

### 8.5 舊寫法（相容）

`universe_lang`：0＝自動（預設）、1＝舊寫法、2＝VQL。自動的判法：舊寫法編得過就是 1，否則照 VQL 讀。plan 的 `effective.universe_lang` 回判定結果。

舊寫法只為了相容舊的結果庫，新寫的一律用 VQL。兩者的差別：

| 項 | 舊寫法（1） | VQL（2） |
|---|---|---|
| 值 | 實數；跟缺值比較一律為假 | 定點；算式無效的格不在範圍內 |
| 時序運算 | 沒有 | 全部算子 |
| 函式 | 逐格 `abs sqrt log exp has min max where`；當日橫斷面 `sum mean count quantile min max` | 第 4 章 |
| 例 | `fund.mktcap >= quantile(fund.mktcap, 0.7)`、`industry == 24 && fund.mktcap > 10e9` | `fund.mktcap >= GQUANT(70, 0, fund.mktcap)`、`industry in {24} && fund.mktcap > 10e9` |

**舊寫法怎麼翻。** 舊寫法不另外求值：編譯時翻成 VQL，跟 VQL 走同一條路算；`universe_lang`、記下來的條件文字、面板身分都不變。翻法：

- 每個比較、邏輯都包成「無效當假」（`FILL(…, 0:BOOL)`），重現舊寫法「跟缺值比較一律為假」。
- `mean(x)` ＝ `RATIO(GSUM(0, x), GCOUNT(0, x))`；`quantile(x, q)` ＝ `GQUANT(q × 100, 0, x)`，q × 100 要是 0 到 100 的整數；`sum`、`count`、單引數的 `min`、`max` ＝ 層級 0 的 `GSUM`、`GCOUNT`、`GMIN`、`GMAX`。
- **跨單位的算術先換成比例再算**（例 `eps * 4 * shares / equity`、`close * volume`）：舊寫法沒有單位，照欄的單位直翻過不了單位規則時，數值欄先換成 `RATIO(欄, 1:<該欄單位>)` 再算。
- **比較兩邊最外層的除法會移項**（`x / y > z`）：改成乘法再比，不在比較前取整——分母 > 0 比 `x` 與 `z·y`，分母 < 0 大小反過來，分母 0 當假。兩邊都是除法就各自移項。內層的除法照 VQL 向下取整到 1e-6。
- **界線**：比例是 1e-6 的定點。跨單位算術裡的值（含移項乘出來的積）超過約 9.2e12 的那一格無效、條件當假；總額欄（例 `fund.mktcap`）進了這種算術最容易碰到。
- 翻不成的回 `E_OLD_SYNTAX`，hint 給 VQL 的寫法：`sd`、`var`（樣本版；VQL 的 `GSTD` 是母體）、`pow`、百分位不是整數的 `quantile`、除以常數 0。

---

## 9. 回測（投組函數）

### 9.1 投組函數是什麼

分數公式只產生每天的排名。**怎麼從排名變成持股**由投組函數決定：`stock_backtest portfolio={…}`。全部是具名參數，全部可以省略。

- 頂層的 `rebalance_days`、`max_positions` 是 `portfolio` 裡同名參數的簡寫。
- 回測期間不在 `portfolio` 裡，是 `stock_backtest` 的頂層參數（9.10）。
- 一個投組一個市場。

### 9.2 參數總表

| 參數 | 值域 | 預設 | 意思 | 例子 |
|---|---|---|---|---|
| `rebalance_days` | 1..252 | 不指定：試 5 與 21，挑一組 | 每 N 個交易日全部重排換股 | `{"rebalance_days":5}` |
| `max_positions` | 1..500 | 不指定：試 20 與 50，挑一組 | 最多持有幾檔 | `{"max_positions":10}` |
| `weighting` | `equal`／`score`／`inv_vol` | `equal` | 等權／依排名（越前越重）／依波動倒數（越穩越重） | `{"weighting":"inv_vol","max_weight":15}` |
| `max_weight` | 0..100（%） | 0（不限） | 單檔上限；超過的部分放現金 | `{"max_positions":10,"max_weight":15}` |
| `hold_days` | 2..60 | 不用 | 每天選、每天買進的那一份持有 N 個交易日到期賣出（N 份錯開） | `{"hold_days":20}` |
| `hold` | BOOL 算式 | 不用 | 逐日逐檔的條件：排名照全體，不合的不買、到換股日賣 | `{"hold":"close > ROLL_MEAN(60, close)"}` |
| `rebalance_when` | BOOL 算式（每天全市場同值） | 不用 | 成立那天換股 | `{"rebalance_when":"MDAY() < LAG(1, MDAY())"}` |
| `exposure` | RATIO 算式 0～1（每天全市場同值） | 1（滿倉） | 總曝險，其餘放現金 | `{"exposure":"idx.taiex < ROLL_MEAN(200, idx.taiex) ? 0.5:RATIO : 1:RATIO"}` |
| `weight` | 數值算式（逐日逐檔） | 不用 | 換股日每檔的權重照這個值的比例 | `{"max_positions":20,"weight":"fund.mktcap"}` |
| `tie_by` | 數值算式（逐日逐檔） | 不用（同分照固定的打散順序） | 挑前 N 檔時分數相同的照這個值排，大的先選；沒有值的排在同分的最後；值也相同才照 `mix64((entity_id<<32)^日)`、最後檔號。只決定同分的先後，不改分數與權重、不抬高暖機；要小的先寫 `NEG(…)` | `{"tie_by":"fund.mktcap"}` |
| `group_by` | 算式（逐日逐檔；數值或條件，不能用 `pos.*`） | 不用 | 分群的依據：同一天值相同的股票算同一群；跟 `group_cap` 一起給（9.5a） | `{"group_by":"CORR_GROUP(RET(1, close))","group_cap":3}` |
| `group_cap` | 正整數 | 不用 | 同一群最多持有幾檔：換股日照名次往下選，同群滿了就跳過、往下補（9.5a） | 同上 |
| `add_when` | BOOL 算式，可用 `pos.*` 與 `pos.size`、`pos.adds`、`pos.cuts`、`pos.avg_ret` | 不用 | 加碼條件：持有中的檔從不成立變成成立的那個收盤，下一個成交時點加碼一次（9.5b） | `{"add_when":"pos.ret >= 0.1","size_init":0.5}` |
| `cut_when` | 同 `add_when` | 不用 | 減碼條件：成立的那一下賣掉現有部位的 `cut_size`（9.5b） | `{"cut_when":"pos.ret >= 0.2"}` |
| `add_size` | > 0 | 1 | 一次加幾份（進場那一份的倍數） | `{"add_when":"pos.ret >= 0.1","add_size":0.5}` |
| `cut_size` | 0 < x < 1 | 0.5 | 一次賣掉現有部位的幾成；全部賣掉寫 `exit_when` | `{"cut_when":"pos.ret >= 0.2","cut_size":0.3}` |
| `size_init` | 0.01..1 | 1 | 進場先買目標權重的幾成（留現金給加碼） | `{"add_when":"pos.ret >= 0.1","size_init":0.5}` |
| `max_size` | ≥ 1 | 不限 | 一檔最多加到進場那一份的幾倍 | `{"add_when":"pos.ret >= 0.1 * (pos.adds + 1)","max_size":3}` |
| `gross_max` | 0.01..2 | 1 | 總部位上限：各檔權重和最多帳戶的幾成；> 1＝融資（要給 `margin_rate`）（9.5b） | `{"gross_max":0.8}` |
| `cash_min` | 0..0.99 | 0 | 現金下限；跟 `gross_max` 取較嚴的 | `{"cash_min":0.1}` |
| `add_mode` | `cash`／`scale` | `cash` | 加碼超過總部位上限時：只用剩下的額度／照比例縮其他持股（算式寫法見 9.12：加碼項包 `MIN2(…, acct.room)`／乘 `MIN2(1, g / acct.gross_req)`） | `{"add_mode":"scale"}` |
| `max_new` | ≥ 1 | 不限 | 一個換股日最多新買幾檔（只留排名最前的，其餘那幾格留現金） | `{"max_new":3}` |
| `cash_rate` | 年利率，例 0.015 | 0 | 現金年利率（只做多） | `{"cash_rate":0.015}` |
| `margin_rate` | 年利率，例 0.05 | 不用 | 融資年利率（`gross_max` > 1 時必給） | `{"gross_max":1.5,"margin_rate":0.05}` |
| `exit_when` | BOOL 算式，可用 `pos.*` | 不用 | 出場條件：每天收盤看，成立就賣，不等換股日 | `{"exit_when":"pos.ret <= -0.08 \|\| pos.ret >= 0.2"}` |
| `keep_when` | BOOL 算式，可用 `pos.*` 與 `pos.rank` | 不用 | 續抱條件：換股日成立的持股先留下 | `{"max_positions":10,"keep_when":"pos.rank <= 30"}` |
| `reentry_days` | 0..10,000 | 0 | 照 `exit_when` 出場後幾個交易日內的換股不買回同一檔 | `{"exit_when":"pos.ret <= -0.08","reentry_days":20}` |
| `exit_below` | PRICE 算式，可用 `pos.entry`、`pos.peak` | 不用 | 盤中停損價位：當天最低價碰到就賣，不等收盤（9.6a） | `{"exit_below":"pos.entry * 0.92"}` |
| `exit_above` | PRICE 算式，可用 `pos.entry`、`pos.peak` | 不用 | 盤中停利價位：當天最高價碰到就賣（9.6a） | `{"exit_above":"pos.entry * 1.2"}` |
| `lock` | `limit`／`none` | 不給：面板有交易所漲跌停價就判 | 規則出場遇漲跌停鎖死：`limit` 賣不掉順延到下一個交易日、`none` 不模擬（9.6a） | `{"exit_below":"pos.entry * 0.92","lock":"none"}` |
| `grid` | 物件：名字 → 要試的值 | 不用 | 要試的數字：算式裡寫成 `$名字`，這裡給每個名字的值（9.11）；也用來改搜尋的基本格：`pl`（做多門檻）、`reb`（換股天數）、`n`（檔數）、`wk`（權重鍵）（9.11a） | `{"exit_when":"pos.drawdown >= $dd","grid":{"dd":[0.06,0.1,0.14]}}`、`{"grid":{"reb":[5,10,21]}}` |
| 做多門檻 `apx.IN_TOP(p)` | 寫在 `hold`；p 是 0..200 的整數或 `$名字` | 不寫：試 160／170／180／190 | 只買分數名次百分位 ≥ p／200 的股票（180＝前 10%、190＝前 5%）；＝ `GE(CS_PCT(apx.SCORE()), p / 200)` | `{"hold":"apx.IN_TOP(190)"}`、`{"hold":"apx.IN_TOP($pl)","grid":{"pl":[150,170,190]}}` |
| `grid_by` | 篩選語言的式子（排序算式，11.3） | 有 `grid`：`apx.SHARPE`；沒有：合成分數 | 同一個公式的各組照它排（大的優先）；沒有 `grid` 時排各組投組參數 | `{"exit_when":"pos.drawdown >= $dd","grid":{"dd":[0.06,0.1,0.14]},"grid_by":"apx.CALMAR"}` |
| `initial_capital` | 1..1e12（元） | 1,000,000 | 初始資金；帶了就算逐股帳戶 | `{"initial_capital":4000000}` |
| `lot` | `odd`／`board` | `odd` | 零股（1 股）／只買整張（1,000 股） | `{"lot":"board"}` |
| `min_fee` | 0..10,000（元） | 20 | 每筆最低手續費 | `{"min_fee":20}` |
| `contribution` | 0..1e12（元／期） | 0 | 定期投入，下一次換股時買進 | `{"initial_capital":5000,"contribution":5000,"contribution_every":"month","lot":"odd"}` |
| `contribution_every` | `month`／`rebalance` | `month` | 每月第一個交易日／每次換股 | 同上 |

互斥與併用：

| 規則 | 說明 |
|---|---|
| `rebalance_days`、`hold_days`、`rebalance_when` 三選一 | 決定什麼時候換股 |
| `weight` 與 `weighting` 二選一 | 決定權重 |
| `exit_when`、`keep_when` 可以和 `hold_days` 一起用 | 每一份各看自己的買進價、各自出場，賣掉的那一塊留現金到那一份到期；`keep_when` 在每一份到期那天看，成立的在那一份再抱一期；`reentry_days` 只擋同一份（要整體不買回，改用 `rebalance_days` 加 `exit_when="pos.days >= N \|\| …"`）。「持有滿 N 天就賣」的時間停損仍寫 `exit_when="pos.days >= N"` |
| `weight` 可以和 `hold_days` 併用 | 每天買進的那一份照當天的值分配，持有期間不再調 |
| `tie_by` 跟其他參數都能併用 | 它只改挑前 N 檔時同分的先後（精確名次），持股上限、權重、出場規則照舊 |
| `group_by` 與 `group_cap` 要一起給 | 只給一個時回缺哪一個與可照做的例子 |

收算式的參數（`hold`、`rebalance_when`、`exposure`、`weight`、`tie_by`、`exit_when`、`keep_when`、`exit_below`、`exit_above`）的值可以是算式字串，或 `{"expr": "…", "label": "白話一句"}`；`label` 是寫給使用者看的話，卡片只放它。

### 9.3 什麼時候換股（三選一）

| 參數 | 行為 |
|---|---|
| `rebalance_days=N` | 暖機結束後每 N 個交易日換一次 |
| `rebalance_when=<BOOL>` | 條件成立那天換。算式每天要對全市場同一個值（只用日曆、市場層欄、`MKT_MEAN`、`EW_INDEX` 這類）；同一天有兩個以上的值 → `E_NOT_MARKET_WIDE`；整段期間沒有一天成立 → `E_NO_REBALANCE` |
| `hold_days=N` | 每天選股、每天買一份，每份抱滿 N 個交易日到期賣出；N 份錯開 |

`rebalance_when` 常見寫法：

| 白話 | 寫法 |
|---|---|
| 每月第一個交易日 | `MDAY() < LAG(1, MDAY())` 或 `MONTH() != LAG(1, MONTH())` |
| 每季（1、4、7、10 月第一個交易日） | `MDAY() < LAG(1, MDAY()) && MONTH() in {1, 4, 7, 10}` |
| 只在財報月的第一個交易日 | `MONTH() != LAG(1, MONTH()) && MONTH() in {3, 5, 8, 11}` |
| 每週五 | `WEEKDAY() == 4` |
| 每週第一個交易日 | `WEEK_START()` |
| 每年第一個交易日 | `YEAR() > LAG(1, YEAR())` |
| 等權大盤站上 60 日均線那天 | `EW_INDEX(close) > ROLL_MEAN(60, EW_INDEX(close)) && !LAG(1, EW_INDEX(close) > ROLL_MEAN(60, EW_INDEX(close)))` |

`hold_days` 與時間停損不同：`hold_days=20` 是「每天買一份、分 20 份輪流」的另一種換股方式；「照常換股、持有滿 20 天的那檔先賣」是 `exit_when="pos.days >= 20"`。

### 9.4 `hold`、`exposure`、`weight`

**`hold`**（逐日逐檔 BOOL）：排名、因子指標仍照原來的選股範圍算；這支策略只買、只持有條件為真的股票。無效的格當不合。

**`exposure`**（每天全市場同一個 RATIO）：

- 取值＝⌊值 × 100⌋ 夾在 0..100：一個百分點一格，每天可以變。
- 變動只收差額的買賣成本，其餘放現金。只有曝險變動的日子不選股、不算換股次數。
- 當天沒有任何一檔有效 → 沿用前一天。整段都是 100 → 當成沒填。

| 白話 | 寫法 |
|---|---|
| 加權指數跌破 200 日均線只投入五成 | `idx.taiex < ROLL_MEAN(200, idx.taiex) ? 0.5:RATIO : 1:RATIO` |
| 等權大盤跌破年線降到五成 | `EW_INDEX(close) < ROLL_MEAN(250, EW_INDEX(close)) ? 0.5 : 1` |
| 等權大盤從一年高點回落超過 15% 只持有三成 | `RATIO(EW_INDEX(close), ROLL_MAX(250, EW_INDEX(close))) < 0.85 ? 0.3 : 1` |
| 全市場單日平均跌超過 2% 降到三成 | `MKT_MEAN(RATIO(close, LAG(1, close)) - 1) < -0.02 ? 0.3 : 1` |
| 每月 13 號剛好是星期五就出清 | `MDAY() == 13 && WEEKDAY() == 4 ? 0:RATIO : 1:RATIO` |
| 依全市場 20 日波動反比調整，最多滿倉 | `MIN2(RATIO(0.01:RATIO, ROLL_STD(20, FILL(RATIO(GSUM(0, LOGR(close, LAG(1, close))), GSUM(0, MASK(VALID(LOGR(close, LAG(1, close))), 1:RATIO))), 0:RATIO))), 1:RATIO)` |

**`weight`**（逐日逐檔的數值）：

- 換股日每檔的權重＝它的值 ÷ 入選各檔的值的總和，一個基點（萬分之一）一格。
- 值不是正數或沒有值的檔不買。比例不到萬分之一的檔也不買，那一份留現金。
- `max_weight` 照樣夾，夾掉的放現金。
- 整段期間沒有一格是正數 → `E_NO_WEIGHT`。

| 白話 | 寫法 |
|---|---|
| 依市值 | `fund.mktcap` |
| 依市值開根號 | `SQRT(fund.mktcap)` |
| 依近 20 日成交金額 | `ROLL_MEAN(20, px.turnover)` |
| 依均量 | `ROLL_MEAN(20, volume)` |
| 依分數 | `CS_PCT(RET(60, close))` |
| 跟 2330 相關越高買越多 | `CORR_SEED({"2330"}, RATIO(close, LAG(1, close)))` |

價 × 量（`MUL(close, volume)`）單位不合；成交金額直接用欄 `px.turnover`。

**`weighting`**（不寫 `weight` 時）：

| 值 | 權重 |
|---|---|
| `equal` | 每檔 1 ÷ 檔數；有 `max_weight` 而且檔數 × 上限不到 100% 時，每檔＝上限，其餘現金 |
| `score` | 依排名：越前越重；權重以萬分之一為單位取整，餘數給排名最高的那檔 |
| `inv_vol` | 依波動倒數：越穩越重；波動是 0 的檔不進候選。波動＝`INV_VOL_KEY(20, RET(1, close))` 那一個（近 20 日報酬，4.6）；要換視窗或序列，權重鍵直接寫 `INV_VOL_KEY(N, x)` |

### 9.5 出場：`exit_when`、`keep_when`、`reentry_days`

這兩條算式裡多六個只在這裡可用的狀態葉：

| 葉 | 單位 | 定義 | 能用在 |
|---|---|---|---|
| `pos.ret` | RATIO | 進場後的報酬（+0.2＝漲 20%）。進場價＝成交那一刻的價；用還原報酬累計，不含成本 | 兩者 |
| `pos.peak_ret` | RATIO | 進場以來每個收盤的 `pos.ret` 的最高值（≥ 0，含進場那一刻的 0） | 兩者 |
| `pos.drawdown` | RATIO | 從進場以來最高收盤價值回落的比例：1 − 現值 ÷ 最高值（0.1＝回落 10%） | 兩者 |
| `pos.days` | DAYS | 進場後過了幾個收盤（隔天開盤成交時，進場當天的收盤＝1） | 兩者 |
| `pos.rank` | COUNT | 今天在候選裡排第幾名（1＝最前） | 只有 `keep_when` |
| `pos.held` | BOOL | 恆真 | 兩者 |

寫法規則：

| 規則 | 說明 |
|---|---|
| `pos.*` 要放在比較裡 | `>`、`>=`、`<`、`<=`（或 `GT`、`LT`） |
| 比較的一邊是狀態葉的一次式 | 可以乘常數、加減常數；係數絕對值 ≤ 100 |
| 另一邊是常數，或整條只看市場的算式 | 例 `-2 * ROLL_STD(20, RET(1, close))` |
| 幾個比較用 `&&`、`\|\|`、`!` 連起來 | 相同的比較只算一個 |
| 不含 `pos.*` 的子條件 | 整棵當一個市場條件，照一般條件編（例 `close < ROLL_MEAN(20, close)`） |

行為：

- **`exit_when`**：持有中的每一檔每天收盤看一次；成立就在下一個成交時點賣出，不等換股日。賣出的那一份留現金到下一個換股日，其他檔的權重不動。
- **`keep_when`**：換股日，持有中而且條件成立的檔先留下，剩下的名額才照排名補。不合 `hold` 或不在選股範圍的檔照樣賣。
- **`reentry_days`**：照 `exit_when` 出場後 N 個交易日內的換股不買回同一檔。0＝只擋出場當天的換股。
- 有比較讀到無效的格時，這條規則對那一檔不成立。
- 報告的 `summary.rule_exits_by_rule` 是每一條（用 `||` 接的每一段）各出場幾檔次。
- 出場規則是整次搜尋的設定：隨機生成、指定公式、自訂函數格點三種來源的每一支策略都照做。

### 9.5a 群上限：`group_by`、`group_cap`

「同一群最多持有幾檔」。兩個參數要一起給：`group_by` 是分群的依據（逐日逐檔的算式；**同一天值相同的股票算同一群**），`group_cap` 是每一群最多幾檔（正整數）。

- 換股日照精確名次由前往後一檔一檔看：目標已經有 `max_positions` 檔就停；那一檔的群在目標裡已經有 `group_cap` 檔就跳過、往下補。
- **持有中的也算**：`keep_when` 成立的持股先照名次看，同群超過的，名次後面的照常賣出。只有跌停鎖死賣不掉的檔會讓一群暫時超過（它還在手上、佔那一群的名額）。
- 群鍵沒有值的股票（例 `CORR_GROUP` 分不到群的、暖機期）自己一群、不受限。
- 群是照**換股日當天**的值分的；兩次換股之間值變了不會觸發賣出，下一個換股日才重看。
- 之後的換手上限、權重取整為 0 的拿掉、鎖死買不到都照舊，而且不再補人，所以結果一定守得住上限。
- `hold_days = N`：每一份各自算（整體同一群最多 N × `group_cap` 檔）。
- `group_by` 可以是任何算式：自動分群 `CORR_GROUP(…)`（第 6 章）、分桶 `CS_QCUT(n, x)`、代碼欄、一個條件（成立的一群、不成立的一群）；不能用 `pos.*`。

| 白話 | `portfolio` |
|---|---|
| 表現相近的同一群最多買 3 檔 | `{"max_positions":10,"group_by":"CORR_GROUP(RET(1, close))","group_cap":3}` |
| 市值切五等分、每一等分最多 2 檔 | `{"max_positions":10,"group_by":"CS_QCUT(5, fund.mktcap)","group_cap":2}` |
| 成交金額切五等分、每一等分最多 2 檔 | `{"max_positions":10,"group_by":"CS_QCUT(5, ROLL_MEAN(20, px.turnover))","group_cap":2}` |
| 同一群最多 2 檔，而且掉出前 30 名才賣 | `{"max_positions":10,"keep_when":"pos.rank <= 30","group_by":"CORR_GROUP(RET(1, close))","group_cap":2}` |

plan 回 `effective.portfolio.group_by.stats`：`cap`、選股範圍內每天的群數 `groups_avg`／`groups_last`、最大一群的檔數 `largest_group_avg`／`largest_group_last`、沒有值的檔數 `no_key_avg`／`no_key_last`。報告的 `program.group_by`、`program.group_cap` 記這支策略用的分群與上限。

### 9.5b 加減碼與帳戶層

**加減碼**：`add_when`／`cut_when` 是條件從不成立變成成立的那個收盤觸發一次（一路成立只做一次），在下一個成交時點加碼或減碼；之後條件回頭不還原。另一種寫法是讓 `weight` 讀 `pos.*`（例 `"weight": "fund.mktcap * (pos.peak_ret >= 0.1 ? 2 : 1)"`），照狀態定目標權重；兩種擇一。

新葉（只在 `add_when`、`cut_when` 與出場規則裡用）：`pos.size`（現在的部位 ÷ 進場那一份）、`pos.adds`、`pos.cuts`（加碼、減碼成交次數）、`pos.avg_ret`（對加碼後均價的報酬）；盤中觸價的價位另可用 `pos.avg`（均價，沒加過碼＝進場價），例 `"exit_below": "pos.avg * 0.92"`。

**帳戶層**：`gross_max`（總部位上限，> 1＝融資）、`cash_min`（現金下限）、`add_mode`（加碼超過上限時只用剩下的額度或縮其他持股）、`max_new`（一個換股日最多新買幾檔）、`cash_rate`／`margin_rate`（現金與融資利率）。

| 白話 | `portfolio` |
|---|---|
| 漲 10% 加碼一倍（先買半份留額度；題庫 B12） | `{"add_when":"pos.ret >= 0.1","size_init":0.5}` |
| 每再漲 10% 加一次，最多 3 倍 | `{"add_when":"pos.ret >= 0.1 * (pos.adds + 1)","max_size":3}` |
| 跌 5% 補一次（只補一次） | `{"add_when":"pos.ret <= -0.05 && pos.adds == 0","size_init":0.5}` |
| 漲 20% 先賣一半、剩下的回落 10% 全出 | `{"cut_when":"pos.ret >= 0.2","exit_when":"pos.drawdown >= 0.1"}` |
| 總部位最多八成、每次換股最多新買 3 檔 | `{"gross_max":0.8,"max_new":3}` |

換股後依序：跌停鎖死買不到的不進場 → 成交日沒有價格（那天的 K 不完整）的新進檔不成交、那一份留現金 → 一個換股日新進場超過 `max_new` 的只留名次最前的 → 加減碼縮到 0 的不進場。

報告的 `size_events` 是加減碼筆數與最近幾筆。按需讀的說明：`stock_formula_help topic=portfolio name=sizing`（加減碼）、`name=account`（帳戶層）。

### 9.6 常見出場寫法

| 白話 | `portfolio` |
|---|---|
| 固定停損 8% | `{"exit_when":"pos.ret <= -0.08"}` |
| 固定停利 20% | `{"exit_when":"pos.ret >= 0.2"}` |
| 停損停利一起 | `{"exit_when":"pos.ret <= -0.08 \|\| pos.ret >= 0.2"}` |
| 移動停損：從進場後最高點回落 10% | `{"exit_when":"pos.drawdown >= 0.1"}` |
| 移動停利（保本）：賺過 10% 之後跌回進場價 | `{"exit_when":"pos.peak_ret >= 0.1 && pos.ret <= 0"}` |
| 時間停損：持有滿 20 個交易日 | `{"exit_when":"pos.days >= 20"}` |
| 跌破 20 日均線 | `{"exit_when":"close < ROLL_MEAN(20, close)"}` |
| 波動停損：跌超過自己 20 日日報酬標準差的 2 倍 | `{"exit_when":"pos.ret < -2 * ROLL_STD(20, RET(1, close))"}` |
| 抱滿 60 天還賠錢才賣 | `{"exit_when":"pos.days >= 60 && pos.ret < 0"}` |
| 停損後 20 個交易日內不買回 | `{"exit_when":"pos.ret <= -0.08","reentry_days":20}` |
| 緩衝區：前 10 名才買、掉出前 30 名才賣 | `{"max_positions":10,"keep_when":"pos.rank <= 30"}` |
| 賺錢的續抱、賠錢的才換 | `{"keep_when":"pos.ret > 0"}` |

這些都是**收盤確認**：當天收盤看條件，下一個成交時點賣。盤中觸價的版本在 §9.6a。

### 9.6a 盤中觸價：`exit_below`、`exit_above`、`lock`

持有中的每一檔每天看一次：當天最低價 ≤ 停損價位（或最高價 ≥ 停利價位）就在當天盤中出場，不等收盤。價位在前一個收盤算好，不偷看。

| 新葉 | 單位 | 意思 |
|---|---|---|
| `pos.entry` | PRICE | 進場成交價（還原價口徑） |
| `pos.peak` | PRICE | 持有以來最有利的**收盤價**（含進場價） |

價位算式＝一項或幾項取極值（`exit_below` 用 `MAX2`、`exit_above` 用 `MIN2`）；每一項是 `常數 × pos.entry ＋ 常數 × pos.peak ＋ 只看市場的價位算式`。

| 白話 | 收盤確認（`exit_when`） | 盤中觸價 |
|---|---|---|
| 固定停損 8% | `exit_when="pos.ret <= -0.08"` | `exit_below="pos.entry * 0.92"` |
| 固定停利 20% | `exit_when="pos.ret >= 0.2"` | `exit_above="pos.entry * 1.2"` |
| 移動停損：最高收盤回落 10% | `exit_when="pos.drawdown >= 0.1"` | `exit_below="pos.peak * 0.9"` |
| 固定與移動停損取先到的 | — | `exit_below="MAX2(pos.entry * 0.92, pos.peak * 0.9)"` |
| 跌破 20 日均線 | `exit_when="close < ROLL_MEAN(20, close)"` | `exit_below="ROLL_MEAN(20, close)"` |
| 波動停損 | `exit_when="pos.ret < -2 * ROLL_STD(20, RET(1, close))"` | `exit_below="pos.peak - 3 * ROLL_MEAN(14, high - low)"` |

四條成交規則：

1. 開盤跳空過頭用開盤價，否則用觸價價位。
2. 同一天停損、停利都碰到，當作先碰停損。
3. 當天高點不墊高當天的移動停損（價位只用到前一個收盤為止的狀態）。
4. 跌停鎖死賣不掉就順延到隔天。

客戶只說「跌 8% 停損、漲 20% 停利」寫觸價那一種；說「收盤跌破才走」寫 `exit_when`。

`portfolio.lock`（選用）：`"limit"` 照跌停鎖死順延（空單看漲停）、`"none"` 不模擬；不給＝面板有交易所漲跌停價（`raw.high`、`raw.low`、`raw.limit_up`、`raw.limit_down`）就判，沒有就不判，plan 的 `portfolio.lock` 寫實際怎麼算（`limit`／`none`／`unavailable`／`no_limits`）。可以跟 `exit_when`／`keep_when`／`reentry_days` 一起給（`reentry_days` 也管觸價出場）；跟 `hold_days` 一起用時每一份各自觸價、各自出場。價位裡也能寫 `$名字`（門檻格點），跟 `exit_when` 共用一份 `grid`（9.11）。報告的 `rule_exits.items` 觸價那幾筆帶 `touch`（`below`／`above`）與 `fill_vs_prev_close_1e6`（成交價 ÷ 前收）、`touch` 區塊有兩邊筆數與 `lock_unknown`。

### 9.7 成交時點與成本

| 項 | 預設 | 說明 |
|---|---|---|
| 成交時點 | 換股日收盤後選股，**隔天開盤**成交 | 決策日 t 照 t 的排名選股；t+1 的隔夜報酬（開盤 ÷ 前收 − 1）歸舊持股，換股後 t+1 的盤中報酬（收盤 ÷ 開盤 − 1）歸新持股。某格沒有開盤價時那一檔照前一天收盤成交 |
| 買進成本 | 手續費 0.1425% | |
| 賣出成本 | 手續費 0.1425% ＋ 證交稅 0.3% | ETF 的證交稅 0.1% |
| 最低手續費 | 每筆 20 元（`min_fee`） | 只在逐股帳戶裡算 |
| 換手上限 | 不限 | `apx.PICK` 第四個引數（9.12；`0.3`＝新買的檔數最多是持有檔數的三成），或 `config` 的 `max_turnover_bp`（`3000`＝同一件事） |

- 成本與交易規則照面板所屬的市場表（台股是上面的值；其他市場用那個市場的）。
- 可用 `stock_backtest` 的 `config` 覆寫：`fee_bp_x100`（單位 0.01 基點，預設 1425）、`tax_sell_bp_x100`（預設 3000）、`tax_sell_etf_bp_x100`（預設 1000）；`cost_bp` ≥ 0＝舊制，買賣各 N 基點、不另計稅。
- 策略的權益是加法累計（不複利）；報告的年化、最大跌幅用複利口徑（第 11 章）。
- 決策日落在面板最後一天的那次換股不做（隔天還沒到）。

### 9.8 選股、名次與同分

換股日的做法（多方）：

1. 候選＝當天在選股範圍內、分數有效、過 `hold` 的股票。
2. 候選照分數排成當天的**精確名次**：每檔一個不重複的位置；成員少於 2 檔的日子沒有名次。
3. 目標＝候選裡名次最高的 `max_positions` 檔。
4. 進場＝目標 − 現有持股；出場＝現有持股 − 目標。

同分：每天每檔一個不重複的位置，次序是 分數大 →（有給 `tie_by` 時）`tie_by` 的值大、沒有值的排在同分的最後 → 固定的打散順序 `mix64((entity_id << 32) ^ 日)` 大 → 檔號小。選股結果不因代號或面板欄序改變；同一份資料重跑，同分的先後一樣。`tie_by` 只決定同分的先後，不改分數、持股上限與權重，也不抬高暖機；要小的先選寫 `NEG(…)`。分數大量同分時（桶號、群內百分位當整條分數），入選的是誰就由 `tie_by` 或打散順序決定：分數用連續值（報酬、比值），或給 `tie_by`。`CS_PCT`、`pos.rank` 這些讀名次的地方不受 `tie_by` 影響（百分位只看比它小、跟它相同的檔數）。

### 9.9 資金與逐股帳戶

帶了 `initial_capital`、`lot`、`min_fee`、`contribution` 任一個，報告加逐股帳戶：每檔股數、整張／零股、手續費、吃到最低手續費的筆數、期末市值。

| 白話 | `portfolio` |
|---|---|
| 400 萬分 10 檔、零股、最低手續費 20 元 | `{"max_positions":10,"initial_capital":4000000,"lot":"odd","min_fee":20}` |
| 每月定期投入 5,000 元 | `{"initial_capital":5000,"contribution":5000,"contribution_every":"month","lot":"odd"}` |
| 每 5 個交易日換一次、最多 10 檔、400 萬 | `{"rebalance_days":5,"max_positions":10,"initial_capital":4000000}` |

每筆手續費＝max(`min_fee`, ⌊成交金額 × 0.1425%⌋)。

### 9.10 回測期間

`stock_backtest` 的頂層參數：

| 參數 | 值 | 意思 |
|---|---|---|
| `backtest_from` | `YYYY-MM-DD`、`YYYY-MM`（該月第一天）、`YYYY`（1 月 1 日） | 起日；不給＝面板第一天 |
| `backtest_to` | 同上（`YYYY-MM`＝月底、`YYYY`＝12 月 31 日） | 迄日；不給＝最新收盤 |
| `backtest_years` | 1..100 | 取代 `backtest_from`：到迄日為止的最近 N 年 |

- 起日之前的資料只用來暖機；篩選、IC、檢驗、報告都從起日算。
- 一個結果庫綁一段期間。已存的策略換期間看：`my_stock_strategies_change action=report backtest_from=…`（不重搜）。

### 9.11 讓系統試門檻：`$名字`＋`grid`

`exit_when`／`keep_when` 與盤中價位 `exit_below`／`exit_above` 裡要讓系統試的數字寫成 `$名字`，`grid` 給每個名字要試的值（四個參數共用一份 `grid`）。每一組代進去，就等於直接寫那一組門檻。

| 白話 | `portfolio` | 組數 |
|---|---|---|
| 停損停利幫我試幾組 | `{"exit_when":"pos.ret <= -$stop \|\| pos.ret >= $take","grid":{"stop":[0.05,0.08,0.1],"take":[0.15,0.2,0.3]}}` | 9 |
| 從高點回落幾 % 出場，讓系統找 | `{"exit_when":"pos.drawdown >= $dd","grid":{"dd":{"from":0.06,"to":0.14,"step":0.02}}}` | 5 |
| 盤中停損的幅度讓系統試 | `{"exit_below":"pos.entry * $floor","grid":{"floor":[0.9,0.92,0.95]}}` | 3 |
| 幾倍波動停損、最多抱幾天一起試 | `{"exit_when":"pos.ret < -$k * ROLL_STD(20, RET(1, close)) \|\| pos.days >= $n","grid":{"k":[1.5,2,3],"n":[20,40]}}` | 6 |

- 值的給法：陣列 `[0.05, 0.08, 0.1]`，或 `{"from": 0.04, "to": 0.1, "step": 0.02}`。同一個 `$名字` 可以出現多次。
- 上限：基本格組數 × 門檻組數 ≤ 32,767（試驗帳本的格數）；超過回錯，減少值的個數或分幾次搜尋。
- 值不能改變條件要看的天數：均線天數這類視窗長度不收，要分開搜尋。
- 每一組都跑。同公式、同投組參數的各組比 `grid_by`（排序算式，預設 `apx.SHARPE`；11.3），留一組；它用的那組寫在報告的 `program.exit_when`，各組的數字在工作摘要的 `exit_grid`。
- 每一組算一次多重檢定的試驗（10.3）。
- 只要「在原值附近找」就把門檻包起來：`{"exit_when":"pos.ret <= -apx.EVOLVE(0.08)"}` ＝ `{"exit_when":"pos.ret <= -$evolve1","grid":{"evolve1":[0.04,0.06,0.07,0.08,0.09,0.1,0.12,0.16]}}`（原值 × 1/2…2，同 10.1）；`apx.PICK(apx.EVOLVE(10), false)`、`apx.EVERY(apx.EVOLVE(21))` 這類整數照天數格點。只收包一個數字；包了算子的參數（要看幾天，例 `ROLL_MEAN(apx.EVOLVE(20), close)`）回 `E_EVOLVE_WINDOW`，改法給 `$名字`＋`grid` 的寫法。

### 9.11a 搜尋的基本格：`pl`、`reb`、`n`、`wk`

搜尋時每一棵樹都試一組「基本格」：做多門檻 × 換股天數 × 檔數（× 權重鍵）。沒寫時試做多門檻 160／170／180／190、換股 5／21 天、檔數 20／50（16 組）。每一軸都是投組算式裡的一個 `$名字`，值寫在同一份 `grid`；只給 `grid` 不寫算式也行。

| 白話 | `portfolio` | 基本格 |
|---|---|---|
| 換股天數試 5、10、21 天 | `{"grid":{"reb":[5,10,21]}}` | 4 × 3 × 2 ＝ 24 |
| 只買排名前 10%（做多門檻 180） | `{"hold":"apx.IN_TOP(180)"}` | 1 × 2 × 2 ＝ 4 |
| 前 25%～前 2.5% 都試 | `{"hold":"apx.IN_TOP($pl)","grid":{"pl":[150,160,170,180,190,195]}}` | 6 × 2 × 2 ＝ 24 |
| 持股 10、20、30 檔 | `{"weight":"apx.SHARE(apx.PICK($n, false), 1)","grid":{"n":[10,20,30]}}` | 4 × 2 × 3 ＝ 24 |
| 等權與依波動倒數都試 | `{"weight":"apx.SHARE(apx.PICK(20, false), $wk == 0 ? 1 : apx.INV_VOL_KEY())","grid":{"wk":[0,1]}}` | 4 × 2 × 1 × 2 ＝ 16 |
| 依名次加權、門檻跟著試 | `{"hold":"apx.IN_TOP($pl)","weight":"apx.SHARE(apx.PICK(20, false), apx.RANK_KEY($pl))","grid":{"pl":[170,190]}}` | 2 × 2 × 1 ＝ 4 |

- 四個位置：`hold` 的 `apx.IN_TOP($pl)`、`rebalance_when` 的 `apx.EVERY($reb)`、`apx.PICK($n, …)` 的檔數、權重鍵 `$wk == 0 ? 1 : apx.INV_VOL_KEY()`。名字可以自己取；只給 `grid` 時用 `pl`、`reb`、`n`、`wk`。
- 只給一個值＝直接寫那個數字（`{"grid":{"reb":[10]}}` 跟 `{"rebalance_days":10}` 是同一件事、同一個結果庫）。
- 這幾個 `$名字` 只寫在上面的位置；出場條件等別處要試的數字另取名字。那一軸已經寫了具體值（例 `rebalance_days`）時，`grid` 不要再給同一軸。
- `apx.RANK_KEY(p)`（依名次的權重鍵，＝ `FLOOR(CS_PCT(apx.SCORE()) * 200) - p + 1`）的 p 跟 `apx.IN_TOP(p)` 同一個；不寫引數＝跟著做多門檻。
- `apx.IN_TOP`、`apx.RANK_KEY` 讀這支策略自己的分數，只寫在 `portfolio`；分數公式裡要名次百分位寫 `CS_PCT(x)`。
- 每一組基本格算一次多重檢定的試驗（10.3）；plan 的 `effective.portfolio` 列出各軸試的值（`rebalance_days`、`max_positions`、`in_top`）。

### 9.12 回測規則寫成算式：`weight`＋`rebalance_when`

上面的參數都是兩條算式的簡寫。也可以直接寫算式，plan 認回同一套規則（數字逐位元相同）：

- `weight`：每天每檔的目標權重，寫成幾項相乘。一定要有一項 `apx.SHARE(apx.PICK(N, 續抱), 權重鍵)`（挑哪幾檔、怎麼分）；其他項是曝險、出場、加減碼。
- `rebalance_when`：換股日。`apx.EVERY(21)`＝每 21 個交易日；也可以寫全市場同值的條件（9.3）。
- `hold` 照舊是候選條件，多兩種項：`GTOP(k, 群鍵, apx.SCORE())`（同一群最多 k 檔）、`!FILL(pos.exit_days <= n, false)`（出場後 n 個交易日內不買回；n＝0 只擋出場當天）。
- 同分鍵：分數公式外面包 `apx.THEN(分數, 鍵)`。

| 參數 | 算式 |
|---|---|
| `max_positions=10` | `weight="apx.SHARE(apx.PICK(10, false), 1)"`（`1`＝等權） |
| `weighting="score"`／`"inv_vol"` | 權重鍵寫 `apx.RANK_KEY()`／`apx.INV_VOL_KEY()` |
| `weight="SQRT(fund.mktcap)"` | `apx.SHARE(apx.PICK(10, false), SQRT(fund.mktcap))` |
| `max_weight=15` | `apx.CAP(0.15, apx.SHARE(…))` |
| `keep_when="pos.rank <= 30"` | `apx.PICK(10, FILL(pos.rank <= 30, false))` |
| `max_new=3` | `apx.PICK(10, false, 3)` |
| `config.max_turnover_bp=3000`（換手上限） | `apx.PICK(10, false, 0, 0.3)`（第四個引數：一次換股新買的檔數 ≤ ⌊0.3 × 現在持有的檔數⌋；第三個引數 `0`＝新買檔數不限） |
| `exposure="…"` | 乘一項 `(…)`（括號裡的全市場同值算式） |
| `gross_max=0.8`、`cash_min=0.1` | 乘 `0.8`、乘 `(1 - 0.1)` |
| `gross_max=1.5`（融資） | 乘 `MIN2(1, 1.5 / acct.gross_req)`（`acct.gross_req`＝換股時各檔目標權重的和；`margin_rate` 照舊給）；有加碼項時，加碼那一項包 `MIN2(…, acct.room)`＝超過上限只用剩下的額度（`add_mode="cash"`，預設） |
| `add_mode="scale"` | 乘 `MIN2(1, g / acct.gross_req)`（不融資 g 寫 `1`），加碼項不包 `acct.room`＝超過上限照比例縮全部持股 |
| `exit_when="條件"` | 乘 `!(pos.held && FILL(條件, false))` |
| `exit_below`、`exit_above` | 乘 `!(pos.held && FILL(TOUCH_BELOW(pos.entry * 0.92) \|\| TOUCH_ABOVE(pos.entry * 1.2), false))` |
| `add_when="A"`、`size_init=0.5` | 乘 `FILL(pos.size + 1 * (A && !FILL(LAG(1, A), false) && !apx.EVERY(21)), 0.5)`（`!apx.EVERY(21)`：換股日只記不加；`MIN2(上限, …)` 包起來＝`max_size`；融資時加碼項寫 `MIN2(1 * (…), acct.room)`） |
| `cut_when="C"` | 乘 `FILL(pos.size * (1 - 0.5 * (C && !FILL(LAG(1, C), false) && !apx.EVERY(21))), 1)` |
| `weight` 讀 `pos.*`（目標寫法） | 乘 `FILL(pos.peak_ret >= 0.1 ? 2 : 1, 1)` |
| `rebalance_days=21` | `rebalance_when="apx.EVERY(21)"` |
| `hold_days=10` | `weight="apx.STAGGER(10, apx.SHARE(…) * …)"` |
| `reentry_days=20` | `hold` 加 `!FILL(pos.exit_days <= 20, false)` |
| `group_by`＋`group_cap=3` | `hold` 加 `GTOP(3, CORR_GROUP(RET(1, close)), apx.SCORE())` |
| `tie_by="fund.mktcap"` | 公式寫 `apx.THEN(RET(60, close), fund.mktcap)` |

例：漲 20% 先賣一半、剩下的從高點回落 10% 全出、每月換股（題庫 B12c）

```json
{"rebalance_when": "apx.EVERY(21)",
 "weight": "apx.SHARE(apx.PICK(10, false), 1) * !(pos.held && FILL(pos.drawdown >= 0.1, false)) * FILL(pos.size * (1 - 0.5 * (pos.ret >= 0.2 && !FILL(LAG(1, pos.ret >= 0.2), false) && !apx.EVERY(21))), 1)"}
```

- **還沒持有的股票**：`pos.held`＝0，其他 `pos.*` 沒有值（沒出場過的 `pos.exit_days` 也沒有）。讀 `pos.*` 的項一律包 `FILL`，不然新股票永遠買不進來——plan 回 `E_NEVER_ENTER`，`fix` 是整條改好的寫法，照貼就過。三種最常見：`(pos.ret > 0 ? 2 : 1)` → `FILL((pos.ret > 0 ? 2 : 1), 1)`；`!(pos.held && pos.ret <= -0.08)` → `!(pos.held && FILL(pos.ret <= -0.08, false))`；`!(pos.exit_days <= 20)` → `!FILL(pos.exit_days <= 20, false)`。
- 同一件事寫兩次（例 `max_positions` 又寫 `apx.PICK`、`rebalance_days` 又寫 `apx.EVERY`）→ `APX_CONFLICT`，留一種。用了 `weight` 的算式，`max_positions`、`weighting`、`max_weight`、`exposure`、`gross_max`、`cash_min`、`keep_when`、`exit_when`、`exit_below`、`exit_above`、`max_new`、加減碼的參數、`add_mode`、`hold_days` 都寫在算式裡；換手上限寫在 `apx.PICK` 第四個引數，`config.max_turnover_bp` 就不要再寫。
- `acct.room`：這一檔這次加碼分到的剩餘總部位額度（倍數，跟 `pos.size` 同單位）＝ 總部位上限減掉今天加碼前的總部位，照今天各檔要加的量比例分下來、除以這一檔的目標權重；只用今天收盤以前的資料。
- 環境事實照舊寫參數：`initial_capital`、`contribution`、`contribution_every`、`lot`、`min_fee`、`lock`、`cash_rate`、`margin_rate`、`grid`、`grid_by`。
- 這一版收得下上表的形狀；其他寫法（例權重讀 `GSUM(…pos.*…)`、兩個讀 `pos.*` 的倍數）回 `APX_UNSUPPORTED`，訊息講還差哪一塊。

---

## 10. 搜尋

### 10.1 三種公式來源：`trees_from`

| 值 | 公式從哪來 | 另外要給 |
|---|---|---|
| `formulas` | 直接給公式文字 | `formulas=["…", …]` |
| `nouns` | 列舉自訂函數的參數格點 × 輸入綁定 | `nouns=[{"name":"user.me.NEAR_HIGH","bind":{"x":"close"},"grid":{"n":[60,120]}}]` |
| `sample` | 系統自己隨機生成（演算法搜尋） | `trees`（幾棵）、`seed`、`rounds` |

- `nouns`：`bind` 把形式輸入對到欄（不給就試每個同單位的欄）；`grid` 蓋過定義檔的格點；格點 × 綁定最多 20,000 組。
- `formulas`：每條在 plan 時就驗——編不過、根不是數值、暖機太長的都在 plan 回錯並指出是哪一條。
- **公式裡的數字讓系統試**：要試的數字寫成 `$名字`，值放 `grid`（跟 9.11 投組規則的 `$名字` 同一份 `grid`）。例「動能的回看天數試半年和一年、都跳過最近一個月」：`formulas=["RET_SKIP($n, 21, close)"]`、`portfolio={"grid":{"n":[126,252]}}` → 兩棵樹 `RET_SKIP(126, 21, close)`、`RET_SKIP(252, 21, close)`，跟直接寫那兩條是同一棵樹（同一個身分、同一筆結果）。不用先裝自訂函數；`nouns`（自訂函數＋`bind`＋`grid`）是同一件事的舊寫法，照收、展開成同一批樹。
- **同一份 `grid`、照位置分路**：分數公式裡的名字每組值是一棵樹；`hold`／`exposure`／`weight`／`exit_when`／`keep_when`／`exit_below`／`exit_above`／加減碼裡的名字每組值是同一棵樹的變體（9.11）。例「動能天數試 60、120，停損試 8%、10%」：`formulas=["RET($n, close)"]`、`portfolio={"exit_when":"pos.ret <= -$stop","grid":{"n":[60,120],"stop":[0.08,0.1]}}` → 兩棵樹、每棵兩組變體。同一個名字不能兩邊都寫（`E_GRID_SHARED`：兩邊各用自己的名字）。公式裡的名字：一個最多 256 個值、展開後最多 20,000 棵樹（同 `nouns`）；視窗長度也收（每組值是另一棵樹，暖機各自算）。plan 的 `effective.formula_grid` 寫用到的名字、寫了幾條、展開成幾條；組合順序是 `grid` 的第一個名字在最外層。
- **讓系統在原值附近找：`apx.EVOLVE(…)`**（策略優化）：把准系統改的那一個數字或欄包起來，系統從原值附近取候選，每個候選一棵樹；拿掉 `apx.EVOLVE` 就是起點（同一個身分）。
  - 包算子的參數：那個參數的格點上原值左右各兩格。例 `RET(apx.EVOLVE(60), close)` → 20、40、60、120、252 五棵，跟 `RET($n, close)`＋`grid {"n":[20,40,60,120,252]}` 是同一批樹、同一份結果。打包參數只動包的那一個（`RET_SKIP(252, apx.EVOLVE(21), close)` 只調 21）。
  - 包常數：原值 × 1/2、3/4、7/8、9/8、5/4、3/2、2（天數、次數單位的常數照天數格點）。BOOL 常數與 0 沒有鄰居。
  - 包欄：換成面板裡同單位、同類（每股值／總額、頻率、對齊相同）的每一欄。例 `CS_PCT(RET(20, apx.EVOLVE(close)))`。
  - 候選整條要編得過、根是數值、回看守得住才列；一個都沒有＝`E_EVOLVE_EMPTY`（改法：往外包一層）。寫 `EVOLVE(…)` 也收；只收一個引數（`E_EVOLVE_ARGS`）。
  - 一條可以包好幾個位置（各位置的候選相乘，原文第一個在最外層）；可以跟 `$名字` 一起用（`$名字` 先代入）。plan 的 `effective.formula_grid.evolve` 寫每個位置取了哪些值（`kind`：`param`／`const`／`leaf`）。
  - 包一段算式（例 `apx.EVOLVE(CS_PCT(fund.roe))`）回 `APX_UNSUPPORTED`；選股範圍裡不收（`APX_UNSUPPORTED`：選股範圍的門檻用 `$名字`＋`grid` 分開試）。投組規則裡的門檻見 9.11。
- 同一棵樹（展開後相同）只算一次。

每棵樹另外試一組投組設定（基本格 16 組：換股天數 5／21、最多持股 20／50），`screen_pick` 從中挑一組。`rebalance_days`、`max_positions` 指定了，那一軸就固定成指定值。

### 10.2 入庫門檻

門檻是「搜尋」的參數，不是公式的屬性：

| 來源 | 沒寫的門檻 |
|---|---|
| 隨機生成 | 套預設值 |
| 指定公式（`formulas`）、自訂函數格點（`nouns`） | 不套：每支照算、照記，不替使用者篩 |
| 明寫的門檻（`screen`、`screen_pick`、`screen_gate`、`screen_admit`、`config`） | 任何來源都照套 |

隨機生成的預設值：跨檔分數 ≥ 2,000 bp、時間分數 ≥ 2,500 bp、不重疊度 ≥ 5,000 bp、全期淨報酬 > 0、IC > 0、IC 的 t 值 ≥ 4（`config` 的 `min_ic_t`）。

### 10.3 多重檢定的試驗數

每試一組（樹 × 投組設定）記一筆進結果庫的試驗帳本，跨輪跨次累積＝「試了幾次」N。

| 量 | 定義 |
|---|---|
| N | 帳本裡的試驗數。同一棵樹、同一組投組參數重跑不重複計；沒有持股的試驗不計；門檻格點（9.11）的每一組各算一次 |
| N_eff | 有效試驗數：用試驗之間超額報酬的相關把 N 折算（樹內各組高度相關、樹間另估） |
| DSR | Deflated Sharpe Ratio：用 N_eff 算出「這麼多次試驗裡運氣最好的 Sharpe」當基準後的機率 |
| IC p_adj | IC 的單尾 p 值照 N_eff 做 Šidák 調整 |
| PBO | 一輪結束用 CSCV 算的回測過擬合機率 |

預設只記帳、寫進報告當參考，不擋入庫。要當硬門檻：把 `DSR` 放進 `checks`（第 11 章）。

---

## 11. 篩選指標與檢驗

### 11.1 篩選的語言

同一套運算子與優先序（另有 `%`），值是實數（double）。名字是**策略層的原子**：一支策略（一棵樹配一組投組設定）跑完之後的數字與逐日序列。

- 純量與序列可以混算（純量廣播）。除以 0＝缺值（`nan`）；跟缺值比較一律為假。
- 條件式的結果要是一個純量。

**序列原子**（逐日，從暖機後第一天起）：

| 名字 | 意思 | 例 |
|---|---|---|
| `r` | 策略的日淨報酬（成本已扣；加法累計） | `mean(r) / sd(r) * sqrt(252)` |
| `m` | 同範圍股票等權平均的日報酬 | `mean(r - m) * 252` |
| `year` | 每日的曆年（分組鍵） | `by_sum(year, r)` |
| `month` | 每日的 年 × 12 ＋ 月 − 1（分組鍵） | `mean(by_sum(month, r) > 0)` |
| `regime` | 多空頭：選股範圍過去 252 日累計 > 0 → 1、< 0 → −1、不明 0 | `filter(r, regime < 0)` |
| `half` | 前半 1／後半 2 | `filter(r, half == 2)` |
| `day` | 0 起的日序號 | `filter(r, day >= 252)` |
| `pos` | 持股數 | `MEAN(pos)` |
| `b` | 大盤基準的日報酬：面板的市場主指數報酬指數（含息；台股加權報酬指數、美股 S&P 500 Total Return，欄 `mkt.idx_tr`）；面板沒有這欄 → 缺值 | `MEAN(r - b) * 252` |

**純量原子**（引擎算好的數）：

| 群 | 名字 |
|---|---|
| 分數 | `cs_bp`（跨檔分數）、`ts_bp`（時間分數）、`novel_bp`（不重疊度）、`short_bp`（短度）、`total_bp`（合成分數＝`apx.TOTAL`）、`n_nodes`（樹的節點數） |
| 策略摘要 | `net_ret_1e6`（全期淨報酬）、`max_dd_1e6`（最大回撤）、`n_rebalances`、`avg_positions_x100`、`breadth_min`（最少可選檔數：各換股日可選檔數的最小值，沒有有效排名的換股日不算） |
| 投組參數 | `reb_days`、`p_long`、`p_short`、`max_positions`、`cost_bp`、`weight_mode` |
| 面板 | `n_days`、`day_first`、`day_mid`、`n_symbols` |
| 因子指標 | `ic_x1e4`（Rank IC 均值）、`ic_t_x100`（IC 的 t 值）、`ic_pos_x1e4`、`ic_days`、`spread_1e6`（前 10% − 後 10% 年化價差）、`spread_days`、`quint_1e6_1`…`quint_1e6_5`（五分位年化報酬）、`quint_days`、`quint_rho_x1e4`、`quint_mono_x1e4`、`turnover_bp_avg`、`turnover_days`、`breadth_med`、`breadth_days`、`ic_decay_x1e4_1`／`_5`／`_21`／`_63`、`ic_decay_days_1`／`_5`／`_21`／`_63`、`ic_half_life_days`、`ic_h1_x1e4`、`ic_h2_x1e4` |
| 曝險 | `size_rho`（排名與市值的秩相關）、`ind_hhi`、`ind_top`、`n_ind`、`adv_med`、`adv_min` |
| 風險（要風險模型） | `alpha_share_bp`（選股 α 占報酬變異的比例，可為負）、`breadth_eff_x100`（年化有效廣度 × 100） |
| 多重檢定（要 `multitest`） | `mt_dsr`（Deflated Sharpe，0..1）、`mt_ic_padj`（IC 的 Šidák 調整 p 值，0..1） |
| 一輪 | `round_pbo`（這一輪的 PBO，0..1；只在 `screen_gate`） |

名字尾巴標倍率：`_bp`＝萬分之一、`_1e6`＝百萬分之一、`_x100`、`_x1e4`。

**函式**：

| 類 | 函式 |
|---|---|
| 逐格 | `ABS` `SQRT` `LOG` `EXP` `FLOOR`（往負無限大捨去到整數）`POW(x, y)` `MIN2(x, y)` `MAX2(x, y)` `WHERE(cond, a, b)` `HAS(x)` |
| 縮減（序列 → 純量） | `SUM` `MEAN` `SD` `VAR`（樣本，n − 1）`COUNT`（元素數）`FIRST` `LAST` `MIN(x)` `MAX(x)`（略過缺值；`AMIN`／`AMAX` 同義）`QUANTILE(x, q)` `STREAK(x)`（最長連續為真）`LEN` |
| 序列 → 序列 | `CUM` `PEAK` `DRAWDOWN`（加法權益的回落）`NAV`（複利淨值）`DRAWDOWN_NAV`（複利回落）`DIFF` `FILTER(x, cond)` |
| 視窗（參數在前） | `ROLL_SUM(n, x)` `ROLL_MEAN(n, x)` `ROLL_STD(n, x)` `ROLL_MIN(n, x)` `ROLL_MAX(n, x)`（只算完整視窗，長度 N − n + 1）、`SKIP(k, x)`（去掉前 k 格）、`LAG(k, x)`（往回位移 k 格，前 k 格缺值，長度不變） |
| 分組 | `BY_SUM(key, x)` `BY_MEAN` `BY_SD` `BY_MIN` `BY_MAX` `BY_COUNT` `BY_PROD1`（複利）`BY_KEY(key)` |
| 日子 | `RANK_DESC(x)`（逐日由大到小的名次，1 起；平手取日子在前的；缺值不佔名次）、`BOOT(seed, block)`（區塊重抽樣的每日次數：日子切成長 block 的區塊、有放回抽 ⌈天數 ÷ block⌉ 次，每天的值＝它那一塊被抽到幾次；日子順序不變。種子＝seed、這一輪的 `val_alt_seed` 與面板，重算逐位相同） |
| 常數 | `nan` |

- 函數名不分大小寫；小寫舊寫法輸入在前（`roll_mean(r, 63)`）照收，整數常數寫在哪一邊都收。舊 `lag(x, k)` 就是 `SKIP(k, x)`。
- 條件成立幾天寫 `SUM(條件)`、比例寫 `MEAN(條件)`；`COUNT(x)` 是元素數，不看值。

### 11.2 `apx.*` 指標字典

官方用上面的原子寫好的定義式。自訂的照同一種寫法（11.4）。

| 名字 | 定義式 | 意思 |
|---|---|---|
| `apx.SHARPE_OF(x)` | `mean(x) / sd(x) * sqrt(252)` | 年化 Sharpe（無風險利率 0） |
| `apx.ANN_OF(x)` | `pow(last(nav(x)), 252 / count(x)) - 1` | 年化報酬（複利） |
| `apx.ANN_ADD_OF(x)` | `mean(x) * 252` | 年化報酬（加法） |
| `apx.VOL_OF(x)` | `sd(x) * sqrt(252)` | 年化波動 |
| `apx.COV_OF(x, y)` | `sum((x - mean(x)) * (y - mean(y))) / (count(x) - 1)` | 樣本共變異數 |
| `apx.CORR_OF(x, y)` | `apx.COV_OF(x, y) / (sd(x) * sd(y))` | 相關係數 |
| `apx.BETA_OF(x, y)` | `apx.COV_OF(x, y) / var(y)` | x 對 y 的 β |
| `apx.MAX_DD_OF(x)` | `max(drawdown_nav(x))` | 最大跌幅（收盤、複利） |
| `apx.MAX_DD_ADD_OF(x)` | `max(drawdown(x))` | 最大跌幅（加法權益） |
| `apx.TOTAL_RET` | `last(nav(r)) - 1` | 全期淨報酬（複利） |
| `apx.TOTAL_RET_ADD` | `sum(r)` | 全期淨報酬（加法；同 `net_ret_1e6` 的口徑） |
| `apx.ANN_RET` | `apx.ANN_OF(r)` | 年化報酬（複利） |
| `apx.ANN_RET_ADD` | `apx.ANN_ADD_OF(r)` | 年化報酬（加法） |
| `apx.HIT_RATE` | `mean(r > 0)` | 日勝率 |
| `apx.MONTHLY_RET` | `by_prod1(month, r)` | 各曆月報酬（序列） |
| `apx.MONTH_WIN` | `mean(apx.MONTHLY_RET > 0)` | 月勝率 |
| `apx.BEST_MONTH` | `max(apx.MONTHLY_RET)` | 最佳曆月 |
| `apx.WORST_MONTH` | `min(apx.MONTHLY_RET)` | 最差曆月 |
| `apx.N_MONTHS` | `count(apx.MONTHLY_RET)` | 曆月數 |
| `apx.ROLL12_WIN` | `mean(roll_sum(r, 252) > 0)` | 滾動 252 日報酬 > 0 的比例 |
| `apx.YEARS` | `by_key(year)` | 各曆年（序列） |
| `apx.YEARLY_RET` | `by_prod1(year, r)` | 各曆年報酬（序列） |
| `apx.YEARLY_SHARPE` | `by_mean(year, r) / by_sd(year, r) * sqrt(252)` | 各曆年 Sharpe（序列） |
| `apx.YEARLY_DAYS` | `by_count(year, r)` | 各曆年交易日數（序列） |
| `apx.N_YEARS` | `count(apx.YEARLY_RET)` | 曆年數 |
| `apx.YEARS_POS` | `mean(apx.YEARLY_RET > 0)` | 曆年報酬 > 0 的比例 |
| `apx.RET_Y_MIN` | `min(apx.YEARLY_RET)` | 最差曆年 |
| `apx.RET_Y_MAX` | `max(apx.YEARLY_RET)` | 最佳曆年 |
| `apx.ANN_VOL` | `apx.VOL_OF(r)` | 年化波動 |
| `apx.DOWN_VOL` | `sqrt(mean(pow(min(r, 0), 2))) * sqrt(252)` | 年化下行波動 |
| `apx.MAX_DD` | `apx.MAX_DD_OF(r)` | 最大跌幅（收盤、複利；同報告） |
| `apx.MAX_DD_ADD` | `apx.MAX_DD_ADD_OF(r)` | 最大跌幅（加法；同 `max_dd_1e6`） |
| `apx.MAX_DD_DAYS` | `streak(drawdown_nav(r) > 0)` | 最長水下期（交易日） |
| `apx.ULCER` | `sqrt(mean(pow(drawdown_nav(r), 2)))` | Ulcer 指數 |
| `apx.SKEW` | `mean(pow(r - mean(r), 3)) / pow(mean(pow(r - mean(r), 2)), 1.5)` | 日報酬偏態 |
| `apx.KURT` | `mean(pow(r - mean(r), 4)) / pow(mean(pow(r - mean(r), 2)), 2)` | 日報酬峰態（常態 3） |
| `apx.VAR95` | `quantile(r, 0.05)` | 日報酬 5% 分位 |
| `apx.CVAR95` | `mean(filter(r, r <= quantile(r, 0.05)))` | 5% 分位以下那些日的平均 |
| `apx.WORST_DAY` | `min(r)` | 最差單日 |
| `apx.BEST_DAY` | `max(r)` | 最佳單日 |
| `apx.SHARPE` | `apx.SHARPE_OF(r)` | 年化 Sharpe |
| `apx.SORTINO` | `apx.ANN_RET / apx.DOWN_VOL` | Sortino |
| `apx.CALMAR` | `apx.ANN_RET / apx.MAX_DD` | Calmar |
| `apx.ROMAD` | `apx.TOTAL_RET / apx.MAX_DD` | 全期淨報酬 ÷ 最大跌幅 |
| `apx.OMEGA` | `sum(max(r, 0)) / sum(max(-r, 0))` | Omega（門檻 0） |
| `apx.IR` | `apx.SHARPE_OF(r - m)` | 對同範圍等權平均的資訊比率 |
| `apx.TE` | `apx.VOL_OF(r - m)` | 年化追蹤誤差 |
| `apx.EXCESS_ANN` | `apx.ANN_OF(r - m)` | 年化超額報酬 |
| `apx.BETA` | `apx.BETA_OF(r, m)` | 對同範圍等權平均的 β |
| `apx.ALPHA_ANN` | `(mean(r) - apx.BETA * mean(m)) * 252` | 年化 α |
| `apx.CORR_MKT` | `apx.CORR_OF(r, m)` | 與同範圍等權平均的相關 |
| `apx.UP_CAPTURE` | `sum(filter(r, m > 0)) / sum(filter(m, m > 0))` | 上漲捕獲 |
| `apx.DOWN_CAPTURE` | `sum(filter(r, m < 0)) / sum(filter(m, m < 0))` | 下跌捕獲 |
| `apx.MKT_ANN` | `apx.ANN_OF(m)` | 同範圍等權平均的年化報酬 |
| `apx.IC` | `ic_x1e4 / 1e4` | Rank IC 均值 |
| `apx.IC_T` | `ic_t_x100 / 100` | IC 的 t 值 |
| `apx.IC_POS` | `ic_days > 0 ? ic_pos_x1e4 / 1e4 : nan` | IC > 0 的比例 |
| `apx.ICIR` | `ic_days > 0 ? ic_t_x100 / 100 / sqrt(ic_days) : nan` | ICIR |
| `apx.SPREAD` | `spread_days > 0 ? spread_1e6 / 1e6 : nan` | 前 10% − 後 10% 年化價差 |
| `apx.QUINT_RHO` | `quint_days > 0 ? quint_rho_x1e4 / 1e4 : nan` | 五分位秩相關 |
| `apx.QUINT_MONO` | `quint_days > 0 ? quint_mono_x1e4 / 1e4 : nan` | 五分位當日單調的比例 |
| `apx.QUINT_TOP` | `quint_days > 0 ? quint_1e6_5 / 1e6 : nan` | 最高五分位年化報酬 |
| `apx.QUINT_BOT` | `quint_days > 0 ? quint_1e6_1 / 1e6 : nan` | 最低五分位年化報酬 |
| `apx.IC_1`、`apx.IC_5`、`apx.IC_21`、`apx.IC_63` | `ic_decay_days_N > 0 ? ic_decay_x1e4_N / 1e4 : nan` | 持有 N 日的 IC |
| `apx.IC_HALF_LIFE` | `ic_decay_days_1 > 0 && ic_decay_x1e4_1 > 0 ? ic_half_life_days : nan` | IC 半衰期（−1＝63 日內沒掉一半） |
| `apx.IC_H1`、`apx.IC_H2` | `ic_h1_x1e4 / 1e4`、`ic_h2_x1e4 / 1e4` | 前、後半段 IC |
| `apx.TURNOVER` | `turnover_days > 0 ? turnover_bp_avg / 1e4 : nan` | 每次換股的換手 |
| `apx.TURNOVER_ANN` | `apx.TURNOVER * 252 / reb_days` | 年化換手 |
| `apx.HOLD_DAYS` | `reb_days / apx.TURNOVER` | 平均持有天數（估算） |
| `apx.COST_DRAG_ANN` | `apx.TURNOVER_ANN * 2 * cost_bp / 1e4` | 成本拖累（估算） |
| `apx.AVG_POSITIONS` | `avg_positions_x100 / 100` | 平均持股數 |
| `apx.SHARPE_H1`、`apx.SHARPE_H2` | `apx.SHARPE_OF(filter(r, half == 1))`、`… half == 2` | 前、後半段 Sharpe |
| `apx.SHARPE_Y_MIN` | `min(filter(apx.YEARLY_SHARPE, apx.YEARLY_DAYS >= 126))` | 最差曆年 Sharpe（≥ 126 日的年） |
| `apx.SHARPE_BULL`、`apx.SHARPE_BEAR` | `apx.SHARPE_OF(filter(r, regime > 0))`、`… regime < 0` | 多頭日、空頭日的 Sharpe |
| `apx.RET_BULL_ANN`、`apx.RET_BEAR_ANN` | `apx.ANN_OF(filter(r, regime > 0))`、`… regime < 0` | 多頭日、空頭日的年化報酬 |
| `apx.DAYS_BULL`、`apx.DAYS_BEAR` | `count(filter(r, regime > 0))`、`… regime < 0` | 多頭、空頭日數 |
| `apx.EXCESS_B_ANN` | `apx.ANN_OF(r - b)` | 對大盤基準的年化超額報酬（複利） |
| `apx.IR_B` | `apx.SHARPE_OF(r - b)` | 對大盤基準的資訊比率 |
| `apx.BETA_B` | `apx.BETA_OF(r, b)` | 對大盤基準的 β |
| `apx.YEARS_BEAT_B` | `MEAN(BY_PROD1(year, r) > BY_PROD1(year, b))` | 曆年報酬贏大盤基準的年數比例 |
| `apx.ROLL_SHARPE_MIN` | `MIN(ROLL_MEAN(252, r) / ROLL_STD(252, r)) * SQRT(252)` | 滾動一年年化 Sharpe 的最小值 |
| `apx.ROLL_RET_MIN` | `MIN(EXP(ROLL_SUM(252, LOG(1 + r)))) - 1` | 滾動一年報酬（複利）的最小值：任一年持有期最差 |
| `apx.TOTAL` | `FLOOR((3 * cs_bp + 2 * ts_bp + 3 * novel_bp + 2 * short_bp) / 10)` | 合成分數（bp）：沒寫 `grid_by` 時各組照它排；`total_bp` 就是它的值 |

`m`（同範圍等權平均）與 `b`（大盤指數）是兩把不同的尺：`apx.IR`、`apx.BETA`、`apx.EXCESS_ANN` 對 `m`，`*_B` 對 `b`。

### 11.3 一條門檻、一條排序

篩選只有兩條算式：**門檻**（`screen_gate`，結果為真才留）與**排序**（`portfolio.grid_by`，大的先）。

**門檻 `screen_gate`**：每支候選都要過。可以用原子、`apx.*`、`user.*`，也可以用檢驗輸出 `check.<NAME>.<輸出名>`。什麼時候評由它引用了什麼決定：沿頂層 `&&` 切開，含 `check.*` 的段在檢驗跑完後評，其餘在送檢前評（沒過的不送檢）。頂層有 `||` 又用到 `check.*` 時整條在檢驗後評（結果一樣，只是多跑檢驗）。用到 `check.*` 要開那個檢驗（`checks`）。

| 白話 | 寫法 |
|---|---|
| 月勝率至少六成 | `screen_gate="apx.MONTH_WIN >= 0.6"` |
| 每一年都要贏同範圍等權平均 | `screen_gate="MIN(BY_SUM(year, r - m)) > 0"` |
| 七成以上的年份贏大盤指數 | `screen_gate="apx.YEARS_BEAT_B >= 0.7"` |
| 對大盤的年化超額報酬至少 5% | `screen_gate="apx.EXCESS_B_ANN >= 0.05"` |
| 任一年持有都不虧超過 15% | `screen_gate="apx.ROLL_RET_MIN > -0.15"` |
| 多頭和空頭的 Sharpe 都要大於 0.5 | `screen_gate="apx.SHARPE_BULL > 0.5 && apx.SHARPE_BEAR > 0.5"` |
| 最差的一年不低於 −20% | `screen_gate="apx.RET_Y_MIN > -0.2"` |
| 2023 起年化至少 10% | `backtest_from="2023-01-01"`、`screen_gate="apx.ANN_RET >= 0.1"` |
| Sharpe 大於 1，而且同類股替換後報酬保留 26% 以上 | `screen_gate="apx.SHARPE > 1 && check.PEER_SWAP.keep_ret_bp >= 2600"`、`checks=["PEER_SWAP"]` |

**排序 `portfolio.grid_by`**：同一個公式的各組（投組參數、`grid` 試的門檻）照它排，大的在前，算不出數的排最後。沒寫時照合成分數排（有 `grid` 時各組門檻之間比 `apx.SHARPE`）。例：`"grid_by":"apx.CALMAR"`。

**舊寫法照收**：`screen_admit`（檢驗後的門檻）＝接在 `screen_gate` 後面；`screen_pick`（同一個公式的各組裡挑第一支過條件的）照舊。`check.*` 不能用在 `screen_pick` 與 `grid_by`（`E_CHECK_STAGE`）。

- 簡寫 `screen=[{"metric": …, "op": ">=", "value": …}]` 是幾個下限門檻的清單。
- 風險原子與多重檢定原子在 `screen_gate` 照常用；寫在 `screen_pick` 時，讀它們的那一段等算出來才評（沒過換同一個公式的下一組）。`round_pbo` 只能寫在 `screen_gate`：那一段在一輪結束時評，沒過＝這一輪入庫的全部移出庫（例 `screen_gate="round_pbo <= 0.5"`）。
- 設定檔的舊鍵是這幾條算式的簡寫：`w_cs`／`w_ts`／`w_novel`／`w_short`＝換 `apx.TOTAL` 的係數；`min_alpha_share`、`min_breadth_eff`＝`screen_pick` 的風險段；`mt_gate=block`＝`screen_pick` 加 `mt_dsr >= min_dsr_bp/10000 && mt_ic_padj <= max_ic_padj_bp/10000`；`pbo_block=1`＝`screen_gate` 加 `round_pbo * 10000 <= max_pbo_bp`。
- 報告工作（`submit_report`）挑哪幾支也是這兩條：`screen_gate`＝入庫的支裡哪些出報告、`grid_by`＝排序（大的在前、算不出數的排最後、同值照庫的順序；預設 `ic_t_x100`），取前 `top` 支（0＝全部）。值讀 `validation.json` 每支的篩選原子與 `apx.*`；不能用 `check.*`。舊鍵 `min_ic_t=X` 是 `screen_gate="ic_t_x100 / 100 >= X"` 的簡寫（跟 `screen_gate` 二選一）。例：`{"library":"L1","top":3,"screen_gate":"ic_t_x100 >= 300 && max_dd_1e6 <= 250000","grid_by":"apx.SHARPE"}`。

### 11.4 自訂指標 `apx.metric.v1`

```json
{"schema": "apx.metric.v1", "name": "user.local.ROMAD", "version": 1,
 "description": "年化報酬 ÷ 最大回撤", "inputs": [],
 "body": "apx.ANN_RET / apx.MAX_DD", "unit": "", "direction": "max", "group": "mine"}
```

| 欄位 | 意思 |
|---|---|
| `name` | `user.<作者>.NAME`（`apx.*` 是保留的） |
| `version` | ≥ 1 的整數；同名同版內容不可改，改內容要 +1 |
| `inputs` | 形式輸入名的清單；空＝值（`user.local.ROMAD`），非空＝函式（`apx.SHARPE_OF(x)`） |
| `body` | 定義式；可引用原子與其他定義式，不得成環 |
| `direction` | `max`／`min`／`none` |
| `unit`、`group`、`description` | 說明用 |

用法：`screen_gate="user.local.ROMAD >= 1"`。最長水下天數：body `streak(drawdown_nav(r) > 0)`。

### 11.5 檢驗

一個檢驗＝把策略換一種條件重跑，看指標還在不在。

**原子變換**（引擎提供，檢驗定義檔引用）：

| 變換 | 意思 |
|---|---|
| run 的 `with`：一條 BOOL 算式（例 `CS_PCT(fund.mktcap) >= 0.7`） | 只在符合條件的股票裡重排名、重跑；寫幾項取交集 |
| run 的 `with`：`PEER_SIDE() == 0`、`PEER_SIDE() == 1` | 相似股兩兩配對、分半的 A、B 兩組選股範圍 |
| run 的 `with`：`RANDOM_HALF()`、`RANDOM_HALF(k)` | 隨機一半股票的選股範圍（k 不同是不同的一半） |
| run 的 `with`：`RETURNS(算式)` | 換報酬的值，`ret` 代表原本的報酬（例 `RETURNS(SUB(ret, MKT_MEAN(ret)))`） |
| run 的 `with`：`PEER_SWAP()` | 選中某檔時改用跟它最相近那檔的報酬 |
| run 的 `with`：`SHUFFLE()`、`SHUFFLE(k, 群鍵)` | 每天的報酬在股票之間打亂（帶群鍵：只在同一群裡打亂） |
| run 的 `with`：`SCORE(算式)` | 換分數，`score` 代表候選原本的分數（例 `SCORE(ROLL_MEAN(5, score))`）；新分數在這次重跑的範圍裡排名 |
| run 的 `with`：`ENTRY_DELAY(n)`、`ENTRY_DELAY_ASOF(n)` | 進場晚 n 天（＝`SCORE(LAG(n, score))`；`_ASOF` 名次照 n 天前那天的範圍＝`SCORE(LAG(n, CS_PCT(score)))`） |
| run 的 `with`：`REBALANCE_WHEN(條件)`、`PHASE(n, k)` | 換股日改成條件成立那天（同回測的 `rebalance_when`）；`PHASE(n, k)`＝`REBALANCE_WHEN(EVERY(n, k))`，每 n 天、錯開 k 天 |
| run 的 `with`：`DAYS(權重)`、`DROP_TOP_DAYS(k)`、`DROP_BOTTOM_DAYS(k)`、`ONLY_DAYS(條件)`、`BOOTSTRAP(seed, block)` | 哪些日子算數：部位不變，收成指標時每一天乘權重（拿掉最好／最差幾天、只看某些日子、日子重抽樣） |
| run 的 `with`：`DROP_TOP_HELD(run, k)`、`DROP_BOTTOM_HELD(run, k)`、`DROP_MOST_HELD(run, k)` | 拿掉前一次重跑裡賺最多／賠最多／抱最久的 k 檔再重跑（＝範圍條件 `!CS_TOPK(k, HELD_RET(run))`…）；範圍條件裡也能直接用 `HELD_RET(run)`、`HELD_DAYS(run)` |
| run 的 `with`：`RETUNE(f)`、`RETUNE_AT(i, f)` | 公式的天數全部（或只有第 i 個）×f／100 重算，天數改一點還穩嗎；`prog.n_windows`＝天數的個數 |
| run 的 `with`：`SCORE_NOISE(k, amp_bp)` | 名次加雜訊再選（＝`SCORE(ADD(CS_PCT(score), MUL(amp_bp / 10000, NOISE(k))))`） |
| run 的 `with`：`user.<作者>.NAME(引數…)` | 自己存的變換（11.6「自己的變換」），展開成它定義的那幾項 |
| run 的 `with`：`COST(…)`、`HOLD(hold && apx.IN_TOP(p))`、`WEIGHT(…)` | 換回測的單邊成本、做多門檻、權重（檔數、權重鍵、單檔上限、換手上限）；`cost`、`hold`、`weight` 代表原本的 |
| run 的 `with`：`PERIOD(起, 迄)`、`ROLLING_OOS(is, oos, k)`、`ROLLING_IS(is, oos, k)`、`SPLIT(n, k)` | 只在某段日期整支重跑；`k` 寫 `win`＝每段跑一次（walk-forward、分段） |
| 舊欄 `interval`、`windows`、`cost_bp`、`p_long`、`p_short`、`rebalance_days`、`weight_mode`、`max_positions`、`max_weight_bp`、`max_turnover_bp` | 照收，等於上面兩列的寫法（`p_short` 的新寫法等放空規則） |

**內建檢驗**（都是定義檔，可用 `stock_formula_help topic=check name=…` 看全文）：

| 名字 | 做什麼 | 參數（預設） | 通過條件 |
|---|---|---|---|
| `apx.PEER_SWAP` | 同類股替換：選中的每一檔改用最相近那檔的報酬重算 | `keep_bp`（0；0..10,000） | 原版淨報酬 > 0、IC ≠ 0，替身版的淨報酬與 IC 都保留原版的一定比例 |
| `apx.PAIR_SPLIT` | 配對分半：A、B 兩個子範圍各自重排名 | `min_spread_1e6`（1） | A、B 的 IC 都與原範圍同號且不為 0、分位價差 ≥ 下限 |
| `apx.RANDOM_HALF` | 隨機一半股票上重排名 | `min_spread_1e6`（1） | IC 與原範圍同號且不為 0、分位價差 ≥ 下限 |
| `apx.COST_RETEST` | 成本重測：單邊成本 ×1…×mult 各跑一次 | `mult`（3；1..4）、`min_ret_1e6`（1） | ×mult 的全期淨報酬 ≥ 下限 |
| `apx.WALK_FORWARD` | 從最後一天起每 `oos_days` 往前切一段、樣本內緊鄰其前；樣本內選變體、樣本外用它跑 | `is_days`（504）、`oos_days`（126）、`min_windows`（2）、`min_pos_bp`（5,000） | 段數夠；樣本外 IC > 0 與淨值 > 0 的段數比例都 ≥ 下限 |
| `apx.PLACEBO` | 報酬逐日在範圍內打亂後重算 | `min_gap_x100`（200） | 原版 IC 的 t 值比打亂版高出下限 |
| `apx.SEGMENTS` | 全期等分 n 段各自算指標 | `n`（4；2..32）、`min_pos_bp`（7,500） | IC > 0 的段數比例 ≥ 下限 |
| `apx.DSR` | Deflated Sharpe Ratio | `min_dsr_bp`（9,500） | DSR ≥ 下限（沒開帳本時不擋） |

- `checks` 預設只開同類股替換：`checks=["peer_swap"]`；`checks=[]`＝只篩選。
- 內建檢驗＝**帶參數的保留檢驗函數**，跟原子同一個文法：`checks=["PEER_SWAP(5000)", "SEGMENTS(4, 7500)"]`。參數照上表的順序、都是整數，後面的可省（＝預設）；不帶括號＝全部預設。用戶檢驗同一套：`checks=["user.local.HALVES(3000)"]`（參數照它定義檔 `params` 的順序）。
- 同類股替換用哪把尺量「最相近」：尺是**輸入**，寫在參數後面——`checks=["PEER_SWAP(5000, inst_net)"]`，只換尺不改參數寫 `PEER_SWAP(inst_net)`。尺是 `ret`（日報酬，預設）、`ret_rank`、`vol_chg`、`amount_chg`、`turnover`、`turnover_rate`、`rev_yoy`、`inst_net`、`size`、`group_rank`，或一條算式（例 `PEER_SWAP(DIFF(1, LOG(volume)))`：全市場名次相關、每期前 252 日）。配對分半與用戶檢驗的 `PEER_SWAP()`、`PEER_SIDE()` 用同一把；檢驗紀錄的 `peer_ruler` 寫實際用的那把。
- 一支策略同時最多 8 個檢驗。
- 舊寫法 `check_params={"PEER_SWAP.keep_bp": 5000, "PEER_SWAP.ruler": "inst_net"}` 照收，展開成同一份（同一組參數、`content_hash` 同、結果與紀錄逐 byte 相同）；同一個參數兩種寫法同給 → `APX_CONFLICT`。

| 白話 | 寫法 |
|---|---|
| 同類股替換後報酬至少保留五成 | `checks=["PEER_SWAP(5000)"]` |
| 同類股用法人買賣超量相似、至少保留五成 | `checks=["PEER_SWAP(5000, inst_net)"]` |
| 交易成本加倍後淨報酬仍為正 | `checks=["COST_RETEST(2)"]` |
| 分 4 段，至少 3 段 IC 為正 | `checks=["SEGMENTS(4, 7500)"]` |

### 11.6 自己組一個檢驗：`apx.check.v1`

內建的 8 個檢驗本身就是這種定義檔（`stock_formula_help topic=check name=PEER_SWAP` 看全文），用戶寫的檢驗跟它們走同一條路。一個檢驗＝「換哪些條件重跑幾次」＋「從重跑的結果算幾個整數」＋「整數要滿足的條件」。

```json
{"schema": "apx.check.v1", "name": "user.local.HALVES", "version": 1,
 "description": "前後兩半 IC 同號",
 "runs": [{"id": "seg", "with": "SPLIT(2, win)"}],
 "outputs": {"seg_ic": "seg.ic_days > 0 ? seg.ic_x1e4 : 0", "lo": "AMIN(seg_ic)", "hi": "AMAX(seg_ic)"},
 "pass": "lo > 0 || hi < 0"}
```

| 欄位 | 意思 |
|---|---|
| `runs` | 幾次重跑、每次換哪些東西（沒寫的＝原版）；最多 8 次 |
| `outputs` | 從各次重跑的指標算出幾個整數；最多 16 個。就是「檢驗後的篩選指標」，門檻 `screen_gate` 用 `check.<NAME>.<輸出名>` 讀 |
| `pass` | 對 `outputs`（與 `$參數`、`env.*`）的條件式 |
| `params` | 參數：`{"name","default","min","max"}`，算式裡寫 `$名字` |

**第 1 步：每一次重跑換什麼（`runs`）。**每一項是一次重跑，`id` 必填（算式裡用它引用）。能換的：

| 鍵 | 值 | 意思 |
|---|---|---|
| `with` | 逗號分隔的變換清單，一項一個變換（下表） | 換選股範圍、報酬、分數、換股日、日子、投組設定、區間都寫在這裡；沒寫＝原版 |
| `vary` | `{"k": [1, 2, 3]}` 或 `{"k": {"from": 1, "to": "$mult"}}` | 同一次重跑展成幾組（一個變數、1..16 個值）；`with` 裡寫 `RUN.k` 取值。變數名是舊投組欄名（`p_long`、`rebalance_days`…）時值直接套進去 |
| 舊欄 | `interval`（`full`、`window.is`、`window.oos`、`window.seg`、`{"begin","end"}`）、`cost_bp`、`p_long`、`p_short`、`rebalance_days`、`weight_mode`、`max_positions`、`max_weight_bp`、`max_turnover_bp`；定義檔頂層的 `windows` | 照收，是 `with` 寫法的別名：`window.is`／`window.oos`＝`ROLLING_IS`／`ROLLING_OOS(is, oos, win)`、`window.seg`＝`SPLIT(n, win)`、`{"begin","end"}`＝`PERIOD(begin, end)`、`"cost_bp":"prog.cost_bp * 3"`＝`COST(cost * 3)`、`"p_long":"170"`＝`HOLD(hold && apx.IN_TOP(170))`…（同一份定義） |

`with` 的每一項（括號裡的逗號不算分隔）：

| 寫法 | 換的是 | 意思 |
|---|---|---|
| 一條 BOOL 算式，或 `UNIVERSE(算式)` | 選股範圍 | 只在算式為真的股票裡重排名、重跑。文法、單位同第 8 章的 `universe=`；跨檔運算（`CS_PCT`…）的成員是原範圍，算一次。一個 run 寫幾項＝取交集 |
| `PEER_SIDE() == 0`、`PEER_SIDE() == 1` | 選股範圍 | 相似股兩兩配對、每對拆兩邊的 A、B 組。`PEER_SIDE(尺)` 用那條算式量相似（不帶＝`PEER_SWAP.ruler` 那把） |
| `RANDOM_HALF()`、`RANDOM_HALF(k)` | 選股範圍 | 隨機一半股票（跨期固定）；k 是第幾份，k 不同是不同的一半 |
| `RETURNS(算式)` | 報酬 | 算式要回 RATIO；`ret` 代表原本的報酬。全日、隔夜、盤中各代入一次；盤中觸價的價格路徑照原本的走 |
| `PEER_SWAP()`、`PEER_SWAP(尺)` | 報酬 | 選中某檔時改用最相近那檔同一天的整根 K；排名與持股不變。等於 `REMAP(PEER_IDX())` |
| `PEER_SWAP(回看, 每幾天, 取樣間隔, 名次式, 最低相關bp, 輸入[, 群鍵])` | 報酬 | 配對怎麼找全寫出來（`PEER_IDX`、`PEER_SIDE` 收同一組引數）：每期開始前 `回看` 日、每 `取樣間隔` 日取一點算 `輸入` 的相關（`名次式` 0＝報酬相關、1＝名次相關；`ret`＝日報酬），每 `每幾天` 重配一次；相關低於 `最低相關bp` 不算相似（`-10000`＝不設下限）；`群鍵` 不寫＝全市場找，寫了＝每期第一天群鍵相同的檔之間找（`PARENT_ID(1)` 同產業、`CORR_GROUP(…)` 相關分群）。預設那把尺＝`PEER_SWAP(252, 126, 1, 0, 3000, ret)` |
| `SHUFFLE()`、`SHUFFLE(k)`、`SHUFFLE(k, 群鍵)` | 報酬 | 每天把報酬在股票之間打亂（跨檔訊號消失、大盤與分佈不變）。k 是第幾份；帶群鍵＝只在群鍵相同的股票之間打亂。等於 `REMAP(SHUFFLE_IDX(…))` |
| `REMAP(檔號算式)` | 報酬 | 每一格改用算式指到的那一檔同一天的整根 K。`PEER_IDX()`、`SHUFFLE_IDX()` 回檔號（整數）；算式無效的格用自己的。只換一部分：`REMAP(MASK(條件, PEER_IDX()))` |
| `SCORE(算式)` | 分數 | `score` 代表候選原本的分數（逐日逐檔）；算出來的新分數在這次重跑的範圍裡排名、照同一條路選股。文法同分數公式（大寫、定點）。`SCORE(score)`＝不換 |
| `ENTRY_DELAY(n)` | 分數 | 進場晚 n 天：n 天前的分數、在今天的範圍排名。等於 `SCORE(LAG(n, score))` |
| `ENTRY_DELAY_ASOF(n)` | 分數 | 進場晚 n 天、名次照 n 天前那天的範圍。等於 `SCORE(LAG(n, CS_PCT(score)))` |
| `REBALANCE_WHEN(條件)` | 換股日 | 換股日改成條件成立那天。條件要每天全市場同值（月初、`EVERY(n, k)`、大盤條件…），跟回測寫 `rebalance_when` 同一條規則、同一份數字；曝險照原本的 |
| `PHASE(n, k)` | 換股日 | 每 n 天換一次、錯開 k 天。等於 `REBALANCE_WHEN(EVERY(n, k))`（`EVERY` 從面板第一天 0 起算） |
| `DAYS(權重)` | 日子 | 每一天算幾次（收淨報酬、IC、價差）：權重是 11.1 篩選語言（小寫那門、實數）對**這次重跑自己**的逐日序列 `r`、`m`、`year`、`month`、`regime`、`half`、`day`、`b` 算的式子；0 或缺值＝不算、1＝算一次、2＝算兩次（取整，上限 32）；BOOL 當 0／1。部位與持股照原樣跑，只改收成 |
| `DROP_TOP_DAYS(k)` | 日子 | 拿掉這次重跑每日淨報酬最好的 k 天（＝`DAYS(rank_desc(r) > k)`） |
| `DROP_BOTTOM_DAYS(k)` | 日子 | 拿掉最差的 k 天（＝`DAYS(rank_desc(-r) > k)`） |
| `ONLY_DAYS(條件)` | 日子 | 只看條件成立的日子（＝`DAYS(條件)`），例 `ONLY_DAYS(regime > 0)` 只看多頭日 |
| `BOOTSTRAP(seed, block)` | 日子 | 日子區塊重抽樣（＝`DAYS(boot(seed, block))`）；`vary` seed 抽幾次，用 `PCT` 收成一個數 |
| `DROP_TOP_HELD(run, k)` | 選股範圍 | 拿掉 run 裡賺最多的 k 檔重跑：範圍變了，重排名、改買別檔。等於 `!CS_TOPK(k, HELD_RET(run))` |
| `DROP_BOTTOM_HELD(run, k)` | 選股範圍 | 拿掉 run 裡賠最多的 k 檔重跑。等於 `!CS_TOPK(k, NEG(HELD_RET(run)))` |
| `DROP_MOST_HELD(run, k)` | 選股範圍 | 拿掉 run 裡抱最久的 k 檔重跑。等於 `!CS_TOPK(k, HELD_DAYS(run))` |
| `HELD_RET(run)`、`HELD_DAYS(run)` | 選股範圍條件裡的值 | run 每檔的持有報酬和（那條回測自己記在這檔上的每一段報酬加總：續抱日全日、隔天開盤成交日的隔夜或盤中段、觸價出場那一段；RATIO）、持有天數（收盤持有的天數，COUNT）。每檔一個值、每天相同，沒持有過＝0；不帶權重 |
| `RETUNE(f)` | 分數 | 同一支公式、每個天數引數 ×f／100（往下取整）後重算的分數，在這次重跑的範圍裡排名。天數引數＝第 4 章標 N 的引數（日數、期數、`RWIN`／`EWIN`／`RWIN_PCT` 的期數）與演算法型的視窗 W；改完夾到那個算子的合法範圍（例 `ROLL_STD` 2..1,260、`EMA` 2..84、`RET_SKIP` 的 N 要大於 M），夾過的次數寫進檢驗紀錄 `clamped`。f 1..1000；`RETUNE(100)`＝不換 |
| `RETUNE_AT(i, f)` | 分數 | 只改第 i 個天數引數（0 起，照公式的文字順序；同一個子式出現幾次都只算一個）。`prog.n_windows`＝這支公式天數引數的個數 |
| `SCORE_NOISE(k, amp_bp)` | 分數 | 名次加雜訊再選，等於 `SCORE(ADD(CS_PCT(score), MUL(amp_bp / 10000, NOISE(k))))`。雜訊只由（這份檢驗的定義、run 的 id、k、日、檔）決定：同一份檢驗重跑逐位相同、k 不同是另一份 |
| `COST(算式)` | 成本 | 單邊成本（bp）換成算式的值；`cost` 代表原本的，例 `COST(cost * 3)`、`COST(cost * c.k)` |
| `HOLD(hold && apx.IN_TOP(p))` | 做多門檻 | 名次百分位 ≥ p／200（180＝前 10%）才買；`hold` 代表原本的候選條件。p 可以寫 `RUN.變數`、`WINNER(…)` |
| `WEIGHT(算式)` | 權重 | `weight` 代表原本那條。認 `apx.SHARE(apx.PICK(N, false[, 0[, 換手上限]]), 鍵)`：N＝最多幾檔（寫 `prog.max_positions`＝不換）、換手上限是比例（`0.3`）；鍵 `1` 等權、`apx.RANK_KEY()` 依名次、`apx.INV_VOL_KEY()` 依波動倒數、`weight` 照原本；`apx.CAP(單檔上限, …)` 單檔上限（比例，`0.1`＝一成）。例 `WEIGHT(apx.SHARE(apx.PICK(10, false), 1))`、`WEIGHT(apx.CAP(0.1, weight))` |
| `PERIOD(起, 迄)`、`PERIOD(起, 迄, 段數)` | 區間 | 整支在日序號 [起, 迄) 重跑，部位在起點從空倉開始（不是只收那段的指標；只收那段寫 `ONLY_DAYS(day >= …)`）。可用 `env.first`（暖機後第一天）、`env.n_days`、`$參數`、這個 run 的 `vary` 變數；起訖用到 `win`（第幾段，0 起）＝每段跑一次，段數＝第三個引數（`n_windows`，最多 32） |
| `ROLLING_OOS(is, oos, k[, mx])`、`ROLLING_IS(is, oos, k[, mx])` | 區間 | walk-forward 的樣本外／樣本內段：從最後一天起每 oos 天往前切、樣本內 is 天緊鄰其前、樣本內起點 ≥ 暖機，最早的一段 k＝0，多於 mx（預設 32）只留最晚的。k 寫 `win`＝每段跑一次（`WINNER` 取同一段）；寫 `RUN.k`＋`vary {"k":{"from":0,"to":"n_windows - 1"}}`＝vary 展開；寫數字＝只跑那一段。段數＝`ROLLING_N(is, oos[, mx])` |
| `SPLIT(n, k[, mx])` | 區間 | 全期 [暖機, 末日) 等分 n 段的第 k 段；k 寫 `win`＝每段跑一次。段數＝`SPLIT_N(n[, mx])` |
| `user.<作者>.NAME(引數…)`、`user.<作者>.NAME@版本(…)` | 看定義 | 自己的變換（下面「自己的變換」）：定義的 `body` 代入引數後展開成那幾項，跟直接寫展開式是同一個變換、同一個 `content_hash`；沒寫版本＝最新版 |

- 一個 run 的選股範圍條件可以寫幾項，報酬變換最多一項（要兩種就拆成兩個 run）。
- 分數變換可以寫幾項，照寫的順序疊（後一個的 `score` 是前一個的結果）；換股日最多一項。`score` 只能寫在 `SCORE(…)` 裡。
- 日子變換（`DAYS` 類）可以寫幾項，權重相乘；同一個 run 也可以同時換範圍或報酬。帶日子變換的 run 只讀得到照權重重收的欄：`net_ret_1e6`（Σ 權重 × 日淨報酬）、`ic_x1e4`、`ic_t_x100`、`ic_pos_x1e4`、`ic_days`（換股日的 IC 帶權重）、`spread_1e6`、`spread_days`；其餘欄（含 `max_dd_1e6`）引用到回 `E_DAYS_FIELD`（要看就引用不帶日子變換的 run；照權重的回撤寫篩選語言 `MAX(DRAWDOWN(r * 權重))`）。權重全是 1（例 `DAYS(1)`、`DROP_TOP_DAYS(0)`）這幾欄就是原 run 的值。
- 日子權重用小寫篩選語言；寫成大寫分數公式（`DAYS(TOP_K(R, 5))`）或一支策略一個數的名字（`ic_x1e4`）回 `E_CHECK_LANG`。保留函數跟展開式是同一個檢驗身分。
- 算式裡可以寫 `$參數` 與這個 run 的 `vary` 變數（`RUN.變數`），代入整數後才算；`vary` 的值要是數字或 `$參數`，範圍的兩端也可以寫 `prog.n_windows`（例 `{"from":0,"to":"prog.n_windows - 1"}`：每支公式的成員數跟著它的天數個數走）。
- 重新調參（`RETUNE`、`RETUNE_AT`）是分數變換的一種，同一個 run 裡永遠先套（改的是公式本身），再接 `SCORE` 類；寫在前面或後面是同一個變換。公式的天數不夠（沒有天數，或 `RETUNE_AT` 的 i ≥ `prog.n_windows`）時，那支公式這個檢驗不過，紀錄寫 `"error":"E_FORMULA_NO_WINDOW"`；用 `RETUNE` 類的檢驗紀錄多一欄 `clamped`。`vary` 範圍寫到 `prog.n_windows` 時分數變換照上限預先排 16 種，要跟別的分數變換一起開就收小，例 `"to":"MIN2(prog.n_windows, 4) - 1"`。
- 一輪裡不同的選股範圍最多 2 種、不同的報酬變換最多 2 種、不同的分數變換最多 16 種、不同的換股日最多 32 種（同一個變換被幾個 run、幾個檢驗用到算一種；`ENTRY_DELAY(2)` 跟 `SCORE(LAG(2, score))` 是同一種）。
- 引用前一次重跑（`HELD_RET`、`HELD_DAYS`、`DROP_*_HELD`）：被引用的 run 要先跑完，所以引用它的 run 自動排到下一階段（最多 4 階）；被引用的 run 不能帶 `vary` 或視窗、不能引用自己（`E_RUN_REF_SELF`）。範圍每支候選各算一次，一輪最多 16 種。
- 舊的 `universe`、`ret` 兩個鍵與舊名（`universe.peer_a`、`universe.peer_b`、`universe.random_half`、`ret.peer_swap`、`ret.shuffle`）照收，等於寫在 `with` 裡。
- 投組設定（`COST`、`HOLD`、`WEIGHT`）同一欄一個 run 只換一次，區間一個 run 只換一次（重複回 `E_WITH_TWICE`）。`WEIGHT`、`HOLD` 的引數照回測的 `weight`、`hold` 認（同一張形狀表）：`COST` 的算式、`apx.PICK` 的檔數、`apx.IN_TOP` 的門檻可以寫檢驗算式（`$參數`、`RUN.變數`、`prog.欄名`、`WINNER(…)`）；單檔上限、換手上限寫數字。
- 一份檢驗裡用到 `win` 的區間段數要同一組（樣本內、樣本外寫同一組 is、oos；不同回 `E_WINDOWS`）；`n_windows` 就是那個段數。`ROLLING_N`、`SPLIT_N` 是段數（整數），寫在 `vary` 的 `to` 或 `PERIOD` 的第三個引數，不是變換。

| 白話 | run 的寫法 |
|---|---|
| 只在大型股重跑 | `{"id":"big","with":"CS_PCT(fund.mktcap) >= 0.7"}` |
| 排除金融股重跑 | `{"id":"x","with":"!(industry in {17})"}` |
| 只在大型股、而且站上年線的股票裡重跑 | `{"id":"bu","with":"CS_PCT(fund.mktcap) >= 0.7, close > ROLL_MEAN(250, close)"}` |
| 扣掉全市場平均之後還要賺 | `{"id":"ex","with":"RETURNS(SUB(ret, MKT_MEAN(ret)))"}`；`pass` 寫 `ex.net_ret_1e6 > 0` |
| 單日漲跌夾在 ±10% 還要賺（不是靠幾天暴漲） | `{"id":"c","with":"RETURNS(MIN2(MAX2(ret, -0.1), 0.1))"}` |
| 同市值帶裡打亂報酬，原版 IC 的 t 值要高出 2 | `{"id":"sh","with":"SHUFFLE(0, CS_QCUT(10, fund.mktcap))"}`；輸出 `"gap":"base.ic_t_x100 - sh.ic_t_x100"`、`pass` 寫 `gap >= 200` |
| 兩份不同的隨機一半，IC 都要跟原版同號 | `{"id":"h","vary":{"s":[1,2]},"with":"RANDOM_HALF(h.s)"}`；`pass` 寫 `COUNT(SIGN(h.ic_x1e4) == SIGN(base.ic_x1e4)) == 2` |
| 只在大型股、而且換成同類股的報酬 | `{"id":"bs","with":"CS_PCT(fund.mktcap) >= 0.7, PEER_SWAP()"}` |
| 用成交量變化量相似度的同類股替換 | `{"id":"sw","with":"PEER_SWAP(DIFF(1, LOG(volume)))"}` |
| 進場晚 1～3 天，晚 3 天還要賺 | `{"id":"late","vary":{"n":[1,2,3]},"with":"ENTRY_DELAY(late.n)"}`；`pass` 寫 `LAST(late.net_ret_1e6) > 0` |
| 換股日錯開也要穩（月換股，錯開 0～15 天至少八成賺） | `{"id":"ph","vary":{"k":{"from":0,"to":15}},"with":"PHASE(21, ph.k)"}`；`pass` 寫 `COUNT(ph.net_ret_1e6 > 0) * 10 >= 16 * 8` |
| 改成每月第一個交易日換股還要賺 | `{"id":"mo","with":"REBALANCE_WHEN(MONTH_START())"}`；`pass` 寫 `mo.net_ret_1e6 > 0` |
| 分數先平滑 5 天、IC 還要同號 | `{"id":"sm","with":"SCORE(ROLL_MEAN(5, score))"}`；`pass` 寫 `SIGN(sm.ic_x1e4) == SIGN(base.ic_x1e4)` |
| 只在大型股、晚 2 天進場一起試 | `{"id":"bl","with":"CS_PCT(fund.mktcap) >= 0.7, ENTRY_DELAY(2)"}` |
| 拿掉最好的 5 天還賺嗎 | `{"id":"nobest","with":"DROP_TOP_DAYS(5)"}`；`pass` 寫 `nobest.net_ret_1e6 > 0` |
| 拿掉最差的 5 天，淨報酬多多少 | `{"id":"noworst","with":"DROP_BOTTOM_DAYS(5)"}`；輸出 `"lift":"noworst.net_ret_1e6 - base.net_ret_1e6"` |
| 只看空頭日，IC 還要為正 | `{"id":"bear","with":"ONLY_DAYS(regime < 0)"}`；`pass` 寫 `bear.ic_days > 0 && bear.ic_x1e4 > 0` |
| 日子重抽樣 16 次，九成五都賺 | `{"id":"bs","vary":{"s":{"from":1,"to":16}},"with":"BOOTSTRAP(bs.s, 21)"}`；輸出 `"p05":"PCT(bs.net_ret_1e6, 500)"`、`pass` 寫 `p05 > 0` |
| 拿掉賺最多的 5 檔重跑，還要賺 | `{"id":"base"},{"id":"cut","with":"DROP_TOP_HELD(base, 5)"}`；`pass` 寫 `cut.net_ret_1e6 > 0` |
| 拿掉抱最久的 3 檔，IC 還要同號 | `{"id":"base"},{"id":"m","with":"DROP_MOST_HELD(base, 3)"}`；`pass` 寫 `SIGN(m.ic_x1e4) == SIGN(base.ic_x1e4)` |
| 公式天數 ×0.9、×1.1 各跑一次，淨報酬都要保留七成 | `{"id":"base"},{"id":"j","vary":{"f":[90,110]},"with":"RETUNE(j.f)"}`；輸出 `"worst":"base.net_ret_1e6 > 0 ? AMIN(KEEP_BP(j.net_ret_1e6, base.net_ret_1e6)) : 0"`、`pass` 寫 `worst >= 7000` |
| 一次只動一個天數（×0.9），每個都還要賺 | `{"id":"q","vary":{"i":{"from":0,"to":"prog.n_windows - 1"}},"with":"RETUNE_AT(q.i, 90)"}`；`pass` 寫 `ALL(q.net_ret_1e6 > 0)` |
| 分數加 5% 雜訊抽 8 次，至少 7 次還賺 | `{"id":"nz","vary":{"s":{"from":1,"to":8}},"with":"SCORE_NOISE(nz.s, 500)"}`；`pass` 寫 `COUNT(nz.net_ret_1e6 > 0) >= 7` |
| 把「只在大型股、晚 2 天進場一起試」存成自己的變換，晚幾天當引數 | `{"id":"bl","vary":{"n":[1,2,3]},"with":"user.local.BIG_LATE(bl.n)"}`（定義檔見下） |

**自己的變換（`apx.noun.v1`、`out_unit` `TRANSFORM`）＝自己的保留函數。**常用的一組變換存成一個名字，跟第 12 章自訂函數同一種定義檔；官方的 `ENTRY_DELAY`、`ENTRY_DELAY_ASOF`、`PHASE`、`DROP_TOP_DAYS`、`DROP_BOTTOM_DAYS`、`ONLY_DAYS`、`BOOTSTRAP`、`DROP_TOP_HELD`、`DROP_BOTTOM_HELD`、`DROP_MOST_HELD`、`SCORE_NOISE` 也是這種定義（例 `ENTRY_DELAY` 的 `body` 是 `SCORE(LAG($n, score))`、`DROP_TOP_DAYS` 的是 `DAYS(rank_desc(r) > $k)`），兩邊同一條代入展開的路、同一個 `content_hash` 規則：

```json
{"schema": "apx.noun.v1", "name": "user.local.BIG_LATE", "version": 1, "out_unit": "TRANSFORM",
 "description": "只在大型股、晚 n 天進場",
 "params": [{"name": "n", "default": 1}],
 "body": "CS_PCT(fund.mktcap) >= 0.7, ENTRY_DELAY($n)"}
```

- `body`＝一項或幾項 `with`（逗號分隔），參數寫 `$名字`（只能用自己 `params` 宣告的）；可以用別的用戶變換（最多 8 層）。沒有 `inputs`。
- run 的 `with` 整項寫 `user.local.BIG_LATE(2)`：引數照 `params` 的順序，後面的可省＝預設；可以帶這個 run 的 `vary` 變數（`user.local.BIG_LATE(bl.n)`，每個值一種變換）。釘版本寫 `user.local.BIG_LATE@1(2)`。
- 存法同自訂函數（驗 → 存）；回測或搜尋帶用到它的檢驗時，存好的變換一起送給引擎，不佔 8 個檢驗的名額。
- 錯誤碼：`E_XFN`（不是 `TRANSFORM`、沒有 `body`、有 `inputs`、一層套一層超過 8 層）、引數個數不對回 `E_WITH_TYPE`（帶寫法，同官方保留函數）、`E_PARAM`（`body` 用了沒宣告的 `$名字`）、沒有那一版回 `E_WITH_UNKNOWN`；`body` 每一項照 `with` 的規則驗（同上表的錯誤碼），都帶位置與可貼的改法。

**檢驗的身分與檢驗帳。**

- 一個檢驗的身分＝名字@版本＋正規化定義的 `content_hash`＋參數值＋那把尺（用到不帶尺的 `PEER_SWAP()`、`PEER_SIDE()`、`PEER_IDX()` 才算）＋種子 `val_alt_seed`＋用到的用戶變換名字@版本。
- 檢驗紀錄（結果庫 `validation.json` 每支候選的 `checks`）在用戶檢驗、或變換帶引數時，`name`、`version` 後面多三欄：`id`（身分，16 位十六進位）、`uses`（用到的用戶變換名字@版本）、`seed`。內建 8 個的紀錄跟以前逐 byte 相同。
- `env.n_checks_tried`＝這支候選在這個結果庫裡、這次搜尋之前跑過幾種不同的檢驗；`env.n_checks_pass`＝其中過了幾種。同一個身分跑幾次算一種、過過一次就算過；內建以名字@版本算；沒有結果庫時是 −1。內建檢驗都不讀它們。
- 檢驗的重跑不是試驗、不進多重檢定的試驗數。要扣「同一支策略換很多種檢驗、過一個就收」，自己寫進 `pass`（例 `tried <= 3 && …`）。

**第 2 步：要切時間就在 `with` 寫區間。**`ROLLING_IS(504, 126, win)`、`ROLLING_OOS(504, 126, win)`：從最後一天起每 126 天往前切一段、樣本內 504 天緊鄰在前，每段跑一次；`SPLIT(4, win)`：全期等分 4 段。舊寫法（頂層 `windows`＋`interval`）照收。

**第 3 步：算輸出（`outputs`）。**`RUN.FIELD` 讀某次重跑的指標，欄名同 11.1 的純量原子（`net_ret_1e6`、`max_dd_1e6`、`ic_x1e4`、`ic_t_x100`、`ic_days`、`spread_1e6`、`quint_1e6_1`…`5`、`turnover_bp_avg`…）。有視窗或 `vary` 的重跑是一串值，要用聚合收成一個數；一條輸出只能引用寫在它前面的輸出；名字不能跟欄名、run id、參數名、`env`、`prog` 撞。

**第 4 步：寫通過條件（`pass`）。**

**第 5 步：驗、存、用。**`stock_formula_help topic=validate document={…}` 驗（回每一個錯）→ `my_stock_strategies_change action=install document={…}` 存 → 回測或搜尋帶 `checks=["user.local.HALVES"]`；帶參數照定義檔 `params` 的順序寫在括號裡 `checks=["user.local.HALVES(3000)"]`。同一支策略最多 8 個檢驗。

檢驗算式是整數（i64）：除以 0＝0；比較與邏輯回 0／1。

| 類 | 能寫的 |
|---|---|
| 值 | 數字、`$參數`、`env.NAME`、`prog.FIELD`、`RUN.FIELD`（某次重跑的指標，欄名同 11.1 的純量原子）、`RUN.VAR`（`vary` 的變數） |
| 運算 | `+ - * / %`、比較、`&& \|\| !`、`?:` |
| 函數 | `ABS(x)`、`MIN2(a, b)`、`MAX2(a, b)`（跟分數公式同名）、`SIGN(x)`（−1／0／1）、`KEEP_BP(變體, 原版)`（變體 ÷ 原版，萬分之一為單位，例 `KEEP_BP(swap.net_ret_1e6, base.net_ret_1e6)`） |
| 聚合 | `COUNT` `SUM` `ALL` `ANY` `AMIN` `AMAX`（視窗維度優先，否則 `vary` 維度）、`FIRST` `LAST`、`NTH(expr, k)`（`vary` 的第 k 個，0 起）、`PCT(expr, q_bp)`（`vary` 維度由小到大的分位：第 ⌊(n − 1) × q_bp ÷ 10,000⌋ 個，0 起；q_bp 夾到 0..10,000，例 `PCT(bs.net_ret_1e6, 500)`＝第 5 百分位）、`WINNER(RUN, FIELD, VAR)`（那次重跑裡 FIELD 最大那組的變數值，用在另一次重跑的投組設定＝樣本內挑、樣本外用，例 `HOLD(hold && apx.IN_TOP(WINNER(is, net_ret_1e6, p)))`）、`n_windows` |
| 舊寫法 | 小寫名字照收、身分照大寫寫法（同一份定義、同一個 `content_hash`）：`min(a, b)`＝`MIN2(a, b)`、`min(x)`＝`AMIN(x)`、`keep_bp(…)`＝`KEEP_BP(…)`…；`MIN(a, b, c)` 回錯附改法 `MIN2(MIN2(a, b), c)` |
| `env` | `n_days`（面板日數）、`first`（暖機後第一天）、`n_symbols`、`peer_nn_corr_bp`（平均替身相關）、`peer_fallback_bp`、`peer_paired_bp`、`mt_dsr_bp`、`mt_ic_padj_bp`、`mt_n_eff_x100`、`ic_t_x100`、`net_ret_1e6`、`n_checks_tried`、`n_checks_pass`（檢驗帳，見上） |

| 白話 | `runs` → `outputs` → `pass` |
|---|---|
| 最近一年 IC 也要為正 | `[{"id":"late","with":"PERIOD(env.n_days - 252, env.n_days)"}]` → `{"late_ic":"late.ic_days > 0 ? late.ic_x1e4 : 0"}` → `late_ic > 0` |
| 每週換股和每月換股都要賺錢 | `[{"id":"rb","vary":{"rebalance_days":[5,21]}}]` → `{"worst":"AMIN(rb.net_ret_1e6)"}` → `worst > 0` |
| 隨機一半股票、成本三倍還要賺錢 | `[{"id":"h","with":"RANDOM_HALF(), COST(cost * 3)"}]` → `{"r":"h.net_ret_1e6"}` → `r > 0` |
| 這支策略之前試過的檢驗不超過 3 種，最近一年 IC 也要為正 | `[{"id":"late","with":"PERIOD(env.n_days - 252, env.n_days)"}]` → `{"tried":"env.n_checks_tried","late_ic":"late.ic_days > 0 ? late.ic_x1e4 : 0"}` → `tried <= 3 && late_ic > 0` |
| 只持有 10 檔也要賺、最大跌幅 ≤ 40% | `[{"id":"n10","with":"WEIGHT(apx.SHARE(apx.PICK(10, false), weight))"}]` → `{"r":"n10.net_ret_1e6","dd_bp":"n10.max_dd_1e6 / 100"}` → `r > 0 && dd_bp <= 4000` |
| 單檔上限一成、換手上限三成也要賺 | `[{"id":"cap","with":"WEIGHT(apx.CAP(0.1, apx.SHARE(apx.PICK(prog.max_positions, false, 0, 0.3), weight)))"}]` → `{"r":"cap.net_ret_1e6"}` → `r > 0` |
| 樣本內挑做多門檻、樣本外驗證，兩段都要賺 | `[{"id":"is","vary":{"p":[170,180,190]},"with":"ROLLING_IS(504, 252, win, 2), HOLD(hold && apx.IN_TOP(is.p))"},{"id":"oos","with":"ROLLING_OOS(504, 252, win, 2), HOLD(hold && apx.IN_TOP(WINNER(is, net_ret_1e6, p)))"}]` → `{"oos_ret":"oos.net_ret_1e6","ok":"ALL(oos_ret > 0)"}` → `n_windows >= 2 && ok == 1` |
| 全期切 4 段，每段 IC 都為正 | `[{"id":"seg","with":"SPLIT(4, win)"}]` → `{"ic":"seg.ic_days > 0 ? seg.ic_x1e4 : 0","ok":"ALL(ic > 0)"}` → `n_windows == 4 && ok == 1` |

---

## 12. 自訂函數

### 12.1 定義檔 `apx.noun.v1`

自訂函數是一份 JSON 定義檔，不是程式。題庫 R17「自己組 14 日 RSI」：

```json
{"schema": "apx.noun.v1", "requires": "apx-std>=1.0",
 "name": "user.local.RSI", "version": 1, "description": "Wilder RSI（α = 1/14）",
 "inputs": [{"name": "x", "unit": "PRICE"}],
 "params": [{"name": "k", "default": 42, "min": 0, "max": 756}],
 "body": "RATIO(SMOOTH($k, 0.071429:RATIO, MAX2(DIFF(1, x), 0:PRICE)), ADD(SMOOTH($k, 0.071429:RATIO, MAX2(DIFF(1, x), 0:PRICE)), SMOOTH($k, 0.071429:RATIO, MAX2(NEG(DIFF(1, x)), 0:PRICE))))",
 "out_unit": "RATIO"}
```

用法：`NEG(user.local.RSI(42, close))`。

| 欄位 | 規則 |
|---|---|
| `name` | `user.<作者小寫>.<大寫名>` |
| `version` | ≥ 1 的整數。`name@version` 不可變：同內容再裝一次沒有動作，不同內容要換版本號 |
| `inputs` | 形式輸入：`name` 與 `unit`（單位名，或 `X`＝任意數值單位）。可以沒有 |
| `params` | 整數參數：`name`、`default`，可帶 `min`、`max`、`grid`（搜尋時要試的值） |
| `body` | 定義式：第 2 章的文法；用 `$參數名` 代參數、形式輸入名代輸入的子樹 |
| `out_unit` | 展開後根的單位；要跟實際的一致 |
| `description` | 說明 |

### 12.2 怎麼寫、怎麼呼叫

| 工具 | 做什麼 |
|---|---|
| `stock_formula_help topic=define` | 格式與例子 |
| `stock_formula_help topic=validate document={…}` | 只檢查、不存：回展開後的原子式、暖機、層級；或每一個錯誤（代碼、欄位、位置、提示、最接近的名字） |
| `my_stock_strategies_change action=install document={…}` | 存檔 |
| `stock_formula_help topic=definitions`／`get` | 列出、讀回 |

呼叫：`user.<作者>.NAME(參數…, 輸入…)`。

- 參數在前、輸入在後。前面的引數（引數個數 − 輸入個數）是參數，照宣告順序；缺的用預設。
- 參數值要在宣告的 `min`..`max` 之內。
- 輸入的單位要跟宣告的一樣（宣告 MONEY 的輸入也收 AMOUNT）；常數輸入照宣告的單位讀。
- 可帶版本：`user.local.RSI@1(42, close)`。不帶＝最新版；一個自訂函數引用另一個時，安裝當下記下被引用的版本，之後對方改版也不影響。
- 自訂函數可以引用其他自訂函數，不能成環；展開深度上限 128 層。

### 12.3 展開

自訂函數在編譯時展開：定義式代入參數與輸入，成為同一棵原子樹。相同的子式只算一個節點。

- 展開後的大小算進公式的節點上限（第 13 章）。
- 展開後跟直接寫原子是同一棵樹，結果相同；結果庫記下用到的 `名字@版本`。

### 12.4 條件用的自訂函數

`out_unit` 是 BOOL、不帶 `inputs`、body 用到代碼欄或 `in {…}` 的自訂函數是**條件名詞**：只用在條件（選股範圍、`hold`），不能放進分數公式。

例：把種子籃子存起來，body＝`GSUM(CORR_GROUP(RATIO(close, LAG(1, close))), symbol in {"2330", "2382", "3231"}) > 0`；之後寫 `universe="user.<你>.NAME()"` 或 `universe="!FILL(user.<你>.NAME(), 0:BOOL)"`。

### 12.5 拿自訂函數去搜尋

`trees_from="nouns"`：列舉參數格點 × 輸入綁定，一組一棵樹。

```json
{"trees_from": "nouns",
 "nouns": [{"name": "user.me.NEAR_HIGH", "bind": {"x": "close"}, "grid": {"n": [60, 120]}}]}
```

這是 2 棵樹。`bind` 不給：每個同單位的欄都試一次。

---

## 13. 上限與護欄

這些是技術上限，不是策略規則。

### 13.1 公式大小

| 項 | 上限 |
|---|---|
| 一條公式的節點數 | 128 |
| 深度 | 128 |
| 帶參數的算子數 | 128 |
| 一個算子的輸入數 | 2 |
| 文字的巢狀（呼叫、括號、一元負號、三元各算一層） | 128 層 |
| 自訂函數展開深度 | 128 層 |

- 節點數在**展開到原子之後**算：自訂函數與文字層展開的保留函數（5.2）都展開；第 4 章標「保」的保留函數（`RET`、`CS_PCT`…）算一個節點。欄與常數也算；相同的子式只算一次。
- 例：`MKT_MEAN(欄)` 展開後 4 個節點、`EW_INDEX(close)` 約 17 個。
- 每支策略附複雜度數據（`complexity`：節點數、深度、帶參數算子數），列表、報告、比較都帶。
- 超過時回 `APX_ENGINE_LIMIT`，訊息帶實際的節點數。

### 13.2 視窗與參數

| 上限 | 算子 |
|---|---|
| 1,260（5 年） | `LAG`、`DIFF`、`WSUM`、`ROLL_MEAN`、`ROLL_MAX`、`ROLL_MIN`、`ROLL_STD`、`ROLL_RANK`、`WRANK`、`COUNT`、`FIRST_IN_WINDOW`、`WDROP` |
| 252（1 年） | `RET`、`RET_SKIP`、`RVOL`、`PARK`、`GK`、`SKEW`、`KURT`、`ACORR`、`WSKEW`、`WKURT`、`WACORR`、`SLOPE`、`R2`、`MAXDD`、`DIST_HIGH`、`DIST_LOW`、`UP_STREAK`、`DOWN_STREAK`、`CORR`、`VOL_RATIO`、`AMIHUD`、`BETA`、`RESID_MOM`、`CS_REL_RET`、`WARG`、`WQUANT`、`WMAD`、`FN_YOY`、`FN_EGROW`、`FN_GM_CHG`、`FN_ACCRUAL`、`FN_DY`、`EV_EARN_DRIFT`、`EV_IDX_CHG` |
| 250 | `CS_RANK_CHG`、`CS_RANK_MEAN`、`EV_POST_N`、`EV_DAYS_SINCE` |
| 84 | `EMA`（暖機 3N−1） |
| 756 | `SMOOTH` 的暖機步數 K |
| 256 | 演算法型的視窗 W |
| 12 | `RWIN`、`RWIN_PCT` 的期數 k |
| 20 | `EWIN` 的期數 k |
| 100 | `CS_QCUT` 的桶數、`CS_WINZ` 的 k、`WQUANT`、`GQUANT` 的 q |

兩個參數的算子，每個參數各要放得進 16 位（第一個 0..65,535、第二個 0..32,767）。

### 13.3 暖機

一條公式的暖機（各節點暖機沿樹相加）要**小於面板天數 ÷ 3**。

- 台股面板 4,127 個交易日 → 暖機上限 1,375 天，放得下 1,260 日的視窗。
- 超過時 plan 回錯，訊息帶：這條要回看幾天、這張面板最多幾天。
- 可用 `config={"lookback_div": 2}` 放寬到面板天數 ÷ 2（題庫 R22：5 年均線再看一年變化）。
- 回測算式（`hold`、`exposure`、`exit_when` 的市場算式…）也有暖機，暖機期內那條條件不成立。

### 13.4 常數與金額

| 項 | 範圍 |
|---|---|
| 一般常數 | 定點後放得進 32 位（1e6 定點的單位約 ±2,147） |
| 大常數 | 7 位有效數字（尾數 ≤ 8,388,607）× 10 的 0～12 次方；太精細的回錯 |
| 常數的小數位 | 不超過該單位的定點精度（1e6 定點 6 位；整數單位 0 位） |
| MONEY | 約 ±9.22 兆元；超過的總額欄讀成 AMOUNT（3.2） |

### 13.5 其他

| 項 | 上限 |
|---|---|
| `CORR_GROUP`／`CORR_SEED` 的點數（lookback ÷ step） | 20..1,024（quant 0：20..500） |
| `CORR_SEED` 的種子數 | 1..64 |
| 選股範圍逐檔列的代號 | 20 檔（整條條件合計） |
| `exit_when`／`keep_when` 狀態葉的係數 | 絕對值 ≤ 100 |
| 自訂函數格點 × 綁定 | 20,000 組 |
| 門檻格點 `grid` | 投組規則的名字：基本格組數 × 門檻組數 ≤ 32,767；分數公式的名字：每個 1..256 個值、展開後最多 20,000 棵樹 |
| 檢驗 | 一支策略 8 個檢驗；一個檢驗 8 次重跑、16 個輸出、32 段、16 組 `vary` |
| `rounds` | 1..1,000 |
| 投組參數 | 第 9.2 節的值域 |

---

## 14. 錯誤碼與怎麼改

錯誤回應都帶：代碼、哪一條公式、第幾個字、訊息、提示；名字打錯的會附最接近的名字。plan 階段就驗，不會等到跑了才失敗。

### 14.1 公式與自訂函數

| 代碼 | 什麼時候 | 怎麼改 |
|---|---|---|
| `APX_SYNTAX` | 括號或逗號不對、公式不完整、`?:` 少了 `:`、不認得的字元、數字格式不對、公式後面多了東西 | 檢查括號與逗號；`NAME(參數…, 輸入…)`；`條件 ? 值1 : 值2` |
| `APX_SYNTAX` | 公式只是常數 | 至少要有一個欄 |
| `APX_UNKNOWN_COLUMN` | 面板沒有這一欄 | 用 `stock_formula_help topic=column` 查欄鍵；回應附最接近的欄名 |
| `APX_UNKNOWN_NOUN` | 名字不是算子、別名，也不是已安裝的自訂函數 | 用 `stock_formula_help topic=noun query=…` 查；回應附最接近的名字。別家語言的名字與小寫的名字附照這條寫好的寫法、`user.*` 沒安裝的附安裝的定義檔形狀（2.7） |
| `APX_UNKNOWN_PARAM` | `$名字` 沒宣告，或用在自訂函數的 body 以外 | `$參數` 要在 `params` 宣告，而且只能出現在 body |
| `APX_UNIT_MISMATCH` | 單位不合（例 PRICE ＋ QTY、PRICE × QTY） | 兩邊換成同單位或不帶單位的數：`RATIO(a, b)`、`CS_PCT(x)`；成交金額用欄 `px.turnover` |
| `APX_UNIT_MISMATCH` | 一邊 MONEY、一邊 AMOUNT | MONEY 那邊的總額欄寫成 `欄名:AMOUNT`；每股值不能跟它混用 |
| `APX_UNIT_MISMATCH` | `&&`、`\|\|`、`!`、`?:` 的條件不是 BOOL | 兩邊要是比較的結果；數字相乘寫 `*` |
| `APX_UNIT_MISMATCH` | 欄名後面的 `:單位` 跟欄不一樣 | 拿掉，或改成欄的單位 |
| `APX_UNIT_MISMATCH` | 代碼欄拿去做數學 | 代碼欄只能 `==`、`!=`、`in {…}` 或當群鍵 |
| `APX_PARAM_RANGE` | 參數超出算子的範圍（例 `ROLL_STD(300, x)`） | 第 4 章各列的參數範圍；訊息帶合法範圍。`RET`、`DIST_HIGH`、`DIST_LOW`、`VOL_RATIO`、`FN_YOY` 的 N 在 253..1,260 時附同一個量的長視窗寫法（2.7） |
| `APX_PARAM_TYPE` | 參數不是整數 | 參數寫整數；小數的倍數放進輸入（`0.5:RATIO`） |
| `APX_ARITY` | 輸入不夠、引數太多、少了參數 | 參數在前、輸入在後；例 `ROLL_MEAN(20, close)` |
| `APX_ARITY` | `LE`、`AND`、`IF`、`MIN`、`MAX`、`GPCT` 這些照收的寫法引數個數不對（2.5） | 訊息說要幾個，附寫法 |
| `APX_UNIT_MISMATCH` | 群內名次 `GRANK` 跟比例相加或比大小；事件天數的輸入給了數值欄；`FILL(條件, 0:RATIO)` | 照訊息附的改法：`GPCT(g, x)`、`EV_CHANGE(欄)`、`0:BOOL`（2.7） |
| `APX_UNSUPPORTED` | 寫了 `SELECT(…)` 或四輸入的 `REC2` | `SELECT` 改寫 `c ? a : b`；`REC2` 用 `SMOOTH` 或兩輸入的組法 |
| `APX_ENGINE_LIMIT` | 展開後超過 128 個節點 | 回應帶實際節點數；標「保」的保留函數算一個節點，可用它取代一段原子 |
| `APX_CYCLE` | 自訂函數互相引用成環，或巢狀超過 128 層 | 拆開引用 |
| `APX_FIELD` | 定義檔欄位不對（`name`、`version`、`schema`、`body`、`out_unit`） | `name`＝`user.<作者小寫>.<大寫名>`；`version` ≥ 1；`out_unit` 用單位名 |
| `APX_CONFLICT` | 同名同版本但內容不同 | `version` +1 再安裝 |
| `APX_INVALID_DEFINITION` | `formulas` 裡有公式驗不過 | 回應逐條列出哪一條、為什麼 |

幾條常見的訊息（代碼是上面其中之一）：

| 訊息的意思 | 怎麼改 |
|---|---|
| 根節點的單位是 BOOL：排名要的是分數 | 當篩選條件寫 `MASK(條件, 分數)`；用成立次數排名寫 `COUNT(20, 條件)` |
| 這條要回看 N 個交易日，這張面板最多 M 天 | 縮短視窗，或 `config={"lookback_div": 2}` |
| `FN_EP`／`FN_BP`／`FN_DY` 的第一個輸入要是每股值 | 用 `eps`、`dps` 或 `FN_PS(總額, shares)`；總額比市值寫 `RATIO(總額, fund.mktcap)` |
| 兩個輸入一個是每股值、一個是公司總額 | 兩邊換成同一種 |
| `FN_DY` 的第一個輸入要是事件欄 | 用只在除息日有值的 `dps`；向前填的欄加總會重複算 |
| `EV_PULSE` 的子節點要是欄 | 直接放欄名；算式的事件用 `EV_CHANGE` 或比較 |
| `"字串"` 只能寫在 `in {…}` 裡或代碼欄 `==`／`!=` 的右邊 | `symbol == "2330"`、`symbol in {"2330"}` |
| `{…}` 只能當 `CORR_SEED` 的第一個引數，或寫在 `in` 後面 | |
| 集合成員寫成 `x in {v1, v2, …}` | 不寫 `IN_SET(…)` |
| `MOD` 的兩邊不能都是常數 | 至少一邊是欄或算式 |
| 常數太精細 | 大常數只收 7 位有效數字 |
| 條件名詞放進分數公式 | 條件名詞只用在 `universe` 或 `portfolio.hold` |
| `CORR_GROUP`／`CORR_SEED` 的輸入不能只是常數；不能巢狀 | 輸入寫一條用面板欄的數值算式 |

### 14.2 條件、選股範圍、回測

| 代碼 | 什麼時候 | 怎麼改 |
|---|---|---|
| `E_NOT_BOOL` | 條件的結果不是 BOOL | 寫成比較：`close > 0`、`ROLL_MEAN(20, volume) > 1000000`、`industry in {24, 25}` |
| `E_UNKNOWN_CODE` | 代碼欄沒有這個名字 | 產業寫代碼數字、交易所寫 `TW`／`TWO`、代號用雙引號 |
| `E_AMBIGUOUS_NAME` | 不帶前綴的欄名對到兩個以上的欄 | 寫完整欄名 |
| `E_EMPTY_UNIVERSE` | 條件下每天都沒有股票，或平均每天不到 1 檔 | 放寬門檻；先看產業代碼與欄名 |
| `APX_INVALID_ARGUMENT` | 選股範圍逐檔列超過 20 檔代號 | 改用類別名詞（`!cls.FIN()`）或種子籃子 |
| `E_NOT_MARKET_WIDE` | `rebalance_when`／`exposure` 同一天有兩個以上的值 | 只用每檔同值的東西：日曆、`idx.*`、`MKT_MEAN`、`EW_INDEX`；逐檔的條件放 `hold` |
| `E_NO_REBALANCE` | `rebalance_when` 整段期間沒有一天成立 | 檢查條件 |
| `E_NO_WEIGHT` | `weight` 整段期間沒有一格是正數 | 權重要 > 0 的檔才會被持有 |
| `E_POS_RULE` | `pos.*` 沒放在比較裡、不是一次式、另一邊不是常數或只看市場的算式；`pos.rank` 用在 `exit_when` | 照 9.5 的寫法規則；例 `pos.ret <= -0.08 \|\| pos.ret >= 0.2` |
| `E_RANGE` | `reentry_days` 不在 0..10,000；`group_cap` 不是正整數 | |
| `E_POS_GROUP` | `group_by`、`group_cap` 只給了一個；`group_by` 用了 `pos.*` | 兩個一起給，例 `{"group_by":"CORR_GROUP(RET(1, close))","group_cap":3}`；群鍵只看當天的資料 |
| `E_POS_SIZE` | 加減碼或帳戶層參數超出範圍或缺搭配（例 `size_init` 沒配 `add_when`、`gross_max` > 1 沒給 `margin_rate`） | 照 9.2 的值域與 9.5b 的例子 |
| `APX_INVALID_ARGUMENT` | 三選一的換股參數給了兩個；`weight` 與 `weighting` 都給 | 持有滿 N 天寫 `exit_when="pos.days >= N"` |
| `APX_INVALID_ARGUMENT` | `portfolio` 裡有不認得的鍵 | 回應列出全部合法的參數名；寫成別的系統的鍵名（`stop_loss`、`top_n`…）時說這件事在這裡的參數與例子 |
| `APX_CHECK_E_SCORE_COND_TIMES` | 分數的根是「條件 × 數值」 | 回應附整條改好的 `MASK(條件, 數值)` |
| `APX_CHECK_E_SCORE_ALL_INVALID` | 最後一年選股範圍內每一天都沒有任何一檔有分數 | 回應指出從哪一段開始沒值與改法：這張面板沒有的群層級 → 改成有的最上面那一層；欄沒資料；重置條件從不成立 → `SCANMAX(DAY_INDEX() < 1, x)`；`RWIN` 期數不夠 |
| `APX_CHECK_E_SCORE_RANK_TIES` | 分數含群內名次（`CS_PEER_RANK`、帶群的 `GRANK`、`GPCT`），每個抽樣日並列最高分的檔數都超過要選的檔數，又沒給 `tie_by` | 給 `portfolio.tie_by`（被排名的那個值）；`CS_PEER_RANK` 也可改 `CS_REL_PARENT(L, x)` |
| `APX_CONFLICT` | 結果庫綁的選股範圍、期間、單檔上限或衍生欄清單跟這次不同 | 換庫名；回應附建議的庫名 |

### 14.3 篩選與檢驗

| 代碼 | 什麼時候 | 怎麼改 |
|---|---|---|
| `E_UNKNOWN_NAME` | 不是原子、序列、形式輸入或定義式 | 回應附「你是不是要…」；`stock_formula_help topic=metric` |
| `E_UNKNOWN_FN` | 不認得的函式；篩選槽寫了分數公式的大寫名字（`ROLL_MEAN(3, r)`、`WSUM`） | 11.1 的函式表；回應附這個槽的寫法（2.8） |
| `E_FN_AS_VALUE` | 函式型定義式沒加括號當值用 | `apx.SHARPE_OF(r)` |
| `E_ARITY`、`E_ARGS` | 引數個數不對 | |
| `E_CYCLE` | 定義式互相引用成環 | |
| `E_CHECK_STAGE` | `check.*` 用在 `screen_pick`／`grid_by`，或門檻用了 `check.*` 卻沒開檢驗 | 寫在門檻 `screen_gate`、`checks` 開那個檢驗 |
| `E_EVOLVE_ARGS` | `apx.EVOLVE` 不是一個引數 | 回應的 `fix` 是改好的整條（兩個以上各包一個） |
| `E_EVOLVE_EMPTY` | `apx.EVOLVE` 包的東西沒有鄰居（BOOL 常數、0、面板沒有第二個同類的欄、格點上回看守得住的只有它） | 回應的 `fix` 是往外包一層的整條 |
| `E_EVOLVE_WINDOW` | 投組規則裡包了算子的參數（要看幾天） | 回應的 `fix` 是 `$名字`＋`grid` 的寫法；或把那個視窗搬進分數公式 |
| `E_RESERVED`、`E_NAMESPACE`、`E_NAME` | 定義檔的名字不合 | `user.<作者>.NAME`；`apx.*` 是保留的 |
| `E_DUP`、`E_HASH` | 同名同版本但內容不同 | `version` +1 |
| `APX_CHECK_*` | 檢驗定義檔的錯（`E_EXPR`、`E_UNIT`、`E_PARAM`、`E_INTERVAL`、`E_WINDOWS`、`E_VARY`、`E_RESOURCE`、`E_CHECK_ATOM_PLACE`（變換放錯地方）、`E_CHECK_LANG`（`with` 寫成小寫篩選語言）、`E_RET_MIX`、`E_WITH_TYPE`（`with` 的一項不是條件；報酬要包 `RETURNS(…)`）、`E_WITH_TWICE`（一個 run 換了兩次報酬）、`E_WITH_UNKNOWN`（沒有這個變換，附最接近的名字）、`E_RET_UNIT`（`RETURNS` 的算式不是 RATIO）、`E_UNIVERSE_EMPTY`（範圍平均每天不到 1 檔）、`E_RES_LIMIT`（一輪超過 2 種範圍或 2 種報酬，附已用的）、`E_DAYS_FIELD`（帶日子變換的 run 讀了跟部位有關的欄）、`E_OUTPUT`、`E_PASS`…加上 `APX_CHECK_` 前綴） | 回應帶路徑（例 `runs[1].interval`）、位置與建議 |

---

## 15. 還不能寫的

下面這些功能已經定案、還不能寫；寫了 plan 會回錯。上線後寫進前面對應的章。

| 功能 | 白話 |
|---|---|
| 換股日的漲跌停鎖死 | 換股那天跌停賣不掉的續抱、漲停買不到的留現金（盤中觸價與收盤規則出場的鎖死順延已經能用，見 9.6a 的 `lock`） |
| 分 K | 盤中某個時點進出場、用分 K 算的欄 |
| 三、四輸入的原子 | `SELECT`、`REC2`、`STATESCAN` 直接寫在公式裡（現在寫 `c ? a : b`） |
| `CORR` 長視窗 | `CORR` 的 N 放寬到 1,260（現在 3..252） |

---

## 16. 附錄：完整範例

每題是一組完整的 `stock_backtest` 引數（題號同範例庫 `examples.json`）。題庫六類各挑幾題。

### 16.1 選股範圍（U）

**U02　只看上櫃，而且排除上市不到 60 個交易日的新股**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "universe": "market == TWO && cal.tday_idx - list.tday_idx >= 60"}
```

**U12　收盤剛站上 250 日均線（20 天內才站上）的才選**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "universe": "close > ROLL_MEAN(250, close) && COUNT(20, close > ROLL_MEAN(250, close)) < 20"}
```

**U16　只選市值前 30%，但不要台積電、鴻海**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "universe": "CS_PCT(fund.mktcap) >= 0.7 && !(symbol in {\"2330\", \"2317\"})"}
```

### 16.2 排名（R）

**R01　資本額相近的互比：市值切 10 組，組內比 20 日漲幅**

```json
{"trees_from": "formulas",
 "formulas": ["SUB(RET(20, close), RATIO(GSUM(CS_QCUT(10, fund.mktcap), RET(20, close)), GSUM(CS_QCUT(10, fund.mktcap), MASK(VALID(RET(20, close)), 1:RATIO))))"]}
```

**R04　季報公布後第 5 個交易日，照公布以來的漲幅排名**

```json
{"trees_from": "formulas",
 "formulas": ["MASK(LAG(5, EV_CHANGE(revenue)), RATIO(close, LATCH(EV_CHANGE(revenue), LAG(1, close))))"]}
```

**R13　毛利率連續 3 季上升的，照最近一季上升幅度排**

```json
{"trees_from": "formulas",
 "formulas": ["MASK(RATIO(fund.gross, revenue) > RWIN(1, 0, RATIO(fund.gross, revenue)) && RWIN(1, 0, RATIO(fund.gross, revenue)) > RWIN(2, 0, RATIO(fund.gross, revenue)) && RWIN(2, 0, RATIO(fund.gross, revenue)) > RWIN(3, 0, RATIO(fund.gross, revenue)), RWIN(1, 7, RATIO(fund.gross, revenue)))"]}
```

**R15　價值、品質、動能三個因子等權排名**

```json
{"trees_from": "formulas",
 "formulas": ["ADD(ADD(CS_PCT(FN_EP(eps, close)), CS_PCT(FN_ROE(net_income, equity))), CS_PCT(RET_SKIP(252, 21, close)))"]}
```

### 16.3 熱度與籌碼（H）

**H04　連續 3 天量增價漲**

```json
{"trees_from": "formulas",
 "formulas": ["MASK(COUNT(3, volume > LAG(1, volume) && close > LAG(1, close)) >= 3, RET(3, close))"]}
```

**H05　冷卻後再起：曾經爆量、之後至少 20 天量縮，今天再度爆量**

```json
{"trees_from": "formulas",
 "formulas": ["MASK(VOL_RATIO(20, volume) >= 3 && COUNT(20, LAG(1, VOL_RATIO(20, volume) >= 3)) == 0 && COUNT(120, VOL_RATIO(20, volume) >= 3) >= 2, VOL_RATIO(20, volume))"]}
```

**H03　法人同步買的一群：用外資買賣超的相關性分群，群近 5 日合計買超最多的優先**

```json
{"trees_from": "formulas", "formulas": ["GSUM(CORR_GROUP(foreign_net), WSUM(5, foreign_net))"]}
```

### 16.4 回測（B）

**B03　等權大盤跌破 200 日均線就降到五成持股**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "portfolio": {"max_positions": 10, "rebalance_days": 21,
               "exposure": "EW_INDEX(close) < ROLL_MEAN(200, EW_INDEX(close)) ? 0.5 : 1"}}
```

**B09　買進後跌 8% 停損、漲 20% 停利**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "portfolio": {"max_positions": 10, "rebalance_days": 21,
               "exit_when": "pos.ret <= -0.08 || pos.ret >= 0.2"}}
```

**B13　緩衝區：排進前 10 名才買、掉出前 30 名才賣**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "portfolio": {"max_positions": 10, "rebalance_days": 21, "keep_when": "pos.rank <= 30"}}
```

**B19　依 20 日均成交金額加權，但單檔最多 10%**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "portfolio": {"max_positions": 20, "rebalance_days": 21, "max_weight": 10,
               "weight": "ROLL_MEAN(20, px.turnover)"}}
```

### 16.5 篩選與檢驗（F）

**F04　每一年都要贏同範圍等權平均**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "screen_gate": "min(by_sum(year, r - m)) > 0"}
```

**F01　年化報酬 ÷ 最大回撤至少 1 才收**（先用 `stock_formula_help topic=validate_change` 安裝 11.4 的 `user.local.ROMAD`）

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "screen_gate": "user.local.ROMAD >= 1"}
```

**F09　同類股替換後報酬至少保留五成**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "checks": ["peer_swap"], "check_params": {"PEER_SWAP.keep_bp": 5000}}
```

### 16.6 奇怪的策略（W）

**W01　只買股號尾數跟今天日期尾數相同的股票**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "universe": "sym.code_num % 10 == MDAY() % 10"}
```

**W08　星期一跌最多的股票，星期二買**

```json
{"trees_from": "formulas", "formulas": ["MASK(WEEKDAY() == 0, NEG(RET(1, close)))"],
 "portfolio": {"max_positions": 10, "rebalance_when": "WEEKDAY() == 0"}}
```

**W12　過去 5 年每年 12 月都上漲的股票**

```json
{"trees_from": "formulas",
 "formulas": ["MASK(COUNT(1260, MONTH() == 1 && TDAY_OF_MONTH() == 1 && AT_MONTH_START(LAG(1, close)) > AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, close))))) >= 5, RET(20, close))"]}
```

**W14　用「月營收年增」的相關性分群，群內比股價 20 日漲幅，只看跟群平均落差最大的**

```json
{"trees_from": "formulas",
 "formulas": ["ABS(SUB(RET(20, close), RATIO(GSUM(CORR_GROUP(1260, 126, 21, 3000, 1, RWIN_PCT(12, revenue_m)), RET(20, close)), GSUM(CORR_GROUP(1260, 126, 21, 3000, 1, RWIN_PCT(12, revenue_m)), MASK(VALID(RET(20, close)), 1:RATIO)))))"]}
```

**R17　自己組 14 日 RSI，越低越前**（先安裝 12.1 的 `user.local.RSI`）

```json
{"trees_from": "formulas", "formulas": ["NEG(user.local.RSI(42, close))"]}
```
