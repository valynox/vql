# VQL example library: plain words -> formula

Each item is a complete set of backtest arguments (`formulas` and `universe` of `stock_backtest`, everything else in `advanced`) that produces a plan as is. The same library, machine-readable, is `examples.json`. Items that show a condition, a backtest rule, a screen or a check use `RET(60, close)` as the score. Each item writes only the conditions in the user's own words, nothing added.

VQL does four jobs; find examples by job:

| Job | Where it is written | Examples |
|---|---|---|
| Search | `formulas` (a few of your own), `trees_from=nouns` (parameter grid of a user function), `trees_from=sample` (random generation, through `gpu_algo_stock_search` in MCP; `config.sample_new` adds market-level columns and correlation groups) | S |
| Backtest | score formula, `universe`, `portfolio` (rebalancing, weights, exits, money) | U, R, H, B, W |
| Screen | `screen_gate`, `screen_pick`, `screen_admit`, user metric `apx.metric.v1` | F01-F05, F08, F11, P |
| Check | `checks`, `check_params`, your own `apx.check.v1` | F06, F07, F09, F10, C |

## Search: trying many formulas at once

**S01　Backtest these three ideas together, one result each**

```json
{"formulas": ["RET_SKIP(252, 21, close)", "NEG(RVOL(60, close))", "FN_EP(eps, close)"]}
```

**S02　Let the system generate 200 random formulas and search**

```json
{"trees_from": "sample", "trees": 200, "seed": 7}
```

**S03　Random generation that also samples market-level columns and groups of similar stocks**

```json
{"trees_from": "sample", "trees": 200, "seed": 7, "config": {"sample_new": 3}}
```

**S04　Distance from the N-day high, trying N = 60, 120 and 250**

```json
{"trees_from": "nouns", "nouns": [{"name": "user.local.NEAR_HIGH", "bind": {"x": "close"}, "grid": {"n": [60, 120, 250]}}]}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.noun.v1", "name": "user.local.NEAR_HIGH", "version": 1, "inputs": [{"name": "x", "unit": "PRICE"}], "params": [{"name": "n", "default": 250, "min": 2, "max": 1260}], "body": "RATIO(x, ROLL_MAX($n, x))", "out_unit": "RATIO"}
```

**S05　Let the system search the momentum lookback around 60 days**

```json
{"formulas": ["RET(apx.EVOLVE(60), close)"]}
```

**S06　Let the system search the stop-loss level around 8%**

```json
{"formulas": ["RET_SKIP(252, 21, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.ret <= -apx.EVOLVE(0.08)"}}
```

**S07　The same momentum on the other price columns of the same kind**

```json
{"formulas": ["CS_PCT(RET(60, apx.EVOLVE(close)))"]}
```

## Stock pool

**U03　Traded every day for the past 20 days, with 20-day average volume above 500 lots**

```json
{"formulas": ["RET(60, close)"], "universe": "COUNT(20, volume > 0) >= 20 && ROLL_MEAN(20, volume) > 500000"}
```

**U04　Mid caps only: market cap between the 30th and 70th percentile of the market**

```json
{"formulas": ["RET(60, close)"], "universe": "CS_PCT(fund.mktcap) >= 0.3 && CS_PCT(fund.mktcap) <= 0.7"}
```

**U06　20-day average turnover rate between 1% and 5%**

```json
{"formulas": ["RET(60, close)"], "universe": "ROLL_MEAN(20, px.turnover_rate) >= 0.01 && ROLL_MEAN(20, px.turnover_rate) <= 0.05"}
```

**U07　Exclude financials and construction, common stocks only**

```json
{"formulas": ["RET(60, close)"], "universe": "!(industry in {17, 14})", "common_only": true}
```

**U08　Pick only from my own list of 10 stocks**

```json
{"formulas": ["RET(60, close)"], "universe": "symbol in {\"2330\", \"2317\", \"2454\", \"3711\", \"2382\", \"2886\", \"2881\", \"2882\", \"2891\", \"3231\"}"}
```

**U09　Only stocks whose daily returns over the past year correlate above 0.5 on average with TSMC, MediaTek and ASE**

```json
{"formulas": ["RET(60, close)"], "universe": "ADD(ADD(CORR(250, RET(1, close), BASKET_MEAN(symbol == \"2330\", RET(1, close))), CORR(250, RET(1, close), BASKET_MEAN(symbol == \"2454\", RET(1, close)))), CORR(250, RET(1, close), BASKET_MEAN(symbol == \"3711\", RET(1, close)))) > 1.5"}
```

**U10　Only stocks whose monthly revenue growth correlates highly with my three seed stocks**

```json
{"formulas": ["RET(60, close)"], "universe": "CORR_SEED({\"2330\", \"2454\", \"3711\"}, 1260, 126, 21, 1, 0, RWIN_PCT(12, revenue_m)) > 0.5"}
```

**U11　Only stocks whose largest drop over the past year is under 40%**

```json
{"formulas": ["RET(60, close)"], "universe": "WDROP(250, LOG(close)) < 0.5108"}
```

**U12　Only stocks that just closed above the 250-day average (within the last 20 days)**

```json
{"formulas": ["RET(60, close)"], "universe": "close > ROLL_MEAN(250, close) && COUNT(20, close > ROLL_MEAN(250, close)) < 20"}
```

**U13　Margin utilization under 30% and short-to-margin ratio above 10%**

```json
{"formulas": ["RET(60, close)"], "universe": "RATIO(margin_bal, margin_limit) < 0.3 && RATIO(short_bal, margin_bal) > 0.1"}
```

**U16　Top 30% by market cap, but without TSMC and Hon Hai**

```json
{"formulas": ["RET(60, close)"], "universe": "CS_PCT(fund.mktcap) >= 0.7 && !(symbol in {\"2330\", \"2317\"})"}
```

**U18　Only stocks with foreign investors net buying for 3 or more days in a row**

```json
{"formulas": ["RET(60, close)"], "universe": "STREAK(foreign_net > 0) >= 3"}
```

**U19　Only stocks within 20 trading days after monthly revenue hits a one-year high**

```json
{"formulas": ["RET(60, close)"], "universe": "SINCE(EV_CHANGE(revenue_m) && revenue_m >= ROLL_MAX(250, revenue_m)) <= 20"}
```

**U20　Only stocks priced (as quoted that day) between 50 and 300**

```json
{"formulas": ["RET(60, close)"], "universe": "BETWEEN(raw.close, 50, 300)"}
```

**U21　Only on the day the close crosses above the 20-day moving average (golden cross)**

```json
{"formulas": ["RET(60, close)"], "universe": "CROSS_UP(close, ROLL_MEAN(20, close))"}
```

**U22　Only stocks closing above the 20-day volume-weighted average price**

```json
{"formulas": ["RET(60, close)"], "universe": "raw.close > VWAP(20)"}
```

## Ranking

**R01　Compare stocks of similar size: split by market cap into 10 groups, rank 20-day return within each group**

```json
{"formulas": ["SUB(RET(20, close), RATIO(GSUM(CS_QCUT(10, fund.mktcap), RET(20, close)), GSUM(CS_QCUT(10, fund.mktcap), MASK(VALID(RET(20, close)), 1:RATIO))))"]}
```

**R02　Compare stocks that behave alike: group by daily-return correlation, rank 60-day return within each group**

```json
{"formulas": ["MASK(VALID(GRANK(CORR_GROUP(LOGR(close, LAG(1, close))), RET(60, close))), SUB(RET(60, close), RATIO(GSUM(CORR_GROUP(LOGR(close, LAG(1, close))), RET(60, close)), GSUM(CORR_GROUP(LOGR(close, LAG(1, close))), MASK(VALID(RET(60, close)), 1:RATIO)))))"]}
```

**R03　Two layers: average of the 20-day return rank within the industry and within the market-cap group**

```json
{"formulas": ["ADD(CS_PEER_RANK(1, RET(20, close)), RATIO(GRANK(CS_QCUT(10, fund.mktcap), RET(20, close)), ADD(SUB(GCOUNT(CS_QCUT(10, fund.mktcap), RET(20, close)), 1:COUNT), SUB(GCOUNT(CS_QCUT(10, fund.mktcap), RET(20, close)), 1:COUNT))))"]}
```

**R04　On the 5th trading day after a quarterly report, rank by the return since the report**

```json
{"formulas": ["MASK(LAG(5, EV_CHANGE(revenue)), RATIO(close, LATCH(EV_CHANGE(revenue), LAG(1, close))))"]}
```

**R05　Since monthly revenue last hit a 12-month high, the pullback from the highest price to today; smaller pullback ranks higher**

```json
{"formulas": ["RATIO(close, SCANMAX(EV_CHANGE(revenue_m) && revenue_m >= ROLL_MAX(250, revenue_m), close))"]}
```

**R06　20-day return minus the industry's average return (stock vs its group)**

```json
{"formulas": ["CS_REL_PARENT(1, RET(20, close))"]}
```

**R07　The industry's 20-day average return minus the market's average (group vs market); every stock of the industry gets the same score**

```json
{"formulas": ["SUB(XL_PARENT(1, RET(20, close)), RATIO(GSUM(0, RET(20, close)), GSUM(0, MASK(VALID(RET(20, close)), 1:RATIO))))"]}
```

**R08　60-day momentum while the equal-weight market is above its 200-day average, 60-day low volatility below it**

```json
{"formulas": ["idx.taiex > ROLL_MEAN(200, idx.taiex) ? CS_PCT(RET(60, close)) : CS_PCT(NEG(ROLL_STD(60, LOGR(close, LAG(1, close)))))"]}
```

**R09　Log of the squared 20-day return times the volume ratio**

```json
{"formulas": ["LOG(MUL(MUL(RET(20, close), RET(20, close)), VOL_RATIO(20, volume)))"]}
```

**R10　In reporting months (Mar, May, Aug, Nov) rank by earnings growth, otherwise by 60-day momentum**

```json
{"formulas": ["MONTH() in {3, 5, 8, 11} ? CS_PCT(FN_EGROW(252, eps)) : CS_PCT(RET(60, close))"]}
```

**R11　Short-term reversal in the first 5 trading days of each month, 20-day momentum otherwise**

```json
{"formulas": ["TDAY_OF_MONTH() <= 5 ? CS_PCT(NEG(RET(5, close))) : CS_PCT(RET(20, close))"]}
```

**R12　Highest score for a 5-10% pullback from the 52-week high, lower for more or less (piecewise score)**

```json
{"formulas": ["(DIST_HIGH(250, close) <= -0.05 && DIST_HIGH(250, close) >= -0.1) ? 0:RATIO : NEG(MIN2(ABS(ADD(DIST_HIGH(250, close), 0.05:RATIO)), ABS(ADD(DIST_HIGH(250, close), 0.1:RATIO))))"]}
```

**R13　Gross margin up three quarters in a row, ranked by the latest quarter's rise**

```json
{"formulas": ["MASK(RATIO(fund.gross, revenue) > RWIN(1, 0, RATIO(fund.gross, revenue)) && RWIN(1, 0, RATIO(fund.gross, revenue)) > RWIN(2, 0, RATIO(fund.gross, revenue)) && RWIN(2, 0, RATIO(fund.gross, revenue)) > RWIN(3, 0, RATIO(fund.gross, revenue)), RWIN(1, 7, RATIO(fund.gross, revenue)))"]}
```

**R14　Industry percentile of revenue growth minus industry percentile of the one-year price return (fundamentals ahead of price)**

```json
{"formulas": ["SUB(CS_PEER_RANK(1, RWIN_PCT(12, revenue_m)), CS_PEER_RANK(1, RET(250, close)))"]}
```

**R15　Value, quality and momentum equally weighted (earnings yield, ROE, 12-month momentum)**

```json
{"formulas": ["ADD(ADD(CS_PCT(FN_EP(eps, close)), CS_PCT(FN_ROE(net_income, equity))), CS_PCT(RET_SKIP(252, 21, close)))"]}
```

**R17　A self-built 14-day RSI, lower ranks higher**

```json
{"formulas": ["NEG(user.local.RSI(42, close))"]}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.noun.v1", "requires": "apx-std>=1.0", "name": "user.local.RSI", "version": 1, "description": "Wilder RSI（α = 1/14）", "inputs": [{"name": "x", "unit": "PRICE"}], "params": [{"name": "k", "default": 42, "min": 0, "max": 756}], "body": "RATIO(SMOOTH($k, 0.071429:RATIO, MAX2(DIFF(1, x), 0:PRICE)), ADD(SMOOTH($k, 0.071429:RATIO, MAX2(DIFF(1, x), 0:PRICE)), SMOOTH($k, 0.071429:RATIO, MAX2(NEG(DIFF(1, x)), 0:PRICE))))", "out_unit": "RATIO"}
```

**R18　Days of consecutive foreign net buying; more ranks higher**

```json
{"formulas": ["STREAK(foreign_net > 0)"]}
```

**R19　60-day change in foreign ownership**

```json
{"formulas": ["DIFF(60, foreign_ratio)"]}
```

**R21　Low P/E, compared only within the same industry**

```json
{"formulas": ["CS_REL_PARENT(1, FN_EP(eps, close))"]}
```

**R22　Price's deviation from its 5-year average, and how that deviation changed over a year**

```json
{"formulas": ["DIFF(250, RATIO(close, ROLL_MEAN(1260, close)))"], "config": {"lookback_div": 2}}
```

**R24　Stocks of strong industries first; within one industry the larger market cap first (second sort key for ties)**

```json
{"formulas": ["SUB(XL_PARENT(1, RET(20, close)), RATIO(GSUM(0, RET(20, close)), GSUM(0, MASK(VALID(RET(20, close)), 1:RATIO))))"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "tie_by": "fund.mktcap"}}
```

**R25　Higher correlation with TSMC's daily returns ranks first**

```json
{"formulas": ["CORR(250, RET(1, close), BASKET_MEAN(symbol == \"2330\", RET(1, close)))"]}
```

**R26　Split market cap into ten groups; rank by 20-day return above the group average**

```json
{"formulas": ["GDEMEAN(CS_QCUT(10, fund.mktcap), RET(20, close))"]}
```

**R27　Biggest gainers of last month**

```json
{"formulas": ["PREV_MONTH_RET(close)"]}
```

**R28　Average foreign net buying over the last 20 days (days without data not counted)**

```json
{"formulas": ["ROLL_MEAN_AVAIL(20, foreign_net)"]}
```

**R29　20-day average return after capping each day's move at ±10%**

```json
{"formulas": ["ROLL_MEAN(20, CLIP(RET(1, close), -0.1, 0.1))"]}
```

**R30　Industry average 20-day return (industry momentum)**

```json
{"formulas": ["GMEAN(1, RET(20, close))"]}
```

**R31　Up days minus down days over the past 20 days**

```json
{"formulas": ["WSUM(20, SIGN(RET(1, close)))"]}
```

**R32　Let the system try momentum lookbacks of half a year (126 days) and one year (252 days), both skipping the latest month**

```json
{"formulas": ["RET_SKIP($n, 21, close)"], "portfolio": {"grid": {"n": [126, 252]}}}
```

**R33　A self-built MACD histogram (12, 26, 9); larger as a share of price ranks higher**

```json
{"formulas": ["RATIO(user.local.MACD_HIST(78, close), close)"]}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.noun.v1", "requires": "apx-std>=1.0", "name": "user.local.MACD_HIST", "version": 1, "description": "MACD 柱 = DIF − DIF 的 EMA9；DIF = EMA12 − EMA26（α = 2/13、2/27、0.2）；單位：價格", "inputs": [{"name": "x", "unit": "PRICE"}], "params": [{"name": "k", "default": 78, "min": 0, "max": 756}], "body": "SUB(SUB(SMOOTH(0, 0.153846:RATIO, x), SMOOTH(0, 0.074074:RATIO, x)), SMOOTH($k, 0.2:RATIO, SUB(SMOOTH(0, 0.153846:RATIO, x), SMOOTH(0, 0.074074:RATIO, x))))", "out_unit": "PRICE"}
```

**R34　A self-built 20-day Bollinger z (standard deviations from the middle band); lower ranks higher**

```json
{"formulas": ["NEG(user.local.BOLL_Z(20, close))"]}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.noun.v1", "requires": "apx-std>=1.0", "name": "user.local.BOLL_Z", "version": 1, "description": "布林 Z = (x − n 日均) / n 日標準差（母體）；±2 = 布林上下緣；單位：倍數", "inputs": [{"name": "x", "unit": "PRICE"}], "params": [{"name": "n", "default": 20, "min": 2, "max": 1260}], "body": "RATIO(SUB(x, ROLL_MEAN($n, x)), ROLL_STD($n, x))", "out_unit": "RATIO"}
```

**R35　A self-built 9-day KD %K; lower ranks higher**

```json
{"formulas": ["NEG(user.local.KD_K(9, 27, high, low, close))"]}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.noun.v1", "requires": "apx-std>=1.0", "name": "user.local.KD_K", "version": 1, "description": "KD 的 K：RSV = (c − n 日最低) / (n 日最高 − n 日最低)，K = RSV 的 1/3 平滑；0～1（×100 = 常見刻度）；D = SMOOTH(k, 0.333333:RATIO, K)", "inputs": [{"name": "h", "unit": "PRICE"}, {"name": "l", "unit": "PRICE"}, {"name": "c", "unit": "PRICE"}], "params": [{"name": "n", "default": 9, "min": 2, "max": 252}, {"name": "k", "default": 27, "min": 0, "max": 756}], "body": "SMOOTH($k, 0.333333:RATIO, RATIO(SUB(c, ROLL_MIN($n, l)), SUB(ROLL_MAX($n, h), ROLL_MIN($n, l))))", "out_unit": "RATIO"}
```

**R36　A self-built 14-day ATR as a share of price; lower (calmer) ranks higher**

```json
{"formulas": ["NEG(RATIO(user.local.ATR(42, high, low, close), close))"]}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.noun.v1", "requires": "apx-std>=1.0", "name": "user.local.ATR", "version": 1, "description": "Wilder ATR（α = 1/14）；真實區間 = max(h − l, |h − 昨收|, |l − 昨收|)；單位：價格", "inputs": [{"name": "h", "unit": "PRICE"}, {"name": "l", "unit": "PRICE"}, {"name": "c", "unit": "PRICE"}], "params": [{"name": "k", "default": 42, "min": 0, "max": 756}], "body": "SMOOTH($k, 0.071429:RATIO, MAX2(SUB(h, l), MAX2(ABS(SUB(h, LAG(1, c))), ABS(SUB(l, LAG(1, c))))))", "out_unit": "PRICE"}
```

## Heat and flows

**H01　Score only stocks trading 3x their 20-day average volume today, ranked by return**

```json
{"formulas": ["MASK(VOL_RATIO(20, volume) >= 3, RET(1, close))"]}
```

**H02　Money flowing in together: group by volume correlation, the group with the biggest average volume surge first**

```json
{"formulas": ["RATIO(GSUM(CORR_GROUP(volume), VOL_RATIO(20, volume)), GSUM(CORR_GROUP(volume), MASK(VALID(VOL_RATIO(20, volume)), 1:RATIO)))"]}
```

**H03　Institutions buying together: group by foreign net-buy correlation, the group with the most 5-day net buying first**

```json
{"formulas": ["GSUM(CORR_GROUP(foreign_net), WSUM(5, foreign_net))"]}
```

**H04　Volume and price both up three days in a row**

```json
{"formulas": ["MASK(COUNT(3, volume > LAG(1, volume) && close > LAG(1, close)) >= 3, RET(3, close))"]}
```

**H05　Back after cooling down: a past volume spike, at least 20 quiet days, and a spike again today**

```json
{"formulas": ["MASK(VOL_RATIO(20, volume) >= 3 && COUNT(20, LAG(1, VOL_RATIO(20, volume) >= 3)) == 0 && COUNT(120, VOL_RATIO(20, volume) >= 3) >= 2, VOL_RATIO(20, volume))"]}
```

**H07　Margin balance up sharply over 5 days while the price did not rise**

```json
{"formulas": ["MASK(RET(5, margin_bal) > 0.2 && RET(5, close) <= 0, RET(5, margin_bal))"]}
```

**H08　First day of investment-trust net buying (none in the previous 10 days)**

```json
{"formulas": ["MASK(FIRST_IN_WINDOW(11, trust_net > 0), RATIO(trust_net, volume))"]}
```

## Backtest

**B01　Rebalance on the first trading day of each month**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "MONTH_START()"}}
```

**B02　Rebalance only on the day the equal-weight market crosses above its 60-day average**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "EW_INDEX(close) > ROLL_MEAN(60, EW_INDEX(close)) && !LAG(1, EW_INDEX(close) > ROLL_MEAN(60, EW_INDEX(close)))"}}
```

**B03　Cut to 50% invested when the equal-weight market falls below its 200-day average**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exposure": "EW_INDEX(close) < ROLL_MEAN(200, EW_INDEX(close)) ? 0.5 : 1"}}
```

**B04　Scale exposure by the inverse of 20-day market volatility (more volatile, less invested; at most fully invested)**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exposure": "MIN2(RATIO(0.01:RATIO, ROLL_STD(20, FILL(MKT_MEAN(LOGR(close, LAG(1, close))), 0:RATIO))), 1:RATIO)"}}
```

**B05　Hold only 30% when the equal-weight market is more than 15% below its one-year high**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exposure": "RATIO(EW_INDEX(close), ROLL_MAX(250, EW_INDEX(close))) < 0.85 ? 0.3 : 1"}}
```

**B06　Weight by the square root of market cap**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "weight": "SQRT(fund.mktcap)"}}
```

**B07　Stop holding names that close below the 20-day average**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "hold": "close >= ROLL_MEAN(20, close)"}}
```

**B08　Pick every day, each name held 10 trading days**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "hold_days": 10}}
```

**B09　Stop loss at -8%, take profit at +20% after entry**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_below": "pos.entry * 0.92", "exit_above": "pos.entry * 1.2"}}
```

**B10　At most 3 holdings among stocks that move together**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "group_by": "CORR_GROUP(RET(1, close))", "group_cap": 3}}
```

**B11　Weight by score, each daily slice held 10 days**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "hold_days": 10, "weight": "CS_PCT(RET(60, close))"}}
```

**B12　Pyramiding: add one more unit when a holding rises another 10%**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "add_when": "pos.ret >= 0.1", "size_init": 0.5}}
```

**B12b　Average down once after a 5% drop from entry**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "add_when": "pos.ret <= -0.05 && pos.adds == 0", "size_init": 0.5}}
```

**B12c　Sell half at +20%, the rest on a 10% fall from the peak**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "cut_when": "pos.ret >= 0.2", "exit_when": "pos.drawdown >= 0.1"}}
```

**B12d　At most 80% invested, at most 3 new names per rebalance**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "gross_max": 0.8, "max_new": 3}}
```

**B13　Buffer: buy in the top 10, sell when out of the top 30**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "keep_when": "pos.rank <= 30"}}
```

**B14　Rebalance only on the first trading day of reporting months (Mar, May, Aug, Nov)**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "MONTH() != LAG(1, MONTH()) && MONTH() in {3, 5, 8, 11}"}}
```

**B15　Rebalance every Friday**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "WEEKDAY() == 4"}}
```

**B16　Invest 5,000 TWD every month, 10 stocks, odd lots**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "initial_capital": 5000, "contribution": 5000, "contribution_every": "month", "lot": "odd"}}
```

**B17　At most 10 stocks, at most 15% each, inverse-volatility weights**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "max_weight": 15, "weighting": "inv_vol"}}
```

**B18　Trailing stop: exit after a 10% fall from the highest close since entry**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.drawdown >= 0.1"}}
```

**B19　Weight by 20-day average traded value, at most 10% per stock**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 20, "rebalance_days": 21, "max_weight": 10, "weight": "ROLL_MEAN(20, px.turnover)"}}
```

**B23　Sell only on a close 8% below entry or 20% above (close-confirmed)**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.ret <= -0.08 || pos.ret >= 0.2"}}
```

**B24　Sell intraday once the price falls below the 20-day average**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_below": "ROLL_MEAN(20, close)"}}
```

**B25　Sell at -8% or after a 10% fall from the highest close, whichever comes first**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_below": "MAX2(pos.entry * 0.92, pos.peak * 0.9)"}}
```

**B20　Try a few stop and take-profit levels: stops 5%, 8%, 10%; take profit 15%, 20%, 30%**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.ret <= -$stop || pos.ret >= $take", "grid": {"stop": [0.05, 0.08, 0.1], "take": [0.15, 0.2, 0.3]}}}
```

**B21　Trailing stop level between 6% and 14%, let the system find it**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.drawdown >= $dd", "grid": {"dd": {"from": 0.06, "to": 0.14, "step": 0.02}}}}
```

**B22　Try the volatility multiple of the stop and the longest holding period together**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.ret < -$k * ROLL_STD(20, RET(1, close)) || pos.days >= $n", "grid": {"k": [1.5, 2, 3], "n": [20, 40]}}}
```

**B26　Let the system try intraday stop and take-profit levels: 90%, 92%, 95% and 115%, 125% of the entry price**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_below": "pos.entry * $floor", "exit_above": "pos.entry * $cap", "grid": {"floor": [0.9, 0.92, 0.95], "cap": [1.15, 1.25]}}}
```

**B27　Buy the top 10 every day, hold each 10 days, stop loss at -8% meanwhile**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "hold_days": 10, "exit_below": "pos.entry * 0.92"}}
```

**B28　Traded-value quintiles, at most 2 holdings per quintile**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "group_by": "CS_QCUT(5, ROLL_MEAN(20, px.turnover))", "group_cap": 2}}
```

**B29　At most 2 per group, and sell only when out of the top 30**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 5, "keep_when": "pos.rank <= 30", "group_by": "CORR_GROUP(RET(1, close))", "group_cap": 2}}
```

**B30　Hold only 50% when fewer than 30% of the market is above its 60-day moving average**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "exposure": "MKT_SHARE(close > ROLL_MEAN(60, close)) < 0.3 ? 0.5 : 1"}}
```

**B31　Rebalance only on days with more advancers than decliners**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "MKT_COUNT(close > LAG(1, close)) > MKT_COUNT(close < LAG(1, close))"}}
```

**B32　Rebalance on the first trading day of each quarter**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "QUARTER_START()"}}
```

**B33　Rebalance on the first trading day of each year**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_when": "YEAR_START()"}}
```

**B34　Hold after crossing above the 250-day line; sell only after falling below the 60-day line**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "hold": "STATE(close > ROLL_MEAN(250, close), close < ROLL_MEAN(60, close))"}}
```

**B35　Try momentum lengths of 60 and 120 days together with stop losses of 8% and 10%**

```json
{"formulas": ["RET($n, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exit_when": "pos.ret <= -$stop", "grid": {"n": [60, 120], "stop": [0.08, 0.1]}}}
```

**B36　Rebalance on the first trading day of each week (Tuesday when Monday is a holiday), picking only from 2020 on**

```json
{"formulas": ["MASK(YEAR() >= 2020, RET(60, close))"], "portfolio": {"max_positions": 10, "rebalance_when": "WEEK_START()"}}
```

**B37　Use up to 1.5x capital (margin), add one unit after a 4% gain; adds beyond the cap use only the room left**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "add_when": "pos.ret >= 0.04", "gross_max": 1.5, "margin_rate": 0.05}}
```

**B38　Use up to 1.5x capital (margin), add one unit after a 4% gain; beyond the cap shrink all holdings in proportion**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "add_when": "pos.ret >= 0.04", "gross_max": 1.5, "margin_rate": 0.05, "add_mode": "scale"}}
```

**B39　Each rebalance buys new names up to 30% of the number of names held**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21}, "config": {"max_turnover_bp": 3000}}
```

## Screening and built-in checks

**F01　Keep only strategies with RoMaD (annual return / max drawdown) of at least 1**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "user.local.ROMAD >= 1"}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.metric.v1", "requires": "apx-std>=1.1", "name": "user.local.ROMAD", "version": 1, "description": "年化報酬 ÷ 最大回撤", "inputs": [], "body": "apx.ANN_RET / apx.MAX_DD", "unit": "", "direction": "max", "group": "mine"}
```

**F02　Monthly win rate of at least 60%**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "apx.MONTH_WIN >= 0.6"}
```

**F03　Longest time under water (from a peak back to that peak) at most 250 days**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "user.local.UNDERWATER <= 250"}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.metric.v1", "requires": "apx-std>=1.1", "name": "user.local.UNDERWATER", "version": 1, "description": "最長水下天數", "inputs": [], "body": "streak(drawdown_nav(r) > 0)", "unit": "days", "direction": "min", "group": "mine"}
```

**F04　Beat the equal-weight market every single year**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "min(by_sum(year, r - m)) > 0"}
```

**F05　Sharpe above 0.5 in both bull and bear markets**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "apx.SHARPE_BULL > 0.5 && apx.SHARPE_BEAR > 0.5"}
```

**F06　Net return still positive with trading costs doubled**

```json
{"formulas": ["RET(60, close)"], "checks": ["cost_retest"], "check_params": {"COST_RETEST.mult": 2}}
```

**F07　IC with the same sign in both halves of the period**

```json
{"formulas": ["RET(60, close)"], "checks": ["user.local.HALVES"]}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.check.v1", "name": "user.local.HALVES", "version": 1, "description": "前後兩半 IC 同號", "windows": {"kind": "split", "n": 2}, "runs": [{"id": "seg", "interval": "window.seg"}], "outputs": {"seg_ic": "seg.ic_days > 0 ? seg.ic_x1e4 : 0", "lo": "AMIN(seg_ic)", "hi": "AMAX(seg_ic)"}, "pass": "lo > 0 || hi < 0"}
```

**F08　Worst year no lower than -20%**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "apx.RET_Y_MIN > -0.2"}
```

**F09　Keep at least half the return after the peer-swap check**

```json
{"formulas": ["RET(60, close)"], "checks": ["peer_swap"], "check_params": {"PEER_SWAP.keep_bp": 5000}}
```

**F10　Four segments, IC positive in at least three**

```json
{"formulas": ["RET(60, close)"], "checks": ["segments"], "check_params": {"SEGMENTS.n": 4, "SEGMENTS.min_pos_bp": 7500}}
```

**F11　Last three years (from 2023) at least 10% a year**

```json
{"formulas": ["RET(60, close)"], "backtest_from": "2023-01-01", "screen_gate": "apx.ANN_RET >= 0.1"}
```

**F12　Beat the market index (the weighted total-return index) in at least 70% of the years**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "apx.YEARS_BEAT_B >= 0.7"}
```

**F13　Annualized excess return over the market index of at least 5%**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "apx.EXCESS_B_ANN >= 0.05"}
```

**F14　No one-year holding period may lose more than 15%**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "apx.ROLL_RET_MIN > -0.15"}
```

**F15　Sharpe above 1 and at least 26% of the return kept after same-group stock swaps (one threshold)**

```json
{"formulas": ["RET(60, close)"], "checks": ["peer_swap"], "screen_gate": "apx.SHARPE > 1 && check.PEER_SWAP.keep_ret_bp >= 2600"}
```

**F16　Even the worst rolling half-year average daily return must be above 0**

```json
{"formulas": ["RET(60, close)"], "screen_gate": "MIN(ROLL_MEAN(126, r)) > 0"}
```

**F17　Pick among several portfolio settings by Calmar**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"grid_by": "apx.CALMAR"}}
```

## Admission after checks

**P01　Keep a strategy only if the peer swap keeps at least 60% of the net return**

```json
{"formulas": ["RET(60, close)"], "checks": ["peer_swap"], "screen_admit": "check.PEER_SWAP.keep_ret_bp >= 6000"}
```

## Checks you build yourself

**C01　IC must also be positive over the last year (last 252 trading days)**

```json
{"formulas": ["RET(60, close)"], "checks": ["user.local.LAST_YEAR"]}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.check.v1", "name": "user.local.LAST_YEAR", "version": 1, "description": "最後 252 個交易日 IC 為正", "runs": [{"id": "late", "interval": {"begin": "env.n_days - 252", "end": "env.n_days"}}], "outputs": {"late_ic": "late.ic_days > 0 ? late.ic_x1e4 : 0"}, "pass": "late_ic > 0"}
```

**C02　Must make money with both weekly and monthly rebalancing**

```json
{"formulas": ["RET(60, close)"], "checks": ["user.local.BOTH_FREQ"]}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.check.v1", "name": "user.local.BOTH_FREQ", "version": 1, "description": "每 5 日與每 21 日換股的淨報酬都 > 0", "runs": [{"id": "rb", "vary": {"rebalance_days": [5, 21]}}], "outputs": {"worst": "AMIN(rb.net_ret_1e6)"}, "pass": "worst > 0"}
```

**C03　Still profitable on a random half of the stocks with costs tripled**

```json
{"formulas": ["RET(60, close)"], "checks": ["user.local.HALF_X3"]}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.check.v1", "name": "user.local.HALF_X3", "version": 1, "description": "隨機一半股票、單邊成本 ×3 的淨報酬 > 0", "runs": [{"id": "h", "universe": "universe.random_half", "cost_bp": "prog.cost_bp * 3"}], "outputs": {"r": "h.net_ret_1e6"}, "pass": "r > 0"}
```

**C04　Profitable with only 10 holdings, max drawdown at most 40%**

```json
{"formulas": ["RET(60, close)"], "checks": ["user.local.TEN"]}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.check.v1", "name": "user.local.TEN", "version": 1, "description": "只持有 10 檔的淨報酬 > 0、最大回撤 ≤ $max_dd_bp", "params": [{"name": "max_dd_bp", "default": 4000, "min": 0, "max": 10000}], "runs": [{"id": "n10", "max_positions": "10"}], "outputs": {"r": "n10.net_ret_1e6", "dd_bp": "n10.max_dd_1e6 / 100"}, "pass": "r > 0 && dd_bp <= $max_dd_bp"}
```

**C05　Pick the long threshold in-sample and confirm out-of-sample; both windows must make money**

```json
{"formulas": ["RET(60, close)"], "checks": ["user.local.WF2"]}
```

Validate (`stock_formula_help topic=validate`), then install the user definition (`my_stock_strategies_change action=install`):

```json
{"schema": "apx.check.v1", "name": "user.local.WF2", "version": 1, "description": "滾動兩段：IS 挑 p_long、OOS 淨報酬都 > 0", "windows": {"kind": "rolling", "is_days": 504, "oos_days": 252, "max": 2}, "runs": [{"id": "is", "interval": "window.is", "vary": {"p_long": [170, 180, 190]}}, {"id": "oos", "interval": "window.oos", "p_long": "WINNER(is, net_ret_1e6, p_long)"}], "outputs": {"oos_ret": "oos.net_ret_1e6", "ok": "ALL(oos_ret > 0)"}, "pass": "n_windows >= 2 && ok == 1"}
```

## Odd strategies

**W02　In last month's best industry, the stocks whose volume shrank most this month**

```json
{"formulas": ["MASK(XL_PARENT(1, RATIO(AT_MONTH_START(LAG(1, close)), AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, close)))))) >= GMAX(0, XL_PARENT(1, RATIO(AT_MONTH_START(LAG(1, close)), AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, close))))))), NEG(RATIO(RATIO(SCANSUM(TDAY_OF_MONTH() == 1, volume), AT_MONTH_START(LAG(1, SCANSUM(TDAY_OF_MONTH() == 1, volume)))), RATIO(TDAY_OF_MONTH(), AT_MONTH_START(LAG(1, TDAY_OF_MONTH()))))))"]}
```

**W03　High correlation of revenue growth with three seed stocks but low price correlation**

```json
{"formulas": ["MASK(CORR_SEED({\"2330\", \"2454\", \"3711\"}, 1260, 126, 21, 1, 0, RWIN_PCT(12, revenue_m)) > 0.5, NEG(CORR_SEED({\"2330\", \"2454\", \"3711\"}, 252, 21, 1, 0, 0, RET(1, close))))"]}
```

**W04　Go to cash whenever the 13th of a month is a Friday (exposure 0 that day)**

```json
{"formulas": ["RET(60, close)"], "portfolio": {"max_positions": 10, "rebalance_days": 21, "exposure": "MDAY() == 13 && WEEKDAY() == 4 ? 0:RATIO : 1:RATIO"}}
```

**W05　Stocks whose close ends in the digit 8**

```json
{"formulas": ["MASK(FLOOR(raw.close % 10) == 8, RET(20, close))"]}
```

**W06　Rank by the return on the day monthly revenue is published**

```json
{"formulas": ["MASK(EV_CHANGE(revenue_m), RET(1, close))"]}
```

**W07　First day of foreign net buying after 10 days of net selling**

```json
{"formulas": ["MASK(foreign_net > 0 && COUNT(10, LAG(1, foreign_net < 0)) >= 10, RATIO(foreign_net, volume))"]}
```

**W08　Buy on Tuesday the stocks that fell most on Monday**

```json
{"formulas": ["MASK(WEEKDAY() == 0, NEG(RET(1, close)))"], "portfolio": {"max_positions": 10, "rebalance_when": "WEEKDAY() == 0"}}
```

**W09　The lowest-priced stock in each industry**

```json
{"formulas": ["MASK(close <= GMIN(1, close), NEG(close))"]}
```

**W10　Stocks whose market-cap rank swings the most (60-day std of the market-cap percentile)**

```json
{"formulas": ["ROLL_STD(60, CS_PCT(fund.mktcap))"]}
```

**W11　Most negative 60-day correlation between price and volume**

```json
{"formulas": ["NEG(CORR(60, close, volume))"]}
```

**W12　Stocks that rose every December for the past five years**

```json
{"formulas": ["MASK(COUNT(1260, MONTH() == 1 && TDAY_OF_MONTH() == 1 && AT_MONTH_START(LAG(1, close)) > AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, close))))) >= 5, RET(20, close))"]}
```

**W13　Largest gap between price rank and market-cap rank within the industry**

```json
{"formulas": ["ABS(SUB(CS_PEER_RANK(1, close), CS_PEER_RANK(1, fund.mktcap)))"]}
```

**W14　Group by monthly-revenue-growth correlation, compare 20-day returns within each group, the largest gap from the group average first**

```json
{"formulas": ["ABS(SUB(RET(20, close), RATIO(GSUM(CORR_GROUP(1260, 126, 21, 3000, 1, RWIN_PCT(12, revenue_m)), RET(20, close)), GSUM(CORR_GROUP(1260, 126, 21, 3000, 1, RWIN_PCT(12, revenue_m)), MASK(VALID(RET(20, close)), 1:RATIO)))))"]}
```

**W15　Largest average reaction (same-day return) on the last 4 quarterly-report days**

```json
{"formulas": ["EWIN(4, 2, EV_CHANGE(revenue), RET(1, close))"]}
```

**W16　Stocks that still get picked when 5% noise is added to the 60-day momentum rank (the same seed reruns identically)**

```json
{"formulas": ["ADD(CS_PCT(RET(60, close)), MUL(0.05, NOISE(1)))"]}
```

**W17　Within each industry, only the 3 largest by market cap; among them, the biggest 20-day gainers**

```json
{"formulas": ["MASK(GTOP(3, 1, fund.mktcap), RET(20, close))"]}
```

**W18　Among stocks priced above their industry median, the largest 20-day return above the industry median**

```json
{"formulas": ["MASK(close > GQUANT(50, 1, close), RET(20, close) - GQUANT(50, 1, RET(20, close)))"]}
```

**W19　Taking a new 60-day high as the entry, the close furthest above the average close since entry**

```json
{"formulas": ["RATIO(close, SCANMEAN(close > ROLL_MAX(60, LAG(1, close)), close)) - 1"]}
```

**W20　Average volume since a new one-year high, most expanded against the 20-day average volume before that high**

```json
{"formulas": ["RATIO(SCANMEAN(close >= ROLL_MAX(250, close), volume), LATCH(close >= ROLL_MAX(250, close), LAG(1, ROLL_MEAN(20, volume))))"]}
```

**W21　Taking a new 60-day high as the entry, the steadiest daily moves since entry (lowest standard deviation)**

```json
{"formulas": ["NEG(SCANSTD(close > ROLL_MAX(60, LAG(1, close)), RET(1, close)))"]}
```
