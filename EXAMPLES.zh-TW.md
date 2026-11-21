# VQL 範例庫：白話 → 寫法

每一題是一組完整的回測引數（`stock_backtest` 的 `formulas`、`universe`，其餘放 `advanced`），照抄就能出計畫。機器讀的同一份在 `examples.json`。示範條件、回測規則、篩選或檢驗的題目，分數用 `RET(60, close)` 代替。每題只寫客戶原話講到的條件，沒講的不加。

VQL 做四件事，範例照這四件找：

| 事 | 寫在哪 | 範例 |
|---|---|---|
| 搜尋策略 | `formulas`（自己給幾條）、`trees_from=nouns`（自訂函數的參數格點）、`trees_from=sample`（系統隨機生成，在 MCP 走 `gpu_algo_stock_search`；`config.sample_new` 加抽大盤層與相關性群） | S |
| 回測策略 | 分數公式、`universe`、`portfolio`（換股、權重、出場、資金） | U、R、H、B、W |
| 篩選策略 | `screen_gate`、`screen_pick`、`screen_admit`、自訂指標 `apx.metric.v1` | F01～F05、F08、F11、P |
| 檢驗策略 | `checks`、`check_params`、自己組的 `apx.check.v1` | F06、F07、F09、F10、C |

## 搜尋：一次試很多個公式

**S01　這三個想法一起回測，各自出結果**

```json
{"formulas": ["RET_SKIP(252, 21, close)", "NEG(RVOL(60, close))", "FN_EP(eps, close)"]}
```

**S02　讓系統自己隨機生成 200 個公式來找**

```json
{"trees_from": "sample", "trees": 200, "seed": 7}
```

**S03　隨機生成時也抽大盤層的欄與表現相近的股票群**

```json
{"trees_from": "sample", "trees": 200, "seed": 7, "config": {"sample_new": 3}}
```

**S04　離 N 日高點的距離，N 試 60、120、250 三種**

```json
{"trees_from": "nouns", "nouns": [{"name": "user.local.NEAR_HIGH", "bind": {"x": "close"}, "grid": {"n": [60, 120, 250]}}]}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.noun.v1", "name": "user.local.NEAR_HIGH", "version": 1, "inputs": [{"name": "x", "unit": "PRICE"}], "params": [{"name": "n", "default": 250, "min": 2, "max": 1260}], "body": "RATIO(x, ROLL_MAX($n, x))", "out_unit": "RATIO"}
```

**S05　動能的回看天數讓系統在 60 天附近找**

```json
{"formulas": ["RET(apx.EVOLVE(60), close)"]}
```

**S06　停損幅度讓系統在 8% 附近找**

```json
{"formulas": ["RET_SKIP(252, 21, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.ret <= -apx.EVOLVE(0.08)"}}
```

**S07　同一個動能，價格欄換別的同類欄試試**

```json
{"formulas": ["CS_PCT(RET(60, apx.EVOLVE(close)))"]}
```

## 選股範圍

**U03　過去 20 天每天都有成交、20 日均量超過 500 張的才選**

```json
{"formulas": ["RET(60, close)"], "universe": "COUNT(20, volume > 0) >= 20 && ROLL_MEAN(20, volume) > 500000"}
```

**U04　只選中型股：市值排在全市場第 30～70 百分位之間**

```json
{"formulas": ["RET(60, close)"], "universe": "CS_PCT(fund.mktcap) >= 0.3 && CS_PCT(fund.mktcap) <= 0.7"}
```

**U06　20 日平均週轉率在 1%～5% 之間**

```json
{"formulas": ["RET(60, close)"], "universe": "ROLL_MEAN(20, px.turnover_rate) >= 0.01 && ROLL_MEAN(20, px.turnover_rate) <= 0.05"}
```

**U07　排除金融與營建，只要普通股**

```json
{"formulas": ["RET(60, close)"], "universe": "!(industry in {17, 14})", "common_only": true}
```

**U08　只在我給的 10 檔自選股裡面挑**

```json
{"formulas": ["RET(60, close)"], "universe": "symbol in {\"2330\", \"2317\", \"2454\", \"3711\", \"2382\", \"2886\", \"2881\", \"2882\", \"2891\", \"3231\"}"}
```

**U09　跟台積電、聯發科、日月光三檔過去一年日報酬相關性平均超過 0.5 的股票才選**

```json
{"formulas": ["RET(60, close)"], "universe": "ADD(ADD(CORR(250, RET(1, close), BASKET_MEAN(symbol == \"2330\", RET(1, close))), CORR(250, RET(1, close), BASKET_MEAN(symbol == \"2454\", RET(1, close)))), CORR(250, RET(1, close), BASKET_MEAN(symbol == \"3711\", RET(1, close)))) > 1.5"}
```

**U10　跟我給的三檔種子股，月營收年增序列相關性高的才選**

```json
{"formulas": ["RET(60, close)"], "universe": "CORR_SEED({\"2330\", \"2454\", \"3711\"}, 1260, 126, 21, 1, 0, RWIN_PCT(12, revenue_m)) > 0.5"}
```

**U11　近一年最大跌幅小於 40% 的才選**

```json
{"formulas": ["RET(60, close)"], "universe": "WDROP(250, LOG(close)) < 0.5108"}
```

**U12　收盤剛站上 250 日均線（20 天內才站上）的才選**

```json
{"formulas": ["RET(60, close)"], "universe": "close > ROLL_MEAN(250, close) && COUNT(20, close > ROLL_MEAN(250, close)) < 20"}
```

**U13　融資使用率低於 30%、券資比高於 10% 的才選**

```json
{"formulas": ["RET(60, close)"], "universe": "RATIO(margin_bal, margin_limit) < 0.3 && RATIO(short_bal, margin_bal) > 0.1"}
```

**U16　只選市值前 30%，但不要台積電、鴻海**

```json
{"formulas": ["RET(60, close)"], "universe": "CS_PCT(fund.mktcap) >= 0.7 && !(symbol in {\"2330\", \"2317\"})"}
```

**U18　只選外資已經連買 3 天以上的股票**

```json
{"formulas": ["RET(60, close)"], "universe": "STREAK(foreign_net > 0) >= 3"}
```

**U19　月營收創一年新高之後 20 個交易日內的才選**

```json
{"formulas": ["RET(60, close)"], "universe": "SINCE(EV_CHANGE(revenue_m) && revenue_m >= ROLL_MAX(250, revenue_m)) <= 20"}
```

**U20　股價（當天看盤價）介於 50 元到 300 元之間的才選**

```json
{"formulas": ["RET(60, close)"], "universe": "BETWEEN(raw.close, 50, 300)"}
```

**U21　收盤剛站上 20 日均線那天（黃金交叉）的才選**

```json
{"formulas": ["RET(60, close)"], "universe": "CROSS_UP(close, ROLL_MEAN(20, close))"}
```

**U22　收盤高於 20 日成交均價的才選**

```json
{"formulas": ["RET(60, close)"], "universe": "raw.close > VWAP(20)"}
```

## 排名

**R01　資本額相近的互比：市值切 10 組，組內比 20 日漲幅**

```json
{"formulas": ["SUB(RET(20, close), RATIO(GSUM(CS_QCUT(10, fund.mktcap), RET(20, close)), GSUM(CS_QCUT(10, fund.mktcap), MASK(VALID(RET(20, close)), 1:RATIO))))"]}
```

**R02　表現相近的互比：用日報酬相關性分群，群內比 60 日漲幅**

```json
{"formulas": ["MASK(VALID(GRANK(CORR_GROUP(LOGR(close, LAG(1, close))), RET(60, close))), SUB(RET(60, close), RATIO(GSUM(CORR_GROUP(LOGR(close, LAG(1, close))), RET(60, close)), GSUM(CORR_GROUP(LOGR(close, LAG(1, close))), MASK(VALID(RET(60, close)), 1:RATIO)))))"]}
```

**R03　兩層疊加：產業內 20 日漲幅名次、市值組內 20 日漲幅名次，兩個名次平均**

```json
{"formulas": ["ADD(CS_PEER_RANK(1, RET(20, close)), RATIO(GRANK(CS_QCUT(10, fund.mktcap), RET(20, close)), ADD(SUB(GCOUNT(CS_QCUT(10, fund.mktcap), RET(20, close)), 1:COUNT), SUB(GCOUNT(CS_QCUT(10, fund.mktcap), RET(20, close)), 1:COUNT))))"]}
```

**R04　季報公布後第 5 個交易日，照公布以來的漲幅排名**

```json
{"formulas": ["MASK(LAG(5, EV_CHANGE(revenue)), RATIO(close, LATCH(EV_CHANGE(revenue), LAG(1, close))))"]}
```

**R05　上次月營收創 12 個月新高以來，股價最高點到今天的回落幅度，回落越少越前**

```json
{"formulas": ["RATIO(close, SCANMAX(EV_CHANGE(revenue_m) && revenue_m >= ROLL_MAX(250, revenue_m), close))"]}
```

**R06　個股 20 日漲幅減掉同產業平均漲幅（個股對群的相對強度）**

```json
{"formulas": ["CS_REL_PARENT(1, RET(20, close))"]}
```

**R07　所屬產業的 20 日平均漲幅減掉全市場平均（群對市場），群裡每檔拿同一個分數**

```json
{"formulas": ["SUB(XL_PARENT(1, RET(20, close)), RATIO(GSUM(0, RET(20, close)), GSUM(0, MASK(VALID(RET(20, close)), 1:RATIO))))"]}
```

**R08　等權大盤在 200 日均線之上用 60 日動能，之下改用 60 日低波動**

```json
{"formulas": ["idx.taiex > ROLL_MEAN(200, idx.taiex) ? CS_PCT(RET(60, close)) : CS_PCT(NEG(ROLL_STD(60, LOGR(close, LAG(1, close)))))"]}
```

**R09　20 日漲幅平方乘上量比，再取對數**

```json
{"formulas": ["LOG(MUL(MUL(RET(20, close), RET(20, close)), VOL_RATIO(20, volume)))"]}
```

**R10　財報月（3、5、8、11 月）照盈餘成長排，其他月照 60 日動能排**

```json
{"formulas": ["MONTH() in {3, 5, 8, 11} ? CS_PCT(FN_EGROW(252, eps)) : CS_PCT(RET(60, close))"]}
```

**R11　每月前 5 個交易日用短線反轉，其他日子用 20 日動能**

```json
{"formulas": ["TDAY_OF_MONTH() <= 5 ? CS_PCT(NEG(RET(5, close))) : CS_PCT(RET(20, close))"]}
```

**R12　離 52 週高點拉回 5%～10% 的給最高分，拉回更多或更少都扣分（分段函數）**

```json
{"formulas": ["(DIST_HIGH(250, close) <= -0.05 && DIST_HIGH(250, close) >= -0.1) ? 0:RATIO : NEG(MIN2(ABS(ADD(DIST_HIGH(250, close), 0.05:RATIO)), ABS(ADD(DIST_HIGH(250, close), 0.1:RATIO))))"]}
```

**R13　毛利率連續 3 季上升的，照最近一季上升幅度排**

```json
{"formulas": ["MASK(RATIO(fund.gross, revenue) > RWIN(1, 0, RATIO(fund.gross, revenue)) && RWIN(1, 0, RATIO(fund.gross, revenue)) > RWIN(2, 0, RATIO(fund.gross, revenue)) && RWIN(2, 0, RATIO(fund.gross, revenue)) > RWIN(3, 0, RATIO(fund.gross, revenue)), RWIN(1, 7, RATIO(fund.gross, revenue)))"]}
```

**R14　營收年增的產業內百分位，減掉股價年漲幅的產業內百分位（基本面跑贏股價）**

```json
{"formulas": ["SUB(CS_PEER_RANK(1, RWIN_PCT(12, revenue_m)), CS_PEER_RANK(1, RET(250, close)))"]}
```

**R15　價值、品質、動能三個因子等權排名（本益比倒數、ROE、12 個月動能）**

```json
{"formulas": ["ADD(ADD(CS_PCT(FN_EP(eps, close)), CS_PCT(FN_ROE(net_income, equity))), CS_PCT(RET_SKIP(252, 21, close)))"]}
```

**R17　自己組 14 日 RSI，越低越前**

```json
{"formulas": ["NEG(user.local.RSI(42, close))"]}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.noun.v1", "requires": "apx-std>=1.0", "name": "user.local.RSI", "version": 1, "description": "Wilder RSI（α = 1/14）", "inputs": [{"name": "x", "unit": "PRICE"}], "params": [{"name": "k", "default": 42, "min": 0, "max": 756}], "body": "RATIO(SMOOTH($k, 0.071429:RATIO, MAX2(DIFF(1, x), 0:PRICE)), ADD(SMOOTH($k, 0.071429:RATIO, MAX2(DIFF(1, x), 0:PRICE)), SMOOTH($k, 0.071429:RATIO, MAX2(NEG(DIFF(1, x)), 0:PRICE))))", "out_unit": "RATIO"}
```

**R18　外資連續買超的天數，越多越前**

```json
{"formulas": ["STREAK(foreign_net > 0)"]}
```

**R19　外資持股比率 60 日增加幅度**

```json
{"formulas": ["DIFF(60, foreign_ratio)"]}
```

**R21　低本益比，但只在同產業裡比（產業內本益比倒數百分位）**

```json
{"formulas": ["CS_REL_PARENT(1, FN_EP(eps, close))"]}
```

**R22　股價對 5 年均線的偏離，看它一年來的變化**

```json
{"formulas": ["DIFF(250, RATIO(close, ROLL_MEAN(1260, close)))"], "config": {"lookback_div": 2}}
```

**R24　強勢產業的股票優先，同一個產業裡市值大的先選（分數一樣時的第二排序）**

```json
{"formulas": ["SUB(XL_PARENT(1, RET(20, close)), RATIO(GSUM(0, RET(20, close)), GSUM(0, MASK(VALID(RET(20, close)), 1:RATIO))))"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "tie_by": "fund.mktcap"}}
```

**R25　跟台積電日報酬相關性越高排越前面**

```json
{"formulas": ["CORR(250, RET(1, close), BASKET_MEAN(symbol == \"2330\", RET(1, close)))"]}
```

**R26　市值分十組，組內比 20 日漲幅（比同組平均高多少）**

```json
{"formulas": ["GDEMEAN(CS_QCUT(10, fund.mktcap), RET(20, close))"]}
```

**R27　上個月漲最多的**

```json
{"formulas": ["PREV_MONTH_RET(close)"]}
```

**R28　近 20 天外資平均買超（沒資料的日子不算）**

```json
{"formulas": ["ROLL_MEAN_AVAIL(20, foreign_net)"]}
```

**R29　每天漲跌封頂 ±10% 之後的 20 日平均漲幅**

```json
{"formulas": ["ROLL_MEAN(20, CLIP(RET(1, close), -0.1, 0.1))"]}
```

**R30　同產業的平均 20 日漲幅（產業動能）**

```json
{"formulas": ["GMEAN(1, RET(20, close))"]}
```

**R31　過去 20 天上漲天數減下跌天數**

```json
{"formulas": ["WSUM(20, SIGN(RET(1, close)))"]}
```

**R32　動能的回看天數讓系統試：半年（126 天）和一年（252 天），都跳過最近一個月**

```json
{"formulas": ["RET_SKIP($n, 21, close)"], "portfolio": {"grid": {"n": [126, 252]}}}
```

**R33　自己組 MACD 柱（12、26、9），柱子佔股價越大越前**

```json
{"formulas": ["RATIO(user.local.MACD_HIST(78, close), close)"]}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.noun.v1", "requires": "apx-std>=1.0", "name": "user.local.MACD_HIST", "version": 1, "description": "MACD 柱 = DIF − DIF 的 EMA9；DIF = EMA12 − EMA26（α = 2/13、2/27、0.2）；單位：價格", "inputs": [{"name": "x", "unit": "PRICE"}], "params": [{"name": "k", "default": 78, "min": 0, "max": 756}], "body": "SUB(SUB(SMOOTH(0, 0.153846:RATIO, x), SMOOTH(0, 0.074074:RATIO, x)), SMOOTH($k, 0.2:RATIO, SUB(SMOOTH(0, 0.153846:RATIO, x), SMOOTH(0, 0.074074:RATIO, x))))", "out_unit": "PRICE"}
```

**R34　自己組 20 日布林 Z 值（離中線幾個標準差），越低越前**

```json
{"formulas": ["NEG(user.local.BOLL_Z(20, close))"]}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.noun.v1", "requires": "apx-std>=1.0", "name": "user.local.BOLL_Z", "version": 1, "description": "布林 Z = (x − n 日均) / n 日標準差（母體）；±2 = 布林上下緣；單位：倍數", "inputs": [{"name": "x", "unit": "PRICE"}], "params": [{"name": "n", "default": 20, "min": 2, "max": 1260}], "body": "RATIO(SUB(x, ROLL_MEAN($n, x)), ROLL_STD($n, x))", "out_unit": "RATIO"}
```

**R35　自己組 9 日 KD 的 K 值，越低越前**

```json
{"formulas": ["NEG(user.local.KD_K(9, 27, high, low, close))"]}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.noun.v1", "requires": "apx-std>=1.0", "name": "user.local.KD_K", "version": 1, "description": "KD 的 K：RSV = (c − n 日最低) / (n 日最高 − n 日最低)，K = RSV 的 1/3 平滑；0～1（×100 = 常見刻度）；D = SMOOTH(k, 0.333333:RATIO, K)", "inputs": [{"name": "h", "unit": "PRICE"}, {"name": "l", "unit": "PRICE"}, {"name": "c", "unit": "PRICE"}], "params": [{"name": "n", "default": 9, "min": 2, "max": 252}, {"name": "k", "default": 27, "min": 0, "max": 756}], "body": "SMOOTH($k, 0.333333:RATIO, RATIO(SUB(c, ROLL_MIN($n, l)), SUB(ROLL_MAX($n, h), ROLL_MIN($n, l))))", "out_unit": "RATIO"}
```

**R36　自己組 14 日 ATR，佔股價比例越低（波動越小）越前**

```json
{"formulas": ["NEG(RATIO(user.local.ATR(42, high, low, close), close))"]}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.noun.v1", "requires": "apx-std>=1.0", "name": "user.local.ATR", "version": 1, "description": "Wilder ATR（α = 1/14）；真實區間 = max(h − l, |h − 昨收|, |l − 昨收|)；單位：價格", "inputs": [{"name": "h", "unit": "PRICE"}, {"name": "l", "unit": "PRICE"}, {"name": "c", "unit": "PRICE"}], "params": [{"name": "k", "default": 42, "min": 0, "max": 756}], "body": "SMOOTH($k, 0.071429:RATIO, MAX2(SUB(h, l), MAX2(ABS(SUB(h, LAG(1, c))), ABS(SUB(l, LAG(1, c))))))", "out_unit": "PRICE"}
```

## 熱度與籌碼

**H01　今天成交量是 20 日均量 3 倍以上的才給分，照漲幅排**

```json
{"formulas": ["MASK(VOL_RATIO(20, volume) >= 3, RET(1, close))"]}
```

**H02　資金同步湧入的一群：用成交量的相關性分群，群平均量比放大最多那群裡的股票優先**

```json
{"formulas": ["RATIO(GSUM(CORR_GROUP(volume), VOL_RATIO(20, volume)), GSUM(CORR_GROUP(volume), MASK(VALID(VOL_RATIO(20, volume)), 1:RATIO)))"]}
```

**H03　法人同步買的一群：用外資買賣超的相關性分群，群近 5 日合計買超最多的優先**

```json
{"formulas": ["GSUM(CORR_GROUP(foreign_net), WSUM(5, foreign_net))"]}
```

**H04　連續 3 天量增價漲**

```json
{"formulas": ["MASK(COUNT(3, volume > LAG(1, volume) && close > LAG(1, close)) >= 3, RET(3, close))"]}
```

**H05　冷卻後再起：曾經爆量、之後至少 20 天量縮，今天再度爆量**

```json
{"formulas": ["MASK(VOL_RATIO(20, volume) >= 3 && COUNT(20, LAG(1, VOL_RATIO(20, volume) >= 3)) == 0 && COUNT(120, VOL_RATIO(20, volume) >= 3) >= 2, VOL_RATIO(20, volume))"]}
```

**H07　融資餘額 5 日暴增但股價沒漲（散戶追不動）**

```json
{"formulas": ["MASK(RET(5, margin_bal) > 0.2 && RET(5, close) <= 0, RET(5, margin_bal))"]}
```

**H08　投信連續買超第 1 天（之前 10 天都沒買）**

```json
{"formulas": ["MASK(FIRST_IN_WINDOW(11, trust_net > 0), RATIO(trust_net, volume))"]}
```

## 回測

**B01　每個月第一個交易日換股**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "MONTH_START()"}}
```

**B02　等權大盤站上 60 日均線那天才換股，其他時候不動**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "EW_INDEX(close) > ROLL_MEAN(60, EW_INDEX(close)) && !LAG(1, EW_INDEX(close) > ROLL_MEAN(60, EW_INDEX(close)))"}}
```

**B03　等權大盤跌破 200 日均線就降到五成持股**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exposure": "EW_INDEX(close) < ROLL_MEAN(200, EW_INDEX(close)) ? 0.5 : 1"}}
```

**B04　依全市場 20 日波動反比調整持股（波動越大持股越少，最多滿倉）**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exposure": "MIN2(RATIO(0.01:RATIO, ROLL_STD(20, FILL(MKT_MEAN(LOGR(close, LAG(1, close))), 0:RATIO))), 1:RATIO)"}}
```

**B05　等權大盤從一年高點回落超過 15% 時只持有三成**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exposure": "RATIO(EW_INDEX(close), ROLL_MAX(250, EW_INDEX(close))) < 0.85 ? 0.3 : 1"}}
```

**B06　依市值開根號加權**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "weight": "SQRT(fund.mktcap)"}}
```

**B07　收盤跌破 20 日均線的不再持有**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "hold": "close >= ROLL_MEAN(20, close)"}}
```

**B08　每天選，每檔持有 10 個交易日到期出場**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "hold_days": 10}}
```

**B09　買進後跌 8% 停損、漲 20% 停利**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_below": "pos.entry * 0.92", "exit_above": "pos.entry * 1.2"}}
```

**B10　表現相近的同一群最多持有 3 檔**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "group_by": "CORR_GROUP(RET(1, close))", "group_cap": 3}}
```

**B11　依分數加權，同時每檔持有 10 天**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "hold_days": 10, "weight": "CS_PCT(RET(60, close))"}}
```

**B12　金字塔加碼：已持有的股票再漲 10% 就加碼一倍**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "add_when": "pos.ret >= 0.1", "size_init": 0.5}}
```

**B12b　買進後跌 5% 補一次（只補一次）**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "add_when": "pos.ret <= -0.05 && pos.adds == 0", "size_init": 0.5}}
```

**B12c　漲 20% 先賣一半，剩下的從高點回落 10% 全出**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "cut_when": "pos.ret >= 0.2", "exit_when": "pos.drawdown >= 0.1"}}
```

**B12d　最多投入八成資金、每次換股最多新買 3 檔**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "gross_max": 0.8, "max_new": 3}}
```

**B13　緩衝區：排進前 10 名才買、掉出前 30 名才賣**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "keep_when": "pos.rank <= 30"}}
```

**B14　只在財報月（3、5、8、11 月）的第一個交易日換股**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "MONTH() != LAG(1, MONTH()) && MONTH() in {3, 5, 8, 11}"}}
```

**B15　每週五換股**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "WEEKDAY() == 4"}}
```

**B16　每月定期投入 5,000 元，持有 10 檔，零股**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "initial_capital": 5000, "contribution": 5000, "contribution_every": "month", "lot": "odd"}}
```

**B17　最多 10 檔、單檔上限 15%、依波動倒數加權**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "max_weight": 15, "weighting": "inv_vol"}}
```

**B18　追蹤停損：從買進後最高點回落 10% 就出場**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.drawdown >= 0.1"}}
```

**B19　依 20 日均成交金額加權，但單檔最多 10%**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 20, "rebalance_days": 21, "max_weight": 10, "weight": "ROLL_MEAN(20, px.turnover)"}}
```

**B23　收盤跌破買進價 8% 或收盤漲過 20% 才賣（收盤確認）**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.ret <= -0.08 || pos.ret >= 0.2"}}
```

**B24　盤中跌破 20 日均線就賣**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_below": "ROLL_MEAN(20, close)"}}
```

**B25　跌 8% 或從最高收盤回落 10%，先碰到的就賣**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_below": "MAX2(pos.entry * 0.92, pos.peak * 0.9)"}}
```

**B20　停損停利幫我試幾組：停損 5%、8%、10%，停利 15%、20%、30%**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.ret <= -$stop || pos.ret >= $take", "grid": {"stop": [0.05, 0.08, 0.1], "take": [0.15, 0.2, 0.3]}}}
```

**B21　從最高點回落幾 % 出場，6% 到 14% 讓系統自己找**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.drawdown >= $dd", "grid": {"dd": {"from": 0.06, "to": 0.14, "step": 0.02}}}}
```

**B22　跌破幾倍波動就停損、最多抱幾天，兩個一起試**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.ret < -$k * ROLL_STD(20, RET(1, close)) || pos.days >= $n", "grid": {"k": [1.5, 2, 3], "n": [20, 40]}}}
```

**B26　盤中停損停利的幅度讓系統試：跌到買進價的 90%、92%、95%，漲到 115%、125%**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_below": "pos.entry * $floor", "exit_above": "pos.entry * $cap", "grid": {"floor": [0.9, 0.92, 0.95], "cap": [1.15, 1.25]}}}
```

**B27　每天買前 10 名、每檔抱 10 天，期間跌 8% 先停損**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "hold_days": 10, "exit_below": "pos.entry * 0.92"}}
```

**B28　成交金額切五等分，每一等分最多買 2 檔**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "group_by": "CS_QCUT(5, ROLL_MEAN(20, px.turnover))", "group_cap": 2}}
```

**B29　同一群最多 2 檔，而且掉出前 30 名才賣**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 5, "keep_when": "pos.rank <= 30", "group_by": "CORR_GROUP(RET(1, close))", "group_cap": 2}}
```

**B30　全市場站上 60 日均線的股票不到三成時只持有五成**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "exposure": "MKT_SHARE(close > ROLL_MEAN(60, close)) < 0.3 ? 0.5 : 1"}}
```

**B31　上漲家數多於下跌家數那天才換股**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "MKT_COUNT(close > LAG(1, close)) > MKT_COUNT(close < LAG(1, close))"}}
```

**B32　每季第一個交易日換股**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "QUARTER_START()"}}
```

**B33　每年第一個交易日換股**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "YEAR_START()"}}
```

**B34　站上年線之後才持有、跌破季線才賣**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "hold": "STATE(close > ROLL_MEAN(250, close), close < ROLL_MEAN(60, close))"}}
```

**B35　動能天數試 60、120 天，停損試 8%、10%，一起找**

```json
{"formulas": ["RET($n, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.ret <= -$stop", "grid": {"n": [60, 120], "stop": [0.08, 0.1]}}}
```

**B36　每週第一個交易日換股（週一放假就週二），2020 年起才開始選**

```json
{"formulas": ["MASK(YEAR() >= 2020, RET(60, close))"], "portfolio": {"max_positions": 10, "rebalance_when": "WEEK_START()"}}
```

**B37　最多用到一倍半資金（融資），漲 4% 加碼一份，加碼超過上限只用剩下的額度**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "add_when": "pos.ret >= 0.04", "gross_max": 1.5, "margin_rate": 0.05}}
```

**B38　最多用到一倍半資金（融資），漲 4% 加碼一份，超過上限就把全部持股照比例縮**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "add_when": "pos.ret >= 0.04", "gross_max": 1.5, "margin_rate": 0.05, "add_mode": "scale"}}
```

**B39　每次換股新買的檔數最多是持有檔數的三成**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21}, "config": {"max_turnover_bp": 3000}}
```

## 篩選與內建檢驗

**F01　ROMAD（年化報酬 ÷ 最大回撤）至少 1 才收**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "user.local.ROMAD >= 1"}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.metric.v1", "requires": "apx-std>=1.1", "name": "user.local.ROMAD", "version": 1, "description": "年化報酬 ÷ 最大回撤", "inputs": [], "body": "apx.ANN_RET / apx.MAX_DD", "unit": "", "direction": "max", "group": "mine"}
```

**F02　月勝率至少六成**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "apx.MONTH_WIN >= 0.6"}
```

**F03　最長水下天數（從前高跌下到回到前高的最長天數）不超過 250 天**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "user.local.UNDERWATER <= 250"}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.metric.v1", "requires": "apx-std>=1.1", "name": "user.local.UNDERWATER", "version": 1, "description": "最長水下天數", "inputs": [], "body": "streak(drawdown_nav(r) > 0)", "unit": "days", "direction": "min", "group": "mine"}
```

**F04　每一年都要贏等權大盤**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "min(by_sum(year, r - m)) > 0"}
```

**F05　多頭和空頭市況下 Sharpe 都要大於 0.5**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "apx.SHARPE_BULL > 0.5 && apx.SHARPE_BEAR > 0.5"}
```

**F06　交易成本加倍後淨報酬仍然要是正的**

```json
{"formulas": ["RET(60, close)"], "checks": ["cost_retest"], "check_params": {"COST_RETEST.mult": 2}}
```

**F07　前後兩半期間的 IC 要同號**

```json
{"formulas": ["RET(60, close)"], "checks": ["user.local.HALVES"]}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.check.v1", "name": "user.local.HALVES", "version": 1, "description": "前後兩半 IC 同號", "windows": {"kind": "split", "n": 2}, "runs": [{"id": "seg", "interval": "window.seg"}], "outputs": {"seg_ic": "seg.ic_days > 0 ? seg.ic_x1e4 : 0", "lo": "AMIN(seg_ic)", "hi": "AMAX(seg_ic)"}, "pass": "lo > 0 || hi < 0"}
```

**F08　最差的一年報酬不能低於 −20%**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "apx.RET_Y_MIN > -0.2"}
```

**F09　同類股替換後報酬至少保留五成**

```json
{"formulas": ["RET(60, close)"], "checks": ["peer_swap"], "check_params": {"PEER_SWAP.keep_bp": 5000}}
```

**F10　分 4 段，至少 3 段 IC 為正**

```json
{"formulas": ["RET(60, close)"], "checks": ["segments"], "check_params": {"SEGMENTS.n": 4, "SEGMENTS.min_pos_bp": 7500}}
```

**F11　近三年回測（2023 起）年化至少 10%**

```json
{"formulas": ["RET(60, close)"], "backtest_from": "2023-01-01", "screen_gate": "apx.ANN_RET >= 0.1"}
```

**F12　七成以上的年份要贏大盤指數（加權報酬指數）**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "apx.YEARS_BEAT_B >= 0.7"}
```

**F13　對大盤指數的年化超額報酬至少 5%**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "apx.EXCESS_B_ANN >= 0.05"}
```

**F14　任何一年的持有期都不能虧超過 15%**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "apx.ROLL_RET_MIN > -0.15"}
```

**F15　Sharpe 大於 1，而且同類股替換後報酬保留 26% 以上（一條門檻）**

```json
{"formulas": ["RET(60, close)"], "checks": ["peer_swap"], "screen_gate": "apx.SHARPE > 1 && check.PEER_SWAP.keep_ret_bp >= 2600"}
```

**F16　滾動半年的平均日報酬最差也要大於 0**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "MIN(ROLL_MEAN(126, r)) > 0"}
```

**F17　幾組投組設定裡照 Calmar 挑**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"grid_by": "apx.CALMAR"}}
```

## 檢驗後入庫

**P01　同類股替換後淨報酬至少保留六成才收**

```json
{"formulas": ["RET(60, close)"], "checks": ["peer_swap"], "screen_admit": "check.PEER_SWAP.keep_ret_bp >= 6000"}
```

## 自己組的檢驗

**C01　最近一年（最後 252 個交易日）IC 也要為正**

```json
{"formulas": ["RET(60, close)"], "checks": ["user.local.LAST_YEAR"]}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.check.v1", "name": "user.local.LAST_YEAR", "version": 1, "description": "最後 252 個交易日 IC 為正", "runs": [{"id": "late", "interval": {"begin": "env.n_days - 252", "end": "env.n_days"}}], "outputs": {"late_ic": "late.ic_days > 0 ? late.ic_x1e4 : 0"}, "pass": "late_ic > 0"}
```

**C02　每週換股和每月換股都要賺錢**

```json
{"formulas": ["RET(60, close)"], "checks": ["user.local.BOTH_FREQ"]}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.check.v1", "name": "user.local.BOTH_FREQ", "version": 1, "description": "每 5 日與每 21 日換股的淨報酬都 > 0", "runs": [{"id": "rb", "vary": {"rebalance_days": [5, 21]}}], "outputs": {"worst": "AMIN(rb.net_ret_1e6)"}, "pass": "worst > 0"}
```

**C03　只在隨機一半的股票裡選、成本又變三倍，還要賺錢**

```json
{"formulas": ["RET(60, close)"], "checks": ["user.local.HALF_X3"]}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.check.v1", "name": "user.local.HALF_X3", "version": 1, "description": "隨機一半股票、單邊成本 ×3 的淨報酬 > 0", "runs": [{"id": "h", "universe": "universe.random_half", "cost_bp": "prog.cost_bp * 3"}], "outputs": {"r": "h.net_ret_1e6"}, "pass": "r > 0"}
```

**C04　只持有 10 檔時也要賺錢，而且最大跌幅不超過 40%**

```json
{"formulas": ["RET(60, close)"], "checks": ["user.local.TEN"]}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.check.v1", "name": "user.local.TEN", "version": 1, "description": "只持有 10 檔的淨報酬 > 0、最大回撤 ≤ $max_dd_bp", "params": [{"name": "max_dd_bp", "default": 4000, "min": 0, "max": 10000}], "runs": [{"id": "n10", "max_positions": "10"}], "outputs": {"r": "n10.net_ret_1e6", "dd_bp": "n10.max_dd_1e6 / 100"}, "pass": "r > 0 && dd_bp <= $max_dd_bp"}
```

**C05　前後兩半各自用樣本內挑做多門檻、樣本外驗證，兩段都要賺**

```json
{"formulas": ["RET(60, close)"], "checks": ["user.local.WF2"]}
```

先驗（`stock_formula_help topic=validate`）再存自訂定義（`my_stock_strategies_change action=install`）：

```json
{"schema": "apx.check.v1", "name": "user.local.WF2", "version": 1, "description": "滾動兩段：IS 挑 p_long、OOS 淨報酬都 > 0", "windows": {"kind": "rolling", "is_days": 504, "oos_days": 252, "max": 2}, "runs": [{"id": "is", "interval": "window.is", "vary": {"p_long": [170, 180, 190]}}, {"id": "oos", "interval": "window.oos", "p_long": "WINNER(is, net_ret_1e6, p_long)"}], "outputs": {"oos_ret": "oos.net_ret_1e6", "ok": "ALL(oos_ret > 0)"}, "pass": "n_windows >= 2 && ok == 1"}
```

## 奇怪的策略

**W02　上個月漲最多的那個產業裡，這個月量縮最多的股票**

```json
{"formulas": ["MASK(XL_PARENT(1, RATIO(AT_MONTH_START(LAG(1, close)), AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, close)))))) >= GMAX(0, XL_PARENT(1, RATIO(AT_MONTH_START(LAG(1, close)), AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, close))))))), NEG(RATIO(RATIO(SCANSUM(TDAY_OF_MONTH() == 1, volume), AT_MONTH_START(LAG(1, SCANSUM(TDAY_OF_MONTH() == 1, volume)))), RATIO(TDAY_OF_MONTH(), AT_MONTH_START(LAG(1, TDAY_OF_MONTH()))))))"]}
```

**W03　跟三檔種子股的營收年增相關性高、但股價相關性低的股票**

```json
{"formulas": ["MASK(CORR_SEED({\"2330\", \"2454\", \"3711\"}, 1260, 126, 21, 1, 0, RWIN_PCT(12, revenue_m)) > 0.5, NEG(CORR_SEED({\"2330\", \"2454\", \"3711\"}, 252, 21, 1, 0, 0, RET(1, close))))"]}
```

**W04　每月 13 號剛好是星期五就全部出清（那天曝險 0）**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exposure": "MDAY() == 13 && WEEKDAY() == 4 ? 0:RATIO : 1:RATIO"}}
```

**W05　收盤價個位數是 8 的股票**

```json
{"formulas": ["MASK(FLOOR(raw.close % 10) == 8, RET(20, close))"]}
```

**W06　月營收公布當天的漲幅排名（看公告當天的反應）**

```json
{"formulas": ["MASK(EV_CHANGE(revenue_m), RET(1, close))"]}
```

**W07　外資連賣 10 天之後第一天轉買**

```json
{"formulas": ["MASK(foreign_net > 0 && COUNT(10, LAG(1, foreign_net < 0)) >= 10, RATIO(foreign_net, volume))"]}
```

**W08　星期一跌最多的股票，星期二買**

```json
{"formulas": ["MASK(WEEKDAY() == 0, NEG(RET(1, close)))"], "portfolio": {"max_positions": 10, "rebalance_when": "WEEKDAY() == 0"}}
```

**W09　每個產業裡股價最低的那一檔**

```json
{"formulas": ["MASK(close <= GMIN(1, close), NEG(close))"]}
```

**W10　市值排名天天大幅變動的（市值百分位 60 日標準差最大）**

```json
{"formulas": ["ROLL_STD(60, CS_PCT(fund.mktcap))"]}
```

**W11　價格跟成交量的 60 日相關性最負的**

```json
{"formulas": ["NEG(CORR(60, close, volume))"]}
```

**W12　過去 5 年每年 12 月都上漲的股票**

```json
{"formulas": ["MASK(COUNT(1260, MONTH() == 1 && TDAY_OF_MONTH() == 1 && AT_MONTH_START(LAG(1, close)) > AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, close))))) >= 5, RET(20, close))"]}
```

**W13　股價在同產業裡排名跟市值在同產業裡排名差最多的（價格偏離規模）**

```json
{"formulas": ["ABS(SUB(CS_PEER_RANK(1, close), CS_PEER_RANK(1, fund.mktcap)))"]}
```

**W14　用「月營收年增」的相關性分群，群內比股價 20 日漲幅，只看跟群平均漲幅落差最大的**

```json
{"formulas": ["ABS(SUB(RET(20, close), RATIO(GSUM(CORR_GROUP(1260, 126, 21, 3000, 1, RWIN_PCT(12, revenue_m)), RET(20, close)), GSUM(CORR_GROUP(1260, 126, 21, 3000, 1, RWIN_PCT(12, revenue_m)), MASK(VALID(RET(20, close)), 1:RATIO)))))"]}
```

**W15　最近 4 次季報公布當天，平均反應（當天漲幅）最大的股票**

```json
{"formulas": ["EWIN(4, 2, EV_CHANGE(revenue), RET(1, close))"]}
```

**W16　60 日動能的名次加 5% 雜訊後還選得出來的股票（同一份種子重跑結果不變）**

```json
{"formulas": ["ADD(CS_PCT(RET(60, close)), MUL(0.05, NOISE(1)))"]}
```

**W17　每個產業只看市值前 3 大，裡面 20 日漲最多的**

```json
{"formulas": ["MASK(GTOP(3, 1, fund.mktcap), RET(20, close))"]}
```

**W18　股價高於同產業中位數的股票裡，20 日漲幅比產業中位數高最多的**

```json
{"formulas": ["MASK(close > GQUANT(50, 1, close), RET(20, close) - GQUANT(50, 1, RET(20, close)))"]}
```

**W19　突破 60 日新高當作進場，收盤價比進場以來平均收盤價高出最多的**

```json
{"formulas": ["RATIO(close, SCANMEAN(close > ROLL_MAX(60, LAG(1, close)), close)) - 1"]}
```

**W20　創一年新高以來的平均成交量，比創新高前 20 日均量放大最多的**

```json
{"formulas": ["RATIO(SCANMEAN(close >= ROLL_MAX(250, close), volume), LATCH(close >= ROLL_MAX(250, close), LAG(1, ROLL_MEAN(20, volume))))"]}
```

**W21　突破 60 日新高當作進場，進場以來每天漲跌波動最小的**

```json
{"formulas": ["NEG(SCANSTD(close > ROLL_MAX(60, LAG(1, close)), RET(1, close)))"]}
```
