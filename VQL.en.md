# Valynox Quant Language (VQL)

- VQL is the stock-selection language of AlphaPrime Stocks: one language for score formulas, stock pools, backtest rules, screening and checks. The author is usually an AI: a person speaks plain words, and the AI writes VQL through MCP.
- The same content exists as a Traditional Chinese edition `VQL.zh-TW.md`, a plain-text edition for AI readers `vql.txt`, and an example library from plain words to formulas `EXAMPLES.en.md` (machine-readable `examples.json`).
- Only forms that work today are described; every example in the text passes plan (the form is checked, nothing is run).
- In MCP: score formulas go in `formulas` of `stock_backtest`, the stock pool in `universe`, exits in `exit_when` / `keep_when` / `grid`, other backtest and screening arguments (`portfolio`, `screen_gate`, `checks`, `check_params`...) in `advanced`; look forms up with `stock_formula_help`.

Contents: 1 What VQL is | 2 Grammar | 3 Values and units | 4 All operators | 5 Reserved functions as atoms | 6 Similarity | 7 Data columns | 8 Stock pool | 9 Backtest | 10 Search | 11 Screening metrics and checks | 12 User functions | 13 Limits | 14 Error codes and fixes | 15 Not writable yet | 16 Appendix: complete examples

---

## 1. What VQL is

### 1.1 Design intent

Valynox Quant Language (VQL) is the strategy language of AlphaPrime Stocks. Its author is an AI, working entirely through MCP: people speak in plain words and the AI writes VQL.

The design intent: **every kind of stock-selection strategy must be writable, including very odd ones.** The language therefore offers only the most basic atoms and operators; every strategy is composed from them. Three rules follow:

1. Operators stay pure. There are no technical-indicator operators (RSI, MACD, Bollinger bands); build them from atoms as a user function (chapter 12 shows RSI).
2. There is no node-count rule. A formula can be as large as you like, with one technical limit (chapter 13); every strategy carries complexity figures.
3. Sectors and themes are not hand-made lists. Baskets are defined by correlation (chapter 6).

### 1.2 Four jobs, one language

| Job | Where it is written | Value type | Output |
|---|---|---|---|
| Search | score formulas: `formulas=[...]` of `stock_backtest`, `trees_from` (your own, a user-function grid, random generation; chapter 10), the body of a user function | i64 fixed point (chapter 3) | one score per day per stock, higher ranks first; many tried at once |
| Backtest | `portfolio={...}` of `stock_backtest` (chapter 9) | i64 fixed point | how the ranking becomes holdings: rebalance timing, number of names, weights, exits |
| Screening | threshold `screen_gate`, ordering `grid_by`, `apx.metric.v1` (chapter 11) | real (double) | whether a strategy is kept |
| Checks | `checks`, `apx.check.v1` (chapter 11) | integer or ratio | metrics after re-running under changed conditions, and a pass condition |

All four share one grammar, one set of operators and one precedence table. They differ in value type and in the names available: search and backtest use panel columns and the operators of chapter 4; screening and checks use strategy-level atoms (`r`, `m`, `mean`, `sd`...). The stock pool (chapter 8) is a generation formula whose result is BOOL.

### 1.3 Three layers of names

| Layer | Looks like | Defined by | Examples |
|---|---|---|---|
| Atom | a name without a dot | the engine; adding one means engine code | `LAG`, `WSUM`, `GT`, `LOGR`, `SCANMAX` |
| Reserved function | a name without a dot, or the full name `apx.NAME` | official; composed of atoms | `RET` (= `apx.RET`), `ROLL_MEAN`, `CS_PCT` |
| User function | `user.<author>.NAME` | the user, stored through `my_stock_strategies_change action=install` | `user.local.RSI` |

- The atom definitions of the reserved functions are in chapter 5: every reserved function decomposes back into atoms, and the definition is bit-identical to writing the name.
- Data is not a computation: it is information obtained from outside (one set per market), that is, the columns of chapter 7 (the column key `LEAF` is the column itself), the level-L parent id `PARENT_ID(L)` (section 4.8), which day the row is `EPOCH_DAY()` and the panel row number `DAY_INDEX()` (section 4.11: the date is data; splitting it into year, month and day with `MONTH()` and the like is an atom), plus the 5 financial ratios of section 7.4, derived by dividing data columns (derived data, e.g. `FN_EP`).
- A user function expanded to atoms is the same tree as writing the atoms directly, with the same results.
- There are also official class nouns `cls.NAME()` (chapter 8), used only in conditions; they exist where the market has industry codes.

### 1.4 One language, one set of data per market

Markets live now: Taiwan (`market=TW`) and the United States (`market=US`). Operators, reserved functions and user functions mean the same in every market; only the **data** differs: it is obtained from outside, one set per market. So there is one rule:

> A formula can be used in market m if and only if every "leaf" it uses exists in m.

| Leaf | Differs by market | Where to look |
|---|---|---|
| data columns | Taiwan has institutional flows, margin and short balances, monthly revenue and limit prices; the US does not | chapter 7; `stock_formula_help topic=column market=US` |
| code values | the values of `symbol`, `market`, `industry` belong to each market (Taiwan `"2330"`, `TWO`; US `"AAPL"`, `XNAS`) | section 2.4 |
| class nouns `cls.*` | defined on that market's industry codes: they exist where the market has industry codes | section 8.3; `stock_formula_help topic=universe` |
| level 1 (industry) `PARENT_ID(1)` | exists where the market has industry codes; elsewhere `GSUM(1, x)`, `CS_PEER_RANK(1, x)` have no groups | section 4.8 |

- Write the generic name where there is one: the market index is `mkt.idx` (price index) and `mkt.idx_tr` (total return index); in Taiwan they resolve to TAIEX (`idx.taiex`, `idx.taiex_tr`), in the US to the S&P 500 (`idx.spx`, `idx.spx_tr`): the same column, bit-identical results; a panel without that column does not compile, and the error lists each market's column name.
- Exchange rules (commission and taxes, price limits, filing deadlines, currency) are not written in formulas: the backtest applies each stock's own market automatically (sections 7.3, 9.7).

---

## 2. Grammar

### 2.1 Nodes

```
node = column[:UNIT]  |  number[:UNIT]  |  NAME(params..., inputs...)  |  $param
```

| Item | Rule | Examples |
|---|---|---|
| Column key | a panel column name, case-insensitive; may carry `:UNIT` as a check | `close`, `fund.mktcap`, `fund.mktcap:AMOUNT` |
| Number | decimal, may have a fraction and an `e` exponent; may carry `:UNIT` | `20`, `0.5:RATIO`, `-0.08`, `5e9`, `1.5e-3` |
| Call | **parameters first, inputs after**; zero-input names keep the parentheses | `ROLL_MEAN(20, close)`, `MONTH()`, `PARENT_ID(1)` |
| Two-parameter operators | the two numbers are written separately | `RET_SKIP(252, 21, close)`, `RWIN(4, 1, revenue)` |
| `$param` | only in the body of a user function (chapter 12) | `WSUM($m, x)` |
| String | `"..."`, only inside `in {...}` or on the right of `==` / `!=` on a code column | `symbol == "2330"` |
| White space | spaces and line breaks are free | |

- Parameters are always integers. Number of parameters = number of arguments - number of inputs.
- An operator has at most three inputs: write the three-input `STATESCAN` (section 4.13) and `GRESID` (section 4.8) directly; write `SELECT` as `c ? a : b`; the four-input `REC2` is not accepted in formula text.
- Parentheses nest at most 64 levels.

### 2.2 Operator form and function form are the same tree

| Operator form | Equals | Note |
|---|---|---|
| `a + b` | `ADD(a, b)` | same unit on both sides |
| `a - b` | `SUB(a, b)` | same unit on both sides |
| `a * b` | `MUL(a, b)` | unit rules in 3.3 |
| `a / b` | `RATIO(a, b)` | b = 0 -> invalid |
| `a % b` | `MOD(a, b)` = `SUB(a, MUL(b, FLOOR(RATIO(a, b))))` | the remainder has the sign of b; b = 0 -> invalid |
| `a > b` | `GT(a, b)` | result BOOL |
| `a < b` | `LT(a, b)` | |
| `a >= b` | `NOT(LT(a, b))` | |
| `a <= b` | `NOT(GT(a, b))` | |
| `a == b` | `NOT(OR(GT(a, b), LT(a, b)))` | `==` on code columns: see 2.4 |
| `a != b` | `OR(GT(a, b), LT(a, b))` | |
| `a && b` | `AND(a, b)` | both sides BOOL; two booleans are not multiplied: `(a > b) * (c > d)` does not compile (fix: `a && b`). A product of two booleans in a universe, portfolio formula or user function saved by an older version (which printed `&&` as `MUL`) is read back as and when the saved text is loaded |
| `a \|\| b` | `OR(a, b)` | both sides BOOL |
| `!a` | `NOT(a)` | a is BOOL |
| `-a` | `NEG(a)` | `-` directly before a digit is a negative constant |
| `c ? a : b` | `FILL(MASK(c, a), MASK(NOT(c), b))` | c is BOOL; c invalid -> invalid |
| `x in {v1, v2, ...}` | set membership (one node) | result BOOL; x invalid -> invalid |

**Precedence, low to high** (left-associative within a level):

| Level | Operators |
|---|---|
| 1 (lowest) | `? :` |
| 2 | `\|\|` |
| 3 | `&&` |
| 4 | `==`  `!=` |
| 5 | `>`  `<`  `>=`  `<=`  `in {...}` |
| 6 | `+`  `-` |
| 7 | `*`  `/`  `%` |
| 8 (highest) | unary `-`  `!` |

Parentheses group. Example: `close / LAG(20, close) > 1.1 && volume > ROLL_MEAN(20, volume)` reads as `((close / LAG(20, close)) > 1.1) && (volume > ROLL_MEAN(20, volume))`.

Two things to note:

- `&&` and `||` with an invalid (unknown) side: when the other side already decides the answer, that is the answer — `false && unknown = false`, `true || unknown = true`; when the answer depends on the unknown side (`true && unknown`, `false || unknown`) the result is invalid. `!` of an invalid value stays invalid. Where a condition is finally used (universe, `hold`, exits, `MASK`) an invalid value counts as not met.
- In the middle part of `?:`, a `:` followed by a known unit name is a unit suffix; otherwise it is the colon of `?:`. Both `c ? 0.5:RATIO : 1:RATIO` and `c ? 0.5 : 1` are correct.

### 2.3 Constants and units

| Written | Read as |
|---|---|
| `0.5:RATIO`, `20:DAYS`, `1:COUNT`, `0:PRICE` | constants with an explicit unit |
| `close > 100` | a number without a unit takes the unit of the other side (here `100:PRICE`) |
| `WREG(20, 0, close, 100)`, `CLIP(close, 10, 100)` | inputs of one group (add and subtract, comparisons, `MAX2` / `MIN2`, `FILL`, the two branches of `?:`, y and x of `WREG`, the X inputs of one reserved function) are the same kind of quantity: a constant without a unit takes the unit of the one that has it (PRICE here). Atoms and reserved functions follow the same rule |
| `FILL(MASK(r, close), MASK(NOT(r), 0))`, `close + LAG(1, 5)` | the same holds for a constant inside an atom whose output has its input's unit (the value of `MASK`, `LAG`, `NEG`, `ROLL_*`, the value of `SCANSUM` ...): `0` and `5` are PRICE here |
| `close / LAG(20, close) > 1.1` | the left side is RATIO, so `1.1` is `1.1:RATIO` |
| `MUL(x, 2)`, `RATIO(x, 3)` | a bare multiplier or divisor with no other side to follow is RATIO (a unitless factor) |
| `fund.mktcap > 5e9` | large constants are handled automatically: constants that do not fit 32 bits take 7 significant digits x 10 to the power 0..12 |

- The limits: a constant inside an atom follows numeric units only, not BOOL (`FILL(c, MASK(d, 0))` needs `0:BOOL`; a bare `FILL(c, 0)` still takes BOOL); a weight (w of `GWMEAN`, `GWMEAN_BY`) and the x and z of `GRESID` and `CS_NEUTRAL_SIZE` are each a group of their own, so without a unit they are RATIO.
- A constant with more fractional digits than the unit's fixed-point precision (6 digits for x1e6 units; none for integer units) is rejected.
- A formula cannot be only a constant; it needs at least one column.

### 2.4 Code columns, sets, strings

`industry`, `market`, `common`, `symbol` are code columns of unit CODE: they support only `==`, `!=`, `in {...}`, or use as a group key. No arithmetic. One code column = one classification (industry, exchange, common stock or not, which stock); every one of them can be a condition and a group key, written the same way.

| Written | Meaning |
|---|---|
| `market in {TW, TWO}` | Taiwan: listed on the main board or the OTC market (exchange names are constants) |
| `market == XNAS` | US: Nasdaq. US exchanges are written as their international codes: `XNAS` Nasdaq, `XNYS` NYSE, `XASE` NYSE American, `ARCX` NYSE Arca, `BATS` Cboe BZX |
| `symbol in {"2330", "2317"}` | these two stocks (codes in double quotes; US tickers, e.g. `"AAPL"`) |
| `symbol == "2330"` | this one stock |
| `industry in {24, 28}` | industry codes: each market's own code table (look it up with `stock_formula_help topic=universe market=...`; the example uses Taiwan codes) |
| `MONTH() in {3, 5, 8, 11}` | `in` also works on numeric values; the values are read in the unit of x |

- Accepted in conditions (stock pool, `hold`, `rebalance_when`, `exposure`) and in score formulas and user functions.
- In a score formula, `x in {...}` and `symbol == "2330"` are a BOOL column computed at the start of the round and appended to the panel; it is part of the result library's identity. Use it as the condition of `MASK` or `?:`, or multiply by it.
- As a group key write the name: `GSUM(industry, x)`, the top 3 of each industry `GTOP(3, industry, x)`. `industry` is level 1: `GSUM(industry, x)` and `GSUM(1, x)` are the same tree, in conditions and in score formulas alike. `market`, `common`, `symbol` are written the same way: the sum within each exchange `GSUM(market, x)`, the top 3 of each exchange `GTOP(3, market, x)`, with the same answer in conditions and in score formulas; in a score formula it is a code column appended to the panel (one value per stock, the same every day) and part of the library identity.
- Classifications that exist where the market's data has that column: `sector`, `subindustry`, `sectype` (security type), `hq_country` (country of incorporation). Written like `industry`: `sector == code`, `sector in {...}` (a name from the code table also works on the right), as a group key `GSUM(sector, x)`, the top 3 of each sector `GTOP(3, sector, x)`, in conditions and in score formulas alike. Which of them a market has, and their codes: `stock_formula_help topic=universe`; a market without that data answers that it has no such classification. Today neither the TW nor the US data has these four columns. `country` (= level 3): every cell is invalid today.
- Code values belong to each market: a code of another market returns `E_UNKNOWN_CODE`.

### 2.5 Aliases

| Alias | Equals |
|---|---|
| `DIV` | `RATIO` |
| `WMAX` | `ROLL_MAX` |
| `WMIN` | `ROLL_MIN` |
| `apx.NAME` | `NAME` (the full name of a reserved function: `apx.RET` = `RET`) |
| `GSUM(k, x)`, `GCOUNT`, `GRANK`, `GMAX`, `GMIN`, `GSTD` whose first argument is not "an integer without a unit" | the key forms `GSUM_BY(k, x)` ... `GMIN_BY(k, x)`, `GSTD_BY(k, x)` (section 4.8) |
| `ROLL_SUM(N, x)` | `WSUM(N, x)` |
| `WMEAN`, `WSTD`, `ROLL_QUANT`, `ROLL_ARG`, `ROLL_MAD`, `ROLL_DROP`, `ROLL_REG`; `RWIN_SUM(k, x)` and the rest, `EWIN_SUM(k, b, x)` and the rest | regular names (2.5a) |
| `LE(a, b)`, `GE(a, b)`, `EQ(a, b)`, `NE(a, b)` | `a <= b`, `a >= b`, `a == b`, `a != b` |
| `AND(a, b, ...)` | `a && b && ...` |
| `IF(c, a, b)`, `IIF(c, a, b)`, `WHERE(c, a, b)` | `c ? a : b` |
| `MIN(a, b, ...)`, `MAX(a, b, ...)` | nested `MIN2` / `MAX2`: the smaller / larger value cell by cell. With two inputs where one is a unitless integer 2..1,260 and the other a series (`MAX(20, close)`) it does not compile; the error gives both rewrites (`ROLL_MAX(20, close)`, or against a constant `MAX2(20:PRICE, close)`). The minimum / maximum over N days is `ROLL_MIN(N, x)` / `ROLL_MAX(N, x)`, over the whole market on one day `GMIN(0, x)` / `GMAX(0, x)` |
| `GPCT(g, x)`, `apx.GPCT(g, x)` | percentile inside a group (section 5.2) |

A condition (BOOL) inside a window, or compared with a unitless 0 / 1, expands to the same tree as below:

| Written | Equals | Meaning |
|---|---|---|
| `ROLL_MAX(N, c)` | `WSUM(N, c) > 0` | true on at least one of the N days |
| `ROLL_MIN(N, c)` | `!(WSUM(N, c) < N)` | true on every one of the N days |
| `ROLL_MEAN(N, c)` | `RATIO(WSUM(N, c), N:COUNT)` | share of the N days on which it is true |
| `c > 0`, `c == 1`, `c != 0`, `c >= 1` | `c` | |
| `c == 0`, `c < 1`, `c != 1`, `c <= 0` | `!c` | |
| `FILL(c, 0)` | `FILL(c, 0:BOOL)` | a unitless fill value takes the other side's unit (a condition is BOOL) |

- One invalid day inside the window -> invalid (as `WSUM`).
- Comparisons that are always true or always false (`c >= 0`, `c > 0.5`) are not accepted: unit mismatch.
- Everything in this section prints under the canonical name (2.6): `ROLL_SUM(20, x)` prints as `WSUM(20, x)`.

### 2.5a Range x algorithm: regular names

<!-- range-algo:begin -->

A function that reads a batch of cells and returns one number = a range x an algorithm. Names follow one pattern: for the last N days the prefixes `ROLL_...` and `W...` are interchangeable; the last k publications are `RWIN_...` (`RWIN_SUM(4, eps)` = `RWIN(4, 1, eps)`), the last k events `EWIN_...`, a group `G...`. Names joined by "=" in one cell are the same tree and print as the first name. "—" = not written by one name yet (build it from other functions).

| Algorithm | Last N days | Since a condition was last true | Last k publications / events | Same day, same group |
|---|---|---|---|---|
| sum | `WSUM` = `ROLL_SUM` | `SCANSUM` | `RWIN_SUM`, `EWIN_SUM` | `GSUM` |
| count | `COUNT` | — | — | `GCOUNT` |
| mean | `ROLL_MEAN` = `WMEAN` | `SCANMEAN` | `RWIN_MEAN`, `EWIN_MEAN` | `GMEAN`, `XL_PARENT` |
| weighted mean | — | — | — | `GWMEAN` |
| max | `ROLL_MAX` = `WMAX` | `SCANMAX` | `RWIN_MAX`, `EWIN_MAX` | `GMAX` |
| min | `ROLL_MIN` = `WMIN` | `SCANMIN` | `RWIN_MIN`, `EWIN_MIN` | `GMIN` |
| std | `ROLL_STD` = `WSTD` | `SCANSTD` | `RWIN_STD`, `EWIN_STD` | `GSTD` |
| rank of today | `WRANK`, `ROLL_RANK` | — | `RWIN_RANK`, `EWIN_RANK` | `GRANK`, `ORD`, `GPCT` |
| quantile (median) | `WQUANT` = `ROLL_QUANT` | — | `RWIN_QUANT`, `EWIN_QUANT` | `GQUANT` |
| value at one cell | `LAG` | `LATCH` | `RWIN_LAG`, `EWIN_LAG` | n/a |
| change versus one cell | `DIFF`, `RET` | — | `RWIN_DIFF`, `EWIN_DIFF`, `RWIN_PCT` | n/a |
| days since the extreme | `WARG` = `ROLL_ARG` | — | — | n/a |
| relation of two series | `WREG` = `ROLL_REG`, `CORR` | — | — | `GRESID` |
| skew / kurtosis / autocorrelation | `WSKEW`, `WKURT`, `WACORR` | — | — | — |
| sum of absolute deviations | `WMAD` = `ROLL_MAD` | — | — | — |
| largest drop | `WDROP` = `ROLL_DROP`, `MAXDD` | — | — | n/a |

Pairs that look alike but mean different things are not interchangeable: `ROLL_RANK` (0..1 share) != `WRANK` (count); `SKEW` `KURT` `ACORR` (of the log returns of a price) != `WSKEW` `WKURT` `WACORR` (of x itself); `COUNT` (days true in N) != `GCOUNT` (valid members of a group).

<!-- range-algo:end -->

### 2.6 What is printed reads back

Formulas printed in lists, reports and search logs are canonical text; pasted back into `formulas` they compile to the same tree.

| Printed as | Example |
|---|---|
| always function form, single spaces | `GT(RATIO(close, LAG(20, close)), 1.1:RATIO)` |
| constants always carry a unit | `1.1:RATIO`, `20:DAYS` |
| two parameters printed separately | `RET_SKIP(252, 21, close)` |
| sets | `(MONTH() in {3, 5, 8, 11})` |
| key forms | `GSUM_BY(PARENT_ID(1), volume)` |
| a derived column (chapter 6) prints as the whole call | `CORR_GROUP(252, 126, 1, 3000, 1, RATIO(close, LAG(1, close)))` |

### 2.7 Errors: what is wrong + a fix to paste

An error message reads "what is wrong; fix: <VQL to paste>". In MCP the diagnostic is split into `msg` (what is wrong), `hint` (what to change) and `write` (the VQL itself).

Names from other languages are not aliases (argument order and meaning differ between languages, so accepting them would compile into something else); the error carries the form written from this call's arguments, with window and input swapped into place:

| Written | Fix | Example |
|---|---|---|
| `MA`, `SMA`, `AVG`, `MEAN`, `ROLL_AVG`, `TS_MEAN` | `ROLL_MEAN(N, x)` | `MA(close, 60)` -> `ROLL_MEAN(60, close)` |
| `SUM`, `MOVING_SUM`, `TS_SUM` | `WSUM(N, x)` | `SUM(volume, 5)` -> `WSUM(5, volume)` |
| `STD`, `STDEV`, `ROLL_STDEV`, `TS_STD` | `ROLL_STD(N, x)` | |
| `TS_MAX`, `TS_MIN`, `TS_RANK` | `ROLL_MAX`, `ROLL_MIN`, `ROLL_RANK` | |
| `PCT_CHANGE`, `ROC` | `RET(N, x)` | |
| `SHIFT`, `REF`, `DELAY` | `LAG(N, x)` | `SHIFT(close, 5)` -> `LAG(5, close)` |
| `RANK`, `PCT_RANK`, `CS_RANK` | `CS_PCT(x)` | |
| lower-case names (the screening vocabulary: `roll_mean(close, 20)`, `lag(close, 5)`, `min(a, b)`) | upper case, parameters first | `roll_mean(close, 20)` -> `ROLL_MEAN(20, close)` |

Other errors that carry a fix:

| Error | Fix |
|---|---|
| N of `RET`, `DIST_HIGH`, `DIST_LOW`, `VOL_RATIO`, `FN_YOY` above 252 (<= 1,260) | the same quantity with a long window: `RET(300, close)` -> `SUB(RATIO(close, LAG(300, close)), 1)`; `DIST_HIGH(500, close)` -> `SUB(RATIO(close, ROLL_MAX(500, close)), 1)`; `VOL_RATIO(300, vol)` -> `RATIO(vol, LAG(1, ROLL_MEAN(300, vol)))`; `FN_YOY(504, rev)` -> `LOGR(rev, LAG(504, rev))` |
| a rank inside a group `GRANK(g, x)` added to or compared with a ratio | `GPCT(g, x)` |
| a numeric column as the input of `EV_DAYS_SINCE`, `EV_POST_N` | `EV_CHANGE(column)` (the event = the day the column's value changes) |
| `FILL(c, 0:RATIO)` with c a condition | `FILL(c, 0:BOOL)` |
| BOOL constant other than 0 / 1 (`2:BOOL`) | `1:BOOL` (for a count write `2:COUNT`) |
| wrong argument count for `LE`, `AND`, `IF`, `MIN`, `MAX`, `GPCT` | the message states how many and gives the form |
| a `user.*` function not installed yet | the hint carries the whole definition shape to install; for one-off use write the formula inline |
| a hand-made correlation with `CORR` + `MASK(symbol == ...)` leaving an empty pool | `CORR_SEED({"2330", "2454"}, RATIO(close, LAG(1, close))) >= 0.5` (chapter 6) |
| a check parameter out of range or a wrong key | the hint carries a `check_params={...}` to paste (value clamped to the bound, defaults, every parameter's range) |
| a `portfolio` key named as in another system (`stop_loss`, `take_profit`, `top_n`, `rebalance`, `weight_cap`...) | the argument for that here and an example (e.g. `"exit_below": "pos.entry * 0.9"`); a typo gets the closest key |

Mismatches whose pasted fix would compile but most likely mean something else (a condition compared with a number, a count added to a ratio) get no fix, only the unit mismatch.

### 2.8 Vocabulary of the three slots

| Slot | Vocabulary | Example | Argument order |
|---|---|---|---|
| score formulas, conditions (stock pool, `hold`, `exposure`, `weight`, exits) | upper-case atoms and reserved functions | `ROLL_MEAN(20, close)`, `LAG(5, x)` | parameters first, inputs after |
| screening (threshold `screen_gate`, ordering `grid_by`, metric definitions; chapter 11) | upper-case functions, the same names as score formulas | `ROLL_MEAN(63, r)`, `SKIP(252, r)`, `BY_SUM(year, r)`, `MIN(x)` (aggregate) | parameters first, input after (as in score formulas) |
| check definitions (chapter 11) | upper-case functions, same names as score formulas | `AMIN(x)` (aggregate), `MIN2(a, b)` (two numbers), `COUNT`, `SUM`, `ALL`, `ANY`, `KEEP_BP` | — |

- One set of names across the three slots: `LAG(k, x)` shifts back in screening and in score formulas alike; the smaller of two numbers is `MIN2(a, b)`, the minimum of a set is `MIN(x)` (= `AMIN(x)`).
- In the screening and check slots function names are case-insensitive and the canonical form is upper case (11.1, 11.6). Old lower-case screening forms still work: input-first `roll_mean(r, 63)` is the same tree as `ROLL_MEAN(63, r)`, and an integer constant is accepted on either side. The old screening `lag(x, k)` (drop the first k cells) is renamed `SKIP(k, x)`; old expressions are rewritten automatically and their identity follows the rewrite (the same `content_hash` as writing `SKIP`). A score-formula-only name (`WSUM`, `RET`, `CS_PCT`...) answers that the name belongs to score formulas and conditions, with this slot's form where one exists (`WSUM` -> `ROLL_SUM(n, x)`).
- A score formula given a lower-case screening name answers with the upper-case, parameters-first form (2.7).

---

## 3. Values and units

### 3.1 Fixed point

Values in generation, backtest and the stock pool are **64-bit integer fixed point**, never floating point. The same formula therefore gives bit-identical results every time.

- Each atom floors once; division rounds toward negative infinity.
- A step that leaves i64, or divides by 0, makes that cell invalid.
- A cell is in one of two states: valid (with a value) or invalid.

### 3.2 The ten units

| Unit | Stored | Meaning | Examples |
|---|---|---|---|
| `PRICE` | x1e6 | money per share: prices and per-share values | `close`, `raw.limit_up`, `eps`, `dps` |
| `QTY` | integer (shares) | share counts | `volume`, `shares`, `foreign_net` |
| `AMOUNT` | integer (whole currency units) | money amounts, company totals | `revenue`, `fund.mktcap`, `px.turnover` |
| `RATIO` | x1e6 | a ratio, a unitless number | `px.turnover_rate`, `RET(20, close)` |
| `PCT` | x1e6 | percentage points | |
| `INDEX` | x1e6 | index points | `mkt.idx` |
| `DAYS` | integer | trading days | `EV_DAYS_SINCE(...)`, `WARG(...)` |
| `COUNT` | integer | counts, ordinals | `MONTH()`, `COUNT(...)`, `GRANK(...)` |
| `BOOL` | integer 0 / 1 | true or false | `close > open` |
| `CODE` | integer | codes (equality, `in`, group key only) | `industry`, `symbol`, `PARENT_ID(1)` |

"Numeric unit" (written X below) = the eight units other than BOOL and CODE.

A BOOL constant is 0 or 1 only (BOOL stores true / false): `2:BOOL` and `FILL(c, 2)` with c a condition are rejected with the fix `1:BOOL`; for a count write `2:COUNT`.

**A unit is a quantity**: `PRICE`, `QTY`, `AMOUNT`, `INDEX`, `PCT` are each one quantity -- money per share, shares, money, index points, percentage points; `RATIO`, `COUNT`, `DAYS`, `BOOL` are pure numbers without a quantity. Multiplication and division follow the quantities (3.3). How a unit is stored depends only on its quantity, never on the data: whole quantities (shares, money, counts, days, true/false, codes) are integers; divisible ones (money per share, ratios, percentage points, index points) are x1e6 fixed point.

- Per-share values (`eps`, `dps`, `FN_PS(...)`) are the same quantity as a price: they compare, add and divide with prices; a company total (`revenue`, `fund.mktcap`) is money, a share count away from a price.
- Shares x price is a money amount, AMOUNT (see "Money = shares x price" in 3.3).
- The old spelling `MONEY` (the unit per-share values and totals used to share) is still accepted: `column:MONEY` means the column's own unit, and a constant `5:MONEY` counts as a constant without a unit (read from the other side).

### 3.3 Unit algebra: what adds, multiplies, compares

| Operation | Inputs -> output | Rejected example |
|---|---|---|
| `ADD`, `SUB`, `MAX2`, `MIN2` | (X, X) same unit -> X | `close + volume` (PRICE + QTY); `revenue + close` (a total + a price) |
| `GT`, `LT` (and `>= <= == !=`) | (X, X) same unit -> BOOL | `close > volume`; `revenue > close` (a company total is a share count away from a price: per-share values are `FN_PS(revenue, shares)`, the average traded price is `VWAP(N)`) |
| `MUL` | quantities add: (QTY, PRICE), (PRICE, QTY) -> AMOUNT (shares x price = money); times a pure number: (RATIO or BOOL, X) -> X, (X, RATIO or BOOL) -> X (a condition times a weight), (COUNT or DAYS, X) -> X, (X, COUNT or DAYS) -> X; (BOOL, BOOL) is not accepted (write "and" as `&&`); a sum that is none of the quantities above is rejected | `volume * volume` (shares x shares); a total x shares (`revenue * shares`); `close * close` |
| `RATIO` (`/`), `MULDIV` | quantities subtract: (AMOUNT, QTY) -> PRICE (money / shares = a per-share value, = `FN_PS` except shares <= 0 are not rejected); (AMOUNT, PRICE) -> QTY (money / price = shares); (X, X) same unit -> RATIO; divided by a pure number: (X, RATIO or COUNT or DAYS) -> X; a difference that is no quantity is rejected | `close / volume` (money per share / shares); `RATIO(close, revenue)` (per-share value / total); `RATIO(1:RATIO, close)` |
| `LOGR` | (X, X) same unit -> RATIO | |
| `LOG`, percentiles, z-scores, returns | X -> RATIO | |
| `SQRT` | RATIO -> RATIO; PRICE / PCT / INDEX -> RATIO; QTY / AMOUNT / COUNT / DAYS also accepted (expanded at text level) -> RATIO | |
| `FLOOR`, `ROUND`, `CEIL` | x1e6 X -> X; integer units are themselves | |
| `EXP`, `SIN`, `COS`, `ATAN` | RATIO -> RATIO | `EXP(close)` |
| `NEG`, `ABS`, `DIFF`, window max / min / mean | X -> X | |
| `LAG` | any non-CODE (BOOL included) -> same unit | |
| `NOT` | BOOL -> BOOL | |
| `AND` | (BOOL, BOOL) -> BOOL | |
| `OR` | (BOOL, BOOL) -> BOOL | |
| `FILL` | (X, X) same unit (BOOL included, CODE not) -> same unit | |
| `MASK` | (BOOL, any non-CODE) -> the unit of the second input | |
| `VALID`, `EV_CHANGE`, `in {...}` | anything (CODE included) -> BOOL | |

When both sides of a product are integer units: COUNT x COUNT -> COUNT, COUNT x DAYS -> DAYS, DAYS x DAYS -> DAYS. Dividing two values of the same unit always gives RATIO (COUNT / COUNT is a ratio).

**Money = shares x price**: money per share (PRICE: prices and per-share values) times shares (QTY) is a money amount, unit AMOUNT (whole currency units, rounded down; negative values too); a column that only has share counts (institutional net buying, margin balance) times a price is a money amount, and `eps * shares` is total earnings. A threshold without a unit is read as AMOUNT from the other side; money / shares is back to money per share (PRICE) and compares with a price; money / price is shares. `close` is the adjusted price; for the day's traded price write `raw.close`.

| In words | Formula |
|---|---|
| foreign investors bought more than 500 million net over the last 5 days | `WSUM(5, foreign_net * close) > 5e8` |
| the average price of foreign net buying over 20 days is above the close | `WSUM(20, foreign_net * close) / WSUM(20, foreign_net) > close` |

To compare different units, first convert to unitless numbers: `RATIO(close, LAG(20, close))`, `CS_PCT(x)`, `LOG(x)`.

Each operator's own unit rule is in its row in chapter 4.

### 3.4 Invalid values

An invalid value takes part in nothing: **any invalid input -> invalid output.** Where invalid cells come from:

| Source | Example |
|---|---|
| the panel cell has no value | `close` before listing or on a halted day; statements not yet published |
| window not full, warm-up | the first 19 days of `ROLL_MEAN(20, close)` |
| any invalid cell inside the window | a halted day invalidates every window that spans it |
| division by 0, logarithm of a value <= 0, square root of a value < 0 | `RATIO(a, 0)`, `LOG(0)` |
| a step leaves i64 | `GSUM(0, equity)`: the whole-market sum does not fit i64; the means are not in this row (below) |
| a non-member of a cross-section | `CS_PCT(x)` of a stock outside the stock pool |

The means (`ROLL_MEAN`, `ROLL_STD`, `XL_PARENT`, `GMEAN_BY`, `GSTD`, `GSTD_BY`, `CS_REL_PARENT`, `CS_Z`, `CS_PEER_Z`, `CS_WINZ`) add up in exact 128-bit arithmetic and then divide; a mean always fits i64, so a sum that leaves i64 does not make them invalid. `WSUM` and `GSUM` output the sum itself and stay invalid when it does not fit i64. The weighted means `GWMEAN_BY`, `GWMEAN` (and `CS_NEUTRAL_IND`) shift the weights and x right until both sums fit i64, then divide: a member always has a value.

Four atoms handle invalid values explicitly:

| Atom | Behaviour | Example |
|---|---|---|
| `VALID(x)` | 1 if x is valid, else 0; always valid | `WSUM(250, VALID(close)) >= 250` |
| `FILL(x, c)` | x if valid, else c | `FILL(foreign_net, 0:QTY)` |
| `MASK(c, x)` | x if c is valid and true, else invalid | `MASK(VOL_RATIO(20, volume) >= 3, RET(1, close))` |
| `EV_CHANGE(x)` | 1 on the day the value changes, else 0; always valid | `EV_CHANGE(revenue_m)` |

What happens to an invalid cell:

| Place | An invalid cell |
|---|---|
| score formula | that stock is not ranked that day |
| stock pool (VQL form) | outside the pool (warm-up included) |
| `hold` | not bought, not held |
| `weight` | not bought |
| `rebalance_when` | no stock valid that day -> no rebalance |
| `exposure` | no stock valid that day -> the previous day's value; with no previous day, fully invested |

`MASK` versus multiplying: `MUL(condition, score)` gives 0 where the condition is false, and 0 still ranks; `MASK(condition, score)` is invalid there and does not rank. Use `MASK` to filter.

`!(condition)` versus `!FILL(condition, 0:BOOL)`: where the condition is invalid the first is invalid (dropped) and the second is true (kept).

---

## 4. All operators

### 4.1 How to read the tables

| Column | Content |
|---|---|
| Signature | parameters (integers) first, inputs after. `N` = window or days, `L` = level, `x a b` = inputs |
| Layer | A = atom (smallest, implemented directly by the engine); R = reserved function (composed of atoms; the atom definition of every one is in chapter 5); D = data (the thing obtained from outside itself, not a computation) |
| Params | the legal range of each parameter (code constants) |
| Units | inputs -> output. X = any numeric unit; "same" = the output has the unit of the input |
| Invalid and warm-up | rules beyond "any invalid input -> invalid"; warm-up = the first days this node makes invalid |
| Example | one complete score formula that compiles |

Common rules:

- Any invalid input -> invalid output. Window operators: any invalid cell inside the window -> invalid (rows with another rule say so).
- A window of N includes today: `[t-N+1, t]`. `LAG(N, x)` is N trading days ago.
- Warm-up adds up: the warm-up of an outer node = the warm-up of its children + its own.
- Return-type operators (`RVOL`, `SKEW`, `KURT`, `ACORR`, `RESID_MOM`) use log returns `ln(x[t]/x[t-1])` internally; a price <= 0 -> invalid. For the skewness, kurtosis or autocorrelation of any series itself (volume, a score) use `WSKEW`, `WKURT`, `WACORR` (no returns taken, values need not be > 0).
- Level L: 0 = everything (inside the stock pool), 1 = industry (or write `industry`), 2 = the panel's whole market (the exchange is the code column `market`, not this level), 3 = country (or write `country`), 4 = global. Levels 3 and 4 have no data in today's panels: every cell is invalid.
- Members of a cross-section or a group = stocks inside the stock pool that day whose input is valid. Cell-wise results (ranks, `CS_Z`, x - group mean, residuals) are invalid for non-members (in `CS_NEUTRAL_IND` w only weights the mean: a pool stock with a group and a valid x has a value, also with its own w <= 0); reductions that give one number per group (`GSUM`, `GCOUNT`, `GMAX`, `GMIN`, `GQUANT`, `GSTD`, `GMEAN`, `GWMEAN` and the key versions) are written to every stock in the pool that has the group, even when its own input is invalid, and are invalid only when the group has 0 members that day (`GSUM_ALL`, `GCOUNT_ALL` ignore the pool).
- Nodes whose output takes only a few values (bucket numbers, flags, group sums and means, which are the same for every member of a group) produce massive ties when used as a whole score; they are parts: use them as a condition, a group key, or divide / subtract a per-stock value by them.

### 4.2 Leaves and constants

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| column key (`LEAF`) | D | — | the column's unit | no value in that cell -> invalid | the column's value that day | `NEG(close)` |
| number (`CONST`) | A | fits 32 bits after scaling | declared unit | always valid | a constant | `MASK(close > 10, RET(20, close))` |
| large number (`CONSTE`) | A | 7 significant digits x 10^0..12 | declared unit | always valid | a constant that does not fit 32 bits; just write the number, it is chosen automatically | `MASK(fund.mktcap > 5e9, RET(20, close))` |

`LEAF`, `CONST`, `CONSTE` cannot be written as calls; write the column name or the number.

### 4.3 Element-wise arithmetic

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `ADD(a, b)` | A | — | (X, X) same unit -> same | overflow -> invalid | a + b | `ADD(CS_PCT(RET(20, close)), CS_PCT(RET(60, close)))` |
| `SUB(a, b)` | A | — | (X, X) same unit -> same | overflow -> invalid | a - b | `SUB(RET(60, close), RET(60, mkt.idx_tr))` |
| `MUL(a, b)` | A | — | see 3.3 | product leaves i64 -> invalid | a x b; with scaled units divided by 1e6, then floored | `MUL(RET(20, close), VOL_RATIO(20, volume))` |
| `RATIO(a, b)` (`DIV`) | A | — | see 3.3 | b = 0 -> invalid | a / b, floored | `RATIO(low, close)` |
| `MULDIV(K, a, b)` | A | K 1..1,000,000,000 | same as `RATIO(a, b)` | b = 0 or result leaves i64 -> invalid | K x a / b, 128-bit product, floored once; K x a never overflows (`RATIO(MUL(K:COUNT, a), b)` is invalid where K x a leaves i64 and equal everywhere else) | `MULDIV(4, net_income, equity)` |
| `NEG(a)` | A | — | X -> same | i64 minimum -> invalid | -a | `NEG(RVOL(60, close))` |
| `ABS(a)` | A | — | X -> same | | absolute value | `ABS(SUB(CS_PEER_RANK(1, close), CS_PEER_RANK(1, fund.mktcap)))` |
| `SQRT(a)` | A | — | RATIO -> RATIO; PRICE / PCT / INDEX -> RATIO; QTY / AMOUNT / COUNT / DAYS -> RATIO (expanded at text level) | a < 0 -> invalid | floor of the square root (fixed point); the output is a unitless number | `SQRT(fund.mktcap)` |
| `LOG(a)` | A | — | X -> RATIO | a <= 0 -> invalid | natural logarithm | `LOG(MUL(MUL(RET(20, close), RET(20, close)), VOL_RATIO(20, volume)))` |
| `EXP(a)` | A | — | RATIO -> RATIO | leaves i64 -> invalid | e to the power a | `EXP(NEG(RVOL(60, close)))` |
| `LOGR(a, b)` | A | — | (X, X) same unit -> RATIO | either <= 0 -> invalid | ln(a / b), rounded once (`LOG(RATIO(a, b))` rounds twice and can differ in the last digit) | `LOGR(close, LAG(1, close))` |
| `FLOOR(a)` | A | — | x1e6 X -> same; integer units are themselves | result leaves i64 -> invalid | to an integer, toward negative infinity (12.7 -> 12, -12.3 -> -13) | `MASK(FLOOR(raw.close % 10) == 8, RET(20, close))` |
| `MAX2(a, b)` | R | — | (X, X) same unit -> same (no BOOL) | | the larger of the two, cell by cell | `MAX2(DIFF(1, close), 0:PRICE)` |
| `MIN2(a, b)` | R | — | (X, X) same unit -> same (no BOOL) | | the smaller of the two | `MIN2(RET(20, close), RET(60, close))` |
| `SIN(a)` | A | — | RATIO -> RATIO | | sin a (radians) | `SIN(RET(20, close))` |
| `COS(a)` | A | — | RATIO -> RATIO | | cos a | `COS(RET(20, close))` |
| `ATAN(a)` | A | — | RATIO -> RATIO | | arctan a, in (-pi/2, pi/2) | `ATAN(RET(20, close))` |

The three rounding reserved functions `MOD`, `ROUND`, `CEIL` are in 5.2.

### 4.4 Comparison, logic, invalid values

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `GT(a, b)` | A | — | (X, X) same unit -> BOOL | | 1 if a > b, else 0 | `MASK(GT(close, open), RET(5, close))` |
| `LT(a, b)` | A | — | (X, X) same unit -> BOOL | | 1 if a < b, else 0 | `MASK(LT(close, ROLL_MEAN(20, close)), NEG(RET(5, close)))` |
| `NOT(a)` | A | — | BOOL -> BOOL | | not | `MASK(NOT(close < ROLL_MEAN(60, close)), RET(20, close))` |
| `AND(a, b)` | A | — | (BOOL, BOOL) -> BOOL | one side known false -> false; else either invalid -> invalid | and (`a && b`) | `MASK(AND(RET(5, close) > 0.1, VOL_RATIO(20, volume) >= 3), RET(5, close))` |
| `OR(a, b)` | R | — | (BOOL, BOOL) -> BOOL | one side known true -> true; else either invalid -> invalid | or. "And" is `AND(a, b)` | `MASK(OR(RET(5, close) > 0.1, VOL_RATIO(20, volume) >= 3), RET(5, close))` |
| `VALID(x)` | A | — | anything (CODE included) -> BOOL | always valid | 1 if x is valid, else 0 | `MASK(WSUM(250, VALID(close)) >= 250, RET(60, close))` |
| `FILL(x, c)` | A | — | (X, X) same unit (BOOL included, CODE not) -> same | invalid only when both are invalid | x if valid, else c | `ADD(FILL(foreign_net, 0:QTY), FILL(trust_net, 0:QTY))` |
| `MASK(c, x)` | A | — | (BOOL, any non-CODE) -> the unit of x | c invalid or false -> invalid | x only where c is true | `MASK(VOL_RATIO(20, volume) >= 3, RET(1, close))` |
| `x in {v, ...}` (`IN_SET`) | A | — | anything (CODE included) -> BOOL | x invalid -> invalid | x equals one of the values | `MONTH() in {3, 5, 8, 11} ? CS_PCT(FN_EGROW(252, eps)) : CS_PCT(RET(60, close))` |

`IN_SET` cannot be written as a call; always write `x in {...}`.

### 4.5 Time-series windows

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `LAG(N, x)` | A | N 1..1,260 | any non-CODE -> same | warm-up N | the value N days ago | `RATIO(close, LAG(20, close))` |
| `DIFF(N, x)` | R | N 1..1,260 | X -> same | warm-up N | x[t] - x[t-N] | `DIFF(60, foreign_ratio)` |
| `WSUM(N, x)` | A | N 1..1,260 | X -> same; BOOL -> COUNT | sum leaves i64 -> invalid; warm-up N-1 | sum of the last N days | `WSUM(5, foreign_net)` |
| `ROLL_MEAN(N, x)` | A | N 2..1,260 | X -> same | warm-up N-1 | mean of the last N days (floored) | `RATIO(close, ROLL_MEAN(60, close))` |
| `ROLL_MAX(N, x)` (`WMAX`) | A | N 2..1,260 | X -> same | warm-up N-1 | maximum of the last N days | `RATIO(close, ROLL_MAX(250, close))` |
| `ROLL_MIN(N, x)` (`WMIN`) | A | N 2..1,260 | X -> same | warm-up N-1 | minimum of the last N days | `RATIO(close, ROLL_MIN(250, close))` |
| `ROLL_STD(N, x)` | A | N 2..1,260 | X -> same | warm-up N-1 | population standard deviation of the last N days | `ROLL_STD(60, CS_PCT(fund.mktcap))` |
| `ROLL_RANK(N, x)` | R | N 2..1,260 | X -> RATIO 0..1 | warm-up N-1 | percentile of x[t] within the last N days | `ROLL_RANK(60, open)` |
| `COUNT(N, b)` | R | N 1..1,260 | BOOL -> COUNT | warm-up N-1 | number of true days in the last N | `MASK(COUNT(3, volume > LAG(1, volume) && close > LAG(1, close)) >= 3, RET(3, close))` |
| `FIRST_IN_WINDOW(N, b)` | R | N 2..1,260 | BOOL -> BOOL | warm-up N-1 | true today and false on the N-1 days before | `MASK(FIRST_IN_WINDOW(11, trust_net > 0), RATIO(trust_net, volume))` |
| `WRANK(N, x)` | A | N 2..1,260 | X -> COUNT | warm-up N-1 | mid-rank of x[t] within the last N days: 2 x (number smaller) + (number equal) - 1 | `WRANK(60, close)` |
| `WARG(N, side, x)` | A | N 2..252; side 0 highest / 1 lowest | X -> DAYS | warm-up N-1 | days since the highest (or lowest) of the last N days; ties take the most recent | `NEG(WARG(60, 0, close))` |
| `WQUANT(N, q, x)` | A | N 2..252; q 0..100 | X -> same | warm-up N-1 | element floor(q x (N-1) / 100) of the last N days sorted ascending (q = 50 is the lower median) | `RATIO(close, WQUANT(60, 50, close))` |
| `WMAD(N, x, c)` | A | N 2..252 | (X, X) same unit -> same | warm-up N-1 | sum over the last N days of the absolute difference between x and today's c | `RATIO(WMAD(20, close, ROLL_MEAN(20, close)), close)` |
| `WDROP(N, x)` | A | N 2..1,260 | X -> same | subtraction overflow -> invalid; warm-up N-1 | largest high-then-low drop inside the last N days (>= 0) | `NEG(WDROP(250, LOG(close)))` |
| `EMA(N, x)` | A | N 2..84 | X -> same | warm-up 3N-1 | exponential mean with alpha = 2/(N+1), restarted 3N cells back for every cell | `RATIO(close, EMA(20, close))` |
| `RET(N, x)` | R | N 1..1,260 | X -> RATIO | x[t-N] <= 0 -> invalid; warm-up N | x[t] / x[t-N] - 1 | `RET(60, close)` |
| `RET_SKIP(N, M, x)` | R | N 2..1,260; 1 <= M < N | X -> RATIO | x[t-N] <= 0 -> invalid; warm-up N | x[t-M] / x[t-N] - 1 (skips the last M days) | `CS_PCT(RET_SKIP(252, 21, close))` |
| `RVOL(N, x)` | R | N 2..252 | X -> RATIO | warm-up N | population standard deviation of N log returns | `NEG(RVOL(60, close))` |
| `PARK(N, a, b)` | A | N 2..252 | (X, X) same unit -> RATIO | warm-up N-1 | Parkinson volatility: a = high, b = low | `NEG(PARK(20, high, low))` |
| `GK(N, a, b)` | A | N 2..252 | (RATIO, RATIO) -> RATIO | warm-up N-1 | Garman-Klass volatility: a = high / low, b = close / open | `NEG(GK(20, RATIO(high, low), RATIO(close, open)))` |
| `SKEW(N, x)` | R | N 3..252 | X -> RATIO | warm-up N | skewness of N log returns (= `WSKEW(N, LOGR(x, LAG(1, x)))`) | `SKEW(60, close)` |
| `KURT(N, x)` | R | N 4..252 | X -> RATIO | warm-up N | excess kurtosis of N log returns (= `WKURT(N, LOGR(x, LAG(1, x)))`) | `NEG(KURT(60, close))` |
| `ACORR(N, k, x)` | R | N 3..252; 1 <= k <= N-2 | X -> RATIO | warm-up N | lag-k autocorrelation of N log returns (= `WACORR(N, k, LOGR(x, LAG(1, x)))`) | `ACORR(60, 1, close)` |
| `WSKEW(N, x)` | A | N 3..252 | X -> RATIO | any invalid cell in the window, or x constant -> invalid; warm-up N-1 | skewness of the last N values of x itself (population moments; any series, need not be > 0) | `WSKEW(60, volume)` |
| `WKURT(N, x)` | A | N 4..252 | X -> RATIO | as `WSKEW` | excess kurtosis of the last N values of x itself | `NEG(WKURT(60, RET(1, close)))` |
| `WACORR(N, k, x)` | A | N 3..252; 1 <= k <= N-2 | X -> RATIO | as `WSKEW` | lag-k autocorrelation (sample ACF) of the last N values of x itself | `WACORR(60, 1, volume)` |
| `SLOPE(N, x)` | A | N 2..252 | X -> same | warm-up N-1 | slope of the linear regression on time over the last N days (per day) | `RATIO(SLOPE(60, close), close)` |
| `R2(N, x)` | A | N 3..252 | X -> RATIO 0..1 | warm-up N-1 | R squared of the same regression | `R2(60, close)` |
| `WREG(N, mode, y, x)` | A | N 3..1,260; mode 0 slope, 1 intercept, 2 today's residual, 3 R squared, 4 t-stat of the slope | (X, X) same unit -> modes 0, 3, 4 RATIO, 1, 2 the unit of y | warm-up N-1; any invalid cell in the window, or x constant -> invalid | regression of y on x over the last N days (least squares; exact sums, rounded once); write it by name, `REG_SLOPE` ... `REG_T` (section 5.2) | `REG_SLOPE(120, RET(1, close), RET(1, mkt.idx))` |
| `INV_VOL_KEY(N, x)` | A | N 2..1,260 | RATIO -> COUNT (16,384..1,073,741,824) | warm-up N-1; fewer than half of the window's cells valid -> invalid | the inverse of x's volatility over the last N days, as a weight key (steadier is larger): the sum of squares Q of the k valid cells, vol = floor(sqrt(floor(Q / k)) / 10) (in 1e-5, clamped to 1..65,535), value = floor(2^30 / vol). `weighting="inv_vol"` is the weight key `INV_VOL_KEY(20, RET(1, close))` | `apx.SHARE(apx.PICK(10, false), INV_VOL_KEY(60, RET(1, close)))` |
| `MAXDD(N, x)` | A | N 2..252 | X -> RATIO | warm-up N-1 | maximum drawdown inside the last N days (positive; old definition) | `NEG(MAXDD(250, close))` |
| `DIST_HIGH(N, x)` | R | N 2..1,260 | X -> RATIO (<= 0) | warm-up N-1 | x[t] / highest of the last N days - 1 | `DIST_HIGH(250, close)` |
| `DIST_LOW(N, x)` | R | N 2..1,260 | X -> RATIO (>= 0) | warm-up N-1 | x[t] / lowest of the last N days - 1 | `DIST_LOW(250, close)` |
| `UP_STREAK(N, x)` | R | N 1..252 | X -> COUNT | warm-up N | consecutive rising days up to today, capped at N | `UP_STREAK(20, close)` |
| `DOWN_STREAK(N, x)` | R | N 1..252 | X -> COUNT | warm-up N | consecutive falling days, capped at N | `NEG(DOWN_STREAK(20, close))` |
| `CORR(N, a, b)` | A | N 3..252 | (X, Y) any numeric -> RATIO | warm-up N-1 | Pearson correlation of a and b over the last N days | `NEG(CORR(60, close, volume))` |
| `VOL_RATIO(N, x)` | R | N 1..252 | X -> RATIO | warm-up N | x[t] / mean of the previous N days (today excluded) | `VOL_RATIO(20, volume)` |
| `AMIHUD(N, a, b)` | A | N 1..252 | (X, AMOUNT) -> RATIO | warm-up N | mean over N days of \|daily return of a\| x 1e9 / traded value b (larger = less liquid) | `AMIHUD(20, close, px.turnover)` |
| `RESID_MOM(N, a, b)` | A | N 3..252 | (X, Y) any numeric -> RATIO | warm-up N | N-day cumulative residual of a after removing beta x the return of b | `RESID_MOM(120, close, mkt.idx)` |

### 4.6 Recursion and reset

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `SCANSUM(r, x)` | A | — | (BOOL, X) -> unit of x | invalid before the first true r; r or x invalid -> invalid that day, the chain breaks until r is next true | restarts from x on a day r is true, then adds x day by day | `SCANSUM(foreign_net <= 0, MUL(foreign_net > 0, 1:COUNT))` |
| `SCANMAX(r, x)` | A | — | (BOOL, X) -> unit of x | as `SCANSUM` | highest x since r was last true (that day included) | `RATIO(close, SCANMAX(EV_CHANGE(revenue_m) && revenue_m >= ROLL_MAX(250, revenue_m), close))` |
| `SCANMIN(r, x)` | A | — | (BOOL, X) -> unit of x | as `SCANSUM` | lowest x since r was last true (that day included) | `RATIO(close, SCANMIN(EV_CHANGE(revenue_m), close))` |
| `SCANSTD(r, x)` | A | — | (BOOL, X) -> unit of x | as `SCANSUM`; only one value on the day r is true -> invalid | population standard deviation of x since r was last true (that day included); exact formula floored once, never invalid because the sums are large | `NEG(SCANSTD(close > ROLL_MAX(60, LAG(1, close)), RET(1, close)))` |
| `LATCH(r, x)` | A | — | (BOOL, X) -> unit of x | invalid before the first sample; x invalid on a sampling day -> invalid until the next sample | records x on a day r is true and holds it | `MASK(LAG(5, EV_CHANGE(revenue)), RATIO(close, LATCH(EV_CHANGE(revenue), LAG(1, close))))` |
| `SMOOTH(K, a, x)` | A | K 0..756 | (RATIO, X) -> unit of x | x invalid -> invalid and restart; invalid for K steps after a start; warm-up K | exponential smoothing y <- y + floor(a x (x - y)), starting at y = x | `RATIO(close, SMOOTH(60, 0.1:RATIO, close))` |
| `RWIN(k, mode, x)` | A | k 1..12; mode 0 value k periods ago / 1 sum of the last k / 2 mean / 3 min / 4 max / 5 standard deviation / 7 change versus k periods ago; quantile and rank are written by name: `RWIN_QUANT`, `RWIN_RANK` (below) | X -> same (rank -> COUNT) | not enough periods -> invalid | a window over publications: one "period" = one change of value (a quarter for statements, a month for monthly revenue) | `RATIO(RWIN(4, 1, fund.div_paid), fund.mktcap)` |
| `RWIN_PCT(k, x)` | A | k 1..12 | X -> RATIO | value k periods ago = 0 -> invalid | (this period - k periods ago) / \|k periods ago\|; monthly revenue year over year uses k = 12 | `RWIN_PCT(12, revenue_m)` |
| `EWIN(k, mode, b, x)` | A | k 1..20; mode as in `RWIN`; quantile and rank by name: `EWIN_QUANT`, `EWIN_RANK` | (BOOL, X) -> unit of x (rank -> COUNT) | not enough periods, or x or b invalid that day -> invalid | a window over events: one "period" = a day where b is true (and x valid); `RWIN(k, m, x)` = `EWIN(k, m, EV_CHANGE(x), x)` | `EWIN(4, 2, EV_CHANGE(revenue_m), RET(1, close))` (mean move on the last 4 monthly-revenue release days) |

- Common reset combinations: a streak `SCANSUM(not holding, MUL(holding, 1:COUNT))`; the highest since an event `SCANMAX(event, x)`; volatility since entry `SCANSTD(entry, RET(1, close))`; the value on the event day `LATCH(event, x)`.
- The warm-up of `RWIN` is counted in periods and cannot be turned into a fixed number of days; cells at the start of the panel without enough periods are invalid.
- Trailing four quarters = `RWIN(4, 1, x)`; the same month last year = `RWIN(12, 0, revenue_m)`; the previous quarter = `RWIN(1, 0, x)`; the change from the previous period = `RWIN(1, 7, x)`.
- When a "period" is a day some condition holds rather than a value change, use `EWIN`: the mean close on the last 3 volume-spike days `EWIN(3, 2, volume > 2 * ROLL_MEAN(20, volume), close)`; `LATCH(c, close)` is the close on the latest such day and `EWIN(1, 0, c, close)` the one before it.
- Example: the mean reaction on the last 4 quarterly-report days = `EWIN(4, 2, EV_CHANGE(revenue), RET(1, close))`. `EV_CHANGE(revenue)` marks the report day (the day quarterly revenue changes); each report day's 1-day return is one period, averaged over the latest 4; a stock with fewer than 4 reports is invalid. `EWIN(4, 3, ...)` is the worst of the 4, `EWIN(1, 0, ...)` the reaction to the previous report.
- Median (quantile) and rank over the last k periods: `RWIN_QUANT(k, q, x)` = the floor(q*(k-1)/100)-th smallest of the last k published values (q 0..100, 50 = median; the same pick as `WQUANT` and `GQUANT`), e.g. EPS above the median of the last 8 quarters `eps > RWIN_QUANT(8, 50, eps)`. `RWIN_RANK(k, x)` = the rank of the current period among the last k = 2 x (periods below it) + (periods equal to it, itself included) - 1 (a COUNT, the same count as `WRANK`; k 2..12; with k distinct values the lowest is 0 and the highest 2k - 2), e.g. quarterly revenue is the highest of the last 12 quarters `RWIN_RANK(12, revenue) >= 22`; for a 0..1 share divide it yourself: `RATIO(RWIN_RANK(12, revenue), 22:COUNT)`. When a period is a day some condition holds use `EWIN_QUANT(k, q, b, x)` and `EWIN_RANK(k, b, x)` (k up to 20). Fewer than k periods, or x (or b) invalid that day -> invalid. The defining formulas are in section 5.2.
- Regression of one series on another: the five names below, each written `(N, y, x)`: y on x, y and x share one unit (return on return, price on price; the slope over time is `SLOPE`). They are reserved functions (section 5.2) that expand to the same atom `WREG(N, mode, y, x)`; the name picks the output (mode):

  | name | mode | output | unit | meaning |
  |---|---|---|---|---|
  | `REG_SLOPE` | 0 | slope | RATIO | how much y moves on average when x moves 1 (on market returns: the sensitivity) |
  | `REG_INTERCEPT` | 1 | intercept | unit of y | the mean of y when x is 0 |
  | `REG_RESID` | 2 | today's residual | unit of y | today's y minus the line at today's x (on market returns: what is left after the market) |
  | `REG_R2` | 3 | R squared | RATIO 0..1 | how much of y's variation the line explains |
  | `REG_T` | 4 | t-stat of the slope | RATIO | the slope over its standard error; a large absolute value = a significant slope |

### 4.7 Cross-section (same day, inside the stock pool)

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `CS_PCT(x)` | R | — | X -> RATIO 0..1 | fewer than 2 members -> invalid | percentile inside the stock pool that day | `CS_PCT(RET(60, close))` |
| `CS_Z(x)` | A | — | X -> RATIO | standard deviation 0 -> invalid | (x - pool mean) / pool standard deviation (= `GZ(0, x)`) | `CS_Z(RET(60, close))` |
| `CS_WINZ(k, x)` | R | k 5..100 | X -> RATIO | | `CS_Z` clipped at +-k/10 standard deviations | `CS_WINZ(30, RET(60, close))` |
| `CS_QCUT(N, x)` | R | N 2..100 | X -> COUNT 0..N-1 | | cuts `CS_PCT` into N buckets and returns the bucket number | `GRANK(CS_QCUT(5, fund.mktcap), RET(20, close))` |
| `CS_RANK_CHG(N, x)` | R | N 1..250 | X -> RATIO | warm-up N | today's `CS_PCT` - the `CS_PCT` of N days ago | `CS_RANK_CHG(20, RET(60, close))` |
| `CS_RANK_MEAN(N, x)` | R | N 2..250 | X -> RATIO | today must be valid and at least ceil(N/2) of the last N days valid; warm-up N-1 | mean of `CS_PCT` over the last N days | `CS_RANK_MEAN(20, RET(5, close))` |
| `CS_RANKMIX(L, flags, a, b)` | R | L 0..4; flags 0..3 (1 inverts a, 2 inverts b, 3 inverts both) | (X, Y) any numeric -> RATIO 0..2 | both must be members; fewer than 2 members -> invalid | the percentiles of a and of b inside group L, added (two equal-weight factors) | `CS_RANKMIX(0, 2, FN_EP(eps, close), RVOL(60, close))` |
| `CS_REL_RET(N, L, x)` | R | N 1..252; L 0..4 | X -> RATIO | warm-up N | N-day return - the mean return of the group | `CS_REL_RET(20, 1, close)` |
| `CS_NEUTRAL_IND(L, x, w)` | R | L 1..4 | (X, any numeric) -> unit of x | only w > 0 counts in the mean; valid whenever its own x is valid (also with its own w <= 0); x - mean outside i64 -> invalid | x - the w-weighted mean of the group's members = `SUB(x, GWMEAN(L, w, x))`, bit for bit (section 5.1) | `CS_NEUTRAL_IND(1, RET(20, close), fund.mktcap)` |
| `CS_NEUTRAL_SIZE(x, z)` | R | — | (X, any numeric) -> unit of x | | residual of x regressed on [1, z] inside the pool (= `GRESID(0, x, z)`, the whole market as one group) | `CS_NEUTRAL_SIZE(RET(20, close), LOG(fund.mktcap))` |
| `CS_NEUTRAL_COUNTRY(x, w)` | R | — | (X, any numeric) -> unit of x | | as `CS_NEUTRAL_IND` with the group fixed to the country (for multi-market panels) | `CS_NEUTRAL_COUNTRY(RET(20, close), fund.mktcap)` |
| `CS_NEUTRAL_MODEL(p, x, q)` | A | p = level 1..4 + K x 256, K <= 16 | (X, any numeric) -> unit of x | | group-demeaned, then minus the projection on K exposure columns; q must be a column written back by the risk model | — |

`CS_NEUTRAL_MODEL` needs columns that exist only on a panel with a risk model attached; production panels have none today, so it has no example.

### 4.8 Groups

A group can be given in three ways, all with the same membership rule:

| Way | Written | How the group is formed |
|---|---|---|
| Level | `GSUM(L, x)`: the first argument is **an integer without a unit** | L 0 everything, 1 industry (or write `industry`; where the market has industry codes, otherwise group with `CORR_GROUP`), 2 the panel's whole market, 3 country (or write `country`), 4 global (levels 3 and 4 have no data yet, every cell is invalid); a stock without a parent at level L is invalid |
| Group key | `GSUM(k, x)`: the first argument is anything else (a column, a formula, a constant with a unit such as `0:COUNT`); prints as `GSUM_BY(k, x)` | stocks with the same key on the same day form a group; key units CODE / COUNT / BOOL / DAYS |
| Whole-panel members | `GSUM_ALL(L, x)` | groups by level L, but membership **ignores the stock pool** |

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `GSUM(L, x)` | A | L 0..4 | X -> same; BOOL -> COUNT | sum leaves i64 -> the whole group invalid | sum over the group's members, given to every stock of the group in the pool | `RATIO(volume, GSUM(1, volume))` |
| `GCOUNT(L, x)` | A | L 0..4 | anything (BOOL, CODE included) -> COUNT | | number of members in the group | `RATIO(GRANK(1, RET(20, close)), GCOUNT(1, RET(20, close)))` |
| `GRANK(L, x)` | A | L 0..4 | X -> COUNT | fewer than 2 members -> invalid | mid-rank inside the group: 2 x (number smaller) + (number equal) - 1 | `GRANK(1, RET(20, close))` |
| `GMAX(L, x)` | A | L 0..4 | X -> same (no BOOL) | a single member counts | maximum of the group | `RATIO(RET(20, close), GMAX(1, RET(20, close)))` |
| `GMIN(L, x)` | A | L 0..4 | X -> same (no BOOL) | a single member counts | minimum of the group | `MASK(close <= GMIN(1, close), NEG(close))` |
| `GSTD(L, x)` | A | L 0..4 | X -> same (no BOOL) | fewer than 2 members -> invalid; all members equal -> 0 (valid) | population standard deviation of the group: mean = floor(sum / n), (x - mean) shifted right until the sum of squares fits 64 bits, square root, shifted back (same arithmetic as `ROLL_STD`) | `RATIO(SUB(RET(20, close), XL_PARENT(1, RET(20, close))), GSTD(1, RET(20, close)))` |
| `GSUM_BY(k, x)` | A | — | (key, X) -> unit of x; (key, BOOL) -> COUNT | key or x invalid -> in no group | sum over the members with the same key | `RATIO(volume, GSUM(close > open, volume))` |
| `GCOUNT_BY(k, x)` | A | — | (key, anything) -> COUNT | | number of members with the same key | `RATIO(GRANK(CS_QCUT(10, fund.mktcap), RET(20, close)), GCOUNT(CS_QCUT(10, fund.mktcap), RET(20, close)))` |
| `GRANK_BY(k, x)` | A | — | (key, X) -> COUNT | fewer than 2 members -> invalid | mid-rank among the members with the same key | `GRANK(PARENT_ID(1), RET(20, close))` |
| `GMAX_BY(k, x)` | A | — | (key, X) -> unit of x | | largest x among the members with the same key | `RATIO(RET(20, close), GMAX(CS_QCUT(5, fund.mktcap), RET(20, close)))` |
| `GMIN_BY(k, x)` | A | — | (key, X) -> unit of x | | smallest x among the members with the same key | `SUB(RET(20, close), GMIN(CS_QCUT(5, fund.mktcap), RET(20, close)))` |
| `GSTD_BY(k, x)` | A | — | (key, X) -> unit of x | fewer than 2 members -> invalid | population standard deviation among the members with the same key (as `GSTD`) | `GSTD(CS_QCUT(5, fund.mktcap), RET(20, close))` |
| `GMEAN_BY(k, x)` | A | — | (key, X) -> unit of x | a single member counts | mean among the members with the same key (128-bit sum, then divide: valid even when the sum leaves i64). k as in `GSUM`: an integer without a unit = level (0 whole market, 1..4), anything else = group key. `XL_PARENT(L, x)`, `GMEAN(L, x)` and `GMEAN_BY(L, x)` give the same tree | `SUB(RET(20, close), GMEAN_BY(1, RET(20, close)))` |
| `GWMEAN_BY(k, w, x)` | A | — | (key, any numeric, X) -> unit of x | a member needs w valid and > 0; a single member counts | the w-weighted mean of x among the members with the same key: the weights keep 20 bits (sw = max(0, bitlen(max w) - 20)) and x shifts right by sx = max(0, bitlen(max|x|) + 20 + bitlen(n) - 62) so both sums fit i64; value = floor(sum((w>>sw)(x>>sx)) / sum(w>>sw)) * 2^sx, so a member always has a value. k as in `GSUM`: an integer without a unit = level (0 whole market, 1..4), anything else = group key. `GWMEAN(L, w, x)`, `GWMEAN_BY(PARENT_ID(L), w, x)` and `GWMEAN_BY(L, w, x)` give the same tree | `GWMEAN_BY(CS_QCUT(5, fund.mktcap), fund.mktcap, RET(20, close))` |
| `ORD(k, x)` | A | — | (key, X) -> COUNT | non-members invalid (members as in `GRANK_BY`); a one-member group gives 0 | the **exact position** of x among the members with the same key: from 0 = largest, every group exactly 0..n-1. Equal values are broken by a fixed shuffle (the same rule as the stock picking ranks, never by ticker order); k as in `GSUM`: a unitless integer = level (`0` whole market, `1` industry, ...), anything else = group key | `ORD(1, x) == 0` = each industry's top one; for the top N use the reserved functions `GTOP(N, first, x)`, `CS_TOPK(k, x)` (5.2) |
| `GQUANT(q, first, x)` | A | q 0..100; `first` as in `GSUM`: an integer without a unit = level 0..4, anything else = group key | (key, X) -> unit of x (no BOOL) | one member = that member | the floor(q x (n - 1) / 100)-th smallest x among the group's members (as `WQUANT`); 50 = median, 0 = `GMIN`, 100 = `GMAX` | `close > GQUANT(50, 1, close)` |
| `GRESID(k, x, z)` | A | — | (key, X, any numeric) -> unit of x | non-members invalid (members: k, x, z all valid); a group of one -> 0; x minus the group mean outside i64 -> the whole group invalid | least-squares residual of x on [1, z] among the members of the same group (128-bit sums for the means, rounded once). k as in `GSUM`: an integer without a unit = level (`0` whole market, `1` industry, ...), anything else = group key | `GRESID(1, RET(20, close), LOG(fund.mktcap))` = size-neutral inside each industry |
| `PARENT_ID(L)` | D | L 1..4 | -> CODE | no parent at level L -> invalid | the id of the level-L parent; a group key only | `RATIO(volume, GSUM(PARENT_ID(1), volume))` |
| `GSUM_ALL(L, x)` | A | L 0..4 | X -> same; BOOL -> COUNT | no member in the group that day -> the whole group invalid; sum leaves i64 -> the whole group invalid | members = every stock with x valid that day (the stock pool is not applied); the result is written to every cell of the group, including cells where x is invalid, outside the pool, or halted. With L = 0 every stock gets the same value | `SUB(RET(20, close), RATIO(GSUM_ALL(0, RET(20, close)), GCOUNT_ALL(0, RET(20, close))))` |
| `GCOUNT_ALL(L, x)` | A | L 0..4 | anything -> COUNT | no member -> invalid (not 0) | the member count of `GSUM_ALL` | `MASK(RATIO(GSUM_ALL(1, close > ROLL_MEAN(60, close)), GCOUNT_ALL(1, close)) > 0.5, RET(20, close))` |
| `CS_PEER_RANK(L, x)` | R | L 1..4 | X -> RATIO 0..1 | fewer than 2 members -> invalid | percentile inside the group | `CS_PEER_RANK(1, RET(20, close))` |
| `CS_PEER_Z(L, x)` | A | L 1..4 | X -> RATIO | | z-score inside the group (= `GZ(L, x)`) | `CS_PEER_Z(1, FN_EP(eps, close))` |
| `CS_REL_PARENT(L, x)` | R | L 1..4 | X -> same | | x - the mean of the group's members | `CS_REL_PARENT(1, RET(20, close))` |
| `XL_PARENT(L, x)` | A | L 1..4 | X -> same | | the mean of the group's members; every member gets the same value; = `GMEAN_BY(PARENT_ID(L), x)`, both spellings give the same tree | `RATIO(volume, XL_PARENT(1, volume))` |
| `GWMEAN(L, w, x)` | R | L 1..4 | (any numeric, X) -> unit of x | a member needs w > 0 | the w-weighted mean of x among the group's members (industry cap-weighted return); = `GWMEAN_BY(PARENT_ID(L), w, x)`, the same tree; a first argument of 0 or a group key gives `GWMEAN_BY` | `GWMEAN(1, fund.mktcap, RET(20, close))` |

Common combinations:

| Wanted | Written |
|---|---|
| percentile inside a group | `GPCT(k, x)` (= `RATIO(GRANK(k, x), ADD(SUB(GCOUNT(k, x), 1:COUNT), SUB(GCOUNT(k, x), 1:COUNT)))`, section 5.2) |
| mean inside a group | `GMEAN(k, x)` (k = level or group key; same as `GMEAN_BY(k, x)`) |
| weighted mean inside a group (e.g. cap-weighted) | `GWMEAN(k, w, x)` (k = level or group key; same as `GWMEAN_BY(k, w, x)`) |
| minus the weighted group mean (industry neutralization) | `CS_NEUTRAL_IND(L, x, w)` = `SUB(x, GWMEAN(L, w, x))` |
| z-score inside a group | `GZ(k, x)` (section 5.2; level 0 = `CS_Z(x)`, 1..4 = `CS_PEER_Z(L, x)`) |
| how far above the group mean | `SUB(x, group mean)` |
| median / quantile inside a group | `GQUANT(50, k, x)`; the top-25% cut `GQUANT(75, k, x)` |
| remove another quantity's effect inside a group (regression residual) | `GRESID(k, x, z)`; the whole market as one group = `CS_NEUTRAL_SIZE(x, z)` |
| everything as one group | `GSUM(0, x)`, or a constant key `GSUM(0:COUNT, x)` |
| example keys | `PARENT_ID(1)` (industry), `CS_QCUT(5, ROLL_MEAN(20, volume))` (five volume buckets), `close > open` (up or down candle), `MONTH()`, `CORR_GROUP(...)` (chapter 6) |

- CODE (code columns, `PARENT_ID`) is only a group key: no arithmetic, and it cannot be a whole formula.
- Write the industry key as `industry` (accepted in conditions and score formulas; the same grouping as `1` and `PARENT_ID(1)`).
- Example: price above the industry median = `close > GQUANT(50, 1, close)` (`1` is the industry level; `industry` and `PARENT_ID(1)` are the same). How far above the median (not pulled by outliers, steadier than "minus the mean") = `RET(20, close) - GQUANT(50, 1, RET(20, close))`; the median of each market-cap decile `GQUANT(50, CS_QCUT(10, fund.mktcap), x)`; the whole-market median `GQUANT(50, 0, x)`.
- The percentile inside a group is 1 for the best stock of every group, so as a whole score it ties massively; rank by "how far above the group mean". When a percentile inside a group is wanted (e.g. `GPCT(g, x)`, inside the industry `CS_PEER_RANK(1, x)`), give `portfolio.tie_by` with the value being ranked (e.g. `"tie_by": "x"`) so ties are ordered by it.

### 4.9 Events

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `EV_CHANGE(x)` | A | — | anything -> BOOL | always valid; warm-up 1 | the day the value changes: x valid today and it is the first day, or yesterday was invalid, or the value differs from yesterday. For a carried statement or monthly revenue column this is the publication day | `MASK(EV_CHANGE(revenue_m), RET(1, close))` |
| `EV_PULSE(c)` | R | — | column -> BOOL | always valid; c must be a column itself | 1 when the column has a value that day (event columns: ex-dividend) | `MASK(EV_PULSE(dps), RET(5, close))` |
| `EV_POST_N(N, b)` | R | N 1..250 | BOOL -> BOOL | warm-up N | b was true N days ago (the N-th day after the event) | `MASK(EV_POST_N(5, EV_CHANGE(revenue_m)), RET(5, close))` |
| `EV_DAYS_SINCE(N, b)` | R | N 1..250 | BOOL -> DAYS | no event within N days -> invalid; warm-up N | days since b was last true (0 on the day) | `NEG(EV_DAYS_SINCE(60, EV_CHANGE(revenue_m)))` |
| `EV_EARN_DRIFT(N, a, b)` | R | N 1..252 | (X, Y) any numeric -> RATIO | no publication within N days -> invalid; warm-up N | a = price, b = statement column: the return of a from the day before the latest publication (value change) of b to today | `EV_EARN_DRIFT(60, close, eps)` |
| `EV_IDX_CHG(N, b)` | R | N 1..252 | BOOL -> RATIO -1..1 | warm-up N | b = a membership flag: positive for the last false-to-true change within N days, negative for true-to-false, larger in size the more recent; 0 without a change | `EV_IDX_CHG(60, close > ROLL_MEAN(200, close))` |

### 4.10 Fundamentals

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `FN_YOY(N, x)` | R | N 1..1,260 | X -> RATIO | either <= 0 -> invalid; warm-up N | ln(x[t] / x[t-N]): log growth (0.1 is about 10%) | `FN_YOY(252, revenue_m)` |
| `FN_EGROW(N, x)` | R | N 1..252 | X -> RATIO | base 0 -> invalid; warm-up N | (x[t] - x[t-N]) / \|x[t-N]\|: works with a negative base | `CS_PCT(FN_EGROW(252, eps))` |
| `FN_GM_CHG(N, a, b)` | R | N 1..252 | (X, X) same unit -> RATIO (gross profit and revenue are AMOUNT) | b <= 0 -> invalid; warm-up N | a = gross profit, b = revenue: gross margin - the gross margin N days ago | `FN_GM_CHG(252, fund.gross, revenue)` |
| `FN_DY(N, a, b)` | R | N 1..252 | (PRICE, PRICE) -> RATIO | b <= 0 -> invalid; a must be a per-share dividend column that has a value only on ex-dividend days; warm-up N-1 | dividends of the last N days / price = dividend yield | `FN_DY(250, dps, raw.close)` |

Per-share values and totals are different quantities: `FN_EP`, `FN_BP`, `FN_DY` follow division and must come out as a ratio, so the first input must be the same quantity as a price (a per-share value: `eps`, `dps`, `FN_PS(...)`); a total / a price is shares, not a ratio, and is rejected. To compare a total with market cap write `RATIO(total, fund.mktcap)`.

The financial ratios derived by dividing data columns, `FN_PS`, `FN_EP`, `FN_BP`, `FN_ROE`, `FN_ACCRUAL`, are data, in section 7.4. The four left here all carry a window (N periods ago, a sum over the last N days) and are computations built from atoms.

### 4.11 Calendar

The date is data; splitting it is an atom: which day the row is (`EPOCH_DAY()`, the exchange calendar) and the panel row number (`DAY_INDEX()`) come from outside, layer D; `MONTH()`, `WEEKDAY()`, `MDAY()`, `YEAR()` split the date into month, weekday, day of month and year, a computation, layer A. The names, spellings and results do not change with the classification.

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `MONTH()` | A | — | -> COUNT 1..12 | panel without a calendar -> invalid | month | `MASK(MONTH() == 1, NEG(RET(20, close)))` |
| `WEEKDAY()` | A | — | -> COUNT 0..6 | as above | weekday, 0 = Monday, 4 = Friday | `MASK(WEEKDAY() == 0, NEG(RET(1, close)))` |
| `MDAY()` | A | — | -> COUNT 1..31 | as above | day of the month | `MASK(MDAY() <= 10, RET(5, close))` |
| `YEAR()` | A | — | -> COUNT | as above | calendar year | `MASK(YEAR() >= 2020, RET(60, close))` |
| `EPOCH_DAY()` | D | — | -> DAYS | as above | days since 1970-01-01 (calendar days, market holidays included); week number = `FLOOR((EPOCH_DAY() + 3) / 7)` (weeks start on Monday) | `EPOCH_DAY() - LAG(1, EPOCH_DAY()) >= 4` (first trading day after a long holiday) |
| `DAY_INDEX()` | D | — | -> DAYS | always valid | panel day number (0 on the first day) | `SCANSUM(DAY_INDEX() < 1, volume)` |

The reserved functions for calendar months and listing age (`TDAY_OF_MONTH`, `AT_MONTH_START`, `LISTED_TDAYS`, `LISTED_DAYS`) are in 5.2.

Reproducible noise (for checks such as "does the pick survive noise on the score"):

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `NOISE(k)` | A | k 0..65535 | -> RATIO -1..1 | always valid | uniform noise fixed by (the check's seed, k, the day, the stock): rerunning the same check gives the same bits, another k is another draw; outside a check the seed is 0 | `ADD(CS_PCT(RET(60, close)), MUL(0.05, NOISE(1)))` |

### 4.12 Algorithmic operators (62)

Bounded algorithmic features of a window, in four families: LZ (Lempel-Ziv complexity, 19), RT (record test, 21), ENT (Shannon entropy, 3), VG (visibility graph, 19). All are atoms.

Common rules:

| Item | Rule |
|---|---|
| Signature | `NAME(W, a)`, `NAME(W, a, b)`; those that read a threshold: `NAME(W, level, a)`, `NAME(W, level, a, b)` |
| W | window 2..256 |
| level | the threshold, in hundredths (`80` = 0.8, `150` = 1.5); 0..32,767. Operators that read no threshold cannot take a level |
| Inputs | any numeric unit; two inputs may have different units |
| Output | "fx" -> RATIO; "int" -> COUNT; "+-1" -> COUNT (-1 / 0 / 1); "0/1" -> BOOL |
| Invalid and warm-up | valid only when the whole window is valid. Warm-up: W for LZ, ENT, VG; W-1 for RT |

A BOOL output cannot be a whole score formula; put it in `MASK(..., score)` or in a condition. A "+-1" output takes only three values and ties massively as a whole score; multiply another score by it or use it as a condition.

| Name | Inputs | Threshold | Output | Meaning |
|---|---|---|---|---|
| `LZ_C_DIFF` | a | — | fx | how disordered the up / flat / down sequence of a is over W days (about 1 looks random; lower = more regular) |
| `LZ_C_SIGN` | a | — | fx | how disordered the positive / zero / negative sequence of a is (a = close - open: the pattern of up and down candles) |
| `LZ_C_MED` | a | — | fx | how disordered the above / below the window median sequence of a is |
| `LZ_C_JOINT` | a, b | — | fx | how disordered the joint sequence of the moves of a and the highs / lows of b is |
| `LZ_N_DIFF` | a | — | int | number of distinct phrases cut from the up / down sequence of a (fewer = more regular) |
| `LZ_N_SIGN` | a | — | int | number of distinct phrases in the sign sequence of a |
| `LZ_N_MED` | a | — | int | number of distinct phrases in the above / below median sequence of a |
| `LZ_N_JOINT` | a, b | — | int | number of distinct phrases in the joint sequence |
| `LZ_GAP` | a, b | — | fx | disorder of the high / low sequence of b - disorder of the up / down sequence of a |
| `LZ_ORDERED_DIFF` | a | read | +-1 | the net direction of a over W days when the disorder of its moves is below the threshold, else 0 |
| `LZ_ORDERED_SIGN` | a | read | +-1 | the majority sign of a when the disorder of its signs is below the threshold, else 0 |
| `LZ_ORDERED_JOINT` | a, b | read | +-1 | the net direction of a when the disorder of the joint sequence is below the threshold, else 0 |
| `LZ_ORDERED_X` | a, b | read | +-1 | the net direction of a when the disorder of the moves of b is below the threshold, else 0 |
| `LZ_LEADS_MED` | a, b | read | 0/1 | 1 when the highs / lows of b are regular while the moves of a are still disordered |
| `LZ_LEADS_DIFF` | a, b | read | 0/1 | 1 when the moves of b are regular while the moves of a are still disordered (b = volume, for instance) |
| `LZ_QUIET` | a, b | read | 0/1 | 1 when the moves of a and the highs / lows of b are both below the threshold |
| `LZ_ECHO_DIFF` | a | — | +-1 | today's direction of a when the up / down sequence of a ends in a long repeated phrase, else 0 |
| `LZ_ECHO_MED` | a, b | — | +-1 | today's direction of a when the high / low sequence of b ends in a long repeated phrase, else 0 |
| `LZ_ECHO_DOUBLE` | a, b | — | +-1 | today's direction of a when both sequences end in a long repeated phrase, else 0 |
| `RT_ZF` | a | — | fx | z-value of (new highs - new lows), counted from old to new |
| `RT_ZB` | a | — | fx | z-value of (new highs - new lows), counted backwards from today |
| `RT_ZS` | a | — | fx | z-value of the total number of new highs and new lows |
| `RT_HIGH_SHARE` | a | — | fx | share of new highs among the records, 0..1 |
| `RT_RECENT_SHARE` | a | — | fx | share of new lows among the records counted backwards from today, 0..1 |
| `RT_DIVERGENCE` | a | — | fx | the old-to-new z-value - the backward z-value |
| `RT_TREND` | a | read | +-1 | +1 when the old-to-new z-value is above the threshold, -1 below its negative, else 0 |
| `RT_RECENT` | a | read | +-1 | +1 when the backward z-value is above the threshold, -1 below its negative, else 0 |
| `RT_QUIET` | a | — | 0/1 | 1 when there are fewer records than random (total z < 0) |
| `RT_TURN` | a | read | +-1 | the backward direction when the old-to-new and backward directions disagree, else 0 |
| `RT_BURST` | a | read | 0/1 | 1 when the z-value of the total number of records is above the threshold (records are dense) |
| `RT_HIGH_SIDE` | a | — | +-1 | with enough records: +1 when new highs are 80% or more, -1 when 20% or less, else 0 |
| `RT_RECENT_SIDE` | a | — | +-1 | with enough backward records: +1 / -1 by the share of new lows, else 0 |
| `RT_AGREE` | a | read | +-1 | the direction when the old-to-new and backward directions agree, else 0 |
| `RT_LATE` | a | — | +-1 | the side of the last record when it falls in the newest quarter of the days (new high +1, new low -1), else 0 |
| `RT_EARLY` | a | — | 0/1 | 1 when there is a record and the last one falls in the older half |
| `RT_BURST_SIDE` | a | read | +-1 | when records are dense, +1 / -1 for whichever of new highs or new lows is more frequent, else 0 |
| `RT_STAIR` | a | — | +-1 | +1: no backward new-high record and at least 2 old-to-new new-high records (a staircase up); -1: the mirror for new lows; else 0 |
| `RT_NET_F` | a | — | int | new highs - new lows, counted from old to new |
| `RT_NET_B` | a | — | int | new lows - new highs, counted backwards from today |
| `RT_COUNT_F` | a | — | int | total number of new highs and new lows |
| `ENT_DIFF3` | a | — | fx | how even the distribution of up / flat / down of a is (Shannon entropy in bits; higher = more random) |
| `ENT_SIGN3` | a | — | fx | how even the distribution of positive / zero / negative of a is |
| `ENT_JOINT6` | a, b | — | fx | how even the distribution over the six joint states of the moves of a and the highs / lows of b is |
| `VG_IRREV` | a | — | fx | how different the series looks forwards and backwards (visibility-graph irreversibility) |
| `VG_IRREV_SKEW` | a | — | fx | irreversibility of rising stretches - that of falling stretches |
| `VG_DEGREE` | a | — | fx | mean number of links in the visibility graph |
| `VG_DEGREE_BIAS` | a | — | fx | bias between forward and backward link counts |
| `VG_LAMBDA` | a | — | fx | decay rate of the link-count distribution (bits) |
| `VG_REACH_DIFF` | a | — | int | points visible forwards from today - points visible backwards |
| `VG_SPAN_MEAN` | a | — | fx | mean number of days a link spans |
| `VG_REGIME` | a | — | +-1 | +1 when the mean link count is 5% or more above the expectation for a random series, -1 when 5% or more below, else 0 |
| `VG_REACH` | a | — | +-1 | +1 when today links to clearly more points than expected in the graph of highs; -1 when the same holds in the graph of lows; else 0 |
| `VG_HUB` | a | — | +-1 | +1 when only the graph of highs has a hub (a few points linked to most others), -1 when only the graph of lows has one, else 0 |
| `VG_FULL_REACH` | a | — | +-1 | today links to the oldest day of the window: +1 in the graph of highs, -1 in the graph of lows, else 0 |
| `VG_IRREV_SIDE` | a | — | +-1 | the direction +1 / -1 when irreversibility is large enough, else 0 |
| `VG_STACK` | a | — | +-1 | +1 when the graph of highs is stacked at least 2 layers deeper than the graph of lows, -1 the other way round, else 0 |
| `VG_SPAN` | a | — | +-1 | +1 when links in the graph of highs span clearly longer than in the graph of lows, -1 the other way round, else 0 |
| `VG_LINK_BREAK` | a | — | 0/1 | 1 when today links only to yesterday in both graphs |
| `VG_LONE` | a | — | +-1 | +1 when today links only to yesterday in the graph of lows and to 3 or more points in the graph of highs; -1 for the mirror; else 0 |
| `VG_EXTREME` | a | — | +-1 | +1 when today is the highest of the window, -1 when the lowest, else 0 |
| `VG_TILT` | a | — | +-1 | the direction of the difference between forward and backward link counts |
| `VG_L1` | a | — | int | total distance between the forward and backward link-count distributions |

Examples:

| Plain words | Written |
|---|---|
| the more regular the last 60 days of moves, the higher | `NEG(LZ_C_DIFF(60, close))` |
| rank by the 20-day return only when moves are regular (negated when the direction is down) | `MUL(LZ_ORDERED_DIFF(60, 80, close), RET(20, close))` |
| score only where volume has become regular while price is still disordered | `MASK(LZ_LEADS_DIFF(60, 80, close, volume), RET(20, close))` |
| frequent new highs over the last 120 days | `RT_ZF(120, close)` |
| the more concentrated the joint price-volume states, the higher | `NEG(ENT_JOINT6(60, close, volume))` |

### 4.13 Operators with three or four inputs

| Name | Inputs | Meaning | Status |
|---|---|---|---|
| `SELECT(c, a, b)` | (BOOL, X, X) -> X | c invalid -> invalid; a where c is true, else b | reserved function (section 5.1); not accepted in formula text, write `c ? a : b` (= `FILL(MASK(c, a), MASK(NOT(c), b))`) |
| `REC2(K, x, c0, c1, c2)` | x: X; coefficients RATIO -> X; K 0..756 | second-order linear recursion y[t] = floor((c1 x y[t-1] + c2 x y[t-2] + c0 x x[t]) / 1e6); y = x on the two starting days; invalid for K steps after a start | not accepted in formula text |
| `GRESID(k, x, z)` | (key, X, any numeric) -> X | within-group regression residual (per day, per group; section 4.8) | accepted |
| `STATESCAN(x, up, lo)` | all three the same numeric unit -> X | conditional state scan (per symbol, along the days): the first day is up and the value is up; in the up state, x below the previous value switches to the down state and the value becomes lo, otherwise the value is max(previous, up); the down state mirrors it (x above the previous value switches to up with the value up, otherwise min(previous, lo)). Any input invalid -> invalid that day, restart on the next day all three are valid | accepted |

`STATESCAN` example: a ratcheting trailing stop `STATESCAN(close, ROLL_MAX(20, close) * 0.9, ROLL_MIN(20, close) * 1.1)`: in the up state the stop only moves up, a close below it flips the line above the price; `close > STATESCAN(close, ROLL_MAX(20, close) * 0.9, ROLL_MIN(20, close) * 1.1)` = currently in the up state.

A node of a formula tree has only two children: a three-input operator gets one extra internal node in the tree that holds its second and third inputs (not counted in the complexity node count, never shown). The four-input `REC2` still returns `APX_UNSUPPORTED`.

---

## 5. Reserved functions as atoms

There are only two layers: an **atom** is a smallest operation, implemented directly by the engine; a **reserved function** is composed of atoms and always decomposes back into atoms. There are two kinds of reserved function:

- **Marked R in chapter 4**: their names are in the operator tables. 5.1 lists the atom definition of every one, bit-identical to writing the name.
- **Expanded at text level** (5.2): replaced by atoms when the formula is read, with no operator number of their own. The expanded nodes count toward the formula size.

### 5.1 Reserved functions marked R and their atom definitions (41)

The reserved function and the atom form on the right are bit-identical (value, valid bits, unit). An empty "Condition" = for all inputs; otherwise bit-identical within that range.

| Reserved function | Atom definition | Condition |
|---|---|---|
| `DIFF(N, x)` | `SUB(x, LAG(N, x))` | |
| `OR(a, b)` | `NOT(AND(NOT(a), NOT(b)))` | |
| `MAX2(a, b)` | `FILL(MASK(GT(a, b), a), MASK(VALID(a), b))` | |
| `MIN2(a, b)` | `FILL(MASK(LT(a, b), a), MASK(VALID(a), b))` | |
| `SELECT(c, a, b)` (written `c ? a : b`) | `FILL(MASK(c, a), MASK(NOT(c), b))` | |
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
| `CS_RANKMIX(L, flags, a, b)` | the same four forms with `CS_PEER_RANK(L, ...)` in place of `CS_PCT` | L 1..4 |
| `RET(N, x)` | `MASK(GT(LAG(N, x), 0), RATIO(SUB(x, LAG(N, x)), LAG(N, x)))` | |
| `RET_SKIP(N, M, x)` | `MASK(GT(LAG(N, x), 0), RATIO(SUB(LAG(M, x), LAG(N, x)), LAG(N, x)))` | |
| `DIST_HIGH(N, x)` | `MASK(GT(ROLL_MAX(N, x), 0), RATIO(SUB(x, ROLL_MAX(N, x)), ROLL_MAX(N, x)))` | |
| `DIST_LOW(N, x)` | `MASK(GT(ROLL_MIN(N, x), 0), RATIO(SUB(x, ROLL_MIN(N, x)), ROLL_MIN(N, x)))` | |
| `FN_YOY(N, x)` | `LOGR(x, LAG(N, x))` | |
| `FIRST_IN_WINDOW(N, b)` | `MASK(VALID(b), MASK(VALID(WSUM(N-1, LAG(1, b))), AND(b, LT(WSUM(N-1, LAG(1, b)), 1:COUNT))))` | b only 0/1 (comparisons, `NOT`, `OR`, `VALID`, `EV_PULSE`, `EV_CHANGE`, `AND` and their `LAG`, `MASK`, `FILL`) |
| `COUNT(N, b)` | `WSUM(N, b)` | |
| `EV_POST_N(N, b)` | `LAG(N, b)` | |
| `EV_PULSE(c)` | `VALID(c)` | c is a column |
| `FN_DY(N, a, b)` | `MASK(GT(b, 0), RATIO(WSUM(N, FILL(a, 0:PRICE)), b))` | a >= 0 (dividends) |
| `FN_EGROW(N, x)` | `RATIO(SUB(x, LAG(N, x)), ABS(LAG(N, x)))` |  |
| `FN_GM_CHG(N, a, b)` | `DIFF(N, MASK(GT(b, 0), RATIO(a, b)))` |  |
| `CS_REL_RET(N, L, x)` | r = `RET(N, x)`: `MASK(GT(GCOUNT(L, r), 1:COUNT), SUB(r, RATIO(GSUM(L, r), GCOUNT(L, r))))` |  |
| `CS_RANK_MEAN(N, x)` | p = `CS_PCT(x)`: `MASK(AND(VALID(p), NOT(LT(MUL(2:COUNT, WSUM(N, VALID(p))), N:COUNT))), RATIO(WSUM(N, FILL(p, 0:RATIO)), WSUM(N, VALID(p))))` |  |
| `RVOL(N, x)` | `ROLL_STD(N, LOGR(x, LAG(1, x)))` |  |
| `SKEW(N, x)` | `WSKEW(N, LOGR(x, LAG(1, x)))` |  |
| `KURT(N, x)` | `WKURT(N, LOGR(x, LAG(1, x)))` |  |
| `ACORR(N, k, x)` | `WACORR(N, k, LOGR(x, LAG(1, x)))` |  |
| `EV_DAYS_SINCE(N, b)` | s = `SUB(SCANSUM(FILL(b, 0:BOOL), 1:DAYS), 1:DAYS)`: `MASK(NOT(GT(s, N:DAYS)), s)` |  |
| `UP_STREAK(N, x)` | c = `FILL(GT(x, LAG(1, x)), 0:BOOL)`: `MASK(AND(VALID(x), VALID(LAG(N, VALID(x)))), MIN2(SCANSUM(NOT(c), MUL(c, 1:COUNT)), N:COUNT))` |  |
| `DOWN_STREAK(N, x)` | as `UP_STREAK` with `LT` in place of `GT` |  |
| `EV_EARN_DRIFT(N, a, b)` | e = `EV_CHANGE(b)`, base = `LATCH(e, LAG(1, a))`, s = `SUB(SCANSUM(e, 1:DAYS), 1:DAYS)`: `MASK(AND(GT(base, 0), LT(s, N:DAYS)), RATIO(SUB(a, base), base))` |  |
| `EV_IDX_CHG(N, b)` | e = `EV_CHANGE(b)`, s = `SUB(SCANSUM(e, 1:DAYS), 1:DAYS)`, m = `RATIO(SUB(N:DAYS, s), N:DAYS)`: `MASK(VALID(WSUM(N+1, b)), FILL(MASK(LT(s, N:DAYS), FILL(MASK(b, m), MASK(NOT(b), NEG(m)))), 0:RATIO))` | b only 0/1 |
| `CS_NEUTRAL_COUNTRY(a, b)` | `CS_NEUTRAL_IND(3, a, b)` |  |
| `GWMEAN(L, w, x)` | `GWMEAN_BY(PARENT_ID(L), w, x)` |  |
| `CS_NEUTRAL_IND(L, a, b)` | `SUB(a, GWMEAN(L, b, a))` |  |
| `CS_NEUTRAL_SIZE(a, b)` | `GRESID(0:COUNT, a, b)` |  |
| `VOL_RATIO(N, x)` | s = `WSUM(N, LAG(1, x))`: `MASK(GT(s, 0), MULDIV(N, x, s))` | x >= 0 (volume, turnover) |

- `2*(N-1)`, `N-1`, `N+1`, `k/10` in a definition are the numbers after the parameter is substituted (the divisor of `ROLL_RANK(60, x)` is `118:COUNT`).
- A leading "s = ...", "c = ..." in a long definition only writes a repeated sub-formula once; expanded, it is the same atom tree.
- N of `RET`, `RET_SKIP`, `DIST_HIGH`, `DIST_LOW`, `FN_YOY` goes up to 1,260: above 252 the formula is expanded into the atom definition of the table when it is read (`LAG`, `ROLL_MAX` / `ROLL_MIN`, `LOGR` already go to 1,260), and the nodes count as expanded; whatever N is, it is the same definition with the same result.
- The division in `FN_DY` uses the unit rule of chapter 3: a per-share value / a price (the same unit) -> ratio. The reserved function must come out as a ratio, so a total as the first input is rejected; the atom form's `RATIO` follows chapter 3 (a total / a price is shares).
- The financial ratios of section 7.4 (data) also each have a bit-identical atom form, listed in that section.

### 5.2 Reserved functions expanded at text level (40)

Both `NAME` and `apx.NAME` are accepted. They expand to the atoms on the right, so the three hardware targets agree automatically.

| Reserved function | Expansion | Output | Meaning | Example |
|---|---|---|---|---|
| `GPCT(g, x)` | `RATIO(GRANK(g, x), ADD(SUB(GCOUNT(g, x), 1:COUNT), SUB(GCOUNT(g, x), 1:COUNT)))` | RATIO 0..1 | percentile of x within its group (smallest 0, largest 1, ties share the middle; fewer than 2 members -> invalid). g = a level number (0 whole market, 1..4) or a group key (any formula; equal values on a day = one group); same arithmetic as `CS_PCT`, `CS_PEER_RANK` (`GPCT(0, x)` = `CS_PCT(x)`, `GPCT(1, x)` = `CS_PEER_RANK(1, x)`). Every group's best is 1: as the whole score with fewer picks than groups, give `portfolio.tie_by` | `GPCT(CS_QCUT(10, close), RET(20, close))` |
| `MOD(a, n)` (= `a % n`) | `SUB(a, MUL(n, FLOOR(RATIO(a, n))))` | unit of a | the remainder of a divided by n, with the sign of n; n = 0 -> invalid | `MASK(sym.code_num % 10 == MDAY() % 10, RET(20, close))` |
| `ROUND(x)` | `FLOOR(ADD(x, 0.5))` | unit of x | round half toward positive infinity; integer units are x itself | `ROUND(RATIO(volume, 1000))` |
| `CEIL(x)` | `NEG(FLOOR(NEG(x)))` | unit of x | round up; integer units are x itself | `CEIL(RATIO(volume, 1000))` |
| `WEEK_START()` | `GT(FLOOR(RATIO(ADD(EPOCH_DAY(), 3:DAYS), 7:COUNT)), LAG(1, FLOOR(RATIO(ADD(EPOCH_DAY(), 3:DAYS), 7:COUNT))))` | BOOL | the first trading day of the week (Tuesday when Monday is a holiday; a fully closed week follows the calendar): true when the week number differs from the previous trading day's; invalid on the panel's first day | `MASK(WEEK_START(), CS_PCT(NEG(RET(5, close))))` |
| `TDAY_OF_MONTH()` | `SCANSUM(MDAY() < LAG(1, MDAY()), 1:COUNT)` | COUNT | the trading day of the month (the first trading day = 1); the panel's first, partial month is invalid | `TDAY_OF_MONTH() <= 5 ? CS_PCT(NEG(RET(5, close))) : CS_PCT(RET(20, close))` |
| `AT_MONTH_START(x)` | `LATCH(MDAY() < LAG(1, MDAY()), x)` | unit of x | the value of x on the month's first trading day, held all month (invalid that day = invalid all month) | `RATIO(AT_MONTH_START(LAG(1, close)), AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, close))))) - 1` |
| `LISTED_DAYS()` | `SCANSUM(DAY_INDEX() < 1, MUL(VALID(close), 1:COUNT))` | COUNT | trading days with a close: counted from the panel's first day, today included; a halted day adds nothing and resuming does not reset | `SCANMAX(LISTED_DAYS() == 1 && VALID(close), FILL(close, 0:PRICE))` |
| `MIN_PRICE(p)` | `raw.close >= p` | BOOL | the day's unadjusted close (exchange price, local currency) >= p, p may have decimals; an invalid close that day = false. A US pool without it gets `MIN_PRICE(5)` by default (8.2); to include penny stocks write `MIN_PRICE(0)`. Needs the panel column `raw.close` | `MIN_PRICE(10) && ROLL_MEAN(20, volume) > 200000` |
| `LISTED_TDAYS()` | `SUB(cal.tday_idx, list.tday_idx)` | COUNT | trading days since listing: counted from the actual listing date, halted days included; the listing day = 0; stocks listed before 2000-01-04 count from 2000-01-04; invalid before listing. Needs the panel columns `cal.tday_idx` and `list.tday_idx` | `MASK(LISTED_TDAYS() >= 500, RET(60, close))` |
| `MKT_MEAN(x)` | `RATIO(GSUM_ALL(0, x), GCOUNT_ALL(0, x))` | unit of x | the equal-weighted whole-market mean of x (floored), the same value for every stock. Members = every stock with x valid that day; the stock pool is not applied and there is no common-stock filter; no stock valid -> invalid | `RET(20, close) - MKT_MEAN(RET(20, close))` |
| `EW_INDEX(p)` | `EXP(SCANSUM(DAY_INDEX() < 1, FILL(LOG(1 + MKT_MEAN(MASK(common == 1, RATIO(p, LAG(1, p)) - 1))), 0)))` | RATIO | the equal-weighted market: the market's default pool (8.2: TW common stocks; US common stocks that closed at USD 5 or more the day before, i.e. `common == 1` in the definition becomes `common == 1 && LAG(1, (apx.MIN_PRICE(5)))`) rebalanced to equal weight daily, 1 on the panel's first day, the same value for every stock; a stock with p invalid today or yesterday is left out that day; write `close` for p. A panel that cannot tell common stocks -> error | `EW_INDEX(close) > ROLL_MEAN(250, EW_INDEX(close)) ? CS_PCT(RET(60, close)) : CS_PCT(NEG(RET(60, close)))` |
| `STREAK(b)` | `SCANSUM(NOT(b), MUL(b, 1:COUNT))` | COUNT | number of consecutive days b is true (false today = 0); invalid on a day b is invalid and until b is next false; invalid before the first false | `STREAK(foreign_net > 0) >= 3` |
| `SINCE(b)` | `SCANSUM(b, 1:DAYS) - 1:DAYS` | DAYS | trading days since b was last true (0 on a true day); invalid on a day b is invalid and until b is next true; invalid before the first true | `SINCE(EV_CHANGE(revenue_m) && revenue_m >= ROLL_MAX(250, revenue_m)) <= 20` |
| `SCANMEAN(r, x)` | `RATIO(SCANSUM(r, x), SCANSUM(r, 1:COUNT))`; COUNT x: x becomes `RATIO(x, 1:COUNT)`; BOOL x: x becomes `MUL(x, 1:RATIO)` | unit of x (COUNT, BOOL -> RATIO) | mean of x from the day r was last true (included) to today (sum / days, floored once: the same mean as `ROLL_MEAN`, `GMEAN`); the mean of a BOOL is the share of days true. Invalid before the first true r; an invalid r or x breaks it until r is next true; a sum outside i64 breaks it until r is next true (`ROLL_MEAN` sums in 128 bits and never breaks on that). Mean cost since entry, mean volume since a new high | `close > SCANMEAN(close > ROLL_MAX(60, LAG(1, close)), close)` |
| `STATE(on, off)` | `LATCH(on \|\| off, MUL(on, 1:COUNT)) > 0:COUNT` | BOOL | true from a day on is true, false from a day off is true, held otherwise; both true the same day = true; a day with one of them known true and on invalid makes the state invalid until the next known signal; a day with one invalid and the other not true keeps the previous state; invalid before the first on or off | `STATE(close > ROLL_MEAN(250, close), close < ROLL_MEAN(60, close))` |
| `GMEAN(first, x)` | `GMEAN_BY(first, x)`; COUNT x: `GMEAN_BY(first, RATIO(x, 1:COUNT))`; BOOL x: `GMEAN_BY(first, MUL(x, 1:RATIO))` (share true) | unit of x | mean of x over the group's members (floored; valid even when the sum leaves i64), written to every stock of the group in the pool; members = in the stock pool that day with x valid (and the key valid for a group key); first as in GSUM: a unitless integer = level (0 whole market), anything else = group key; COUNT or BOOL x gives RATIO; non-members invalid. `GMEAN(L, x)` and `XL_PARENT(L, x)` give the same tree; for a COUNT x they are different identities (the units differ: `GMEAN` gives RATIO, `XL_PARENT` gives COUNT) | `GMEAN(CS_QCUT(10, fund.mktcap), RET(20, close))` |
| `GDEMEAN(first, x)` | `MASK(GCOUNT(first, x) > 1:COUNT, x - GMEAN(first, x))` | unit of x | x minus its group mean (GMEAN); a group of one -> invalid; non-members invalid. x needs a 1e6 fixed-point unit (PRICE, RATIO, PCT, INDEX) | `GDEMEAN(CORR_GROUP(RATIO(close, LAG(1, close))), RET(20, close))` |
| `GTOP(N, first, x)` | `ORD(first, x) < N` | BOOL | x is among the N largest in its group (exact ranks: exactly N per group, all of a smaller group); first = a level number (0 whole market, 1 industry, ...) or a group key; equal values broken by a fixed shuffle, the same rule as the stock picking ranks; non-members invalid | `MASK(GTOP(3, 1, fund.mktcap), RET(20, close))` (the 3 largest in each industry) |
| `GZ(first, x)` | `RATIO(SUB(x, GMEAN_BY(first, x)), GSTD(first, x))` | RATIO | z-score of x within its group = (x - group mean) / group population std; first as in `GSTD`: a unitless integer = level (`GZ(0, x)` = `CS_Z(x)`, `GZ(L, x)` = `CS_PEER_Z(L, x)`, bit for bit), anything else = group key; fewer than 2 members, std 0, or x - mean out of range -> invalid; non-members invalid | `GZ(CS_QCUT(10, fund.mktcap), RET(20, close))` |
| `CS_TOPK(k, x)` | `ORD(0, x) < k` | BOOL | x is among the k largest in the stock pool that day (exactly k stocks); equal values broken by a fixed shuffle; x invalid or outside the pool -> invalid | `MASK(CS_TOPK(50, fund.mktcap), RET(20, close))` |
| `MKT_SHARE(b)` | `MKT_MEAN(MASK(common == 1, MUL(b, 1:RATIO)))` | RATIO | share of common stocks with b true, 0..1 (floored to 1e-6), one value for every stock; denominator = common stocks with b valid that day, stock pool not applied; none valid -> invalid; no common-stock flag -> error | `MKT_SHARE(close > ROLL_MEAN(60, close)) < 0.3 ? 0.5 : 1` |
| `MKT_COUNT(b)` | `GSUM_ALL(0, MASK(common == 1, b))` | COUNT | number of common stocks with b true, one value for every stock; counts common stocks with b valid, stock pool not applied; none valid -> invalid; no common-stock flag -> error | `MKT_COUNT(raw.close >= raw.limit_up)` |
| `BASKET_MEAN(sel, x)` | `MKT_MEAN(MASK(sel, x))` | unit of x | equal-weighted mean of x over the stocks with sel true (floored), written to every stock; stock pool not applied; none of them with x valid that day -> invalid | `CORR(250, RET(1, close), BASKET_MEAN(symbol == "2330", RET(1, close)))` |
| `CLIP(x, lo, hi)` | `MAX2(MIN2(x, hi), lo)` | unit of x | x limited to lo..hi (lo when lo > hi); any invalid -> invalid; all three in the same unit | `CLIP(RET(1, close), -0.1, 0.1)` |
| `BETWEEN(x, lo, hi)` | `x >= lo && x <= hi` | BOOL | lo <= x <= hi (inclusive); one side known not met -> 0, else any invalid -> invalid; all three in the same unit | `BETWEEN(CS_PCT(fund.mktcap), 0.3, 0.7)` |
| `SIGN(x)` | `MUL(x > 0, 1:COUNT) - MUL(x < 0, 1:COUNT)` | COUNT | sign of x: -1 / 0 / 1; x invalid -> invalid | `GSUM(0, SIGN(RET(1, close)))` |
| `CROSS_UP(a, b)` | `a > b && !(LAG(1, a) > LAG(1, b))` | BOOL | the day a moves above b (a > b today, a <= b yesterday); known not above today, or known above already yesterday -> 0, else any of today's or yesterday's values invalid -> invalid; same unit | `CROSS_UP(close, ROLL_MEAN(20, close))` |
| `CROSS_DOWN(a, b)` | `a < b && !(LAG(1, a) < LAG(1, b))` | BOOL | the day a moves below b (a < b today, a >= b yesterday); known not below today, or known below already yesterday -> 0, else any of today's or yesterday's values invalid -> invalid; same unit | `CROSS_DOWN(foreign_net, 0)` |
| `MONTH_START()` | `MDAY() < LAG(1, MDAY())` | BOOL | 1 on the first trading day of a calendar month, else 0; invalid on the panel's first day | `MONTH_START()` |
| `QUARTER_START()` | `MONTH_START() && (MONTH() - 1) % 3 == 0` | BOOL | 1 on the first trading day of January, April, July and October, else 0; on the panel's first day: 0 outside those four months, invalid in them | `QUARTER_START()` |
| `EVERY(n, k)` | `MOD(DAY_INDEX(), n) == k` | BOOL | once every n trading days, offset by k: day d counted from the panel's first day (0-based) is 1 when d mod n = k, else 0; same for every stock; n 1..1,260, k 0..n-1 | `EVERY(21, 5)` |
| `YEAR_START()` | `MONTH() < LAG(1, MONTH())` | BOOL | 1 on the first trading day of a year, else 0; invalid on the panel's first day | `YEAR_START()` |
| `PREV_MONTH_RET(p)` | `RATIO(AT_MONTH_START(LAG(1, p)), AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, p))))) - 1` | RATIO | last calendar month's return = p on last month's final trading day / p on the final trading day of the month before - 1, same value all month; either p invalid (or a zero denominator) -> invalid all month | `CS_PCT(PREV_MONTH_RET(close))` |
| `ROLL_MEAN_AVAIL(N, x)` | `RATIO(WSUM($N, FILL(x, 0)), WSUM($N, VALID(x)))` | unit of x | mean of x over the days it is valid within the last N days (floored; invalid days are left out of the denominator); invalid for the panel's first N - 1 days; all invalid in the window -> invalid; N 1..1,260 | `ROLL_MEAN_AVAIL(20, foreign_net)` |
| `VWAP(N)` | `RATIO(WSUM($N, px.turnover), WSUM($N, volume))` | PRICE | N-day volume-weighted average price = total turnover / total shares traded (TWD per share, unadjusted: compare with raw.close); any invalid cell in the window, zero shares, or a turnover total outside fixed point -> invalid; N 1..1,260. Needs the panel columns px.turnover and volume | `raw.close > VWAP(20)` |
| `REG_SLOPE(N, y, x)` | `WREG(N, 0, y, x)` | RATIO | slope of the regression of y on x over the last N days (how much y moves per 1 of x); N 3..1,260, y and x share one unit; any invalid cell in the window or x constant -> invalid | `REG_SLOPE(60, RET(1, close), MKT_MEAN(RET(1, close))) < 0.8` |
| `BETA(N, y, x)` | `WREG(N, 0, y, x)` | RATIO | beta = cov(y, x) / var(x) = `REG_SLOPE(N, y, x)` (the same tree); for beta to the market pass both returns; N 3..1,260, y and x share one unit. For two prices (the `BETA` atom up to 1.1.0, retired) write `BETA(N, LOGR(a, LAG(1, a)), LOGR(b, LAG(1, b)))` | `BETA(120, RET(1, close), MKT_MEAN(RET(1, close))) < 0.8` |
| `REG_INTERCEPT(N, y, x)` | `WREG(N, 1, y, x)` | unit of y | intercept (the mean of y when x is 0); otherwise as `REG_SLOPE` | `CS_PCT(REG_INTERCEPT(60, RET(1, close), MKT_MEAN(RET(1, close))))` |
| `REG_RESID(N, y, x)` | `WREG(N, 2, y, x)` | unit of y | today's residual (today's y minus the line at today's x); otherwise as `REG_SLOPE` | `CS_PCT(REG_RESID(60, RET(1, close), MKT_MEAN(RET(1, close))))` |
| `REG_R2(N, y, x)` | `WREG(N, 3, y, x)` | RATIO 0..1 | R squared (how much of y's variation the line explains); otherwise as `REG_SLOPE` | `REG_R2(120, RET(1, close), MKT_MEAN(RET(1, close))) < 0.2` |
| `REG_T(N, y, x)` | `WREG(N, 4, y, x)` | RATIO | t-stat of the slope (slope / standard error); otherwise as `REG_SLOPE` | `REG_T(60, RET(1, close), MKT_MEAN(RET(1, close))) > 2` |
| `RWIN_QUANT(k, q, x)` | `RWIN(k, 8, q, x)` | unit of x | the floor(q*(k-1)/100)-th smallest of the last k published values (q 50 = median, 0 = min, 100 = max); k 1..12, q 0..100; a period = a day x changes; fewer than k periods or x invalid -> invalid | `eps > RWIN_QUANT(8, 50, eps)` |
| `RWIN_RANK(k, x)` | `RWIN(k, 9, x)` | COUNT | rank of the current period among the last k = 2 x (periods below it) + (periods equal to it, itself included) - 1 (the same count as `WRANK`); k 2..12; otherwise as `RWIN_QUANT` | `RWIN_RANK(12, revenue) >= 22` |
| `EWIN_QUANT(k, q, b, x)` | `EWIN(k, 8, q, b, x)` | unit of x | the floor(q*(k-1)/100)-th smallest x over the last k event days (days with b true and x valid); k 1..20, q 0..100; fewer than k events, or b or x invalid that day -> invalid | `EWIN_QUANT(8, 50, EV_CHANGE(revenue_m), RET(1, close))` |
| `EWIN_RANK(k, b, x)` | `EWIN(k, 9, b, x)` | COUNT | rank of x on the latest event day among the last k event days (the same count as `RWIN_RANK`); k 2..20; otherwise as `EWIN_QUANT` | `EWIN_RANK(8, EV_CHANGE(revenue_m), RET(1, close)) >= 14` |

- "Start of month" is `MDAY() < LAG(1, MDAY())` (today's day number is smaller than the previous trading day's). It assumes adjacent trading days are less than 28 calendar days apart.
- Return of the previous calendar month = `PREV_MONTH_RET(close)`; listed for 2 years = `LISTED_TDAYS() >= 500`; the highest price since listing = the `LISTED_DAYS` example.
- `MKT_MEAN`, `EW_INDEX`, `MKT_SHARE`, `MKT_COUNT`, `BASKET_MEAN`, `MONTH_START`, `QUARTER_START`, `YEAR_START` and `EVERY` give the same value for every stock, so they can go straight into `exposure` and `rebalance_when` (chapter 9). `EVERY(n, k)` counts from the panel's first day and ignores warm-up; `apx.EVERY(n)` (one argument) = every n days after warm-up = `rebalance_days=n` (the same in a backtest's `rebalance_when` and a check's `REBALANCE_WHEN`).
- Inputs marked X in one function are the same kind of quantity: a constant without a unit takes the unit of the one that has it (in `CLIP(close, 10, 100)`, 10 and 100 are PRICE; the same rule as for atoms, section 2.3). The first argument of `GMEAN` and `GDEMEAN` works as in `GSUM`: an integer without a unit = level (0 whole market, 1..4), anything else = group key. An input can be declared "X, COUNT turned into RATIO" (the x of `GMEAN` today): a COUNT input is replaced by `RATIO(x, 1:COUNT)` before expansion, and that expansion is printed as written.
- `EW_INDEX` measured on the Taiwan panel (4,127 days): maximum relative error against a double-precision reference 6.07e-3, and 3.86e-4 inside any 250 days.
- `SQRT` on integer units is also a text-level expansion: for QTY / COUNT / DAYS, `SQRT(n)` = `SQRT(RATIO(n, 1:same unit))`; for AMOUNT, `SQRT(a)` = `MUL(1000, SQRT(RATIO(a, 1000000:AMOUNT)))` (three decimals).

---

## 6. Similarity

Sectors and themes are not hand-made lists: **baskets are defined by correlation.** Which series "alike" refers to is the author's choice: returns, volume changes, heat, monthly revenue growth, institutional flows, any numeric formula.

Both functions are **derived columns**: computed by the host at the start of a round and appended to the panel. Both use only data from before each period.

### 6.1 `CORR_GROUP`: groups of stocks that behave alike

```
CORR_GROUP([lookback[, every[, step[, corr_bp[, quant]]]]], <formula>)
```

| Parameter | Default | Meaning |
|---|---|---|
| lookback | 252 | trading days before the period's first day used for each period |
| every | 126 | regroup every this many trading days |
| step | 1 | one point every this many days (21 for monthly data) |
| corr_bp | 3000 | correlation threshold in ten-thousandths (3000 = 0.3) |
| quant | 1 | 1 = turn values into ranks inside the window before correlating (any series); 0 = return style (the input must be a daily return) |
| `<formula>` | — | any numeric formula over panel columns; `CORR_GROUP` / `CORR_SEED` cannot be nested |

- Parameters come before the formula and can be dropped from the end: `CORR_GROUP(RATIO(close, LAG(1, close)))` uses every default.
- The output unit is CODE: **a group key only** (the key-form operators of section 4.8).
- A group: pairwise correlations; stocks whose every pair is at or above the threshold form a group.
- Points = lookback / step, in 20..1,024 (20..500 with quant 0).
- A stock alone in its group: `GRANK` invalid, `GCOUNT` = 1.
- The `CORR_GROUP` list of a result library is fixed and is part of its identity. Without a library name the library follows the list; a named library with a different list answers with a suggested name.

Choosing `<formula>`:

| What the user means by "alike" | Formula |
|---|---|
| prices move together | `RATIO(close, LAG(1, close))` or `LOGR(close, LAG(1, close))` |
| volume swells and shrinks together | `DIFF(1, LOG(volume))` |
| heating up together | `RATIO(volume, ROLL_MEAN(20, volume))`, with `CORR_GROUP(60, 20, 1, 3000, 1, …)` |
| monthly revenue growth in step | `RWIN_PCT(12, revenue_m)`, with `CORR_GROUP(1260, 126, 21, 3000, 1, …)` |
| foreign investors buy and sell together | `foreign_net` or `RATIO(foreign_net, volume)` |

For a ranking score use "how far above the group mean" (corpus R02):

```
MASK(VALID(GRANK(CORR_GROUP(LOGR(close, LAG(1, close))), RET(60, close))),
     SUB(RET(60, close),
         RATIO(GSUM(CORR_GROUP(LOGR(close, LAG(1, close))), RET(60, close)),
               GSUM(CORR_GROUP(LOGR(close, LAG(1, close))), MASK(VALID(RET(60, close)), 1:RATIO)))))
```

The outer `MASK(VALID(GRANK(...)), ...)` drops stocks without a group mate.

### 6.2 Seed baskets: stocks in the same group as a few names

In a condition (stock pool, `portfolio.hold`, `portfolio.weight`) write:

```
GSUM(CORR_GROUP(<formula>), symbol in {"2330", "2382", "3231"}) > 0
```

Meaning: the stocks that share a group with any seed (the seeds included). `GSUM(k, BOOL)` is the number of group members for which the condition holds.

| Wanted | Written |
|---|---|
| exclude this basket (stocks without a group stay) | `!FILL(GSUM(CORR_GROUP(RATIO(close, LAG(1, close))), symbol in {"2881", "2882", "2891"}) > 0, 0:BOOL)` |
| the union of two baskets | put all seeds in one `in {...}`; for two different series write `FILL(A, 0:BOOL) \|\| FILL(B, 0:BOOL)` |
| without seeds: every group that holds a qualifying stock | `GSUM(k, <BOOL per day and stock>) > 0` |
| the recently hot stocks and the group heating up with them | `GSUM(CORR_GROUP(60, 20, 1, 3000, 1, RATIO(volume, ROLL_MEAN(20, volume))), CS_PCT(ROLL_MEAN(20, RATIO(volume, ROLL_MEAN(120, volume)))) >= 0.98) > 0` |

- `GMAX` / `GMIN` do not take BOOL, hence `GSUM(...) > 0`.
- `!(...)` also drops stocks that have no group; to keep them write `!FILL(..., 0:BOOL)`.
- Groups are rebuilt every `every` trading days, so the basket's members change between periods. The plan returns the number of matching stocks per period (mean, minimum, maximum), the change rate between adjacent periods, and the latest period's members.
- A `CORR_GROUP` inside a condition is not part of the library's list and does not split libraries.
- To reuse it, store it with `my_stock_strategies_change action=install` as a user function (`out_unit` BOOL, no `inputs`, body = that condition) and write `user.<you>.NAME()` in conditions. Such a function works only in conditions.

### 6.3 `CORR_SEED`: each stock's correlation with the seeds

```
CORR_SEED({seeds...}[, lookback[, every[, step[, quant[, how]]]]], <formula>)
```

| Parameter | Default | Meaning |
|---|---|---|
| seeds | — | 1..64 stocks, codes in double quotes: `{"2330", "2317"}`; order and duplicates do not matter |
| lookback | 252 | trading days before the period's first day used for each period |
| every | 21 | recompute every this many trading days (>= 1) |
| step | 1 | one point every this many days (>= 1) |
| quant | 1 | 1 = ranks inside the window; 0 = return style |
| how | 0 | with several seeds: 0 = the seeds as one combined series, 1 = maximum, 2 = mean, 3 = minimum |
| `<formula>` | — | any numeric formula |

- The output unit is RATIO in -1..1: a number, usable as a whole formula, as a factor in another score, or against a threshold.
- A seed's own value is 1. Stocks with too little data, and periods without a full window or a usable seed, have no value.
- Points = lookback / step, in 20..1,024 (20..500 with quant 0).
- Two unrelated series give a value within about +-2 / sqrt(points): 20 points +-0.45, 40 points +-0.32, 60 points +-0.26, 252 points +-0.13.
- A `CORR_SEED` in a formula is part of the library identity, like `CORR_GROUP`; in a condition or a weight it is not, and does not split libraries.

| Plain words | Written |
|---|---|
| rank by correlation with 2330 | `CORR_SEED({"2330"}, RATIO(close, LAG(1, close)))` |
| recently heating up with these three | `CORR_SEED({"2330", "2382", "3231"}, 40, 5, DIFF(1, LOG(volume)))` |
| institutional flows in step | `CORR_SEED({"2330"}, 60, 5, ADD(ADD(foreign_net, trust_net), dealer_net))` |
| correlation times momentum | `MUL(CORR_SEED({"2330"}, RATIO(close, LAG(1, close))), RET(20, close))` |
| only stocks correlated enough (stock pool or `hold`) | `CORR_SEED({"2330"}, RATIO(close, LAG(1, close))) >= 0.5` |
| the same, without the seed itself | `CORR_SEED({"2330"}, RATIO(close, LAG(1, close))) >= 0.5 && symbol != "2330"` |
| the higher the correlation, the larger the position (`weight`; values <= 0 are not bought) | `CORR_SEED({"2330"}, RATIO(close, LAG(1, close)))` |

### 6.4 Peers used by checks

The peer-swap check (`apx.PEER_SWAP`, chapter 11) has its own pairing: every selected stock is replaced by the one closest to it and the strategy is recomputed. The input of that yardstick can likewise be any series.

---

## 7. Data columns

### 7.1 How column keys are named

A column key is a leaf of a formula. The common ones have short names without a prefix; the rest are prefixed by source.

| Prefix | What | Examples |
|---|---|---|
| (none) | price and volume, common statement and flow columns | `open high low close volume`, `eps revenue net_income equity assets shares`, `revenue_m`, `dps`, `foreign_net trust_net dealer_net`, `margin_bal short_bal` |
| `raw.` | exchange prices, volume and price limits (unadjusted) | `raw.close`, `raw.limit_up`, `raw.limit_down`, `raw.ref_price` |
| `px.` | columns computed from price and volume | `px.turnover` (traded value), `px.turnover_rate` (turnover rate) |
| `fund.` | statement items and market cap | `fund.mktcap`, `fund.gross`, `fund.cfo`, `fund.debt` |
| `mkt.` | market level, generic names: this market's main index, the same value for every stock | `mkt.idx` (price index), `mkt.idx_tr` (total return, dividends reinvested); Taiwan = `idx.taiex`, `idx.taiex_tr`, US = `idx.spx`, `idx.spx_tr` (S&P 500) |
| `idx.` | market level: indices, the same value for every stock | Taiwan `idx.taiex`, `idx.taiex_tr`, `idx.tpex`, `idx.tpex_tr`; US `idx.spx`, `idx.spx_tr` |
| `etf.` | market level: ETF closes, the same value for every stock | `etf.0050`, `etf.0050_adj` |
| `list.` | listing information | `list.date`, `list.tday_idx` |
| `cal.` | calendar, the same value for every stock | `cal.tday_idx` |
| `sym.` | the numeric value of the stock code | `sym.code_num` (2330 -> 2330) |
| `news.` | material announcements | `news.major_cnt` |
| `tdcc_` | shareholding distribution (weekly) | `tdcc_people`, `tdcc_big1000_pct` |
| `dt_`, `sbl_` | day trading, securities lending | `dt_vol`, `sbl_vol` |

Code columns that are not panel columns (one column = one classification): `industry`, `market`, `common`, `symbol` (section 2.4).

**Taiwan-only columns** (data only the Taiwan exchanges publish): institutional flows (`foreign_net`, `trust_net`, `dealer_net`, `govbank_net`), margin, short and securities lending (`margin_*`, `short_bal`, `sbl_*`, `foreign_shares`, `foreign_ratio`, `unres_loan_bal`), broker branches (`brk.*`, `top15_*`), shareholding distribution (`tdcc_*`), day trading (`dt_*`), monthly revenue `revenue_m`, material announcements `news.*`, limit and reference prices (`raw.limit_up`, `raw.limit_down`, `raw.ref_price`), exchange daily values (`per`, `pbr`, `div_yield`, `mkt_value`), full-delivery, disposition and attention flags (`full_delivery`, `disposition`, `attention`), the OTC index `idx.tpex*`. Which columns a market actually has: `stock_formula_help topic=column market=<TW|US>`.

Data that can be computed (recipes without inputs, defined once): `fund.mktcap` = `MUL(raw.close, shares)`, `px.turnover` = `MUL(raw.close, volume)`, `px.turnover_rate` = `RATIO(volume, shares)`. Write the name as usual: a panel with that column reads the column; a panel without it but with every column the recipe uses computes it from the recipe (the same tree and the same values as writing the recipe yourself); only when those columns are missing too do you get "the panel has no such column" with the missing ones listed. A column lookup (`stock_formula_help topic=column name=...`) adds one line on how it is derived.

### 7.2 How to find the columns

The column table changes with the panel, so it is not copied here.

| Looking for | How |
|---|---|
| the columns and units of a panel | `stock_formula_help topic=column` (optionally `market=`, `panel=`) |
| a column by plain words | `stock_formula_help topic=column query=<words>` (for instance `query=融資`) |
| the definition, method and publication rule of a column | `stock_formula_help topic=column name=<key>` |
| the panels that exist | `stock_formula_help topic=column` |
| the history of one column for one stock | `stock_formula_help topic=column` (`changes_only=true` lists only the days the value changes) |
| industry codes, exchange names, class nouns | `stock_formula_help topic=universe` |

When a formula uses a column the given panel lacks, the plan switches to the panel that has them all.

### 7.3 Publication timing: no look-ahead

When a panel is cut, every cell is aligned to "the value visible after that day's close". Formulas look only at cells and never handle publication dates themselves.

| Kind of column | Visible from | Days without a new value |
|---|---|---|
| price and volume (`close`, `volume`...) | that day's close | days without trading (before listing, holidays, halts) are invalid |
| quarterly statements (`revenue`, `eps`, `fund.*`...) | with a known filing date: the next trading day after it; without one: by that market's deadline: Taiwan, the next trading day after the statutory deadline (annual report 3/31, Q1 5/15, Q2 8/14, Q3 11/14); US, 60 days after the quarter, 90 days after the fiscal year | carried from the previous quarter for at most 155 days from publication, then invalid |
| monthly revenue `revenue_m` | switches on each row's publication date (published by the 10th of each month) | carried until the next row |
| institutional net buying (`foreign_net`...) | published after the close: a day's value is in the next trading day's row | not carried; before 2020 a day without trades is missing, later it is 0 |
| margin columns (`margin_bal`...) | published after the close: a day's value is in the next trading day's row | carried; stocks without margin eligibility have no value |
| shareholding distribution (`tdcc_*`) | the next trading day after the reference date | carried until the next period |
| cash dividend per share `dps` | the next trading day after the ex-dividend date | an event column: a value on that day only, invalid on every other day |
| market level (`idx.*`, `etf.*`) | that day's close, the same value for every stock | holidays are invalid |
| exchange prices (`raw.*`) | that day's close (same row as `close`); limit prices and the reference price are in the row of the trading day they apply to | days without trades are invalid |

Key points:

- `close`, `open`, `high`, `low` are adjusted prices (dividends and splits adjusted to the latest). Thresholds compare against the adjusted price, not the price quoted that day. For the actual quoted level use `raw.*`.
- The publication day of a carried column (statements, monthly revenue) = the day its value changes, `EV_CHANGE(column)`. For windows counted in periods use `RWIN` (section 4.6) rather than guessing a number of days.
- Sum event columns (`dps`) with `FN_DY` or `WSUM`; summing a carried column over N days counts the same figure many times.
- Market-level columns have the same value for every stock; random search does not pick them as leaves by itself, they are used only where a formula names them.
- `eps` is the average single-quarter EPS of the last four quarters, so `FN_EP(eps, close)` = the inverse of the trailing P/E.

### 7.4 Derived data: financial ratios from data columns

These 5 are data: financial ratios derived by dividing data columns, so they are data like the columns. They are still written as calls (`FN_EP(earnings per share, price)`); names, parameters and results are the same as before (the units follow the quantities of chapter 3, see the table); the noun list marks them `data`.

| Signature | Params | Units | Invalid and warm-up | In plain words | Example |
|---|---|---|---|---|---|
| `FN_EP(a, b)` | — | (PRICE, PRICE) -> RATIO | b <= 0 -> invalid; a must be a per-share value | earnings yield, the inverse of P/E: 4 x earnings per share / price (quarter to year) | `FN_EP(eps, close)` |
| `FN_BP(a, b)` | — | (PRICE, PRICE) -> RATIO | a <= 0 or b <= 0 -> invalid; a must be a per-share value | the inverse of P/B: book value per share / price | `FN_BP(FN_PS(equity, shares), close)` |
| `FN_PS(a, b)` | — | (AMOUNT, QTY) -> PRICE | b <= 0 -> invalid | per-share value: total / shares | `FN_BP(FN_PS(equity, shares), close)` |
| `FN_ROE(a, b)` | — | (X, X) same unit -> RATIO (net income and equity are AMOUNT) | b <= 0 -> invalid | return on equity: 4 x net income / equity (quarter to year) | `FN_ROE(net_income, equity)` |
| `FN_ACCRUAL(N, a, b)` | N 1..252 | (X, X) same unit -> RATIO | b <= 0 -> invalid; warm-up N | accrual ratio: a = accruals, b = assets, 4 x a / the mean of today's assets and the assets N days ago | `NEG(FN_ACCRUAL(252, SUB(net_income, fund.cfo), assets))` |

- Per-share values and totals are different quantities: `FN_EP`, `FN_BP` must come out as a ratio, so the first input must be a per-share value (`eps`, `FN_PS(...)`); a total (a total / a price = shares) is rejected. To compare a total with market cap write `RATIO(total, fund.mktcap)`.
- The divisions follow the quantities of chapter 3: money / shares -> a per-share value (PRICE), a per-share value / a price -> ratio.

Bit-identical atom forms (value, valid bits, unit; checked one by one like those of chapter 5). The atom forms' `RATIO` and `MULDIV` follow chapter 3 and need not come out as a ratio. `FN_ACCRUAL` is a statistic fused into a single rounding and has no atom form.

| Name | Atom form |
|---|---|
| `FN_EP(a, b)` | `MASK(GT(b, 0), MULDIV(4, a, b))` |
| `FN_BP(a, b)` | `MASK(AND(GT(a, 0), GT(b, 0)), RATIO(a, b))` |
| `FN_PS(a, b)` | `MASK(GT(b, 0), RATIO(a, b))` |
| `FN_ROE(a, b)` | `MASK(GT(b, 0), MULDIV(4, a, b))` |

---

## 8. Stock pool

### 8.1 Writing the condition

The stock pool is `universe=...` of `stock_backtest`: one BOOL formula per day and stock; cells where it is true stay. It is written in the same language as chapters 2-4.

| Plain words | Written |
|---|---|
| a close on each of the last 250 trading days | `WSUM(250, VALID(close)) >= 250` |
| listed for two years | `cal.tday_idx - list.tday_idx >= 500` (or `LISTED_TDAYS() >= 500`) |
| OTC only, excluding stocks listed for fewer than 60 trading days (Taiwan) | `market == TWO && cal.tday_idx - list.tday_idx >= 60` |
| Nasdaq and NYSE only (US) | `market in {XNAS, XNYS}` |
| 20-day average volume above 500 lots, traded every day | `COUNT(20, volume > 0) >= 20 && ROLL_MEAN(20, volume) > 500000` |
| mid caps: market cap between the 30th and 70th percentile | `CS_PCT(fund.mktcap) >= 0.3 && CS_PCT(fund.mktcap) <= 0.7` |
| top 30% by market cap, without TSMC and Hon Hai | `CS_PCT(fund.mktcap) >= 0.7 && !(symbol in {"2330", "2317"})` |
| only among these names | `symbol in {"2330", "2454", "2317"}` |
| no financials, no construction | `!cls.FIN() && !cls.REALTY()` (in Taiwan also by code: `!(industry in {17, 14})`) |
| largest one-year drop below 40% | `WDROP(250, LOG(close)) < 0.5108` |
| just above the 250-day average (crossed within 20 days) | `close > ROLL_MEAN(250, close) && COUNT(20, close > ROLL_MEAN(250, close)) < 20` |
| margin usage below 30% and short / margin ratio above 10% (Taiwan) | `RATIO(margin_bal, margin_limit) < 0.3 && RATIO(short_bal, margin_bal) > 0.1` |
| no limit-up or limit-down close in the last 60 days (Taiwan) | `COUNT(60, raw.close >= raw.limit_up \|\| raw.close <= raw.limit_down) == 0` |
| no full-delivery, disposition or attention stocks (Taiwan) | `!full_delivery && !disposition && !attention` |
| behaving like these names | `GSUM(CORR_GROUP(RATIO(close, LAG(1, close))), symbol in {"2330", "2382", "3231"}) > 0` |
| the last digit of the code equals the last digit of today's date | `sym.code_num % 10 == MDAY() % 10` |

Rules:

- Cells where the formula is invalid (window not full and warm-up included) are outside the pool.
- Members of cross-section operators (`CS_PCT`, `GSUM`...) = the pool **before** the filter, computed once, not iterated. `CS_PCT(fund.mktcap) >= 0.7` is the top 30% of the whole market.
- One result library is bound to one stock pool. Another pool means another library.
- 0 stocks on every day -> error `E_EMPTY_UNIVERSE`. Fewer than 1 a day on average (for example a single stock or a strict condition) still runs; nothing is held on the 0-stock days and the money stays in cash, the same result as `MASK(condition, score)` in the score.
- At most 20 symbols may be listed one by one (counted over the whole condition). More than that is listing a whole kind of stock: use a class noun or a seed basket.
- The plan returns `effective.universe_stats`: mean stocks per day, stocks on the last day, sample codes. To preview only: `stock_formula_help topic=universe universe="..."`.

### 8.2 Default = all common stocks (US also drops stocks under USD 5 that day)

Without `universe` the pool is every common stock of the market; for US stocks `apx.MIN_PRICE(5)` is added (the day's unadjusted close at USD 5 or more).

- `common_only` (default true): stocks that are certainly not common stocks (preferred shares, ETFs, beneficiary certificates, warrants, depositary receipts) are removed first, then `universe` applies. To include ETFs and the like: `common_only=false`.
- The code column `common`: 1 = common stock, 0 = not, invalid = cannot tell.
- The US default adds `apx.MIN_PRICE(5)`: the system puts it after the common-stock frame and the plan prints it in `effective.universe` (`!FILL(common == 0, 0:BOOL) && apx.MIN_PRICE(5) && (the user's condition)`). If the user wrote `MIN_PRICE` themselves (`apx.MIN_PRICE(1)`, or `apx.MIN_PRICE(0)` for no floor) theirs is used and nothing is added; a pool that only names stocks (`symbol == ...`, `symbol in {...}` joined by `||`) gets nothing added either. The price is each rebalance day's close, never a later price; the data stay, the stocks are just not picked. Taiwan and other markets add nothing.
- Apart from that, conditions the user did not state are not added. A liquidity threshold or a listing-age rule is not a default.

### 8.3 Class nouns

Official nouns `cls.NAME()`: one kind of stock, one meaning in every market; the definition = a condition on that market's industry codes, so **they exist where the market has industry codes** (which markets and their definitions: `stock_formula_help topic=universe market=...`). Used only in conditions. With no matching class, or when the client names a theme or "stocks like these": a seed basket or a correlation group (chapter 6), usable in every market.

| Noun | Meaning | Taiwan definition | US definition |
|---|---|---|---|
| `cls.FIN()` | financials | `industry in {17}` | sub-industries starting `40` |
| `cls.SEMI()` | semiconductors | `industry in {24}` | sub-industries starting `453010` |
| `cls.TECH()` | technology, electronics (semiconductors included) | `industry in {24, 25, 26, 28, 29, 30, 31, 36}` | sub-industries starting `45` |
| `cls.COMM()` | communication | `industry in {27}` | sub-industries starting `50` |
| `cls.HEALTH()` | health care, biotech | `industry in {22}` | sub-industries starting `35` |
| `cls.STAPLES()` | consumer staples | `industry in {2, 18, 33}` | sub-industries starting `30` |
| `cls.DISCR()` | consumer discretionary | `industry in {4, 11, 12, 16, 32, 37, 38}` | sub-industries starting `25` |
| `cls.INDU()` | industrials | `industry in {5, 6, 15, 35}` | sub-industries starting `20` |
| `cls.MATER()` | materials | `industry in {1, 3, 8, 9, 10, 21}` | sub-industries starting `15` |
| `cls.ENERGY()`, `cls.UTIL()` | energy, utilities (one definition in Taiwan) | `industry in {23}` | ENERGY: starting `10`; UTIL: starting `55` (separate in the US) |
| `cls.REALTY()`, `cls.CONSTR()` | real estate, construction | `industry in {14}` | REALTY: sub-industries starting `60`; CONSTR Taiwan only |
| `cls.SHIPPING()` | shipping | `industry in {15}` | — (Taiwan only) |
| `cls.STEEL()` | steel | `industry in {10}` | — (Taiwan only) |
| `cls.PLASTIC()` | plastics | `industry in {3}` | — (Taiwan only) |
| `cls.ECOMP()` | electronic components | `industry in {28}` | — (Taiwan only) |
| `cls.PC()` | computers and peripherals | `industry in {25}` | — (Taiwan only) |
| `cls.OPTO()` | optoelectronics | `industry in {26}` | — (Taiwan only) |
| `cls.BIO()` | biotech | `industry in {22}` | — (Taiwan only) |
| `cls.GREEN()` | green energy | `industry in {35}` | — (Taiwan only) |

- Exclude with `!cls.FIN()`; keep only with `cls.SEMI()`. Classes may overlap (`TECH` contains `SEMI`).
- US industry codes are GICS 8-digit sub-industries (e.g. `45301020` semiconductors); the US has the first 12 classes shared across markets, and a Taiwan-only class (shipping, steel, ...) written on the US returns "a class only TW has". US stocks without a sub-industry (mostly delisted ones and ETFs) belong to no class; their industry is empty, so both `cls.X()` and `!cls.X()` are false for them (neither side keeps them).
- Stocks without an industry are kept by neither side of a class condition; to keep them write `FILL(!cls.FIN(), 1:BOOL)` (no financials, stocks without an industry kept).
- Class nouns carry a version and a stored strategy records the version it used; a changed definition takes effect on a re-run.
- When the user means a theme or "stocks like these" and no class fits: use a seed basket (section 6.2).

### 8.4 `hold` versus `universe`

The same condition in two places means two different things:

| Where | Where ranking happens | Stocks that fail |
|---|---|---|
| `universe="close > ROLL_MEAN(60, close)"` | only inside the pool | not ranked, never bought |
| `portfolio={"hold":"close > ROLL_MEAN(60, close)"}` | on the whole market | not bought; held ones are sold on the rebalance day |

To rank only among a group of stocks use `universe`; to rank on everything but only buy and hold names that pass use `hold`.

### 8.5 The old form (compatibility)

`universe_lang`: 0 = automatic (default), 1 = old form, 2 = VQL. Automatic: 1 when the old form compiles, otherwise read as VQL. `effective.universe_lang` in the plan reports the decision.

The old form exists only for older result libraries; write VQL in anything new. The differences:

| Item | Old form (1) | VQL (2) |
|---|---|---|
| Values | real numbers; a comparison with a missing value is false | fixed point; cells where the formula is invalid are outside the pool |
| Time series | none | every operator |
| Functions | element-wise `abs sqrt log exp has min max where`; cross-section of the day `sum mean count quantile min max` | chapter 4 |
| Example | `fund.mktcap >= quantile(fund.mktcap, 0.7)`, `industry == 24 && fund.mktcap > 10e9` | `fund.mktcap >= GQUANT(70, 0, fund.mktcap)`, `industry in {24} && fund.mktcap > 10e9` |

**How the old form is translated.** The old form has no evaluator of its own: it is translated to VQL at compile time and computed on the same path as VQL; `universe_lang`, the stored condition text and the panel identity stay as they are. The translation:

- Every comparison and logical operator is wrapped as "invalid counts as false" (`FILL(…, 0:BOOL)`), which reproduces the old rule that a comparison with a missing value is false.
- `mean(x)` = `RATIO(GSUM(0, x), GCOUNT(0, x))`; `quantile(x, q)` = `GQUANT(q × 100, 0, x)`, where q × 100 must be an integer from 0 to 100; `sum`, `count` and one-argument `min`, `max` = `GSUM`, `GCOUNT`, `GMIN`, `GMAX` at level 0.
- **Arithmetic across units is converted to ratios first** (for example `eps * 4 * shares / equity`, `close / volume`): the old form has no units, so when the direct translation does not pass the unit rules, every numeric column becomes `RATIO(column, 1:<its unit>)` before the arithmetic.
- **An outermost division on either side of a comparison is cross-multiplied** (`x / y > z`): it becomes a multiplication and is not rounded before the comparison — with a positive denominator `x` is compared with `z·y`, with a negative one the direction flips, and a zero denominator counts as false. With a division on both sides each is moved. Inner divisions floor to 1e-6 as in VQL.
- **Limit**: a ratio is fixed point at 1e-6. A value inside cross-unit arithmetic (including the products made by cross-multiplying) above about 9.2e12 makes that cell invalid and the condition false; a total-amount column (for example `fund.mktcap`) in such arithmetic is the usual way to reach it.
- What cannot be translated returns `E_OLD_SYNTAX` with the VQL spelling in the hint: `sd`, `var` (sample versions; VQL's `GSTD` is the population one), `pow`, a `quantile` whose percentile is not an integer, division by the constant 0.

---

## 9. Backtest (the portfolio function)

### 9.1 What the portfolio function is

A score formula only produces each day's ranking. **How the ranking becomes holdings** is decided by the portfolio function: `stock_backtest portfolio={...}`. All arguments are named and all are optional.

- The top-level `rebalance_days` and `max_positions` are shorthands for the arguments of the same name inside `portfolio`.
- The backtest period is not inside `portfolio`; it is a top-level argument of `stock_backtest` (9.10).
- One portfolio, one market.

### 9.2 All arguments

| Argument | Range | Default | Meaning | Example |
|---|---|---|---|---|
| `rebalance_days` | 1..252 | omitted: tries 5 and 21, keeps one | re-rank and rebalance every N trading days | `{"rebalance_days":5}` |
| `max_positions` | 1..500 | omitted: tries 5 and 10, keeps one | hold at most N stocks | `{"max_positions":10}` |
| `weighting` | `equal` / `score` / `inv_vol` | `equal` | equal weight / by rank (higher weighs more) / by inverse volatility (steadier weighs more) | `{"weighting":"inv_vol","max_weight":15}` |
| `max_weight` | 0..100 (%) | 0 (no cap) | cap per stock; the excess stays in cash | `{"max_positions":10,"max_weight":15}` |
| `hold_days` | 2..60 | not used | pick every day; each day's purchase is one slice held N trading days and then sold (N staggered slices) | `{"hold_days":20}` |
| `hold` | BOOL formula | not used | a condition per day and stock: ranks stay on the whole market, names that fail are not bought and are sold on the rebalance day | `{"hold":"close > ROLL_MEAN(60, close)"}` |
| `rebalance_when` | BOOL formula (one value per day for the market) | not used | rebalance on the days it holds | `{"rebalance_when":"MDAY() < LAG(1, MDAY())"}` |
| `exposure` | RATIO formula 0..1 (one value per day for the market) | 1 (fully invested) | total exposure; the rest stays in cash | `{"exposure":"mkt.idx < ROLL_MEAN(200, mkt.idx) ? 0.5:RATIO : 1:RATIO"}` |
| `weight` | numeric formula (per day and stock) | not used | on a rebalance day each stock's weight is in proportion to this value | `{"max_positions":20,"weight":"fund.mktcap"}` |
| `tie_by` | numeric formula (per day and stock) | not used (ties follow the fixed shuffle) | when picking the top N, stocks with the same score are ordered by this value, larger first; stocks without a value come last among the tied; equal values then `mix64((entity_id<<32)^day)`, then the symbol index. Only orders ties: the score and the weights are unchanged and the warm-up is not raised; for smaller first write `NEG(...)` | `{"tie_by":"fund.mktcap"}` |
| `group_by` | formula (per day and stock; a number or a condition, no `pos.*`) | not used | what a group is: stocks with the same value on a day are one group; given together with `group_cap` (9.5a) | `{"group_by":"CORR_GROUP(RET(1, close))","group_cap":3}` |
| `group_cap` | positive integer | not used | at most this many holdings per group: on a rebalance day names are taken in rank order, a name whose group is full is skipped (9.5a) | as above |
| `add_when` | BOOL formula, may use `pos.*` and `pos.size`, `pos.adds`, `pos.cuts`, `pos.avg_ret` | not used | add condition: at the close where it turns from false to true for a holding, add once at the next fill (9.5b) | `{"add_when":"pos.ret >= 0.1","size_init":0.5}` |
| `cut_when` | as `add_when` | not used | cut condition: when it triggers, sell `cut_size` of the current position (9.5b) | `{"cut_when":"pos.ret >= 0.2"}` |
| `add_size` | > 0 | 1 | units per add (multiples of the entry unit) | `{"add_when":"pos.ret >= 0.1","add_size":0.5}` |
| `cut_size` | 0 < x < 1 | 0.5 | share of the current position sold per cut; to sell all write `exit_when` | `{"cut_when":"pos.ret >= 0.2","cut_size":0.3}` |
| `size_init` | 0.01..1 | 1 | share of the target weight bought at entry (cash kept for adds) | `{"add_when":"pos.ret >= 0.1","size_init":0.5}` |
| `max_size` | >= 1 | no cap | cap per holding, in entry units | `{"add_when":"pos.ret >= 0.1 * (pos.adds + 1)","max_size":3}` |
| `gross_max` | 0.01..2 | 1 | gross cap: weights may sum to at most this share of the account; > 1 = margin (give `margin_rate`) (9.5b) | `{"gross_max":0.8}` |
| `cash_min` | 0..0.99 | 0 | minimum cash share; the stricter of it and `gross_max` applies | `{"cash_min":0.1}` |
| `add_mode` | `cash` / `scale` | `cash` | adds beyond the gross cap: only the room left / shrink the other holdings (as a formula, 9.12: wrap the add term in `MIN2(..., acct.room)` / multiply by `MIN2(1, g / acct.gross_req)`) | `{"add_mode":"scale"}` |
| `max_new` | >= 1 | no limit | at most this many new names per rebalance (the best-ranked; the other slots stay cash) | `{"max_new":3}` |
| `cash_rate` | annual rate, e.g. 0.015 | 0 | annual interest on cash (long-only) | `{"cash_rate":0.015}` |
| `margin_rate` | annual rate, e.g. 0.05 | not used | annual margin rate (required when `gross_max` > 1) | `{"gross_max":1.5,"margin_rate":0.05}` |
| `exit_when` | BOOL formula, may use `pos.*` | not used | exit condition: checked at every close, sold when true without waiting for the rebalance day | `{"exit_when":"pos.ret <= -0.08 \|\| pos.ret >= 0.2"}` |
| `keep_when` | BOOL formula, may use `pos.*` and `pos.rank` | not used | keep condition: on a rebalance day, holdings for which it is true stay | `{"max_positions":10,"keep_when":"pos.rank <= 30"}` |
| `reentry_days` | 0..10,000 | 0 | after an `exit_when` exit the same stock is not bought back at rebalances within N trading days | `{"exit_when":"pos.ret <= -0.08","reentry_days":20}` |
| `exit_below` | PRICE formula, may use `pos.entry`, `pos.peak` | not used | intraday stop level: sold once the day's low touches it, without waiting for the close (9.6a) | `{"exit_below":"pos.entry * 0.92"}` |
| `exit_above` | PRICE formula, may use `pos.entry`, `pos.peak` | not used | intraday take-profit level: sold once the day's high touches it (9.6a) | `{"exit_above":"pos.entry * 1.2"}` |
| `lock` | `limit` / `none` | omitted: simulated when the panel has the exchange limit prices (the US has no price limits; plan says `no_limits`) | rule exits on locked limit days: `limit` moves a blocked sale to the next trading day, `none` does not simulate locks (9.6a) | `{"exit_below":"pos.entry * 0.92","lock":"none"}` |
| `grid` | an object: name -> values to try | not used | numbers to try: written `$name` in the formulas, each name's values given here (9.11); also sets the search's base grid: `pl` (long threshold), `reb` (rebalance days), `n` (positions), `wk` (weight key) (9.11a) | `{"exit_when":"pos.drawdown >= $dd","grid":{"dd":[0.06,0.1,0.14]}}`, `{"grid":{"reb":[5,10,21]}}` |
| long threshold `apx.IN_TOP(p)` | in `hold`; p is an integer 0..200 or a `$name` | not written: tries 160 / 170 / 180 / 190; written: used as written, nothing added | buy only stocks whose score ranks at or above percentile p/200 (180 = top 10%, 190 = top 5%); the top-ranked stock in the pool always passes, so a one-stock pool holds that stock; `apx.IN_TOP(0)` = no threshold; = `GE(q, p / 200)`, q = `FILL(CS_PCT(apx.SCORE()), MASK(EQ(GCOUNT(0, apx.SCORE()), 1:COUNT), 1:RATIO))` | `{"hold":"apx.IN_TOP(190)"}`, `{"hold":"apx.IN_TOP($pl)","grid":{"pl":[150,170,190]}}` |
| `grid_by` | a screening expression (the ordering, 11.3) | with `grid`: `apx.SHARPE`; without: combined score | orders a formula's combinations (larger first); without `grid` it orders the portfolio settings | `{"exit_when":"pos.drawdown >= $dd","grid":{"dd":[0.06,0.1,0.14]},"grid_by":"apx.CALMAR"}` |
| `initial_capital` | 1..1e12 (local currency: TWD in Taiwan, USD in the US; same for `min_fee`, `contribution`) | 1,000,000 | starting capital; given = a share-level account is computed | `{"initial_capital":4000000}` |
| `lot` | `odd` / `board` | `odd` | odd lots (1 share) / board lots only (1,000 shares in Taiwan) | `{"lot":"board"}` |
| `min_fee` | 0..10,000 (local currency) | 20 | minimum commission per order (Taiwan brokers' 20; US zero-commission brokers: write 0) | `{"min_fee":20}` |
| `contribution` | 0..1e12 (per period) | 0 | regular contribution, invested at the next rebalance | `{"initial_capital":5000,"contribution":5000,"contribution_every":"month","lot":"odd"}` |
| `contribution_every` | `month` / `rebalance` | `month` | the first trading day of each month / every rebalance | as above |

Exclusive and combinable arguments:

| Rule | Note |
|---|---|
| one of `rebalance_days`, `hold_days`, `rebalance_when` | decides when to rebalance |
| one of `weight`, `weighting` | decides the weights |
| `tie_by` with any other argument | it only changes the order of tied scores when the top N are picked (the exact rank); positions, weights and exit rules are unchanged |
| `group_by` and `group_cap` go together | with only one, the answer names the missing one and gives an example to follow |
| `exit_when`, `keep_when` may go with `hold_days`  | each slice checks its own entry prices and exits on its own; a sold share stays in cash until that slice expires; `keep_when` is checked on each slice's expiry day and a holding for which it is true stays in that slice for another N days; `reentry_days` blocks only the same slice (to block every buy, use `rebalance_days` with `exit_when="pos.days >= N \|\| ..."`). A time stop ("held N days") is still `exit_when="pos.days >= N"` |
| `weight` may go with `hold_days` | each day's slice is split by that day's values and left as is while held |

Arguments that take a formula (`hold`, `rebalance_when`, `exposure`, `weight`, `tie_by`, `exit_when`, `keep_when`, `exit_below`, `exit_above`) accept either the formula string or `{"expr": "...", "label": "one plain sentence"}`; the `label` is the sentence written for the user and is the only thing the card shows.

### 9.3 When to rebalance (one of three)

| Argument | Behaviour |
|---|---|
| `rebalance_days=N` | every N trading days once warm-up is over |
| `rebalance_when=<BOOL>` | on the days the condition holds. The formula must have one value per day for the whole market (calendar, market-level columns, `MKT_MEAN`, `EW_INDEX` and the like); two or more values on the same day -> `E_NOT_MARKET_WIDE`; never true over the period -> `E_NO_REBALANCE` |
| `hold_days=N` | picks every day, buys one slice every day, each slice held N trading days and then sold; N staggered slices |

Common `rebalance_when` forms:

| Plain words | Written |
|---|---|
| the first trading day of each month | `MDAY() < LAG(1, MDAY())` or `MONTH() != LAG(1, MONTH())` |
| quarterly (first trading day of January, April, July, October) | `MDAY() < LAG(1, MDAY()) && MONTH() in {1, 4, 7, 10}` |
| only on the first trading day of earnings months | `MONTH() != LAG(1, MONTH()) && MONTH() in {3, 5, 8, 11}` |
| every Friday | `WEEKDAY() == 4` |
| the first trading day of each week | `WEEK_START()` |
| the first trading day of each year | `YEAR() > LAG(1, YEAR())` |
| the day the equal-weighted market crosses above its 60-day average | `EW_INDEX(close) > ROLL_MEAN(60, EW_INDEX(close)) && !LAG(1, EW_INDEX(close) > ROLL_MEAN(60, EW_INDEX(close)))` |

`hold_days` is not a time stop: `hold_days=20` is another way of rebalancing ("buy a slice every day, rotate over 20 slices"); "rebalance as usual and sell a holding once it has been held 20 days" is `exit_when="pos.days >= 20"`.

"At most N stocks at a time, each held M days" (never more than N names; each is sold M days after it was bought and its slot is refilled by rank) is not `hold_days` either: `hold_days=M` with `max_positions=N` buys an N-stock slice every day, so up to N x M names are held at once, each slice with 1/M of the money. Look every day, keep what is younger than M days, sell at M days:

| In plain words | Written as |
|---|---|
| at most 8 stocks, each one swapped out after 15 days | `{"rebalance_days":1,"max_positions":8,"keep_when":"pos.days < 15","exit_when":"pos.days >= 15"}` |

`keep_when` keeps a holding younger than 15 days at each daily rebalance wherever its rank falls; `exit_when` sells it at the next fill after its 15th close; the freed slot is refilled by rank the next day. A stock that leaves the stock pool, loses a valid score or fails `hold` is still sold early (9.5).

### 9.4 `hold`, `exposure`, `weight`

**`hold`** (BOOL per day and stock): ranks and factor metrics are still computed on the original stock pool; this strategy only buys and holds stocks for which the condition is true. An invalid cell counts as failing.

**`exposure`** (one RATIO per day for the whole market):

- The value used = floor(value x 100) clamped to 0..100: whole percentage points, and it may change every day.
- A change costs only the difference traded; the rest stays in cash. On a day when only the exposure changes there is no selection and no rebalance is counted.
- No stock valid that day -> the previous day's value. 100 throughout -> treated as not given.

| Plain words | Written |
|---|---|
| half invested while the market index (TAIEX in Taiwan, S&P 500 in the US) is below its 200-day average | `mkt.idx < ROLL_MEAN(200, mkt.idx) ? 0.5:RATIO : 1:RATIO` |
| half invested while the equal-weighted market is below its 250-day average | `EW_INDEX(close) < ROLL_MEAN(250, EW_INDEX(close)) ? 0.5 : 1` |
| 30% invested when the equal-weighted market is more than 15% off its one-year high | `RATIO(EW_INDEX(close), ROLL_MAX(250, EW_INDEX(close))) < 0.85 ? 0.3 : 1` |
| 30% invested on a day the whole market falls more than 2% on average | `MKT_MEAN(RATIO(close, LAG(1, close)) - 1) < -0.02 ? 0.3 : 1` |
| out of the market on a Friday the 13th | `MDAY() == 13 && WEEKDAY() == 4 ? 0:RATIO : 1:RATIO` |
| inversely to the market's 20-day volatility, at most fully invested | `MIN2(RATIO(0.01:RATIO, ROLL_STD(20, FILL(MKT_MEAN(LOGR(close, LAG(1, close))), 0:RATIO))), 1:RATIO)` |

**`weight`** (a number per day and stock):

- On a rebalance day each stock's weight = its value / the sum of the values of the selected stocks, in basis points.
- Stocks whose value is not positive or missing are not bought. Stocks whose share rounds below one basis point are not bought either, and that share stays in cash.
- `max_weight` still caps a stock; the clipped part stays in cash.
- Never positive over the whole period -> `E_NO_WEIGHT`.

| Plain words | Written |
|---|---|
| by market cap | `fund.mktcap` |
| by the square root of market cap | `SQRT(fund.mktcap)` |
| by 20-day traded value | `ROLL_MEAN(20, px.turnover)` |
| by average volume | `ROLL_MEAN(20, volume)` |
| by score | `CS_PCT(RET(60, close))` |
| the higher the correlation with 2330, the more | `CORR_SEED({"2330"}, RATIO(close, LAG(1, close)))` |

Price x volume (`close * volume`) is AMOUNT and works as a weight too; `px.turnover` is the day's actual traded value.

**`weighting`** (when `weight` is not given):

| Value | Weights |
|---|---|
| `equal` | 1 / number of stocks each; with `max_weight` and (number of stocks x cap) below 100%, each = the cap and the rest is cash |
| `score` | by rank: higher weighs more; weights are rounded to basis points and the remainder goes to the top-ranked stock |
| `inv_vol` | by inverse volatility: steadier weighs more. It is the weight key `INV_VOL_KEY(20, RET(1, close))` (20-day returns, 4.6) under the same rules as a `weight` formula: stocks without a value (fewer than half of the last 20 returns valid) are not bought; stocks whose share rounds below one basis point are not bought either, and that share stays in cash. For another window or series write the key `INV_VOL_KEY(N, x)` |

### 9.5 Exits: `exit_when`, `keep_when`, `reentry_days`

Six state leaves exist only inside these two formulas:

| Leaf | Unit | Definition | Usable in |
|---|---|---|---|
| `pos.ret` | RATIO | return since entry (+0.2 = up 20%). Entry price = the price of the fill; accumulated from adjusted returns, costs excluded | both |
| `pos.peak_ret` | RATIO | the highest `pos.ret` at any close since entry (>= 0; includes the 0 of the entry itself) | both |
| `pos.drawdown` | RATIO | the fall from the best closing value since entry: 1 - current value / best value (0.1 = 10% off) | both |
| `pos.days` | DAYS | closes since entry (with fills at the next open, the close of the entry day = 1) | both |
| `pos.rank` | COUNT | today's place among the candidates (1 = best) | `keep_when` only |
| `pos.held` | BOOL | always true | both |

There is one rule: **a position rule is an ordinary condition with the `pos.*` leaves above added.** Combine them with constants, market columns and formulas exactly as elsewhere using cell-wise operations: arithmetic, `%`, comparisons (`==` and `!=` included), `&&`, `||`, `!`, `? :`, `ABS`, `SQRT`, `LOG`, `EXP`, `FLOOR`, `ROUND`, `MIN2`, `MAX2`, `FILL`, `MASK` and any other function that reads only the same day and the same stock, under the unit rules of chapter 3. Position state exists only while a name is held and only for the names held, so two things differ from an ordinary formula:

| Difference | Note |
|---|---|
| functions that read other days or other stocks | `ROLL_*`, `EMA`, `LAG(2 or more)` (other days), `CS_*`, `G*` (other stocks on the same day) and the like take market-only parts; yesterday's value of position state is `LAG(1, …)` (for instance `pos.ret < LAG(1, pos.ret) - 0.03`, `ABS(pos.ret - LAG(1, pos.ret)) > 0.05`) |
| `pos.rank` | the rank on a rebalance day: `keep_when` only, alone against an integer (for instance `pos.rank <= 30`, `pos.rank == 3`); join other conditions beside it with `&&`, `\|\|` |

Examples: `pos.ret == pos.peak_ret` (closing at the best close since entry), `pos.ret * close <= -2 * ROLL_MEAN(14, high - low)` (the price lost exceeds twice the average daily range), `pos.ret > 0 ? close < ROLL_MEAN(20, close) : pos.days >= 60` (winners are sold below the 20-day mean, losers after 60 days).

Behaviour:

- **`exit_when`**: checked at every close for every holding; when true the holding is sold at the next fill without waiting for the rebalance day. Its share stays in cash until the next rebalance and the other weights are untouched.
- **`keep_when`**: on a rebalance day, holdings for which it is true stay, and only the remaining slots are filled by rank. Names that fail `hold` or left the stock pool are still sold.
- **`reentry_days`**: after an `exit_when` exit the same stock is not bought back at rebalances within N trading days. 0 = only the rebalance on the exit day is blocked.
- When part of a rule cannot be computed (the first 20 days of `ROLL_STD(20, …)`, a halted day, a market condition with no value), `&&`, `||` and `!` are three-valued: false && unknown = false, true || unknown = true, anything else that meets an unknown is unknown; an unknown result at the end counts as not true (no exit, no keep, no add or cut). So `pos.ret <= -0.1 || pos.ret >= 2 * ROLL_STD(200, RET(1, close))` still stops out in the first 200 days, and `!(pos.ret > 0 && market condition)` is not true when the market condition is unknown and `pos.ret > 0`. A `? :` condition in `weight` that is unknown leaves that stock's multiple unchanged that day (a new entry gets `size_init`).
- `summary.rule_exits_by_rule` in the report counts the exits per rule (each part joined by `||`).
- Exit rules are a setting of the whole search: every strategy follows them, whether its formula was sampled, given, or enumerated from a user function.

### 9.5a Group cap: `group_by`, `group_cap`

"At most N holdings per group". The two arguments go together: `group_by` says what a group is (a formula per day and stock; **stocks with the same value on a day are one group**), `group_cap` is the number of holdings per group (a positive integer).

- On a rebalance day names are taken in exact-rank order: stop once the target has `max_positions` names; a name whose group already has `group_cap` names in the target is skipped and the next one is taken.
- **Holdings count too**: holdings kept by `keep_when` are walked first in rank order, and the lower-ranked ones of a full group are sold as usual. Only a name locked limit-down can leave a group over the cap (it is still held and takes a slot of its group).
- Names without a group value (e.g. not in any `CORR_GROUP` group, warm-up) are groups of their own and are not limited.
- Groups are those of the **rebalance day**; a value that changes between rebalances does not trigger a sale, it is looked at again on the next rebalance day.
- The turnover cap, dropping names whose weight rounds to 0 and names that cannot be bought at a limit lock all work as before, and nobody is added afterwards, so the cap always holds.
- `hold_days = N`: each tranche counts its own (at most N x `group_cap` per group overall).
- `group_by` is any formula: automatic groups `CORR_GROUP(...)` (chapter 6), buckets `CS_QCUT(n, x)`, a code column, or a condition (two groups); `pos.*` is not allowed.

| Plain words | `portfolio` |
|---|---|
| at most 3 of the stocks that move together | `{"max_positions":10,"group_by":"CORR_GROUP(RET(1, close))","group_cap":3}` |
| market-cap quintiles, at most 2 per quintile | `{"max_positions":10,"group_by":"CS_QCUT(5, fund.mktcap)","group_cap":2}` |
| traded-value quintiles, at most 2 per quintile | `{"max_positions":10,"group_by":"CS_QCUT(5, ROLL_MEAN(20, px.turnover))","group_cap":2}` |
| at most 2 per group, and sell only out of the top 30 | `{"max_positions":10,"keep_when":"pos.rank <= 30","group_by":"CORR_GROUP(RET(1, close))","group_cap":2}` |

The plan returns `effective.portfolio.group_by.stats`: `cap`, the number of groups in the pool per day `groups_avg` / `groups_last`, the size of the largest group `largest_group_avg` / `largest_group_last`, names without a value `no_key_avg` / `no_key_last`. The report's `program.group_by` and `program.group_cap` record the grouping and the cap of the strategy.

### 9.5b Adding, cutting and the account level

**Adding and cutting**: `add_when` / `cut_when` trigger once at the close where the condition turns from false to true (staying true acts once), adding or cutting at the next fill; a later reversal does not undo it. The other way is a `weight` that reads `pos.*` (e.g. `"weight": "fund.mktcap * (pos.peak_ret >= 0.1 ? 2 : 1)"`), setting a target weight by state; use one of the two. The target multiple moves on a 1% grid of the target weight (the same grid as `size_init` and `add_size`): a multiple written out must be a multiple of 0.01 between 0.01 and 40.95 (`pos.ret > 0 ? 0.005 : 1` does not compile); a computed multiple (e.g. `1 + pos.ret`) is floored to 0.01 and capped at `max_size`; a computed multiple below 0.01 (0 and negatives included) counts as 0.01 in this version, so to sell the name write `exit_when`.

New leaves (only in `add_when`, `cut_when`, `exit_when`, `keep_when`): `pos.size` (current position / entry unit), `pos.adds`, `pos.cuts` (adds and cuts filled), `pos.avg_ret` (return against the average price after adds); intraday touch levels may also use `pos.avg` (the average price; the entry price when nothing was added), e.g. `"exit_below": "pos.avg * 0.92"`.

**Account level**: `gross_max` (cap on total positions, > 1 = margin), `cash_min` (minimum cash), `add_mode` (adds beyond the cap use only the room left or shrink the other holdings), `max_new` (most new names per rebalance), `cash_rate` / `margin_rate` (interest on cash and margin).

| Plain words | `portfolio` |
|---|---|
| add one more unit at +10% (enter at half size to leave room; corpus B12) | `{"add_when":"pos.ret >= 0.1","size_init":0.5}` |
| add again every further 10%, up to 3x | `{"add_when":"pos.ret >= 0.1 * (pos.adds + 1)","max_size":3}` |
| average down once at -5% | `{"add_when":"pos.ret <= -0.05 && pos.adds == 0","size_init":0.5}` |
| sell half at +20%, the rest on a 10% drop from the peak | `{"cut_when":"pos.ret >= 0.2","exit_when":"pos.drawdown >= 0.1"}` |
| at most 80% invested, at most 3 new names per rebalance | `{"gross_max":0.8,"max_new":3}` |

After a rebalance, in order: names locked limit-up are not bought -> new names without a price on the fill day (an incomplete bar) are not filled and their share stays in cash -> beyond `max_new` new names only the best-ranked are kept -> names whose size shrinks to 0 are not entered.

The report's `size_events` counts adds and cuts and lists the latest. On-demand help: `stock_formula_help topic=portfolio name=sizing` (adding and cutting), `name=account` (account level).

### 9.6 Common exit forms

| Plain words | `portfolio` |
|---|---|
| stop loss at 8% | `{"exit_when":"pos.ret <= -0.08"}` |
| take profit at 20% | `{"exit_when":"pos.ret >= 0.2"}` |
| both | `{"exit_when":"pos.ret <= -0.08 \|\| pos.ret >= 0.2"}` |
| trailing stop: 10% off the best level since entry | `{"exit_when":"pos.drawdown >= 0.1"}` |
| break-even stop: up 10% at some point, then back to the entry price | `{"exit_when":"pos.peak_ret >= 0.1 && pos.ret <= 0"}` |
| time stop: held 20 trading days | `{"exit_when":"pos.days >= 20"}` |
| below the 20-day average | `{"exit_when":"close < ROLL_MEAN(20, close)"}` |
| volatility stop: down more than 2x its own 20-day daily-return standard deviation | `{"exit_when":"pos.ret < -2 * ROLL_STD(20, RET(1, close))"}` |
| sold only when held 60 days and still losing | `{"exit_when":"pos.days >= 60 && pos.ret < 0"}` |
| no buy-back within 20 trading days of a stop | `{"exit_when":"pos.ret <= -0.08","reentry_days":20}` |
| rank buffer: buy in the top 10, sell out of the top 30 | `{"max_positions":10,"keep_when":"pos.rank <= 30"}` |
| keep the holdings in profit, replace the rest | `{"keep_when":"pos.ret > 0"}` |

All of these are **confirmed at the close**: the condition is checked at that day's close and the sale happens at the next fill. The intraday touch version is in §9.6a.

### 9.6a Intraday touch: `exit_below`, `exit_above`, `lock`

Every holding is looked at once a day: when the day's low <= the stop level (or the day's high >= the target level) it exits intraday that day, without waiting for the close. The level is computed at the previous close, so nothing looks ahead.

| New leaf | Unit | Meaning |
|---|---|---|
| `pos.entry` | PRICE | the entry fill price (adjusted-price convention) |
| `pos.peak` | PRICE | the most favourable **closing price** since entry (the entry price included) |

A level formula = the extreme of one or more terms (`MAX2` for `exit_below`, `MIN2` for `exit_above`); each term is `constant x pos.entry + constant x pos.peak + a market-only price formula`.

| Plain words | Confirmed at the close (`exit_when`) | Intraday touch |
|---|---|---|
| stop loss at 8% | `exit_when="pos.ret <= -0.08"` | `exit_below="pos.entry * 0.92"` |
| take profit at 20% | `exit_when="pos.ret >= 0.2"` | `exit_above="pos.entry * 1.2"` |
| trailing stop: 10% off the best close | `exit_when="pos.drawdown >= 0.1"` | `exit_below="pos.peak * 0.9"` |
| fixed and trailing stop, whichever comes first | — | `exit_below="MAX2(pos.entry * 0.92, pos.peak * 0.9)"` |
| below the 20-day average | `exit_when="close < ROLL_MEAN(20, close)"` | `exit_below="ROLL_MEAN(20, close)"` |
| volatility stop | `exit_when="pos.ret < -2 * ROLL_STD(20, RET(1, close))"` | `exit_below="pos.peak - 3 * ROLL_MEAN(14, high - low)"` |

Four fill rules:

1. A gap through the level at the open fills at the open; otherwise at the level.
2. Stop and target both touched on the same day count as the stop first.
3. The day's high does not raise that day's trailing stop (the level uses state only up to the previous close).
4. A sale blocked by a limit-down lock is deferred to the next day.

When the user says only "stop at 8%, take profit at 20%", write the touch form; when the user says "leave only on a close below", write `exit_when`.

`portfolio.lock` (optional): `"limit"` defers a sale blocked by a limit-down lock (limit-up for shorts), `"none"` does not simulate locks; omitted = simulated when the panel has the exchange limit prices (`raw.high`, `raw.low`, `raw.limit_up`, `raw.limit_down`), otherwise not; plan's `portfolio.lock` says which (`limit` / `none` / `unavailable` / `no_limits`). Can be given with `exit_when` / `keep_when` / `reentry_days` (`reentry_days` also covers touch exits); with `hold_days` each slice is touched and exits on its own . `$name` (threshold grid) may be written inside a level too, sharing one `grid` with `exit_when` (9.11). In the report, touch exits in `rule_exits.items` carry `touch` (`below` / `above`) and `fill_vs_prev_close_1e6` (fill / previous close); the `touch` block has counts per side and `lock_unknown`.

### 9.7 Fills and costs

| Item | Default | Note |
|---|---|---|
| Fill timing | selection after the close of the rebalance day, filled at **the next open** | on decision day t the selection uses the ranking of t; the overnight return of t+1 (open / previous close - 1) goes to the old holdings, and after the rebalance the intraday return of t+1 (close / open - 1) goes to the new holdings. A stock without an open price that day is filled at the previous close |
| Buy cost | Taiwan: commission 0.1425%; US: 0 (zero commission) | |
| Sell cost | Taiwan: commission 0.1425% + transaction tax 0.3%; US: SEC fee (on the amount) | the Taiwan tax on ETFs is 0.1% |
| Minimum commission | 20 per order (`min_fee`, local currency) | counted only in the share-level account |
| Turnover cap | none | the fourth argument of `apx.PICK` (9.12; `0.3` = new names at most 30% of the names held), or `max_turnover_bp` in `config` (`3000` = the same thing) |

- Costs and trading rules follow the cost table of each stock's own market automatically (`market/costs/tw.json`, `us.json`); formulas do not write them.
- A held stock with no price on a day (halt, delisting): that day's return counts as 0 and the stock stays held; it leaves at the next rebalance (with `hold_days`, when its slice expires) at the last price.
- They can be overridden through `config` of `stock_backtest`: `fee_bp_x100` (in 0.01 basis points, default 1425), `tax_sell_bp_x100` (default 3000), `tax_sell_etf_bp_x100` (default 1000); `cost_bp` >= 0 = the legacy model, N basis points per side and no separate tax.
- Buy and hold within a holding period: between rebalances each stock is bought and held and its weight drifts with its price; weights return to target only on rebalance days and days a size change fills. Exposure changes, rule exits and touch exits leave the other holdings' weights alone (an exited stock's value stays as cash). Same definition as the per-share account.
- Strategy equity is the sum of daily returns (each day's return as above); the annual return and maximum drawdown in reports use compounding (chapter 11).
- A rebalance whose decision day is the last day of the panel is not done (the next day has not arrived).

### 9.8 Selection, ranks and ties

On a rebalance day (long side):

1. Candidates = stocks inside the stock pool that day, with a valid score, passing `hold`.
2. Candidates are ordered by score into that day's **exact ranks**: one distinct position per stock; a day with fewer than 2 members has no ranks.
3. Target = the `max_positions` highest-ranked candidates.
4. Entries = target - current holdings; exits = current holdings - target.

Ties: every stock has one distinct position per day. The order is larger score -> (when `tie_by` is given) larger `tie_by` value, cells without a value last among the tied names -> larger value of the fixed shuffle `mix64((entity_id << 32) ^ day)` -> smaller stock index. The selection does not depend on stock codes or on the column order of the panel, and a rerun on the same data orders ties the same way. `tie_by` only orders ties: the score, the position count and the weights are unchanged and the warm-up is not raised; for smaller first write `NEG(...)`. With massive ties (a bucket number or a percentile inside a group as the whole score) the names picked are decided by `tie_by` or by the shuffle: use continuous scores (returns, ratios), or give `tie_by`. Readers of ranks such as `CS_PCT` and `pos.rank` are not affected by `tie_by` (a percentile only counts the names below it and equal to it).

### 9.9 Capital and the share-level account

With any of `initial_capital`, `lot`, `min_fee`, `contribution` the report adds a share-level account: shares per stock, board lots and odd shares, commission, the number of orders at the minimum commission, final value.

| Plain words | `portfolio` |
|---|---|
| 4 million in 10 stocks, odd lots, minimum commission 20 | `{"max_positions":10,"initial_capital":4000000,"lot":"odd","min_fee":20}` |
| 5,000 more every month | `{"initial_capital":5000,"contribution":5000,"contribution_every":"month","lot":"odd"}` |
| every 5 trading days, at most 10 stocks, 4 million | `{"rebalance_days":5,"max_positions":10,"initial_capital":4000000}` |

Commission per order = max(`min_fee`, floor(traded amount x commission rate)) (Taiwan 0.1425%, US 0).

### 9.10 Backtest period

Top-level arguments of `stock_backtest`:

| Argument | Value | Meaning |
|---|---|---|
| `backtest_from` | `YYYY-MM-DD`, `YYYY-MM` (first day of that month), `YYYY` (1 January) | start; omitted = the panel's first day |
| `backtest_to` | the same (`YYYY-MM` = month end, `YYYY` = 31 December) | end; omitted = the latest close |
| `backtest_years` | 1..100 | instead of `backtest_from`: the last N years up to the end |

- Data before the start only warms up; screening, IC, checks and the report count from the start.
- One result library is bound to one period. To see a stored strategy over another period: `my_stock_strategies_change action=report backtest_from=...` (no new search).

### 9.11 Letting the system try thresholds: `$name` + `grid`

Numbers in `exit_when` / `keep_when` and in the intraday levels `exit_below` / `exit_above` that the system should try are written `$name`, and `grid` gives the values to try for each name (one `grid` shared by the four). Each set, substituted, equals writing those thresholds directly.

| Plain words | `portfolio` | Sets |
|---|---|---|
| try a few stop-loss / take-profit pairs | `{"exit_when":"pos.ret <= -$stop \|\| pos.ret >= $take","grid":{"stop":[0.05,0.08,0.1],"take":[0.15,0.2,0.3]}}` | 9 |
| let the system find the percentage off the high | `{"exit_when":"pos.drawdown >= $dd","grid":{"dd":{"from":0.06,"to":0.14,"step":0.02}}}` | 5 |
| let the system try the intraday stop level | `{"exit_below":"pos.entry * $floor","grid":{"floor":[0.9,0.92,0.95]}}` | 3 |
| the volatility multiple and the longest holding, tried together | `{"exit_when":"pos.ret < -$k * ROLL_STD(20, RET(1, close)) \|\| pos.days >= $n","grid":{"k":[1.5,2,3],"n":[20,40]}}` | 6 |

- Values are given as an array `[0.05, 0.08, 0.1]` or as `{"from": 0.04, "to": 0.1, "step": 0.02}`. The same `$name` may appear several times.
- Limit: base-grid settings x threshold sets <= 32,767 (cells of the trial ledger); beyond that an error, use fewer values or split the search.
- A value may also be how many days the condition looks back, for instance `{"exit_when":"close < ROLL_MEAN($n, close)","grid":{"n":[20,60]}}`: every set starts on the start day of the set with the longest warm-up.
- Every set runs. Per formula and portfolio setting the sets are compared by `grid_by` (default `apx.SHARPE`) and one is kept; the set it used is in `program.exit_when` of the report, and the numbers of every set are in `exit_grid` of the job summary.
- Every set is one trial in the multiple-testing count (10.3).
- To search near the original value just wrap the threshold: `{"exit_when":"pos.ret <= -apx.EVOLVE(0.08)"}` = `{"exit_when":"pos.ret <= -$evolve1","grid":{"evolve1":[0.04,0.06,0.07,0.08,0.09,0.1,0.12,0.16]}}` (the original x 1/2 ... 2, as in 10.1); integers such as `apx.PICK(apx.EVOLVE(10), false)`, `apx.EVERY(apx.EVOLVE(21))` use the day grid. Only a single number may be wrapped; wrapping an operator parameter (how many days it looks at, e.g. `ROLL_MEAN(apx.EVOLVE(20), close)`) returns `E_EVOLVE_WINDOW`, whose fix is the `$name` + `grid` form.

### 9.11a The search's base grid: `pl`, `reb`, `n`, `wk`

In a search every tree tries a "base grid": long threshold x rebalance days x positions (x weight key). Not written: thresholds 160 / 170 / 180 / 190, rebalance 5 / 21 days, 5 / 10 positions (16 settings). Each axis is a `$name` in the portfolio formulas with its values in the same `grid`; giving only `grid` works too.

| Plain words | `portfolio` | Base grid |
|---|---|---|
| try rebalancing every 5, 10 and 21 days | `{"grid":{"reb":[5,10,21]}}` | 4 x 3 x 2 = 24 |
| buy only the top 10% (threshold 180) | `{"hold":"apx.IN_TOP(180)"}` | 1 x 2 x 2 = 4 |
| try top 25% down to top 2.5% | `{"hold":"apx.IN_TOP($pl)","grid":{"pl":[150,160,170,180,190,195]}}` | 6 x 2 x 2 = 24 |
| hold 10, 20 or 30 stocks | `{"weight":"apx.SHARE(apx.PICK($n, false), 1)","grid":{"n":[10,20,30]}}` | 4 x 2 x 3 = 24 |
| try equal and inverse-volatility weights | `{"weight":"apx.SHARE(apx.PICK(20, false), $wk == 0 ? 1 : INV_VOL_KEY(20, RET(1, close)))","grid":{"wk":[0,1]}}` | 4 x 2 x 1 x 2 = 16 |
| rank weights with the threshold tried along | `{"hold":"apx.IN_TOP($pl)","weight":"apx.SHARE(apx.PICK(20, false), apx.RANK_KEY($pl))","grid":{"pl":[170,190]}}` | 2 x 2 x 1 = 4 |

- The four places: `apx.IN_TOP($pl)` in `hold`, `apx.EVERY($reb)` in `rebalance_when`, the count of `apx.PICK($n, ...)`, the weight key `$wk == 0 ? 1 : INV_VOL_KEY(20, RET(1, close))`. Any name works; with only `grid` use `pl`, `reb`, `n`, `wk`.
- One value = writing that number (`{"grid":{"reb":[10]}}` and `{"rebalance_days":10}` are the same thing and the same result library).
- These `$name`s go only in those places; numbers to try elsewhere (exits...) take their own names. When an axis already has a fixed value (e.g. `rebalance_days`), do not give that axis in `grid` too.
- `apx.RANK_KEY(p)` (rank weight key, = `FLOOR(q * 200) - p + 1`, q as in `apx.IN_TOP`) uses the same p as `apx.IN_TOP(p)`; no argument = follow the long threshold.
- What you write is used as written: `apx.IN_TOP(p)` in `hold` means no default threshold is added; `grid` with only `pl` / `reb` / `n` / `wk` replaces that axis's defaults, e.g. `{"grid":{"pl":[0]}}` = no threshold.
- `apx.IN_TOP` and `apx.RANK_KEY` read the strategy's own score and go only in `portfolio`; in a score formula use `CS_PCT(x)` for a percentile rank.
- Every base-grid setting is one trial in the multiple-testing count (10.3); plan's `effective.portfolio` lists each axis's values (`rebalance_days`, `max_positions`, `in_top`).

### 9.12 Backtest rules as formulas: `weight` + `rebalance_when`

Every argument above is shorthand for two formulas. You can write the formulas directly; plan maps them back to the same rules (the numbers are bit-identical):

- `weight`: each stock's target weight every day, written as a product of factors. It must have one factor `apx.SHARE(apx.PICK(N, keep), key)` (which names and how to split); the other factors are exposure, exits and sizing.
- `rebalance_when`: rebalance days. `apx.EVERY(21)` = every 21 trading days; or a condition with one value per day for the market (9.3).
- `hold` is still the candidate condition, with two more kinds of factor: `GTOP(k, group key, apx.SCORE())` (at most k per group) and `!FILL(pos.exit_days <= n, false)` (no buy-back within n trading days of an exit; n = 0 blocks the exit day only).
- Tie key: wrap the score formula in `apx.THEN(score, key)`.

| Argument | Formula |
|---|---|
| `max_positions=10` | `weight="apx.SHARE(apx.PICK(10, false), 1)"` (`1` = equal weight) |
| `weighting="score"` / `"inv_vol"` | key `apx.RANK_KEY()` / `INV_VOL_KEY(20, RET(1, close))` |
| `weight="SQRT(fund.mktcap)"` | `apx.SHARE(apx.PICK(10, false), SQRT(fund.mktcap))` |
| `max_weight=15` | `apx.CAP(0.15, apx.SHARE(...))` |
| `keep_when="pos.rank <= 30"` | `apx.PICK(10, FILL(pos.rank <= 30, false))` |
| `max_new=3` | `apx.PICK(10, false, 3)` |
| `config.max_turnover_bp=3000` (turnover cap) | `apx.PICK(10, false, 0, 0.3)` (fourth argument: new names per rebalance <= floor(0.3 x the names held now); third argument `0` = no limit on new names) |
| `exposure="..."` | times `(...)` (the market-wide formula in parentheses) |
| `gross_max=0.8`, `cash_min=0.1` | times `0.8`, times `(1 - 0.1)` |
| `gross_max=1.5` (margin) | times `MIN2(1, 1.5 / acct.gross_req)` (`acct.gross_req` = the sum of target weights at a rebalance; `margin_rate` still given); with an add term, wrap the add in `MIN2(..., acct.room)` = adds beyond the cap use only the room left (`add_mode="cash"`, the default) |
| `add_mode="scale"` | times `MIN2(1, g / acct.gross_req)` (`1` without margin), add term not wrapped in `acct.room` = beyond the cap shrink all holdings in proportion |
| `exit_when="cond"` | times `!(pos.held && FILL(cond, false))` |
| `exit_below`, `exit_above` | times `!(pos.held && FILL(TOUCH_BELOW(pos.entry * 0.92) \|\| TOUCH_ABOVE(pos.entry * 1.2), false))` |
| `add_when="A"`, `size_init=0.5` | times `FILL(pos.size + 1 * (A && !FILL(LAG(1, A), false) && !apx.EVERY(21)), 0.5)` (`!apx.EVERY(21)`: on rebalance days the trigger is recorded, not traded; wrapping in `MIN2(cap, ...)` = `max_size`; with margin the add term is `MIN2(1 * (...), acct.room)`) |
| `cut_when="C"` | times `FILL(pos.size * (1 - 0.5 * (C && !FILL(LAG(1, C), false) && !apx.EVERY(21))), 1)` |
| `weight` reading `pos.*` (target form) | times `FILL(pos.peak_ret >= 0.1 ? 2 : 1, 1)` |
| `rebalance_days=21` | `rebalance_when="apx.EVERY(21)"` |
| `hold_days=10` | `weight="apx.STAGGER(10, apx.SHARE(...) * ...)"` |
| `reentry_days=20` | `hold` gets `!FILL(pos.exit_days <= 20, false)` |
| `group_by` + `group_cap=3` | `hold` gets `GTOP(3, CORR_GROUP(RET(1, close)), apx.SCORE())` |
| `tie_by="fund.mktcap"` | formula `apx.THEN(RET(60, close), fund.mktcap)` |

Example: sell half at +20%, the rest on a 10% fall from the high, monthly rebalance (corpus B12c)

```json
{"rebalance_when": "apx.EVERY(21)",
 "weight": "apx.SHARE(apx.PICK(10, false), 1) * !(pos.held && FILL(pos.drawdown >= 0.1, false)) * FILL(pos.size * (1 - 0.5 * (pos.ret >= 0.2 && !FILL(LAG(1, pos.ret >= 0.2), false) && !apx.EVERY(21))), 1)"}
```

- **Stocks not held**: `pos.held` = 0 and every other `pos.*` has no value (nor does `pos.exit_days` for a stock never exited). Wrap every factor that reads `pos.*` in `FILL`, or no new stock is ever bought: plan answers `E_NEVER_ENTER` with `fix` = the whole formula corrected, to paste as is. The three common ones: `(pos.ret > 0 ? 2 : 1)` -> `FILL((pos.ret > 0 ? 2 : 1), 1)`; `!(pos.held && pos.ret <= -0.08)` -> `!(pos.held && FILL(pos.ret <= -0.08, false))`; `!(pos.exit_days <= 20)` -> `!FILL(pos.exit_days <= 20, false)`.
- The same thing written twice (e.g. `max_positions` and `apx.PICK`, `rebalance_days` and `apx.EVERY`) -> `APX_CONFLICT`; keep one. With a `weight` formula, `max_positions`, `weighting`, `max_weight`, `exposure`, `gross_max`, `cash_min`, `keep_when`, `exit_when`, `exit_below`, `exit_above`, `max_new`, the sizing arguments, `add_mode` and `hold_days` go inside the formula; the turnover cap is the fourth argument of `apx.PICK`, so drop `config.max_turnover_bp`.
- `acct.room`: the remaining gross room this name gets for today's add (in multiples, the unit of `pos.size`) = the gross cap minus the gross before today's adds, split over today's adds in proportion to what each asks for, divided by this name's target weight; it uses only data up to today's close.
- Environment facts stay arguments: `initial_capital`, `contribution`, `contribution_every`, `lot`, `min_fee`, `lock`, `cash_rate`, `margin_rate`, `grid`, `grid_by`.
- This edition accepts the shapes in the table; other forms (e.g. a weight reading `GSUM(...pos.*...)`, two multipliers reading `pos.*`) answer `APX_UNSUPPORTED`, naming the missing piece.

---

## 10. Search

### 10.1 Three sources of formulas: `trees_from`

| Value | Where formulas come from | Also given |
|---|---|---|
| `formulas` | formula texts given directly | `formulas=["...", ...]` |
| `nouns` | a user function enumerated over its parameter grid x input bindings | `nouns=[{"name":"user.me.NEAR_HIGH","bind":{"x":"close"},"grid":{"n":[60,120]}}]` |
| `sample` | generated at random by the system (algorithmic search) | `trees` (how many), `seed`, `rounds` |

- `nouns`: `bind` maps a formal input to a column (omitted: every column of the same unit is tried); `grid` overrides the grid of the definition; grid x bindings is at most 20,000 combinations.
- `formulas`: each one is validated at plan time; one that does not compile, has a non-numeric root, or needs too long a warm-up is reported at plan time with its text.
- **Letting the system try numbers in a formula**: write the number as `$name` and give its values in `grid` (the same `grid` as the `$name`s of portfolio rules, 9.11). Example "try a 6-month and a 1-year momentum lookback, both skipping the last month": `formulas=["RET_SKIP($n, 21, close)"]`, `portfolio={"grid":{"n":[126,252]}}` -> two trees `RET_SKIP(126, 21, close)` and `RET_SKIP(252, 21, close)`, the same trees as writing those two directly (same identity, same results). No user function to install; `nouns` (user function + `bind` + `grid`) is the older way of saying the same thing, still accepted and expanded to the same trees.
- **One `grid`, routed by position**: a name in a score formula gives one tree per value; a name in `hold` / `exposure` / `weight` / `exit_when` / `keep_when` / `exit_below` / `exit_above` / sizing gives variants of the same tree (9.11). Example "try momentum over 60 and 120 days and a stop at 8% and 10%": `formulas=["RET($n, close)"]`, `portfolio={"exit_when":"pos.ret <= -$stop","grid":{"n":[60,120],"stop":[0.08,0.1]}}` -> two trees, two variants each. A name may not be written on both sides (`E_GRID_SHARED`: use a name per side). Names in formulas: up to 256 values each and 20,000 trees after expansion (as `nouns`); window lengths are accepted (each value is another tree with its own warm-up). Plan's `effective.formula_grid` lists the names, how many formulas were written and how many they expanded to; the first name of `grid` is the outermost loop.
- **Letting the system search near the original: `apx.EVOLVE(...)`** (strategy optimization): wrap the one number or column the system may change; the system takes candidates near the original, one tree per candidate. Removing `apx.EVOLVE` gives the starting strategy (the same identity).
  - A wrapped operator parameter: two grid points on each side of the original on that parameter's grid. Example `RET(apx.EVOLVE(60), close)` -> five trees 20, 40, 60, 120, 252, the same trees and the same result as `RET($n, close)` + `grid {"n":[20,40,60,120,252]}`. A packed parameter changes only the wrapped part (`RET_SKIP(252, apx.EVOLVE(21), close)` tunes 21 only).
  - A wrapped constant: the original x 1/2, 3/4, 7/8, 9/8, 5/4, 3/2, 2 (constants in days or counts use the day grid). A BOOL constant and 0 have no neighbours.
  - A wrapped column: every panel column of the same unit and kind (per-share / total, frequency, alignment). Example `CS_PCT(RET(20, apx.EVOLVE(close)))`.
  - A candidate is listed only if the whole formula compiles, has a numeric root and fits the warm-up; none -> `E_EVOLVE_EMPTY` (fix: wrap one level out). `EVOLVE(...)` also works; exactly one argument (`E_EVOLVE_ARGS`).
  - Several positions may be wrapped (their candidates multiply, the first in the text is the outermost loop); `$name` may be used together (it is substituted first). Plan's `effective.formula_grid.evolve` lists the values taken for each position (`kind`: `param` / `const` / `leaf`).
  - Wrapping a whole expression (e.g. `apx.EVOLVE(CS_PCT(fund.roe))`) returns `APX_UNSUPPORTED`; not accepted in the stock pool (`APX_UNSUPPORTED`: try pool thresholds with `$name` + `grid` separately). Thresholds in portfolio rules: 9.11.
- The same tree (equal after expansion) counts once.

Every tree additionally tries a set of portfolio settings (a base grid of 16: rebalance every 5 / 21 days, at most 5 / 10 positions), and `screen_pick` keeps one. When `rebalance_days` or `max_positions` is given, that axis is fixed to the given value.

### 10.2 Admission thresholds

Thresholds are arguments of the search, not properties of a formula:

| Source | Thresholds not stated |
|---|---|
| random generation | defaults apply |
| given formulas (`formulas`), user-function grids (`nouns`) | none applied: every strategy is computed and recorded; nothing is filtered on the user's behalf (only a strategy that holds nothing on every day of the backtest is not admitted, whatever the source) |
| thresholds stated explicitly (`screen`, `screen_pick`, `screen_gate`, `screen_admit`, `config`) | applied whatever the source |

Defaults for random generation: cross-section score >= 2,000 bp, time score >= 2,500 bp, novelty >= 5,000 bp, net return over the period > 0, IC > 0, IC t-value >= 4 (`min_ic_t` in `config`).

### 10.3 The number of trials for multiple testing

Every (tree x portfolio setting) tried is one record in the trial ledger of the result library, accumulated across rounds and runs = "how many times we tried", N.

| Quantity | Definition |
|---|---|
| N | the number of trials in the ledger. The same tree with the same portfolio arguments is not counted twice; a trial with no holdings is not counted; every set of a threshold grid (9.11) counts once |
| N_eff | the effective number of trials: N discounted by the correlation of excess returns between trials (settings of one tree are highly correlated; correlation between trees is estimated separately) |
| DSR | Deflated Sharpe Ratio: the probability after taking "the luckiest Sharpe among that many trials" as the benchmark, computed with N_eff |
| IC p_adj | the one-sided p-value of the IC with a Šidák adjustment for N_eff |
| PBO | the probability of backtest overfitting, computed with CSCV at the end of a round |

By default these are recorded and written into the report as reference and do not block admission. To make it a hard gate, put `DSR` into `checks` (chapter 11).

---

## 11. Screening metrics and checks

### 11.1 The screening language

The same operators and precedence (plus `%`); values are real numbers (double). Names are **strategy-level atoms**: the numbers and daily series left after one strategy (a tree with one portfolio setting) has run.

- Scalars and series mix freely (scalars broadcast). Division by 0 = missing (`nan`); a comparison with a missing value is unknown, `&&` and `||` answer only when the answer is already settled (false && unknown = false, true || unknown = true), and a threshold that comes out unknown does not pass. Series always stay aligned by day.
- The result of a condition must be a scalar.

**Series atoms** (daily, from the first day after warm-up):

| Name | Meaning | Example |
|---|---|---|
| `r` | the strategy's daily net return (costs deducted; additive) | `mean(r) / sd(r) * sqrt(252)` |
| `m` | the equal-weighted mean daily return of the stocks in the same pool | `mean(r - m) * 252` |
| `year` | the calendar year of each day (a grouping key) | `by_sum(year, r)` |
| `month` | year x 12 + month - 1 of each day (a grouping key) | `mean(by_sum(month, r) > 0)` |
| `regime` | bull or bear: the pool's cumulative return over the last 252 days > 0 -> 1, < 0 -> -1, unclear 0 | `filter(r, regime < 0)` |
| `half` | first half 1 / second half 2 | `filter(r, half == 2)` |
| `day` | day number from 0 | `filter(r, day >= 252)` |
| `pos` | number of holdings | `MEAN(pos)` |
| `b` | daily return of the market benchmark: the panel's total return main index (dividends reinvested; TAIEX Total Return in Taiwan, S&P 500 Total Return in the US; column `mkt.idx_tr`); missing when the panel has no such column | `MEAN(r - b) * 252` |

**Scalar atoms** (numbers the engine has computed):

| Group | Names |
|---|---|
| Scores | `cs_bp` (cross-section score), `ts_bp` (time score), `novel_bp` (novelty), `short_bp` (shortness), `total_bp` (combined score = `apx.TOTAL`), `n_nodes` (nodes of the tree) |
| Strategy summary | `net_ret_1e6` (net return over the period), `max_dd_1e6` (maximum drawdown), `n_rebalances`, `avg_positions_x100`, `breadth_min` (fewest selectable stocks: the minimum over rebalance days; days with no valid rank are not counted) |
| Portfolio arguments | `reb_days`, `p_long`, `p_short`, `max_positions`, `cost_bp`, `weight_mode` |
| Panel | `n_days`, `day_first`, `day_mid`, `n_symbols` |
| Factor metrics | `ic_x1e4` (mean rank IC), `ic_t_x100` (IC t-value), `ic_pos_x1e4`, `ic_days`, `spread_1e6` (annualised top 10% - bottom 10% spread), `spread_days`, `quint_1e6_1`...`quint_1e6_5` (annualised quintile returns), `quint_days`, `quint_rho_x1e4`, `quint_mono_x1e4`, `turnover_bp_avg`, `turnover_days`, `breadth_med`, `breadth_days`, `ic_decay_x1e4_1` / `_5` / `_21` / `_63`, `ic_decay_days_1` / `_5` / `_21` / `_63`, `ic_half_life_days`, `ic_h1_x1e4`, `ic_h2_x1e4` |
| Exposure | `size_rho` (rank correlation between ranking and market cap), `ind_hhi`, `ind_top`, `n_ind`, `adv_med`, `adv_min` |
| Risk (needs a risk model) | `alpha_share_bp` (share of return variance from stock-picking alpha, can be negative), `breadth_eff_x100` (annual effective breadth × 100) |
| Multiple testing (needs `multitest`) | `mt_dsr` (Deflated Sharpe, 0..1), `mt_ic_padj` (Šidák-adjusted IC p-value, 0..1) |
| Round | `round_pbo` (this round's PBO, 0..1; `screen_gate` only) |

The tail of a name states the scale: `_bp` = ten-thousandths, `_1e6` = millionths, `_x100`, `_x1e4`.

**Functions**:

| Kind | Functions |
|---|---|
| Element-wise | `ABS` `SQRT` `LOG` `EXP` `FLOOR` (round down toward minus infinity) `POW(x, y)` `MIN2(x, y)` `MAX2(x, y)` `WHERE(cond, a, b)` `HAS(x)` |
| Reductions (series -> scalar) | `SUM` `MEAN` `SD` `VAR` (sample, n - 1) `COUNT` (number of elements) `FIRST` `LAST` `MIN(x)` `MAX(x)` (`AMIN` / `AMAX` are the same) `QUANTILE(x, q)` `STREAK(x)` (longest run of true) `LEN`; one missing cell in the set makes the result missing (to use only the known values write `FILTER(x, HAS(x))`), `COUNT` and `LEN` count cells whatever their values |
| Series -> series | `CUM` `PEAK` `DRAWDOWN` (fall of additive equity) `NAV` (compounded net value) `DRAWDOWN_NAV` (compounded fall) `DIFF` `FILTER(x, cond)` |
| Windows (parameter first) | `ROLL_SUM(n, x)` `ROLL_MEAN(n, x)` `ROLL_STD(n, x)` `ROLL_MIN(n, x)` `ROLL_MAX(n, x)` (complete windows only: the first n - 1 days are not in the set), `SKIP(k, x)` (the first k days are not in the set), `LAG(k, x)` (shift back k days, the first k days are not in the set); `DIFF`: the first day is not in the set |
| Grouping | `BY_SUM(key, x)` `BY_MEAN` `BY_SD` `BY_MIN` `BY_MAX` `BY_COUNT` `BY_PROD1` (compounded) `BY_KEY(key)` |
| Days | `RANK_DESC(x)` (rank of each day, largest first, from 1; ties go to the earlier day; missing values take no rank), `BOOT(seed, block)` (block bootstrap counts: the days are cut into blocks of `block` days, ceil(days / block) blocks are drawn with replacement, each day's value is how often its block was drawn; day order kept. Seeded by seed, the round's `val_alt_seed` and the panel: a recomputation gives the same bits) |
| Constant | `nan` |

- Function names are case-insensitive; the old lower-case input-first forms (`roll_mean(r, 63)`) still work, with an integer constant accepted on either side. The old `lag(x, k)` is `SKIP(k, x)`.
- The number of days a condition holds is `SUM(condition)`, the share is `MEAN(condition)`; `COUNT(x)` counts elements regardless of value.

### 11.2 The `apx.*` metric dictionary

Official definitions written with the atoms above. User metrics are written the same way (11.4).

| Name | Definition | Meaning |
|---|---|---|
| `apx.SHARPE_OF(x)` | `mean(x) / sd(x) * sqrt(252)` | annualised Sharpe (risk-free rate 0) |
| `apx.ANN_OF(x)` | `pow(last(nav(x)), 252 / count(x)) - 1` | annual return (compounded) |
| `apx.ANN_ADD_OF(x)` | `mean(x) * 252` | annual return (additive) |
| `apx.VOL_OF(x)` | `sd(x) * sqrt(252)` | annualised volatility |
| `apx.COV_OF(x, y)` | `sum((x - mean(x)) * (y - mean(y))) / (count(x) - 1)` | sample covariance |
| `apx.CORR_OF(x, y)` | `apx.COV_OF(x, y) / (sd(x) * sd(y))` | correlation |
| `apx.BETA_OF(x, y)` | `apx.COV_OF(x, y) / var(y)` | beta of x on y |
| `apx.MAX_DD_OF(x)` | `max(drawdown_nav(x))` | maximum drawdown (closes, compounded) |
| `apx.MAX_DD_ADD_OF(x)` | `max(drawdown(x))` | maximum drawdown (additive equity) |
| `apx.TOTAL_RET` | `last(nav(r)) - 1` | net return over the period (compounded) |
| `apx.TOTAL_RET_ADD` | `sum(r)` | net return over the period (additive; the convention of `net_ret_1e6`) |
| `apx.ANN_RET` | `apx.ANN_OF(r)` | annual return (compounded) |
| `apx.ANN_RET_ADD` | `apx.ANN_ADD_OF(r)` | annual return (additive) |
| `apx.HIT_RATE` | `mean(r > 0)` | daily hit rate |
| `apx.MONTHLY_RET` | `by_prod1(month, r)` | return of each calendar month (series) |
| `apx.MONTH_WIN` | `mean(apx.MONTHLY_RET > 0)` | share of winning months |
| `apx.BEST_MONTH` | `max(apx.MONTHLY_RET)` | best calendar month |
| `apx.WORST_MONTH` | `min(apx.MONTHLY_RET)` | worst calendar month |
| `apx.N_MONTHS` | `count(apx.MONTHLY_RET)` | number of calendar months |
| `apx.ROLL12_WIN` | `mean(roll_sum(r, 252) > 0)` | share of rolling 252-day returns above 0 |
| `apx.YEARS` | `by_key(year)` | the calendar years (series) |
| `apx.YEARLY_RET` | `by_prod1(year, r)` | return of each calendar year (series) |
| `apx.YEARLY_SHARPE` | `by_mean(year, r) / by_sd(year, r) * sqrt(252)` | Sharpe of each calendar year (series) |
| `apx.YEARLY_DAYS` | `by_count(year, r)` | trading days of each calendar year (series) |
| `apx.N_YEARS` | `count(apx.YEARLY_RET)` | number of calendar years |
| `apx.YEARS_POS` | `mean(apx.YEARLY_RET > 0)` | share of calendar years with a return above 0 |
| `apx.RET_Y_MIN` | `min(apx.YEARLY_RET)` | worst calendar year |
| `apx.RET_Y_MAX` | `max(apx.YEARLY_RET)` | best calendar year |
| `apx.ANN_VOL` | `apx.VOL_OF(r)` | annualised volatility |
| `apx.DOWN_VOL` | `sqrt(mean(pow(min(r, 0), 2))) * sqrt(252)` | annualised downside volatility |
| `apx.MAX_DD` | the report's maximum drawdown | maximum drawdown (the same number as the report's maximum drawdown) |
| `apx.MAX_DD_ADD` | `apx.MAX_DD_ADD_OF(r)` | maximum drawdown (additive; the convention of `max_dd_1e6`) |
| `apx.MAX_DD_DAYS` | `streak(drawdown_nav(r) > 0)` | longest time under water (trading days) |
| `apx.ULCER` | `sqrt(mean(pow(drawdown_nav(r), 2)))` | Ulcer index |
| `apx.SKEW` | `mean(pow(r - mean(r), 3)) / pow(mean(pow(r - mean(r), 2)), 1.5)` | skewness of daily returns |
| `apx.KURT` | `mean(pow(r - mean(r), 4)) / pow(mean(pow(r - mean(r), 2)), 2)` | kurtosis of daily returns (normal = 3) |
| `apx.VAR95` | `quantile(r, 0.05)` | 5% quantile of daily returns |
| `apx.CVAR95` | `mean(filter(r, r <= quantile(r, 0.05)))` | mean of the days at or below the 5% quantile |
| `apx.WORST_DAY` | `min(r)` | worst day |
| `apx.BEST_DAY` | `max(r)` | best day |
| `apx.SHARPE` | `apx.SHARPE_OF(r)` | annualised Sharpe |
| `apx.SORTINO` | `mean(r) / sqrt(mean(pow(min(r, 0), 2))) * sqrt(252)` | Sortino ratio (as in the report) |
| `apx.CALMAR` | `apx.ANN_RET / apx.MAX_DD` | Calmar |
| `apx.ROMAD` | `apx.TOTAL_RET / apx.MAX_DD` | net return over the period / maximum drawdown |
| `apx.OMEGA` | `sum(max(r, 0)) / sum(max(-r, 0))` | Omega (threshold 0) |
| `apx.IR` | `apx.SHARPE_OF(r - m)` | information ratio against the pool's equal-weighted mean |
| `apx.TE` | `apx.VOL_OF(r - m)` | annualised tracking error |
| `apx.EXCESS_ANN` | `apx.ANN_OF(r - m)` | annualised excess return |
| `apx.BETA` | `apx.BETA_OF(r, m)` | beta against the pool's equal-weighted mean |
| `apx.ALPHA_ANN` | `(mean(r) - apx.BETA * mean(m)) * 252` | annualised alpha |
| `apx.CORR_MKT` | `apx.CORR_OF(r, m)` | correlation with the pool's equal-weighted mean |
| `apx.UP_CAPTURE` | `sum(filter(r, m > 0)) / sum(filter(m, m > 0))` | upside capture |
| `apx.DOWN_CAPTURE` | `sum(filter(r, m < 0)) / sum(filter(m, m < 0))` | downside capture |
| `apx.MKT_ANN` | `apx.ANN_OF(m)` | annual return of the pool's equal-weighted mean |
| `apx.IC` | `ic_x1e4 / 1e4` | mean rank IC |
| `apx.IC_T` | `ic_t_x100 / 100` | IC t-value |
| `apx.IC_POS` | `ic_days > 0 ? ic_pos_x1e4 / 1e4 : nan` | share of IC > 0 |
| `apx.ICIR` | `ic_days > 0 ? ic_t_x100 / 100 / sqrt(ic_days) : nan` | ICIR |
| `apx.SPREAD` | `spread_days > 0 ? spread_1e6 / 1e6 : nan` | annualised top 10% - bottom 10% spread |
| `apx.QUINT_RHO` | `quint_days > 0 ? quint_rho_x1e4 / 1e4 : nan` | rank correlation of the quintiles |
| `apx.QUINT_MONO` | `quint_days > 0 ? quint_mono_x1e4 / 1e4 : nan` | share of days on which the quintiles are monotonic |
| `apx.QUINT_TOP` | `quint_days > 0 ? quint_1e6_5 / 1e6 : nan` | annual return of the top quintile |
| `apx.QUINT_BOT` | `quint_days > 0 ? quint_1e6_1 / 1e6 : nan` | annual return of the bottom quintile |
| `apx.IC_1`, `apx.IC_5`, `apx.IC_21`, `apx.IC_63` | `ic_decay_days_N > 0 ? ic_decay_x1e4_N / 1e4 : nan` | IC when held N days |
| `apx.IC_HALF_LIFE` | `ic_decay_days_1 > 0 && ic_decay_x1e4_1 > 0 ? ic_half_life_days : nan` | IC half-life (-1 = not halved within 63 days) |
| `apx.IC_H1`, `apx.IC_H2` | `ic_h1_x1e4 / 1e4`, `ic_h2_x1e4 / 1e4` | IC of the first and second half |
| `apx.TURNOVER` | `turnover_days > 0 ? turnover_bp_avg / 1e4 : nan` | turnover per rebalance |
| `apx.TURNOVER_ANN` | `apx.TURNOVER * 252 / reb_days` | annualised turnover |
| `apx.HOLD_DAYS` | `reb_days / apx.TURNOVER` | average holding days (an estimate) |
| `apx.COST_DRAG_ANN` | `apx.TURNOVER_ANN * 2 * cost_bp / 1e4` | cost drag (an estimate) |
| `apx.AVG_POSITIONS` | `avg_positions_x100 / 100` | average number of holdings |
| `apx.SHARPE_H1`, `apx.SHARPE_H2` | `apx.SHARPE_OF(filter(r, half == 1))`, `… half == 2` | Sharpe of the first and second half |
| `apx.SHARPE_Y_MIN` | `min(filter(apx.YEARLY_SHARPE, apx.YEARLY_DAYS >= 126))` | worst calendar-year Sharpe (years with >= 126 days) |
| `apx.SHARPE_BULL`, `apx.SHARPE_BEAR` | `apx.SHARPE_OF(filter(r, regime > 0))`, `… regime < 0` | Sharpe on bull days and on bear days |
| `apx.RET_BULL_ANN`, `apx.RET_BEAR_ANN` | `apx.ANN_OF(filter(r, regime > 0))`, `… regime < 0` | annual return on bull days and on bear days |
| `apx.DAYS_BULL`, `apx.DAYS_BEAR` | `count(filter(r, regime > 0))`, `… regime < 0` | number of bull days and of bear days |
| `apx.EXCESS_B_ANN` | `apx.ANN_OF(r - b)` | annual excess return over the market benchmark (compounded) |
| `apx.IR_B` | `apx.SHARPE_OF(r - b)` | information ratio against the market benchmark |
| `apx.BETA_B` | `apx.BETA_OF(r, b)` | beta to the market benchmark |
| `apx.YEARS_BEAT_B` | `MEAN(BY_PROD1(year, r) > BY_PROD1(year, b))` | share of calendar years beating the market benchmark |
| `apx.B_ANN` | `apx.ANN_OF(b)` | annual return of the market benchmark over the strategy's own days (dividends reinvested) |
| `apx.BEAT_B_ANN` | `apx.ANN_RET - apx.B_ANN` | strategy annual return minus the benchmark's over the same days (after costs; > 0 = beats the market; not the compounded daily excess of `apx.EXCESS_B_ANN`) |
| `apx.ROLL_SHARPE_MIN` | `MIN(ROLL_MEAN(252, r) / ROLL_STD(252, r)) * SQRT(252)` | minimum rolling one-year annualised Sharpe |
| `apx.ROLL_RET_MIN` | `MIN(EXP(ROLL_SUM(252, LOG(1 + r)))) - 1` | minimum rolling one-year return (compounded): the worst one-year holding period |
| `apx.TOTAL` | `FLOOR((3 * cs_bp + 2 * ts_bp + 3 * novel_bp + 2 * short_bp) / 10)` | combined score (bp): orders the settings when `grid_by` is not given; `total_bp` is its value |

`m` (the pool's equal-weighted mean) and `b` (the market index) are two different yardsticks: `apx.IR`, `apx.BETA`, `apx.EXCESS_ANN` use `m`; the `*_B` metrics use `b`. Whether a strategy beats any yardstick is the same rule: `apx.ANN_RET - apx.ANN_OF(yardstick) > 0` (`apx.BEAT_B_ANN` is the one for `b`).

### 11.3 One threshold, one ordering

Screening has two expressions: the **threshold** (`screen_gate`, kept when true) and the **ordering** (`portfolio.grid_by`, larger first).

**Threshold `screen_gate`**: every candidate must pass. It can use atoms, `apx.*`, `user.*` and check outputs `check.<NAME>.<output>`. When it is evaluated follows from what it refers to: it is split at top-level `&&`; the parts with `check.*` are evaluated after the checks, the rest before the checks (a candidate failing them is not checked). With a top-level `||` and `check.*`, the whole expression is evaluated after the checks (same result, more checks run). Using `check.*` needs that check on (`checks`).

| Plain words | Written |
|---|---|
| at least 60% winning months | `screen_gate="apx.MONTH_WIN >= 0.6"` |
| beats the pool's equal-weighted mean every year | `screen_gate="MIN(BY_SUM(year, r - m)) > 0"` |
| beats the total-return market index over the whole period | `screen_gate="apx.BEAT_B_ANN > 0"` |
| beats the market index in at least 70% of the years | `screen_gate="apx.YEARS_BEAT_B >= 0.7"` |
| at least 5% a year over the market index | `screen_gate="apx.EXCESS_B_ANN >= 0.05"` |
| no one-year holding period loses more than 15% | `screen_gate="apx.ROLL_RET_MIN > -0.15"` |
| Sharpe above 0.5 on both bull and bear days | `screen_gate="apx.SHARPE_BULL > 0.5 && apx.SHARPE_BEAR > 0.5"` |
| the worst year not below -20% | `screen_gate="apx.RET_Y_MIN > -0.2"` |
| at least 10% a year from 2023 | `backtest_from="2023-01-01"`, `screen_gate="apx.ANN_RET >= 0.1"` |
| Sharpe above 1, and keeps 26% or more of the return after a peer swap | `screen_gate="apx.SHARPE > 1 && check.PEER_SWAP.keep_ret_bp >= 2600"`, `checks=["PEER_SWAP"]` |

**Ordering `portfolio.grid_by`**: the combinations of one formula (portfolio settings, thresholds tried with `grid`) are ordered by it, larger first, those without a number last. Without it the combined score orders them (with `grid`, the thresholds compare `apx.SHARPE`). Example: `"grid_by":"apx.CALMAR"`.

**Old forms still work**: `screen_admit` (a threshold after the checks) = appended to `screen_gate`; `screen_pick` (the first setting of a formula that passes) as before. `check.*` cannot be used in `screen_pick` or `grid_by` (`E_CHECK_STAGE`).

- The shorthand `screen=[{"metric": ..., "op": ">=", "value": ...}]` is a list of lower bounds.
- Risk and multiple-testing atoms work in `screen_gate` as usual; in `screen_pick` the parts that read them are evaluated once they are computed (a failing setting moves on to the formula's next one). `round_pbo` only goes in `screen_gate`: that part is evaluated at the end of the round, and failing it moves every formula admitted that round out of the library (e.g. `screen_gate="round_pbo <= 0.5"`).
- Old config keys are shorthands for these expressions: `w_cs`/`w_ts`/`w_novel`/`w_short` = other coefficients for `apx.TOTAL`; `min_alpha_share`, `min_breadth_eff` = the risk part of `screen_pick`; `mt_gate=block` = `screen_pick` plus `mt_dsr >= min_dsr_bp/10000 && mt_ic_padj <= max_ic_padj_bp/10000`; `pbo_block=1` = `screen_gate` plus `round_pbo * 10000 <= max_pbo_bp`.
- The report job (`submit_report`) picks its strategies with the same two expressions: `screen_gate` = which admitted strategies get a report, `grid_by` = the ordering (larger first, those without a number last, ties in library order; default `ic_t_x100`), then the first `top` (0 = all). Values come from each strategy's screening atoms and `apx.*` in `validation.json`; `check.*` is not available. The old key `min_ic_t=X` is shorthand for `screen_gate="ic_t_x100 / 100 >= X"` (use one or the other). Example: `{"library":"L1","top":3,"screen_gate":"ic_t_x100 >= 300 && max_dd_1e6 <= 250000","grid_by":"apx.SHARPE"}`.

### 11.4 User metrics: `apx.metric.v1`

```json
{"schema": "apx.metric.v1", "name": "user.local.ROMAD", "version": 1,
 "description": "年化報酬 ÷ 最大回撤", "inputs": [],
 "body": "apx.ANN_RET / apx.MAX_DD", "unit": "", "direction": "max", "group": "mine"}
```

| Field | Meaning |
|---|---|
| `name` | `user.<author>.NAME` (`apx.*` is reserved) |
| `version` | an integer >= 1; the same name and version cannot change content, a change needs +1 |
| `inputs` | the list of formal input names; empty = a value (`user.local.ROMAD`), non-empty = a function (`apx.SHARPE_OF(x)`) |
| `body` | the definition; may refer to atoms and other definitions, without cycles |
| `direction` | `max` / `min` / `none` |
| `unit`, `group`, `description` | descriptive |

Use: `screen_gate="user.local.ROMAD >= 1"`. Longest time under water: body `streak(drawdown_nav(r) > 0)`.

### 11.5 Checks

A check = re-running the strategy under a changed condition and seeing whether the metrics survive.

**Atomic transforms** (provided by the engine, referred to by check definitions):

| Transform | Meaning |
|---|---|
| a run's `with`: a BOOL expression (for instance `CS_PCT(fund.mktcap) >= 0.7`) | re-rank and re-run only among the stocks where it holds; several items intersect |
| a run's `with`: `PEER_SIDE() == 0`, `PEER_SIDE() == 1` | similar stocks paired and split into two stock pools A and B |
| a run's `with`: `RANDOM_HALF()`, `RANDOM_HALF(k)` | a stock pool of a random half of the stocks (a different k is a different half) |
| a run's `with`: `RETURNS(expression)` | replaces the value of the return; `ret` is the original return (for instance `RETURNS(SUB(ret, MKT_MEAN(ret)))`) |
| a run's `with`: `PEER_SWAP()` | where a stock is selected, the return of the stock closest to it is used instead |
| a run's `with`: `SHUFFLE()`, `SHUFFLE(k, group key)` | each day's returns shuffled across stocks (with a group key: only inside the same group) |
| a run's `with`: `SCORE(expression)` | replaces the score; `score` is the candidate's original score (for instance `SCORE(ROLL_MEAN(5, score))`); the new score is ranked inside this run's pool |
| a run's `with`: `ENTRY_DELAY(n)`, `ENTRY_DELAY_ASOF(n)` | enter n days late (= `SCORE(LAG(n, score))`; `_ASOF` ranks with that earlier day's pool = `SCORE(LAG(n, CS_PCT(score)))`) |
| a run's `with`: `REBALANCE_WHEN(condition)`, `PHASE(n, k)` | rebalance on the days the condition holds (same as `rebalance_when` in a backtest); `REBALANCE_WHEN(apx.EVERY(n))` = every n days after warm-up (= the old key `rebalance_days`); `PHASE(n, k)` = `REBALANCE_WHEN(EVERY(n, k))`, every n days offset by k |
| a run's `with`: `DAYS(weight)`, `DROP_TOP_DAYS(k)`, `DROP_BOTTOM_DAYS(k)`, `ONLY_DAYS(condition)`, `BOOTSTRAP(seed, block)` | which days count: positions unchanged, each day weighted when the metrics are collected (without the best / worst days, only some days, day bootstrap) |
| a run's `with`: `DROP_TOP_HELD(run, k)`, `DROP_BOTTOM_HELD(run, k)`, `DROP_MOST_HELD(run, k)` | re-run without the k stocks that earned the most / lost the most / were held longest in an earlier run (= the pool condition `!CS_TOPK(k, HELD_RET(run))`...); pool conditions can also use `HELD_RET(run)` and `HELD_DAYS(run)` directly |
| a run's `with`: `RETUNE(f)`, `RETUNE_AT(i, f)` | every day count of the formula (or only the i-th) times f/100, re-run: does it hold with slightly different lookbacks; `prog.n_windows` = how many day counts there are |
| a run's `with`: `SCORE_NOISE(k, amp_bp)` | add noise to the rank before picking (= `SCORE(ADD(CS_PCT(score), MUL(amp_bp / 10000, NOISE(k))))`) |
| a run's `with`: `user.<author>.NAME(args...)` | a transform you saved (11.6, "Your own transforms"); expands to the items it defines |
| a run's `with`: `COST(...)`, `HOLD(hold && apx.IN_TOP(p))`, `WEIGHT(...)` | replace the backtest's one-way cost, long threshold, weights (names held, weight key, cap per stock, turnover cap); `cost`, `hold`, `weight` are the originals |
| a run's `with`: `PERIOD(start, end)`, `ROLLING_OOS(is, oos, k)`, `ROLLING_IS(is, oos, k)`, `SPLIT(n, k)` | re-run the whole strategy on one date range only; `k` = `win` runs once per window (walk-forward, segments) |
| old keys `interval`, `windows`, `cost_bp`, `p_long`, `p_short`, `rebalance_days`, `weight_mode`, `max_positions`, `max_weight_bp`, `max_turnover_bp` | still accepted, the same as the two rows above (`p_short` waits for the short-selling rules) |

**Built-in checks** (all are definition files; read the full text with `stock_formula_help topic=check name=...`):

| Name | What it does | Parameters (default) | Pass condition |
|---|---|---|---|
| `apx.PEER_SWAP` | peer swap: each selected stock recomputed with the return of its closest peer | `keep_bp` (0; 0..10,000) | the original has net return > 0 and IC != 0, and the swapped version keeps a given share of both the net return and the IC |
| `apx.PAIR_SPLIT` | pair split: re-ranked inside each of the two sub-pools A and B | `min_spread_1e6` (1) | the IC of A and of B has the sign of the original and is not 0, and the quantile spread >= the bound |
| `apx.RANDOM_HALF` | re-ranked on a random half of the stocks | `min_spread_1e6` (1) | the IC has the sign of the original and is not 0, and the quantile spread >= the bound |
| `apx.COST_RETEST` | cost retest: one run for each of x1 ... xmult of the per-side cost | `mult` (3; 1..4), `min_ret_1e6` (1) | the net return at xmult >= the bound |
| `apx.WALK_FORWARD` | from the last day backwards, one out-of-sample segment every `oos_days` with the in-sample segment right before it; a variant chosen in sample runs out of sample | `is_days` (504), `oos_days` (126), `min_windows` (2), `min_pos_bp` (5,000) | enough segments; the shares of segments with out-of-sample IC > 0 and with net value > 0 are both >= the bound |
| `apx.PLACEBO` | recomputed after shuffling returns across the pool each day | `min_gap_x100` (200) | the original IC t-value exceeds the shuffled one by the bound |
| `apx.SEGMENTS` | the period cut into n equal segments, metrics for each | `n` (4; 2..32), `min_pos_bp` (7,500) | the share of segments with IC > 0 >= the bound |
| `apx.DSR` | Deflated Sharpe Ratio | `min_dsr_bp` (9,500) | DSR >= the bound (not blocking when no ledger is open) |

- By default only the peer swap is on: `checks=["peer_swap"]`; `checks=[]` = screening only.
- A built-in check is a **reserved check function with parameters**, the same grammar as an atom: `checks=["PEER_SWAP(5000)", "SEGMENTS(4, 7500)"]`. Parameters go in the order of the table above, all integers; trailing ones may be left out (= default); no brackets = all defaults. User checks the same way: `checks=["user.local.HALVES(3000)"]` (in the order of its definition's `params`).
- Which ruler the peer swap uses to find the closest stock: the ruler is an **input**, written after the parameters: `checks=["PEER_SWAP(5000, inst_net)"]`; to change only the ruler write `PEER_SWAP(inst_net)`. The ruler is `ret` (daily return, the default), `ret_rank`, `vol_chg`, `amount_chg`, `turnover`, `turnover_rate`, `rev_yoy`, `inst_net`, `size`, `group_rank`, or an expression (e.g. `PEER_SWAP(DIFF(1, LOG(volume)))`: whole-market rank correlation over the 252 days before each period). The pair split and the `PEER_SWAP()`, `PEER_SIDE()` of user checks use the same ruler; the check record's `peer_ruler` names the one used.
- At most 8 checks per strategy.
- The old form `check_params={"PEER_SWAP.keep_bp": 5000, "PEER_SWAP.ruler": "inst_net"}` still works and expands to the same thing (same parameters, same `content_hash`, byte-identical results and records); the same parameter given both ways -> `APX_CONFLICT`.

| Plain words | Written |
|---|---|
| at least half of the return kept after a peer swap | `checks=["PEER_SWAP(5000)"]` |
| peers matched by institutional net buying, at least half kept | `checks=["PEER_SWAP(5000, inst_net)"]` |
| net return still positive with trading costs doubled | `checks=["COST_RETEST(2)"]` |
| 4 segments, IC positive in at least 3 | `checks=["SEGMENTS(4, 7500)"]` |

### 11.6 Building your own check: `apx.check.v1`

The 8 built-in checks are definitions of this very kind (`stock_formula_help topic=check name=PEER_SWAP` shows one in full), and a user's check runs the same way. A check = "re-run a few times with something changed" + "compute a few integers from the re-runs" + "a condition on those integers".

```json
{"schema": "apx.check.v1", "name": "user.local.HALVES", "version": 1,
 "description": "前後兩半 IC 同號",
 "runs": [{"id": "seg", "with": "SPLIT(2, win)"}],
 "outputs": {"seg_ic": "seg.ic_days > 0 ? seg.ic_x1e4 : 0", "lo": "AMIN(seg_ic)", "hi": "AMAX(seg_ic)"},
 "pass": "lo > 0 || hi < 0"}
```

| Field | Meaning |
|---|---|
| `runs` | the re-runs and what each one changes (not written = the original); at most 8 |
| `outputs` | numbers (integer or ratio) computed from the metrics of the runs; at most 16. These are the "screening metrics after the check", read in the threshold `screen_gate` as `check.<NAME>.<output>` |
| `pass` | a condition over `outputs` (and `$params`, `env.*`) |
| `params` | parameters: `{"name","default","min","max"}`, written `$name` in expressions |

**Step 1: what each re-run changes (`runs`).** Each item is one re-run; `id` is required (expressions refer to it). What can change:

| Key | Value | Meaning |
|---|---|---|
| `with` | a comma-separated list of transforms, one transform per item (table below) | the stock pool, returns, score, rebalance days, days, portfolio settings and date range are all changed here; not written = original |
| `vary` | `{"k": [1, 2, 3]}` or `{"k": {"from": 1, "to": "$mult"}}` | expands the re-run into several (several variables allowed, one member per combination; members = window segments x the value counts of the variables, at most 4,096, more is an error. Each different transform also takes one slot of the round, see "One round holds at most" below; only portfolio arguments (`COST`, portfolio key names) take no slot and can reach 4,096); `with` reads it as `RUN.k`. A variable named after an old portfolio key (`p_long`, `rebalance_days`...) is applied directly |
| old keys | `interval` (`full`, `window.is`, `window.oos`, `window.seg`, `{"begin","end"}`), `cost_bp`, `p_long`, `p_short`, `rebalance_days`, `weight_mode`, `max_positions`, `max_weight_bp`, `max_turnover_bp`; the top-level `windows` | still accepted, aliases of the `with` forms: `window.is` / `window.oos` = `ROLLING_IS` / `ROLLING_OOS(is, oos, win)`, `window.seg` = `SPLIT(n, win)`, `{"begin","end"}` = `PERIOD(begin, end)`, `"cost_bp":"prog.cost_bp * 3"` = `COST(cost * 3)`, `"p_long":"170"` = `HOLD(hold && apx.IN_TOP(170))`, `"rebalance_days":"21"` = `REBALANCE_WHEN(apx.EVERY(21))`... (the same definition) |

Each item of `with` (commas inside parentheses do not separate items):

| Written as | Changes | Meaning |
|---|---|---|
| a BOOL expression, or `UNIVERSE(expression)` | the stock pool | re-rank and re-run only among the stocks where it is true. Same grammar and units as `universe=` in chapter 8; cross-sectional operators (`CS_PCT`...) see the original pool, computed once. Several items in one run intersect |
| `PEER_SIDE() == 0`, `PEER_SIDE() == 1` | the stock pool | similar stocks paired two by two, each pair split into groups A and B. `PEER_SIDE(ruler)` measures similarity with that expression (none = the `PEER_SWAP.ruler` one) |
| `RANDOM_HALF()`, `RANDOM_HALF(k)` | the stock pool | a random half of the stocks (fixed across periods); k numbers the draw, a different k is a different half |
| `RETURNS(expression)` | the returns | the expression must give a RATIO; `ret` is the original return. It is applied to the full-day, overnight and intraday return each; the intraday price path for touch exits stays as it was |
| `PEER_SWAP()`, `PEER_SWAP(ruler)` | the returns | when a stock is picked, the whole bar of its closest peer on that day is used instead; ranks and holdings unchanged. Same as `REMAP(PEER_IDX())` |
| `PEER_SWAP(lookback, every, step, rank, min_corr_bp, input[, group key])` | the returns | how peers are found, written out in full (`PEER_IDX` and `PEER_SIDE` take the same arguments): correlation of `input` over the `lookback` days before each period, one point every `step` days (`rank` 0 = return correlation, 1 = rank correlation; `ret` = daily return), re-paired every `every` days; a correlation below `min_corr_bp` is not a peer (`-10000` = no floor); no `group key` = search the whole market, with one = only among stocks with the same key on the first day of the period (`PARENT_ID(1)` same industry, `CORR_GROUP(...)` correlation clusters). The default ruler = `PEER_SWAP(252, 126, 1, 0, 3000, ret)` |
| `SHUFFLE()`, `SHUFFLE(k)`, `SHUFFLE(k, group key)` | the returns | returns shuffled across stocks each day (cross-sectional signal gone, market and distribution unchanged). k numbers the draw; with a group key only stocks with the same key are shuffled among themselves. Same as `REMAP(SHUFFLE_IDX(...))` |
| `REMAP(index expression)` | the returns | each cell takes the whole bar, on the same day, of the stock the expression points at. `PEER_IDX()` and `SHUFFLE_IDX()` give a stock index (an integer); cells where the expression is invalid keep their own. To change only some cells: `REMAP(MASK(condition, PEER_IDX()))` |
| `SCORE(expression)` | the score | `score` is the candidate's original score (per day, per stock); the new score is ranked inside this run's pool and picked the same way. Same grammar as score formulas (upper case, fixed point). `SCORE(score)` = no change |
| `ENTRY_DELAY(n)` | the score | enter n days late: the score from n days ago, ranked in today's pool. Same as `SCORE(LAG(n, score))` |
| `ENTRY_DELAY_ASOF(n)` | the score | enter n days late with the ranks of that earlier day's pool. Same as `SCORE(LAG(n, CS_PCT(score)))` |
| `REBALANCE_WHEN(condition)` | rebalance days | rebalance on the days the condition holds. The condition must be market-wide (month start, `EVERY(n, k)`, a market condition...); same rule and same numbers as `rebalance_when` in a backtest; exposure stays as it was. `REBALANCE_WHEN(apx.EVERY(n))` = every n days after warm-up, the same numbers as the old key `rebalance_days=n` (n may be a `$param` or `RUN.var`) |
| `PHASE(n, k)` | rebalance days | every n days, offset by k. Same as `REBALANCE_WHEN(EVERY(n, k))` (`EVERY` counts from the panel's first day, 0-based) |
| `DAYS(weight)` | the days | how many times each day counts: the weight is an 11.1 screening-language expression (lower case, real numbers) over **this re-run's own** daily series `r`, `m`, `year`, `month`, `regime`, `half`, `day`, `b`; 0 = not counted, 1 = once, 2 = twice (truncated, at most 32); a BOOL is 0/1; warm-up days and days dropped by `FILTER` are not counted. On a day whose weight is invalid (unknown), whether and how often it counts is unknown, so the recollected fields are invalid: net return if any day is unknown, the IC and spread fields if any rebalance day is unknown; to count only days with a known weight write `FILTER(w, HAS(w))`. Positions and holdings run as they are; only the collection changes |
| `DROP_TOP_DAYS(k)` | the days | without this re-run's k best daily net returns (= `DAYS(rank_desc(r) > k)`) |
| `DROP_BOTTOM_DAYS(k)` | the days | without the k worst days (= `DAYS(rank_desc(-r) > k)`) |
| `ONLY_DAYS(condition)` | the days | only the days where the condition holds (= `DAYS(condition)`), e.g. `ONLY_DAYS(regime > 0)` for bull days |
| `BOOTSTRAP(seed, block)` | the days | block bootstrap of the days (= `DAYS(boot(seed, block))`); `vary` the seed for several draws and fold them with `PCT` |
| `DROP_TOP_HELD(run, k)` | the stock pool | re-run without the k stocks that earned the most in run: the pool changes, so ranks are recomputed and other stocks get bought. Same as `!CS_TOPK(k, HELD_RET(run))` |
| `DROP_BOTTOM_HELD(run, k)` | the stock pool | re-run without the k stocks that lost the most in run. Same as `!CS_TOPK(k, NEG(HELD_RET(run)))` |
| `DROP_MOST_HELD(run, k)` | the stock pool | re-run without the k stocks held longest in run. Same as `!CS_TOPK(k, HELD_DAYS(run))` |
| `HELD_RET(run)`, `HELD_DAYS(run)` | values inside a pool condition | per stock in run: the sum of returns while held (every piece that run's backtest itself booked on the stock: full days while held, the overnight or intraday part on next-open trade days, the touch-exit part; RATIO) and the number of days held at the close (COUNT). One value per stock, the same every day, 0 if never held; not weighted |
| `RETUNE(f)` | the score | the same formula with every day-count argument times f/100 (floored), ranked in this run's pool. Day-count arguments = the arguments marked N in chapter 4 (days, periods, the period count of `RWIN` / `EWIN` / `RWIN_PCT`) and the window W of the algorithmic operators; the new value is clamped to that operator's legal range (e.g. `ROLL_STD` 2..1,260, `EMA` 2..84, the N of `RET_SKIP` must exceed M) and the number of clamps goes to the check record's `clamped`. f 1..1000; `RETUNE(100)` = unchanged |
| `RETUNE_AT(i, f)` | the score | only the i-th day-count argument (from 0, in the formula's text order; a sub-expression used several times counts once). `prog.n_windows` = how many day-count arguments the formula has |
| `SCORE_NOISE(k, amp_bp)` | the score | add noise to the rank before picking, same as `SCORE(ADD(CS_PCT(score), MUL(amp_bp / 10000, NOISE(k))))`. The noise depends only on (k, the day, the stock), the same rule as `RANDOM_HALF`, `SHUFFLE`, `BOOTSTRAP`: a re-run gives the same bits, whichever check or run it is written in, and a different k is another draw |
| `COST(expr)` | cost | the one-way cost (bp) becomes the expression's value; `cost` is the original, e.g. `COST(cost * 3)`, `COST(cost * c.k)` |
| `HOLD(hold && apx.IN_TOP(p))` | long threshold | buy only at percentile rank >= p/200 (180 = top 10%); `hold` is the original entry condition. p may be `RUN.var` or `WINNER(...)` |
| `WEIGHT(expr)` | weights | `weight` is the original. Takes `apx.SHARE(apx.PICK(N, false[, 0[, turnover cap]]), key)`: N = names held (`prog.max_positions` keeps it), the turnover cap is a ratio (`0.3`); key `1` equal, `apx.RANK_KEY()` by rank, `INV_VOL_KEY(20, RET(1, close))` inverse volatility, `weight` as before; `apx.CAP(cap, ...)` caps each stock (a ratio, `0.1` = 10%). E.g. `WEIGHT(apx.SHARE(apx.PICK(10, false), 1))`, `WEIGHT(apx.CAP(0.1, weight))` |
| `PERIOD(start, end)`, `PERIOD(start, end, windows)` | date range | the whole strategy re-run on day indices [start, end), positions start from cash at the start (not just the metrics of that range; for that write `ONLY_DAYS(day >= ...)`). May use `env.first` (first day after warm-up), `env.n_days`, `$param` and this run's `vary` variable; bounds that use `win` (window number, from 0) run once per window, the number of windows is the third argument (`n_windows`, at most 32) |
| `ROLLING_OOS(is, oos, k[, mx])`, `ROLLING_IS(is, oos, k[, mx])` | date range | walk-forward out-of-sample / in-sample segments: cut every oos days back from the last day with is days in sample right before, the in-sample start >= warm-up, the earliest is k = 0, more than mx (default 32) keeps the latest. k = `win` runs once per window (`WINNER` reads the same window); `RUN.k` with `vary {"k":{"from":0,"to":"n_windows - 1"}}` expands by vary; a number runs that segment only. Number of windows = `ROLLING_N(is, oos[, mx])` |
| `SPLIT(n, k[, mx])` | date range | segment k of [warm-up, last day) cut into n equal parts; k = `win` runs once per segment. Number of segments = `SPLIT_N(n[, mx])` |
| `user.<author>.NAME(args...)`, `user.<author>.NAME@version(...)` | as defined | your own transform (below, "Your own transforms"): its `body` with the arguments substituted expands to those items; the same transform, and the same `content_hash`, as writing the expansion; no version = the latest |

- One run may carry several stock-pool conditions and at most one return transform (two kinds: write two runs).
- Score transforms may be stacked and apply in the order written (each one's `score` is the previous result); at most one rebalance-day transform per run. `score` is only allowed inside `SCORE(...)`.
- Day transforms (the `DAYS` family) may be written several times; the weights multiply, and the same run may also change the pool or the returns. A run with a day transform reads only the fields that are recollected with the weights: `net_ret_1e6` (sum of weight x daily net return), `ic_x1e4`, `ic_t_x100`, `ic_pos_x1e4`, `ic_days` (the rebalance-day IC, weighted), `spread_1e6`, `spread_days`; the other fields (`max_dd_1e6` included) give `E_DAYS_FIELD` (read them from a run without a day transform; a weighted drawdown is the screening expression `MAX(DRAWDOWN(r * weight))`). All weights 1 (e.g. `DAYS(1)`, `DROP_TOP_DAYS(0)`) gives the original run's value in these fields.
- The weight is in the lower-case screening language; a score formula in upper case (`DAYS(TOP_K(R, 5))`) or a per-strategy scalar (`ic_x1e4`) gives `E_CHECK_LANG`. A reserved function and its expansion are the same check identity.
- Expressions may use `$param` and this run's `vary` variable (`RUN.var`); they are replaced by their integer values first. The `vary` values must then be numbers or `$param`; either end of a range may also use `prog.n_windows` (e.g. `{"from":0,"to":"prog.n_windows - 1"}`: the number of members follows each formula's number of day counts).
- Re-tuning (`RETUNE`, `RETUNE_AT`) is a score transform that always applies first within a run (it changes the formula itself), then the `SCORE` family; writing it before or after is the same transform. When the formula has too few day counts (none, or `RETUNE_AT`'s i ≥ `prog.n_windows`) that formula fails this check and the record says `"error":"E_FORMULA_NO_WINDOW"`; a check that uses the `RETUNE` family also records `clamped`. A `vary` range up to `prog.n_windows` reserves 16 score transforms (the limit); to open it together with other score transforms keep it smaller, e.g. `"to":"MIN2(prog.n_windows, 4) - 1"`.
- One round holds at most 2 different stock pools, 2 different return transforms, 16 different score transforms, 32 different rebalance-day transforms and 255 different day weightings (`BOOTSTRAP` and the like); more returns `E_RES_LIMIT` (a transform used by several runs or checks counts once; `ENTRY_DELAY(2)` and `SCORE(LAG(2, score))` are the same one).
- Referring to an earlier run (`HELD_RET`, `HELD_DAYS`, `DROP_*_HELD`): the referred run has to finish first, so the run that refers to it moves to the next stage by itself (at most 4 stages); the referred run cannot have `vary` or windows and a run cannot refer to itself (`E_RUN_REF_SELF`). The pool is computed once per candidate, at most 16 different ones per round.
- The old keys `universe` and `ret` and the old names (`universe.peer_a`, `universe.peer_b`, `universe.random_half`, `ret.peer_swap`, `ret.shuffle`) are still accepted; they mean the same as writing them in `with`.
- Portfolio settings (`COST`, `HOLD`, `WEIGHT`) change each setting once per run, and the date range once per run (twice gives `E_WITH_TWICE`). `WEIGHT` and `HOLD` arguments are read like the backtest `weight` and `hold` (the same shape table): the `COST` expression, the `apx.PICK` count and the `apx.IN_TOP` threshold may be check expressions (`$param`, `RUN.var`, `prog.FIELD`, `WINNER(...)`); the cap per stock and the turnover cap are numbers.
- The date ranges that use `win` in one check share one number of windows (in-sample and out-of-sample with the same is, oos; otherwise `E_WINDOWS`); `n_windows` is that number. `ROLLING_N` and `SPLIT_N` are counts (integers) for `vary`'s `to` or `PERIOD`'s third argument, not transforms.

| In plain words | The run |
|---|---|
| re-run on large caps only | `{"id":"big","with":"CS_PCT(fund.mktcap) >= 0.7"}` |
| re-run without financials | `{"id":"x","with":"!(industry in {17})"}` |
| re-run on large caps that are above their one-year average | `{"id":"bu","with":"CS_PCT(fund.mktcap) >= 0.7, close > ROLL_MEAN(250, close)"}` |
| still profitable after taking out the market average | `{"id":"ex","with":"RETURNS(SUB(ret, MKT_MEAN(ret)))"}`; `pass` is `ex.net_ret_1e6 > 0` |
| still profitable with daily moves capped at ±10% (not made by a few big days) | `{"id":"c","with":"RETURNS(MIN2(MAX2(ret, -0.1), 0.1))"}` |
| returns shuffled inside market-cap deciles; the original IC t must be 2 higher | `{"id":"sh","with":"SHUFFLE(0, CS_QCUT(10, fund.mktcap))"}`; output `"gap":"base.ic_t_x100 - sh.ic_t_x100"`, `pass` is `gap >= 200` |
| two different random halves, both with the IC sign of the original | `{"id":"h","vary":{"s":[1,2]},"with":"RANDOM_HALF(h.s)"}`; `pass` is `COUNT(SIGN(h.ic_x1e4) == SIGN(base.ic_x1e4)) == 2` |
| large caps only, with the returns of the closest peers | `{"id":"bs","with":"CS_PCT(fund.mktcap) >= 0.7, PEER_SWAP()"}` |
| peer swap with peers matched on volume changes | `{"id":"sw","with":"PEER_SWAP(DIFF(1, LOG(volume)))"}` |
| enter 1 to 3 days late; still profitable at 3 days | `{"id":"late","vary":{"n":[1,2,3]},"with":"ENTRY_DELAY(late.n)"}`; `pass` is `LAST(late.net_ret_1e6) > 0` |
| stable when rebalance days shift (monthly, offsets 0 to 15, at least 80% profitable) | `{"id":"ph","vary":{"k":{"from":0,"to":15}},"with":"PHASE(21, ph.k)"}`; `pass` is `COUNT(ph.net_ret_1e6 > 0) * 10 >= 16 * 8` |
| still profitable when rebalancing on the first trading day of each month | `{"id":"mo","with":"REBALANCE_WHEN(MONTH_START())"}`; `pass` is `mo.net_ret_1e6 > 0` |
| score smoothed over 5 days keeps the IC sign | `{"id":"sm","with":"SCORE(ROLL_MEAN(5, score))"}`; `pass` is `SIGN(sm.ic_x1e4) == SIGN(base.ic_x1e4)` |
| large caps only and 2 days late, together | `{"id":"bl","with":"CS_PCT(fund.mktcap) >= 0.7, ENTRY_DELAY(2)"}` |
| does it still earn without its best 5 days | `{"id":"nobest","with":"DROP_TOP_DAYS(5)"}`; `pass` is `nobest.net_ret_1e6 > 0` |
| without the worst 5 days, how much more it earns | `{"id":"noworst","with":"DROP_BOTTOM_DAYS(5)"}`; output `"lift":"noworst.net_ret_1e6 - base.net_ret_1e6"` |
| on bear days only, the IC must still be positive | `{"id":"bear","with":"ONLY_DAYS(regime < 0)"}`; `pass` is `bear.ic_days > 0 && bear.ic_x1e4 > 0` |
| 16 day bootstraps, 95% of them profitable | `{"id":"bs","vary":{"s":{"from":1,"to":16}},"with":"BOOTSTRAP(bs.s, 21)"}`; output `"p05":"PCT(bs.net_ret_1e6, 500)"`, `pass` is `p05 > 0` |
| still profitable without the 5 stocks that earned the most | `{"id":"base"},{"id":"cut","with":"DROP_TOP_HELD(base, 5)"}`; `pass` is `cut.net_ret_1e6 > 0` |
| IC keeps its sign without the 3 stocks held longest | `{"id":"base"},{"id":"m","with":"DROP_MOST_HELD(base, 3)"}`; `pass` is `SIGN(m.ic_x1e4) == SIGN(base.ic_x1e4)` |
| day counts times 0.9 and 1.1, each keeps 70% of the net return | `{"id":"base"},{"id":"j","vary":{"f":[90,110]},"with":"RETUNE(j.f)"}`; output `"worst":"base.net_ret_1e6 > 0 ? AMIN(KEEP_BP(j.net_ret_1e6, base.net_ret_1e6)) : 0"`, `pass` is `worst >= 7000` |
| one day count at a time (times 0.9), each still earns | `{"id":"q","vary":{"i":{"from":0,"to":"prog.n_windows - 1"}},"with":"RETUNE_AT(q.i, 90)"}`; `pass` is `ALL(q.net_ret_1e6 > 0)` |
| 8 draws of 5% score noise, at least 7 still earn | `{"id":"nz","vary":{"s":{"from":1,"to":8}},"with":"SCORE_NOISE(nz.s, 500)"}`; `pass` is `COUNT(nz.net_ret_1e6 > 0) >= 7` |
| "large caps only and 2 days late, together" saved as your own transform, days late as an argument | `{"id":"bl","vary":{"n":[1,2,3]},"with":"user.local.BIG_LATE(bl.n)"}` (definition file below) |

**Your own transforms (`apx.noun.v1`, `out_unit` `TRANSFORM`) = your own reserved functions.** Save a set of transforms you use often under one name, in the same kind of definition file as the user functions of chapter 12; the official `ENTRY_DELAY`, `ENTRY_DELAY_ASOF`, `PHASE`, `DROP_TOP_DAYS`, `DROP_BOTTOM_DAYS`, `ONLY_DAYS`, `BOOTSTRAP`, `DROP_TOP_HELD`, `DROP_BOTTOM_HELD`, `DROP_MOST_HELD` and `SCORE_NOISE` are definitions of the same kind (e.g. the `body` of `ENTRY_DELAY` is `SCORE(LAG($n, score))`, that of `DROP_TOP_DAYS` is `DAYS(rank_desc(r) > $k)`), expanded the same way with the same `content_hash` rule:

```json
{"schema": "apx.noun.v1", "name": "user.local.BIG_LATE", "version": 1, "out_unit": "TRANSFORM",
 "description": "large caps only, entering n days late",
 "params": [{"name": "n", "default": 1}],
 "body": "CS_PCT(fund.mktcap) >= 0.7, ENTRY_DELAY($n)"}
```

- `body` = one or more `with` items (comma-separated); parameters are written `$name` (only those declared in its own `params`); it may use other user transforms (at most 8 levels deep). No `inputs`.
- Write it as a whole `with` item: `user.local.BIG_LATE(2)`; arguments follow the order of `params`, trailing ones may be left out (= default); this run's `vary` variable may be an argument (`user.local.BIG_LATE(bl.n)`, one transform per value). Pin a version with `user.local.BIG_LATE@1(2)`.
- Validate and install it like a user function; when a backtest or search uses a check that calls it, the saved transforms are sent to the engine with it and do not count towards the 8 checks.
- Error codes: `E_XFN` (not `TRANSFORM`, no `body`, has `inputs`, nested more than 8 levels), a wrong number of arguments returns `E_WITH_TYPE` (with the call form, as for the official reserved functions), `E_PARAM` (`body` uses an undeclared `$name`), a missing version returns `E_WITH_UNKNOWN`; every item of `body` is checked by the `with` rules (the error codes above), each with its position and a fix you can paste.

**The identity of a check, and the check ledger.**

- A check's identity = name@version + the `content_hash` of the normalised definition + parameter values + the ruler (only when it uses `PEER_SWAP()`, `PEER_SIDE()` or `PEER_IDX()` without a ruler) + the seed `val_alt_seed` + the name@version of every user transform it uses.
- In the check record (the `checks` of each candidate in the library's `validation.json`), a user check, or one whose transforms take arguments, has three more fields after `name` and `version`: `id` (the identity, 16 hex digits), `uses` (the name@version of the user transforms used) and `seed`. The records of the 8 built-ins are byte-for-byte as before.
- `env.n_checks_tried` = how many different checks this candidate has been through in this library before this search; `env.n_checks_pass` = how many of them it passed. The same identity run several times counts once, and passing once counts as passed; built-ins count by name@version; invalid when there is no library (the quantity does not exist). No built-in check reads them.
- The re-runs of a check are not trials and do not add to the multiple-testing trial count. To discount "try many checks on one strategy and keep it if one passes", write it into `pass` yourself (e.g. `tried <= 3 && ...`).

**Step 2: to cut time, write the date range in `with`.** `ROLLING_IS(504, 126, win)`, `ROLLING_OOS(504, 126, win)`: from the last day backwards every 126 days, the 504-day in-sample part right before, one run per window; `SPLIT(4, win)`: 4 equal segments. The old form (top-level `windows` plus `interval`) is still accepted.

**Step 3: compute outputs (`outputs`).** `RUN.FIELD` reads a metric of one re-run; field names as the scalar atoms of 11.1 (`net_ret_1e6`, `max_dd_1e6`, `ic_x1e4`, `ic_t_x100`, `ic_days`, `spread_1e6`, `quint_1e6_1`...`5`, `turnover_bp_avg`...). A re-run with windows or `vary` is a series of values and needs an aggregate to become one number; an output may only refer to outputs written before it; names must not clash with field names, run ids, parameter names, `env` or `prog`.

**Step 4: write the condition (`pass`).**

**Step 5: validate, install, use.** `stock_formula_help topic=validate document={...}` validates (every error returned) -> `my_stock_strategies_change action=install document={...}` stores it -> a backtest or search with `checks=["user.local.HALVES"]`; parameters go in the brackets in the order of the definition's `params`: `checks=["user.local.HALVES(3000)"]`. At most 8 checks per strategy.

Check expression values are integers or ratios: division or % by 0 -> invalid (the output is recorded as null and the check fails); comparisons and logic return 0 / 1.

| Kind | What can be written |
|---|---|
| Values | numbers, `$param`, `env.NAME`, `prog.FIELD`, `RUN.FIELD` (a metric of one run; field names as the scalar atoms of 11.1), `RUN.VAR` (a variable of `vary`) |
| Operators | `+ - * / %`, comparisons, `&& \|\| !`, `?:` |
| Functions | `ABS(x)`, `MIN2(a, b)`, `MAX2(a, b)` (same names as score formulas), `SIGN(x)` (-1 / 0 / 1), `KEEP_BP(variant, original)` (variant / original, in ten-thousandths, e.g. `KEEP_BP(swap.net_ret_1e6, base.net_ret_1e6)`; original 0 -> invalid), `HAS(x)` (1 with a value, 0 when invalid; the only function that looks at invalid, as in screening), `FILTER(x, condition)` (as in screening: condition known and true -> member, value = x; known and false -> not a member, aggregates skip it; condition invalid -> member with an unknown value) |
| Aggregates | `COUNT` `SUM` `ALL` `ANY` `AMIN` `AMAX` (the window dimension first, else the `vary` dimension), `FIRST` `LAST`, `NTH(expr, k)` (the k-th of `vary`, from 0), `PCT(expr, q_bp)` (percentile over `vary`, smallest first: the floor((n - 1) x q_bp / 10,000)-th, from 0; q_bp clamped to 0..10,000, e.g. `PCT(bs.net_ret_1e6, 500)` = the 5th percentile), `WINNER(RUN, FIELD, VAR)` (the variable value of the member with the largest FIELD in that run; used in another run's portfolio setting = pick in-sample, use out-of-sample, e.g. `HOLD(hold && apx.IN_TOP(WINNER(is, net_ret_1e6, p)))`; invalid if any member's FIELD is invalid), `LEN(x)` (how many members, regardless of value; how many `vary` values: `LEN(c.k)`), `n_windows`. A member with an invalid value makes the result invalid unless the answer does not depend on it: `FIRST` `LAST` `NTH` look at that one member only, `ALL` is 0 once one member is known false, `ANY` is 1 once one is known true; no members: `COUNT` `SUM` `LEN` 0, `ALL` 1, `ANY` 0, the rest invalid. Only the known values: `MEAN(FILTER(x, HAS(x)))`; let missing values pass: `ALL(!HAS(x) \|\| x > 0)` |
| Old forms | lower-case names are accepted and identified by the upper-case form (same definition, same `content_hash`): `min(a, b)` = `MIN2(a, b)`, `min(x)` = `AMIN(x)`, `keep_bp(...)` = `KEEP_BP(...)`...; `MIN(a, b, c)` answers with the fix `MIN2(MIN2(a, b), c)` |
| `env` | `n_days` (panel days), `first` (first day after warm-up), `n_symbols`, `peer_nn_corr_bp` (average peer correlation), `peer_fallback_bp`, `peer_paired_bp`, `mt_dsr_bp`, `mt_ic_padj_bp`, `mt_n_eff_x100`, `ic_t_x100`, `net_ret_1e6`, `n_checks_tried`, `n_checks_pass` (the check ledger, above). A quantity that does not exist (`mt_*` without the ledger, the check ledger without a library) is invalid |

| Plain words | `runs` -> `outputs` -> `pass` |
|---|---|
| IC must also be positive over the last year | `[{"id":"late","with":"PERIOD(env.n_days - 252, env.n_days)"}]` -> `{"late_ic":"late.ic_days > 0 ? late.ic_x1e4 : 0"}` -> `late_ic > 0` |
| profitable with both weekly and monthly rebalancing | `[{"id":"rb","vary":{"rebalance_days":[5,21]}}]` -> `{"worst":"AMIN(rb.net_ret_1e6)"}` -> `worst > 0` |
| profitable on a random half of the stocks with costs tripled | `[{"id":"h","with":"RANDOM_HALF(), COST(cost * 3)"}]` -> `{"r":"h.net_ret_1e6"}` -> `r > 0` |
| at most 3 different checks tried on this strategy before, and IC positive over the last year too | `[{"id":"late","with":"PERIOD(env.n_days - 252, env.n_days)"}]` -> `{"tried":"env.n_checks_tried","late_ic":"late.ic_days > 0 ? late.ic_x1e4 : 0"}` -> `tried <= 3 && late_ic > 0` |
| profitable with only 10 holdings, max drawdown <= 40% | `[{"id":"n10","with":"WEIGHT(apx.SHARE(apx.PICK(10, false), weight))"}]` -> `{"r":"n10.net_ret_1e6","dd_bp":"n10.max_dd_1e6 / 100"}` -> `r > 0 && dd_bp <= 4000` |
| profitable with a 10% cap per stock and a 30% turnover cap | `[{"id":"cap","with":"WEIGHT(apx.CAP(0.1, apx.SHARE(apx.PICK(prog.max_positions, false, 0, 0.3), weight)))"}]` -> `{"r":"cap.net_ret_1e6"}` -> `r > 0` |
| pick the long threshold in-sample, confirm out-of-sample, both windows profitable | `[{"id":"is","vary":{"p":[170,180,190]},"with":"ROLLING_IS(504, 252, win, 2), HOLD(hold && apx.IN_TOP(is.p))"},{"id":"oos","with":"ROLLING_OOS(504, 252, win, 2), HOLD(hold && apx.IN_TOP(WINNER(is, net_ret_1e6, p)))"}]` -> `{"oos_ret":"oos.net_ret_1e6","ok":"ALL(oos_ret > 0)"}` -> `n_windows >= 2 && ok == 1` |
| 4 segments over the whole period, IC positive in each | `[{"id":"seg","with":"SPLIT(4, win)"}]` -> `{"ic":"seg.ic_days > 0 ? seg.ic_x1e4 : 0","ok":"ALL(ic > 0)"}` -> `n_windows == 4 && ok == 1` |

---

## 12. User functions

### 12.1 The definition file `apx.noun.v1`

A user function is a JSON definition file, not code. Corpus R17, "build a 14-day RSI yourself":

```json
{"schema": "apx.noun.v1", "requires": "apx-std>=1.0",
 "name": "user.local.RSI", "version": 1, "description": "Wilder RSI（α = 1/14）",
 "inputs": [{"name": "x", "unit": "PRICE"}],
 "params": [{"name": "k", "default": 42, "min": 0, "max": 756}],
 "body": "RATIO(SMOOTH($k, 0.071429:RATIO, MAX2(DIFF(1, x), 0:PRICE)), ADD(SMOOTH($k, 0.071429:RATIO, MAX2(DIFF(1, x), 0:PRICE)), SMOOTH($k, 0.071429:RATIO, MAX2(NEG(DIFF(1, x)), 0:PRICE))))",
 "out_unit": "RATIO"}
```

Use: `NEG(user.local.RSI(42, close))`.

| Field | Rule |
|---|---|
| `name` | `user.<author in lower case>.<NAME in upper case>` |
| `version` | an integer >= 1. `name@version` is immutable: installing the same content again does nothing, different content needs a new version number |
| `inputs` | formal inputs: `name` and `unit` (a unit name, or `X` = any numeric unit). May be empty |
| `params` | integer parameters: `name`, `default`, optionally `min`, `max`, `grid` (the values a search tries) |
| `body` | the definition: the grammar of chapter 2; `$param` stands for a parameter and a formal input name for the sub-tree given as input |
| `out_unit` | the unit of the root after expansion; must match the real one |
| `description` | descriptive |

### 12.2 Writing and calling

| Tool | What it does |
|---|---|
| `stock_formula_help topic=define` | the format with an example |
| `stock_formula_help topic=validate document={...}` | checks only, stores nothing: returns the expanded atom formula, warm-up and level; or every error (code, field, position, hint, closest names) |
| `my_stock_strategies_change action=install document={...}` | stores it |
| `stock_formula_help topic=definitions` / `get` | lists, reads back |

Call: `user.<author>.NAME(params..., inputs...)`.

- Parameters first, inputs after. The leading arguments (number of arguments - number of inputs) are parameters, in declaration order; missing ones take their default.
- Parameter values must lie within the declared `min`..`max`.
- The unit of an input must equal the declared one (an input declared MONEY is the old spelling and counts as `X`: any numeric unit); a constant input is read in the declared unit.
- A version may be given: `user.local.RSI@1(42, close)`. Without one = the latest version; when one user function refers to another, the version referred to is recorded at installation, so a later version of that function changes nothing.
- User functions may refer to other user functions, without cycles; expansion is at most 128 levels deep.

### 12.3 Expansion

A user function is expanded at compile time: parameters and inputs are substituted into the definition, giving one tree of atoms. Equal sub-expressions are one node.

- The expanded size counts toward the node limit of a formula (chapter 13).
- After expansion it is the same tree as writing the atoms, so the three hardware targets agree; the result library records each `name@version` used.

### 12.4 User functions for conditions

A user function with `out_unit` BOOL, no `inputs`, and a body that uses code columns or `in {...}` is a **condition noun**: it works only in conditions (stock pool, `hold`) and cannot go into a score formula.

Example: store a seed basket with the body `GSUM(CORR_GROUP(RATIO(close, LAG(1, close))), symbol in {"2330", "2382", "3231"}) > 0`; afterwards write `universe="user.<you>.NAME()"` or `universe="!FILL(user.<you>.NAME(), 0:BOOL)"`.

### 12.5 Searching with a user function

`trees_from="nouns"`: the parameter grid x input bindings is enumerated, one tree per combination.

```json
{"trees_from": "nouns",
 "nouns": [{"name": "user.me.NEAR_HIGH", "bind": {"x": "close"}, "grid": {"n": [60, 120]}}]}
```

That is 2 trees. Without `bind`, every column of the same unit is tried.

---

## 13. Limits

These are technical limits, not strategy rules.

### 13.1 Formula size

| Item | Limit |
|---|---|
| nodes per formula | 128 |
| depth | 128 |
| parameterised operators | 128 |
| inputs per operator | 2 |
| nesting of the text (a call, a parenthesis, a unary minus and a ternary each count one level) | 128 levels |
| user-function expansion depth | 128 levels |

- Nodes are counted **after expansion to atoms**: user functions and the reserved functions expanded at text level (5.2) are expanded; a reserved function marked R in chapter 4 (`RET`, `CS_PCT`...) is one node. Columns and constants count; equal sub-expressions count once.
- For instance `MKT_MEAN(column)` is 4 nodes after expansion and `EW_INDEX(close)` about 17.
- Every strategy carries complexity figures (`complexity`: nodes, depth, parameterised operators) in lists, reports and comparisons.
- Over the limit the answer is `APX_ENGINE_LIMIT`, and the message carries the actual node count.

### 13.2 Windows and parameters

| Limit | Operators |
|---|---|
| 1,260 (5 years) | `LAG`, `DIFF`, `WSUM`, `ROLL_MEAN`, `ROLL_MAX`, `ROLL_MIN`, `ROLL_STD`, `ROLL_RANK`, `WRANK`, `COUNT`, `FIRST_IN_WINDOW`, `WDROP` |
| 252 (1 year) | `RET`, `RET_SKIP`, `RVOL`, `PARK`, `GK`, `SKEW`, `KURT`, `ACORR`, `WSKEW`, `WKURT`, `WACORR`, `SLOPE`, `R2`, `MAXDD`, `DIST_HIGH`, `DIST_LOW`, `UP_STREAK`, `DOWN_STREAK`, `CORR`, `VOL_RATIO`, `AMIHUD`, `RESID_MOM`, `CS_REL_RET`, `WARG`, `WQUANT`, `WMAD`, `FN_YOY`, `FN_EGROW`, `FN_GM_CHG`, `FN_ACCRUAL`, `FN_DY`, `EV_EARN_DRIFT`, `EV_IDX_CHG` |
| 250 | `CS_RANK_CHG`, `CS_RANK_MEAN`, `EV_POST_N`, `EV_DAYS_SINCE` |
| 84 | `EMA` (warm-up 3N-1) |
| 756 | the warm-up steps K of `SMOOTH` |
| 256 | the window W of algorithmic operators |
| 12 | the periods k of `RWIN`, `RWIN_PCT` |
| 20 | the periods k of `EWIN` |
| 100 | the buckets of `CS_QCUT`, the k of `CS_WINZ`, the q of `WQUANT`, `GQUANT` |

For two-parameter operators each parameter must fit 16 bits (the first 0..65,535, the second 0..32,767).

### 13.3 Warm-up

The warm-up of a formula (the warm-up of its nodes added along the tree) must be **less than panel days / 3**.

- The Taiwan panel has 4,127 trading days -> a warm-up limit of 1,375 days, enough for a 1,260-day window.
- Over the limit the plan returns an error stating how many days the formula looks back and how many the panel allows.
- `config={"lookback_div": 2}` relaxes it to panel days / 2 (corpus R22: the change over one year of the gap to a 5-year average).
- Backtest formulas (`hold`, `exposure`, the market part of `exit_when`...) have a warm-up too; during it the condition does not hold.

### 13.4 Constants and money

| Item | Range |
|---|---|
| ordinary constants | fit 32 bits after scaling (about +-2,147 for x1e6 units) |
| large constants | 7 significant digits (mantissa <= 8,388,607) x 10 to the power 0..12; finer values are rejected |
| fractional digits of a constant | no more than the unit's fixed-point precision (6 for x1e6 units; 0 for integer units) |
| PRICE | about +-9.22 trillion per share |
| QTY, AMOUNT | integers, about +-9.22 x 10^18 shares or currency units |

### 13.5 Others

| Item | Limit |
|---|---|
| points of `CORR_GROUP` / `CORR_SEED` (lookback / step) | 20..1,024 (quant 0: 20..500) |
| seeds of `CORR_SEED` | 1..64 |
| symbols listed one by one in a stock pool | 20 (over the whole condition) |
| one formula reading `pos.*` in a position rule (`exit_when`, `keep_when`, `add_when`, `cut_when`) | at most 15 intermediate values held at once (beyond that, split it into comparisons joined with `&&`, `\|\|`) |
| terms in `exit_below` / `exit_above` | 1-3 each |
| user-function grid x bindings | 20,000 combinations |
| threshold grid `grid` | names in portfolio rules: base-grid settings x threshold sets <= 32,767; names in score formulas: 1..256 values each, at most 20,000 trees after expansion |
| checks | 8 checks per strategy; per check 8 runs, 16 outputs, 32 segments; the members one run expands into (window segments x the value counts of its `vary` variables) at most 4,096 |
| `rounds` | 1..1,000 |
| portfolio arguments | the ranges of section 9.2 |

---

## 14. Error codes and fixes

Every error answer carries: the code, which formula, the character position, the message, a hint; a mistyped name also comes with the closest names. Validation happens at plan time, not after a run has started.

### 14.1 Formulas and user functions

| Code | When | Fix |
|---|---|---|
| `APX_SYNTAX` | wrong parentheses or commas, an incomplete formula, `?:` without `:`, an unknown character, a malformed number, something left after the formula | check parentheses and commas; `NAME(params..., inputs...)`; `condition ? value1 : value2` |
| `APX_SYNTAX` | the formula is only a constant | it needs at least one column |
| `APX_UNKNOWN_COLUMN` | the panel has no such column | look up column keys with `stock_formula_help topic=column`; the answer lists the closest names |
| `APX_UNKNOWN_NOUN` | the name is not an operator, an alias or an installed user function | look it up with `stock_formula_help topic=noun query=...`; the answer lists the closest names. Names from other languages and lower-case names get the form written out; a `user.*` not installed gets the definition shape to install (2.7) |
| `APX_UNKNOWN_PARAM` | `$name` is not declared, or is used outside the body of a user function | declare `$params` in `params`; they appear only in a body |
| `APX_UNIT_MISMATCH` | units disagree (PRICE + QTY, QTY x QTY) | bring both sides to the same unit or to unitless numbers: `RATIO(a, b)`, `CS_PCT(x)`; for traded value use the column `px.turnover` |
| `APX_UNIT_MISMATCH` | one side a price or per-share value (PRICE), the other a company total (AMOUNT) | per-share values are `FN_PS(total, shares)`; a total against market cap is `RATIO(total, fund.mktcap)` |
| `APX_UNIT_MISMATCH` | `&&`, `\|\|`, `!` or the condition of `?:` is not BOOL | both sides must be the results of comparisons; multiply numbers with `*` |
| `APX_UNIT_MISMATCH` | the `:UNIT` after a column name differs from the column's unit | remove it or use the column's unit |
| `APX_UNIT_MISMATCH` | arithmetic on a code column | code columns support only `==`, `!=`, `in {...}`, or use as a group key |
| `APX_PARAM_RANGE` | a parameter outside the operator's range (for instance `ROLL_STD(2000, x)`); a look-back length written negative (for instance `LAG(-3, close)`, which looks ahead) | the ranges in the rows of chapter 4; the message states the legal range. N of `RET`, `DIST_HIGH`, `DIST_LOW`, `VOL_RATIO`, `FN_YOY` in 253..1,260 gets the same quantity with a long window (2.7). A negative length: the message says it needs data not yet known when the order is placed, and gives the event-aligned form `MASK(LAG(k, event), RET(k, close))` and the holding period `hold_days`. Out of range never compiles; the server does not clamp to the bound or swap in another length |
| `APX_PARAM_TYPE` | a parameter that is not an integer | parameters are integers; a fractional factor goes into an input (`0.5:RATIO`) |
| `APX_ARITY` | too few inputs, too many arguments, a missing parameter | parameters first, inputs after; for instance `ROLL_MEAN(20, close)` |
| `APX_ARITY` | wrong argument count for the accepted forms `LE`, `AND`, `IF`, `MIN`, `MAX`, `GPCT` (2.5) | the message states how many and gives the form |
| `APX_UNIT_MISMATCH` | a rank inside a group `GRANK` added to or compared with a ratio; a numeric column as the input of an event-days operator; `FILL(condition, 0:RATIO)` | the fix in the message: `GPCT(g, x)`, `EV_CHANGE(column)`, `0:BOOL` (2.7) |
| `APX_UNSUPPORTED` | `SELECT(...)` or the four-input `REC2` | write `SELECT` as `c ? a : b`; use `SMOOTH` or two-input forms instead of `REC2` |
| `APX_ENGINE_LIMIT` | more than 128 nodes after expansion | the answer carries the node count; a reserved function marked R is one node and can replace a stretch of atoms |
| `APX_CYCLE` | user functions refer to each other in a cycle, or nest deeper than 128 levels | break the reference |
| `APX_FIELD` | a wrong field in a definition file (`name`, `version`, `schema`, `body`, `out_unit`) | `name` = `user.<author in lower case>.<NAME in upper case>`; `version` >= 1; `out_unit` is a unit name |
| `APX_CONFLICT` | the same name and version with different content | install again with `version` + 1 |
| `APX_INVALID_DEFINITION` | a formula in `formulas` does not validate | the answer lists each failing formula with its reason |

Frequent messages (the code is one of the above):

| The message means | Fix |
|---|---|
| the unit of the root is BOOL: ranking needs a score | as a filter write `MASK(condition, score)`; to rank by how often it holds write `COUNT(20, condition)` |
| this formula looks back N trading days, the panel allows M | shorten the window, or `config={"lookback_div": 2}` |
| the first input of `FN_EP` / `FN_BP` / `FN_DY` must be a per-share value | use `eps`, `dps` or `FN_PS(total, shares)`; a total against market cap is `RATIO(total, fund.mktcap)` |
| one input is a per-share value, the other a company total | bring both to the same kind |
| the first input of `FN_DY` must be an event column | use `dps`, which has a value only on ex-dividend days; summing a carried column counts the same dividend many times |
| the child of `EV_PULSE` must be a column | put a column name there; for an event defined by a formula use `EV_CHANGE` or a comparison |
| a `"string"` belongs inside `in {...}` or on the right of `==` / `!=` on a code column | `symbol == "2330"`, `symbol in {"2330"}` |
| `{...}` is only the first argument of `CORR_SEED`, or follows `in` | |
| set membership is written `x in {v1, v2, ...}` | never `IN_SET(...)` |
| both sides of `MOD` cannot be constants | at least one side is a column or a formula |
| the constant is too fine | large constants take 7 significant digits |
| a condition noun inside a score formula | condition nouns work only in `universe` or `portfolio.hold` |
| the input of `CORR_GROUP` / `CORR_SEED` cannot be only a constant; no nesting | the input is a numeric formula over panel columns |

### 14.2 Conditions, stock pool, backtest

| Code | When | Fix |
|---|---|---|
| `E_NOT_BOOL` | the result of a condition is not BOOL | write a comparison: `close > 0`, `ROLL_MEAN(20, volume) > 1000000`, `industry in {24, 25}` |
| `E_UNKNOWN_CODE` | the code column has no such name (often: a code of another market) | write this market's codes: Taiwan industries are code numbers, exchanges `TW` / `TWO`; US exchanges `XNAS`, `XNYS` ...; stock codes in double quotes |
| `E_AMBIGUOUS_NAME` | a column name without its prefix matches two or more columns | write the full column key |
| `E_EMPTY_UNIVERSE` | the stock pool has 0 stocks on every day | relax the thresholds; check industry codes and column names first |
| `APX_INVALID_ARGUMENT` | more than 20 symbols listed one by one in the stock pool | use a class noun (`!cls.FIN()`) or a seed basket |
| `E_NOT_MARKET_WIDE` | `rebalance_when` / `exposure` has two or more values on the same day | use only things that are the same for every stock: the calendar, `mkt.idx`, `MKT_MEAN`, `EW_INDEX`; a per-stock condition belongs in `hold` |
| `E_NO_REBALANCE` | `rebalance_when` never holds over the period | check the condition |
| `E_NO_WEIGHT` | `weight` is never positive over the period | only stocks with a weight > 0 are held |
| `E_POS_RULE` | a function that looks back over days takes position state, `LAG(2 or more, pos.*)`, `pos.rank` not alone against an integer or used in `exit_when`, a condition that always or never holds | the one rule of 9.5; yesterday's value is `LAG(1, …)`, for instance `pos.ret < LAG(1, pos.ret) - 0.03` |
| `E_RANGE` | `reentry_days` outside 0..10,000; `group_cap` not a positive integer | |
| `E_POS_GROUP` | only one of `group_by`, `group_cap` given; `group_by` uses `pos.*` | give both, e.g. `{"group_by":"CORR_GROUP(RET(1, close))","group_cap":3}`; the group key reads only that day's data |
| `E_POS_SIZE` | a sizing or account argument out of range or missing its partner (e.g. `size_init` without `add_when`, `gross_max` > 1 without `margin_rate`); a multiple written in `weight` that is not on the 1% grid | follow the ranges in 9.2 and the examples in 9.5b |
| `APX_INVALID_ARGUMENT` | two of the three rebalance arguments given; both `weight` and `weighting` given | "held N days" is `exit_when="pos.days >= N"` |
| `APX_INVALID_ARGUMENT` | an unknown key in `portfolio` | the answer lists every legal argument name; a key named as in another system (`stop_loss`, `top_n`...) gets the argument for that here and an example |
| `APX_CHECK_E_SCORE_COND_TIMES` | the root of the score is "condition x number" | the answer carries the whole fixed `MASK(condition, number)` |
| `APX_CHECK_E_SCORE_ALL_INVALID` | in the last year no stock in the pool has a score on any day | the answer names where the values stop and the fix: a group level the panel lacks -> the highest level it has; a column without data; a reset condition never true -> `SCANMAX(DAY_INDEX() < 1, x)`; too few `RWIN` periods |
| `APX_CHECK_E_SCORE_RANK_TIES` | the score holds a rank inside a group (`CS_PEER_RANK`, `GRANK` with a group, `GPCT`), on every sampled day more names tie at the top score than are picked, and no `tie_by` | give `portfolio.tie_by` (the value being ranked); for `CS_PEER_RANK` also `CS_REL_PARENT(L, x)` |
| `APX_CONFLICT` | the result library is bound to another stock pool, period, per-stock cap or derived-column list | use another library name; the answer suggests one |

### 14.3 Screening and checks

| Code | When | Fix |
|---|---|---|
| `E_UNKNOWN_NAME` | not an atom, a series, a formal input or a definition | the answer adds "did you mean ..."; `stock_formula_help topic=metric` |
| `E_UNKNOWN_FN` | an unknown function; a score-formula upper-case name in a screening slot (`ROLL_MEAN(3, r)`, `WSUM`) | the function table of 11.1; the answer gives this slot's form (2.8) |
| `E_FN_AS_VALUE` | a function-type definition used as a value without parentheses | `apx.SHARPE_OF(r)` |
| `E_ARITY`, `E_ARGS` | the wrong number of arguments | |
| `E_CYCLE` | definitions refer to each other in a cycle | |
| `E_CHECK_STAGE` | `check.*` used in `screen_pick` / `grid_by`, or the threshold uses `check.*` with no check on | write it in the threshold `screen_gate` and turn that check on in `checks` |
| `E_NO_BENCH` | a threshold (`screen_gate` / `screen_pick` / `screen_admit`) uses the market benchmark (series `b`, `apx.BEAT_B_ANN`, `apx.B_ANN`, `apx.EXCESS_B_ANN`, `apx.IR_B`, `apx.BETA_B`, `apx.YEARS_BEAT_B`) and the panel has no total-return index column `mkt.idx_tr`: that term has no value for any candidate, so the plan fails instead of running a search that keeps nothing | drop that term, or use one that does not need the index (e.g. `apx.ANN_RET >= 0.15`); once the panel has the column the original form works |
| `E_EVOLVE_ARGS` | `apx.EVOLVE` without exactly one argument | the response's `fix` is the whole corrected formula (two or more: one wrap each) |
| `E_EVOLVE_EMPTY` | what `apx.EVOLVE` wraps has no neighbour (a BOOL constant, 0, no second column of the same kind, only this value fits the warm-up on the grid) | the response's `fix` wraps one level out |
| `E_EVOLVE_WINDOW` | an operator parameter (days looked at) wrapped in a portfolio rule | the response's `fix` is the `$name` + `grid` form; or move that window into the score formula |
| `E_RESERVED`, `E_NAMESPACE`, `E_NAME` | the name of a definition file is not acceptable | `user.<author>.NAME`; `apx.*` is reserved |
| `E_DUP`, `E_HASH` | the same name and version with different content | `version` + 1 |
| `APX_CHECK_*` | an error in a check definition (`E_EXPR`, `E_UNIT`, `E_PARAM`, `E_INTERVAL`, `E_WINDOWS`, `E_VARY`, `E_RESOURCE`, `E_CHECK_ATOM_PLACE` (a transform in the wrong place), `E_CHECK_LANG` (`with` written in the lowercase screen language), `E_RET_MIX`, `E_WITH_TYPE` (an item of `with` is not a condition; returns go inside `RETURNS(...)`), `E_WITH_TWICE` (one run changes the returns twice), `E_WITH_UNKNOWN` (no such transform; the closest name is given), `E_RET_UNIT` (the `RETURNS` expression is not a RATIO), `E_UNIVERSE_EMPTY` (the pool averages under 1 stock a day), `E_RES_LIMIT` (more than 2 pools or 2 return transforms in a round; the ones in use are listed), `E_DAYS_FIELD` (a run with a day transform reads a field that depends on the positions), `E_OUTPUT`, `E_PASS`... with the prefix `APX_CHECK_`) | the answer carries the path (for instance `runs[1].interval`), the position and a suggestion |

---

## 15. Not writable yet

The features below are decided but cannot be written yet; plan rejects them. Once available they are written into the chapter they belong to.

| Feature | Plain words |
|---|---|
| Limit lock at rebalances | a holding locked limit-down on a rebalance day stays held, a name locked limit-up is not bought and its share stays in cash (the lock deferral of touch and close-rule exits already works, see `lock` in 9.6a) |
| Intraday bars | entries and exits at intraday time points, columns computed from intraday bars |
| Atoms with three or four inputs | `SELECT`, `REC2`, `STATESCAN` written directly in a formula (write `c ? a : b` today) |
| Long `CORR` window | N of `CORR` widened to 1,260 (3..252 today) |

---

## 16. Appendix: complete examples

Each item is a complete set of arguments for `stock_backtest` (item ids as in the example library `examples.json`). A few from each of the six classes.

### 16.1 Stock pool (U)

**U03  Traded every day for the past 20 days, with 20-day average volume above 500 lots**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "universe": "COUNT(20, volume > 0) >= 20 && ROLL_MEAN(20, volume) > 500000"}
```

**U12  Only stocks that have just closed above their 250-day average (crossed within 20 days)**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "universe": "close > ROLL_MEAN(250, close) && COUNT(20, close > ROLL_MEAN(250, close)) < 20"}
```

**U16  Top 30% by market cap, without TSMC and Hon Hai**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "universe": "CS_PCT(fund.mktcap) >= 0.7 && !(symbol in {\"2330\", \"2317\"})"}
```

### 16.2 Ranking (R)

**R01  Compare stocks of similar size: ten market-cap groups, 20-day return inside each group**

```json
{"trees_from": "formulas",
 "formulas": ["SUB(RET(20, close), RATIO(GSUM(CS_QCUT(10, fund.mktcap), RET(20, close)), GSUM(CS_QCUT(10, fund.mktcap), MASK(VALID(RET(20, close)), 1:RATIO))))"]}
```

**R04  On the fifth trading day after a quarterly report, rank by the return since the report**

```json
{"trees_from": "formulas",
 "formulas": ["MASK(LAG(5, EV_CHANGE(revenue)), RATIO(close, LATCH(EV_CHANGE(revenue), LAG(1, close))))"]}
```

**R13  Gross margin up three quarters in a row, ranked by the latest rise**

```json
{"trees_from": "formulas",
 "formulas": ["MASK(RATIO(fund.gross, revenue) > RWIN(1, 0, RATIO(fund.gross, revenue)) && RWIN(1, 0, RATIO(fund.gross, revenue)) > RWIN(2, 0, RATIO(fund.gross, revenue)) && RWIN(2, 0, RATIO(fund.gross, revenue)) > RWIN(3, 0, RATIO(fund.gross, revenue)), RWIN(1, 7, RATIO(fund.gross, revenue)))"]}
```

**R15  Value, quality and momentum, three factors ranked with equal weight**

```json
{"trees_from": "formulas",
 "formulas": ["ADD(ADD(CS_PCT(FN_EP(eps, close)), CS_PCT(FN_ROE(net_income, equity))), CS_PCT(RET_SKIP(252, 21, close)))"]}
```

### 16.3 Heat and flows (H)

**H04  Volume and price up three days in a row**

```json
{"trees_from": "formulas",
 "formulas": ["MASK(COUNT(3, volume > LAG(1, volume) && close > LAG(1, close)) >= 3, RET(3, close))"]}
```

**H05  Rising again after cooling: a volume surge earlier, at least 20 quiet days, a surge again today**

```json
{"trees_from": "formulas",
 "formulas": ["MASK(VOL_RATIO(20, volume) >= 3 && COUNT(20, LAG(1, VOL_RATIO(20, volume) >= 3)) == 0 && COUNT(120, VOL_RATIO(20, volume) >= 3) >= 2, VOL_RATIO(20, volume))"]}
```

**H03  A group bought in step by institutions: grouped by the correlation of foreign net buying, the group with the largest 5-day total first**

```json
{"trees_from": "formulas", "formulas": ["GSUM(CORR_GROUP(foreign_net), WSUM(5, foreign_net))"]}
```

### 16.4 Backtest (B)

**B03  Half invested while the equal-weighted market is below its 200-day average**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "portfolio": {"max_positions": 10, "rebalance_days": 21,
               "exposure": "EW_INDEX(close) < ROLL_MEAN(200, EW_INDEX(close)) ? 0.5 : 1"}}
```

**B09  Stop loss at 8%, take profit at 20%**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "portfolio": {"max_positions": 10, "rebalance_days": 21,
               "exit_when": "pos.ret <= -0.08 || pos.ret >= 0.2"}}
```

**B13  Rank buffer: buy in the top 10, sell out of the top 30**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "portfolio": {"max_positions": 10, "rebalance_days": 21, "keep_when": "pos.rank <= 30"}}
```

**B19  Weights by 20-day average traded value, at most 10% per stock**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "portfolio": {"max_positions": 20, "rebalance_days": 21, "max_weight": 10,
               "weight": "ROLL_MEAN(20, px.turnover)"}}
```

### 16.5 Screening and checks (F)

**F04  Beats the pool's equal-weighted mean every year**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "screen_gate": "min(by_sum(year, r - m)) > 0"}
```

**F01  Annual return / maximum drawdown at least 1** (install `user.local.ROMAD` of 11.4 with `stock_formula_help topic=validate_change` first)

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "screen_gate": "user.local.ROMAD >= 1"}
```

**F09  At least half of the return kept after a peer swap**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "checks": ["peer_swap"], "check_params": {"PEER_SWAP.keep_bp": 5000}}
```

### 16.6 Odd strategies (W)

**W08  The biggest losers of Monday, bought on Tuesday**

```json
{"trees_from": "formulas", "formulas": ["MASK(WEEKDAY() == 0, NEG(RET(1, close)))"],
 "portfolio": {"max_positions": 10, "rebalance_when": "WEEKDAY() == 0"}}
```

**W12  Stocks that rose every December of the last 5 years**

```json
{"trees_from": "formulas",
 "formulas": ["MASK(COUNT(1260, MONTH() == 1 && TDAY_OF_MONTH() == 1 && AT_MONTH_START(LAG(1, close)) > AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, close))))) >= 5, RET(20, close))"]}
```

**W14  Grouped by the correlation of monthly revenue growth; inside each group, the largest gap between a stock's 20-day return and the group mean**

```json
{"trees_from": "formulas",
 "formulas": ["ABS(SUB(RET(20, close), RATIO(GSUM(CORR_GROUP(1260, 126, 21, 3000, 1, RWIN_PCT(12, revenue_m)), RET(20, close)), GSUM(CORR_GROUP(1260, 126, 21, 3000, 1, RWIN_PCT(12, revenue_m)), MASK(VALID(RET(20, close)), 1:RATIO)))))"]}
```

**R17  A 14-day RSI built by hand, lower first** (install `user.local.RSI` of 12.1 first)

```json
{"trees_from": "formulas", "formulas": ["NEG(user.local.RSI(42, close))"]}
```
