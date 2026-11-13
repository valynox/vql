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
| Checks | `checks`, `apx.check.v1` (chapter 11) | integer (i64) | metrics after re-running under changed conditions, and a pass condition |

All four share one grammar, one set of operators and one precedence table. They differ in value type and in the names available: search and backtest use panel columns and the operators of chapter 4; screening and checks use strategy-level atoms (`r`, `m`, `mean`, `sd`...). The stock pool (chapter 8) is a generation formula whose result is BOOL.

### 1.3 Three layers of names

| Layer | Looks like | Defined by | Examples |
|---|---|---|---|
| Atom | a name without a dot | the engine; adding one means engine code | `LAG`, `WSUM`, `GT`, `LOGR`, `SCANMAX` |
| Reserved function | a name without a dot, or the full name `apx.NAME` | official; composed of atoms | `RET` (= `apx.RET`), `ROLL_MEAN`, `CS_PCT` |
| User function | `user.<author>.NAME` | the user, stored through `my_stock_strategies_change action=install` | `user.local.RSI` |

- The atom definitions of the reserved functions are in chapter 5: every reserved function decomposes back into atoms, and the definition is bit-identical to writing the name.
- A user function expanded to atoms is the same tree as writing the atoms directly, with the same results.
- There are also official class nouns `cls.NAME()` (chapter 8), used only in conditions.

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
- An operator has at most two inputs. Atoms with three or four inputs (`SELECT`, `REC2`, `STATESCAN`) are not accepted in formula text today (section 4.13).
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
| `a && b` | `MUL(a, b)` | both sides BOOL |
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

- `&&` and `||` do not short-circuit: if either side is invalid the result is invalid (the same holds for `OR`).
- In the middle part of `?:`, a `:` followed by a known unit name is a unit suffix; otherwise it is the colon of `?:`. Both `c ? 0.5:RATIO : 1:RATIO` and `c ? 0.5 : 1` are correct.

### 2.3 Constants and units

| Written | Read as |
|---|---|
| `0.5:RATIO`, `20:DAYS`, `1:COUNT`, `0:PRICE` | constants with an explicit unit |
| `close > 100` | a number without a unit takes the unit of the other side (here `100:PRICE`) |
| `close / LAG(20, close) > 1.1` | the left side is RATIO, so `1.1` is `1.1:RATIO` |
| `MUL(x, 2)`, `RATIO(x, 3)` | a bare multiplier or divisor with no other side to follow is RATIO (a unitless factor) |
| `fund.mktcap > 5e9` | large constants are handled automatically: constants that do not fit 32 bits take 7 significant digits x 10 to the power 0..12 |

- A constant with more fractional digits than the unit's fixed-point precision (6 digits for x1e6 units; none for integer units) is rejected.
- A formula cannot be only a constant; it needs at least one column.

### 2.4 Code columns, sets, strings

`industry`, `market`, `common`, `symbol` are code columns of unit CODE: they support only `==`, `!=`, `in {...}`, or use as a group key. No arithmetic.

| Written | Meaning |
|---|---|
| `market in {TW, TWO}` | listed on the main board or the OTC market (exchange names are constants) |
| `symbol in {"2330", "2317"}` | these two stocks (codes in double quotes) |
| `symbol == "2330"` | this one stock |
| `industry in {24, 28}` | industry codes (look the table up with `stock_formula_help topic=universe`) |
| `MONTH() in {3, 5, 8, 11}` | `in` also works on numeric values; the values are read in the unit of x |

- Accepted in conditions (stock pool, `hold`, `rebalance_when`, `exposure`) and in score formulas and user functions.
- In a score formula, `x in {...}` and `symbol == "2330"` are a BOOL column computed at the start of the round and appended to the panel; it is part of the result library's identity. Use it as the condition of `MASK` or `?:`, or multiply by it.
- In a score formula the industry group is written `PARENT_ID(1)` (a group key), not `industry`.

### 2.5 Aliases

| Alias | Equals |
|---|---|
| `DIV` | `RATIO` |
| `WMAX` | `ROLL_MAX` |
| `WMIN` | `ROLL_MIN` |
| `apx.NAME` | `NAME` (the full name of a reserved function: `apx.RET` = `RET`) |
| `GSUM(k, x)`, `GCOUNT`, `GRANK`, `GMAX`, `GMIN`, `GSTD` whose first argument is not "an integer without a unit" | the key forms `GSUM_BY(k, x)` ... `GMIN_BY(k, x)`, `GSTD_BY(k, x)` (section 4.8) |
| `ROLL_SUM(N, x)` | `WSUM(N, x)` |
| `LE(a, b)`, `GE(a, b)`, `EQ(a, b)`, `NE(a, b)` | `a <= b`, `a >= b`, `a == b`, `a != b` |
| `AND(a, b, ...)` | `a && b && ...` |
| `IF(c, a, b)`, `IIF(c, a, b)`, `WHERE(c, a, b)` | `c ? a : b` |
| `MIN(a, b, ...)`, `MAX(a, b, ...)` | nested `MIN2` / `MAX2`: the smaller / larger value cell by cell. The minimum / maximum over N days is `ROLL_MIN(N, x)` / `ROLL_MAX(N, x)`, over the whole market on one day `GMIN(0, x)` / `GMAX(0, x)` |
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

### 2.6 What is printed reads back

Formulas printed in lists, reports and search logs are canonical text; pasted back into `formulas` they compile to the same tree.

| Printed as | Example |
|---|---|
| always function form, single spaces | `GT(RATIO(close, LAG(20, close)), 1.1:RATIO)` |
| constants always carry a unit | `1.1:RATIO`, `20:DAYS` |
| two parameters printed separately | `RET_SKIP(252, 21, close)` |
| sets | `(MONTH() in {3, 5, 8, 11})` |
| key forms | `GSUM_BY(PARENT_ID(1), volume)` |
| a money column read in whole currency units | `fund.mktcap:AMOUNT` |
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

### 3.2 The eleven units

| Unit | Stored | Meaning | Examples |
|---|---|---|---|
| `PRICE` | x1e6 | price | `close`, `raw.limit_up` |
| `QTY` | x1e6 | quantity (shares) | `volume`, `shares`, `foreign_net` |
| `MONEY` | x1e6 | money; holds about 9.22 trillion | `revenue`, `eps` (per-share values are MONEY too) |
| `RATIO` | x1e6 | a ratio, a unitless number | `px.turnover_rate`, `RET(20, close)` |
| `PCT` | x1e6 | percentage points | |
| `INDEX` | x1e6 | index points | `idx.taiex` |
| `DAYS` | integer | trading days | `EV_DAYS_SINCE(...)`, `WARG(...)` |
| `COUNT` | integer | counts, ordinals | `MONTH()`, `COUNT(...)`, `GRANK(...)` |
| `BOOL` | integer 0 / 1 | true or false | `close > open` |
| `CODE` | integer | codes (equality, `in`, group key only) | `industry`, `symbol`, `PARENT_ID(1)` |
| `AMOUNT` | integer (whole currency units) | large money amounts | `fund.mktcap:AMOUNT` |

"Numeric unit" (written X below) = the nine units other than BOOL and CODE.

A BOOL constant is 0 or 1 only (BOOL stores true / false): `2:BOOL` and `FILL(c, 2)` with c a condition are rejected with the fix `1:BOOL`; for a count write `2:COUNT`.

**What AMOUNT is**: the x1e6 fixed point of MONEY holds only about +-9.22 trillion. Total columns in currency units (market cap, total assets) that exceed it use AMOUNT (whole currency units).

- Columns are always declared MONEY; AMOUNT exists only as a unit inside formulas.
- If any cell of a column does not fit MONEY, the bare column name is read as AMOUNT automatically; `column:AMOUNT` can also be written explicitly.
- When one input is MONEY and the other AMOUNT, the MONEY side is switched to its AMOUNT form (total columns in whole units, constants). Per-share values (`eps`) cannot be mixed with AMOUNT.

### 3.3 Unit algebra: what adds, multiplies, compares

| Operation | Inputs -> output | Rejected example |
|---|---|---|
| `ADD`, `SUB`, `MAX2`, `MIN2` | (X, X) same unit -> X; per-share money (MONEY) with PRICE, either order -> unit of the first input | `close + volume` (PRICE + QTY) |
| `GT`, `LT` (and `>= <= == !=`) | (X, X) same unit -> BOOL; per-share money (MONEY) against PRICE -> BOOL | `close > volume` |
| `MUL` | (RATIO or BOOL, X) -> X; (X, RATIO or BOOL) -> X; (BOOL, BOOL) -> BOOL; (COUNT or DAYS, X) -> X; (X, COUNT or DAYS) -> X | `close * volume` (PRICE x QTY; for traded value use the column `px.turnover`) |
| `RATIO` (`/`), `MULDIV` | (X, X) same unit -> RATIO; (X, RATIO) -> X; (X, COUNT or DAYS) -> X; (MONEY, QTY) -> MONEY (per-share money, = `FN_PS` except shares <= 0 are not rejected); (MONEY, PRICE), (PRICE, MONEY) -> RATIO | `revenue > close` (a company total is a share count away from a price: per-share values are `FN_PS(revenue, shares)`, the average traded price is `VWAP(N)`) |
| `LOGR` | (X, X) same unit -> RATIO | |
| `LOG`, percentiles, z-scores, returns | X -> RATIO | |
| `SQRT` | RATIO -> RATIO; PRICE / QTY / PCT / MONEY / INDEX -> RATIO; COUNT / DAYS / AMOUNT also accepted (expanded at text level) -> RATIO | |
| `FLOOR`, `ROUND`, `CEIL` | x1e6 X -> X; integer units are themselves | |
| `EXP`, `SIN`, `COS`, `ATAN` | RATIO -> RATIO | `EXP(close)` |
| `NEG`, `ABS`, `DIFF`, window max / min / mean | X -> X | |
| `LAG` | any non-CODE (BOOL included) -> same unit | |
| `NOT` | BOOL -> BOOL | |
| `OR` | (BOOL, BOOL) -> BOOL | |
| `FILL` | (X, X) same unit (BOOL included, CODE not) -> same unit | |
| `MASK` | (BOOL, any non-CODE) -> the unit of the second input | |
| `VALID`, `EV_CHANGE`, `in {...}` | anything (CODE included) -> BOOL | |

When both sides of a product are integer units: COUNT x COUNT -> COUNT, COUNT x DAYS -> DAYS, DAYS x DAYS -> DAYS. Dividing two values of the same unit always gives RATIO (COUNT / COUNT is a ratio).

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

The means (`ROLL_MEAN`, `ROLL_STD`, `XL_PARENT`, `GMEAN_BY`, `GSTD`, `GSTD_BY`, `CS_REL_PARENT`, `CS_Z`, `CS_PEER_Z`, `CS_WINZ`) add up in exact 128-bit arithmetic and then divide; a mean always fits i64, so a sum that leaves i64 does not make them invalid. `WSUM` and `GSUM` output the sum itself and stay invalid when it does not fit i64.

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
| Layer | A = atom (smallest, implemented directly by the engine); R = reserved function (composed of atoms; the atom definition of every one is in chapter 5) |
| Params | the legal range of each parameter (code constants) |
| Units | inputs -> output. X = any numeric unit; "same" = the output has the unit of the input |
| Invalid and warm-up | rules beyond "any invalid input -> invalid"; warm-up = the first days this node makes invalid |
| Example | one complete score formula that compiles |

Common rules:

- Any invalid input -> invalid output. Window operators: any invalid cell inside the window -> invalid (rows with another rule say so).
- A window of N includes today: `[t-N+1, t]`. `LAG(N, x)` is N trading days ago.
- Warm-up adds up: the warm-up of an outer node = the warm-up of its children + its own.
- Return-type operators (`RVOL`, `SKEW`, `KURT`, `ACORR`, `BETA`, `RESID_MOM`) use log returns `ln(x[t]/x[t-1])` internally; a price <= 0 -> invalid.
- Level L: 0 = everything (inside the stock pool), 1 = industry, 2 = market, 3 = country, 4 = global.
- Members of a cross-section or a group = stocks inside the stock pool that day whose input is valid; non-members get an invalid output.
- Nodes whose output takes only a few values (bucket numbers, flags, group sums and means, which are the same for every member of a group) produce massive ties when used as a whole score; they are parts: use them as a condition, a group key, or divide / subtract a per-stock value by them.

### 4.2 Leaves and constants

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| column key (`LEAF`) | A | — | the column's unit | no value in that cell -> invalid | the column's value that day | `NEG(close)` |
| number (`CONST`) | A | fits 32 bits after scaling | declared unit | always valid | a constant | `MASK(close > 10, RET(20, close))` |
| large number (`CONSTE`) | A | 7 significant digits x 10^0..12 | declared unit | always valid | a constant that does not fit 32 bits; just write the number, it is chosen automatically | `MASK(fund.mktcap > 5e9, RET(20, close))` |

`LEAF`, `CONST`, `CONSTE` cannot be written as calls; write the column name or the number.

### 4.3 Element-wise arithmetic

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `ADD(a, b)` | A | — | (X, X) same unit -> same | overflow -> invalid | a + b | `ADD(CS_PCT(RET(20, close)), CS_PCT(RET(60, close)))` |
| `SUB(a, b)` | A | — | (X, X) same unit -> same | overflow -> invalid | a - b | `SUB(RET(60, close), RET(60, idx.taiex_tr))` |
| `MUL(a, b)` | A | — | see 3.3 | product leaves i64 -> invalid | a x b; with scaled units divided by 1e6, then floored | `MUL(RET(20, close), VOL_RATIO(20, volume))` |
| `RATIO(a, b)` (`DIV`) | A | — | see 3.3 | b = 0 -> invalid | a / b, floored | `RATIO(low, close)` |
| `MULDIV(K, a, b)` | A | K 1..1,000,000,000 | same as `RATIO(a, b)` | b = 0 or result leaves i64 -> invalid | K x a / b, 128-bit product, floored once; K x a never overflows (`RATIO(MUL(K:COUNT, a), b)` is invalid where K x a leaves i64 and equal everywhere else) | `MULDIV(4, net_income, equity)` |
| `NEG(a)` | A | — | X -> same | i64 minimum -> invalid | -a | `NEG(RVOL(60, close))` |
| `ABS(a)` | A | — | X -> same | | absolute value | `ABS(SUB(CS_PEER_RANK(1, close), CS_PEER_RANK(1, fund.mktcap)))` |
| `SQRT(a)` | A | — | RATIO -> RATIO; PRICE / QTY / PCT / MONEY / INDEX -> RATIO; COUNT / DAYS / AMOUNT -> RATIO (expanded at text level) | a < 0 -> invalid | floor of the square root (fixed point); the output is a unitless number | `SQRT(fund.mktcap)` |
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
| `OR(a, b)` | R | — | (BOOL, BOOL) -> BOOL | either invalid -> invalid (no short circuit) | or. "And" is `MUL(a, b)` | `MASK(OR(RET(5, close) > 0.1, VOL_RATIO(20, volume) >= 3), RET(5, close))` |
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
| `SKEW(N, x)` | A | N 3..252 | X -> RATIO | warm-up N | skewness of N log returns | `SKEW(60, close)` |
| `KURT(N, x)` | A | N 4..252 | X -> RATIO | warm-up N | excess kurtosis of N log returns | `NEG(KURT(60, close))` |
| `ACORR(N, k, x)` | A | N 3..252; 1 <= k <= N-2 | X -> RATIO | warm-up N | lag-k autocorrelation of N log returns | `ACORR(60, 1, close)` |
| `SLOPE(N, x)` | A | N 2..252 | X -> same | warm-up N-1 | slope of the linear regression on time over the last N days (per day) | `RATIO(SLOPE(60, close), close)` |
| `R2(N, x)` | A | N 3..252 | X -> RATIO 0..1 | warm-up N-1 | R squared of the same regression | `R2(60, close)` |
| `MAXDD(N, x)` | A | N 2..252 | X -> RATIO | warm-up N-1 | maximum drawdown inside the last N days (positive; old definition, frozen) | `NEG(MAXDD(250, close))` |
| `DIST_HIGH(N, x)` | R | N 2..1,260 | X -> RATIO (<= 0) | warm-up N-1 | x[t] / highest of the last N days - 1 | `DIST_HIGH(250, close)` |
| `DIST_LOW(N, x)` | R | N 2..1,260 | X -> RATIO (>= 0) | warm-up N-1 | x[t] / lowest of the last N days - 1 | `DIST_LOW(250, close)` |
| `UP_STREAK(N, x)` | R | N 1..252 | X -> COUNT | warm-up N | consecutive rising days up to today, capped at N | `UP_STREAK(20, close)` |
| `DOWN_STREAK(N, x)` | R | N 1..252 | X -> COUNT | warm-up N | consecutive falling days, capped at N | `NEG(DOWN_STREAK(20, close))` |
| `CORR(N, a, b)` | A | N 3..252 | (X, Y) any numeric -> RATIO | warm-up N-1 | Pearson correlation of a and b over the last N days | `NEG(CORR(60, close, volume))` |
| `VOL_RATIO(N, x)` | R | N 1..252 | X -> RATIO | warm-up N | x[t] / mean of the previous N days (today excluded) | `VOL_RATIO(20, volume)` |
| `AMIHUD(N, a, b)` | A | N 1..252 | (X, MONEY) -> RATIO | warm-up N | mean over N days of \|daily return of a\| x 1e9 / traded value b (larger = less liquid) | `AMIHUD(20, close, px.turnover)` |
| `BETA(N, a, b)` | A | N 3..252 | (X, Y) any numeric -> RATIO | warm-up N | covariance of the log returns of a and b / variance of b | `NEG(BETA(120, close, idx.taiex))` |
| `RESID_MOM(N, a, b)` | A | N 3..252 | (X, Y) any numeric -> RATIO | warm-up N | N-day cumulative residual of a after removing beta x the return of b | `RESID_MOM(120, close, idx.taiex)` |

### 4.6 Recursion and reset

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `SCANSUM(r, x)` | A | — | (BOOL, X) -> unit of x | invalid before the first true r; r or x invalid -> invalid that day, the chain breaks until r is next true | restarts from x on a day r is true, then adds x day by day | `SCANSUM(foreign_net <= 0, MUL(foreign_net > 0, 1:COUNT))` |
| `SCANMAX(r, x)` | A | — | (BOOL, X) -> unit of x | as `SCANSUM` | highest x since r was last true (that day included) | `RATIO(close, SCANMAX(EV_CHANGE(revenue_m) && revenue_m >= ROLL_MAX(250, revenue_m), close))` |
| `SCANMIN(r, x)` | A | — | (BOOL, X) -> unit of x | as `SCANSUM` | lowest x since r was last true (that day included) | `RATIO(close, SCANMIN(EV_CHANGE(revenue_m), close))` |
| `LATCH(r, x)` | A | — | (BOOL, X) -> unit of x | invalid before the first sample; x invalid on a sampling day -> invalid until the next sample | records x on a day r is true and holds it | `MASK(LAG(5, EV_CHANGE(revenue)), RATIO(close, LATCH(EV_CHANGE(revenue), LAG(1, close))))` |
| `SMOOTH(K, a, x)` | A | K 0..756 | (RATIO, X) -> unit of x | x invalid -> invalid and restart; invalid for K steps after a start; warm-up K | exponential smoothing y <- y + floor(a x (x - y)), starting at y = x | `RATIO(close, SMOOTH(60, 0.1:RATIO, close))` |
| `RWIN(k, mode, x)` | A | k 1..12; mode 0 value k periods ago / 1 sum of the last k / 2 mean / 3 min / 4 max / 5 standard deviation / 7 change versus k periods ago | X -> same | not enough periods -> invalid | a window over publications: one "period" = one change of value (a quarter for statements, a month for monthly revenue) | `RATIO(RWIN(4, 1, fund.div_paid), fund.mktcap)` |
| `RWIN_PCT(k, x)` | A | k 1..12 | X -> RATIO | value k periods ago = 0 -> invalid | (this period - k periods ago) / \|k periods ago\|; monthly revenue year over year uses k = 12 | `RWIN_PCT(12, revenue_m)` |
| `EWIN(k, mode, b, x)` | A | k 1..20; mode as in `RWIN` | (BOOL, X) -> unit of x | not enough periods, or x or b invalid that day -> invalid | a window over events: one "period" = a day where b is true (and x valid); `RWIN(k, m, x)` = `EWIN(k, m, EV_CHANGE(x), x)` | `EWIN(4, 2, EV_CHANGE(revenue_m), RET(1, close))` (mean move on the last 4 monthly-revenue release days) |

- Common reset combinations: a streak `SCANSUM(not holding, MUL(holding, 1:COUNT))`; the highest since an event `SCANMAX(event, x)`; the value on the event day `LATCH(event, x)`.
- The warm-up of `RWIN` is counted in periods and cannot be turned into a fixed number of days; cells at the start of the panel without enough periods are invalid.
- Trailing four quarters = `RWIN(4, 1, x)`; the same month last year = `RWIN(12, 0, revenue_m)`; the previous quarter = `RWIN(1, 0, x)`; the change from the previous period = `RWIN(1, 7, x)`.
- When a "period" is a day some condition holds rather than a value change, use `EWIN`: the mean close on the last 3 volume-spike days `EWIN(3, 2, volume > 2 * ROLL_MEAN(20, volume), close)`; `LATCH(c, close)` is the close on the latest such day and `EWIN(1, 0, c, close)` the one before it.
- Example: the mean reaction on the last 4 quarterly-report days = `EWIN(4, 2, EV_CHANGE(revenue), RET(1, close))`. `EV_CHANGE(revenue)` marks the report day (the day quarterly revenue changes); each report day's 1-day return is one period, averaged over the latest 4; a stock with fewer than 4 reports is invalid. `EWIN(4, 3, ...)` is the worst of the 4, `EWIN(1, 0, ...)` the reaction to the previous report.

### 4.7 Cross-section (same day, inside the stock pool)

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `CS_PCT(x)` | R | — | X -> RATIO 0..1 | fewer than 2 members -> invalid | percentile inside the stock pool that day | `CS_PCT(RET(60, close))` |
| `CS_Z(x)` | A | — | X -> RATIO | standard deviation 0 -> invalid | (x - pool mean) / pool standard deviation | `CS_Z(RET(60, close))` |
| `CS_WINZ(k, x)` | R | k 5..100 | X -> RATIO | | `CS_Z` clipped at +-k/10 standard deviations | `CS_WINZ(30, RET(60, close))` |
| `CS_QCUT(N, x)` | R | N 2..100 | X -> COUNT 0..N-1 | | cuts `CS_PCT` into N buckets and returns the bucket number | `GRANK(CS_QCUT(5, fund.mktcap), RET(20, close))` |
| `CS_RANK_CHG(N, x)` | R | N 1..250 | X -> RATIO | warm-up N | today's `CS_PCT` - the `CS_PCT` of N days ago | `CS_RANK_CHG(20, RET(60, close))` |
| `CS_RANK_MEAN(N, x)` | R | N 2..250 | X -> RATIO | today must be valid and at least ceil(N/2) of the last N days valid; warm-up N-1 | mean of `CS_PCT` over the last N days | `CS_RANK_MEAN(20, RET(5, close))` |
| `CS_RANKMIX(L, flags, a, b)` | R | L 0..4; flags 0..3 (1 inverts a, 2 inverts b, 3 inverts both) | (X, Y) any numeric -> RATIO 0..2 | both must be members; fewer than 2 members -> invalid | the percentiles of a and of b inside group L, added (two equal-weight factors) | `CS_RANKMIX(0, 2, FN_EP(eps, close), RVOL(60, close))` |
| `CS_REL_RET(N, L, x)` | R | N 1..252; L 0..4 | X -> RATIO | warm-up N | N-day return - the mean return of the group | `CS_REL_RET(20, 1, close)` |
| `CS_NEUTRAL_IND(L, x, w)` | A | L 1..4 | (X, any numeric) -> unit of x | a member needs w > 0 | x - the w-weighted mean of the group's members | `CS_NEUTRAL_IND(1, RET(20, close), fund.mktcap)` |
| `CS_NEUTRAL_SIZE(x, z)` | A | — | (X, any numeric) -> unit of x | | residual of x regressed on [1, z] inside the pool | `CS_NEUTRAL_SIZE(RET(20, close), LOG(fund.mktcap))` |
| `CS_NEUTRAL_COUNTRY(x, w)` | R | — | (X, any numeric) -> unit of x | | as `CS_NEUTRAL_IND` with the group fixed to the country (for multi-market panels) | `CS_NEUTRAL_COUNTRY(RET(20, close), fund.mktcap)` |
| `CS_NEUTRAL_MODEL(p, x, q)` | A | p = level 1..4 + K x 256, K <= 16 | (X, any numeric) -> unit of x | | group-demeaned, then minus the projection on K exposure columns; q must be a column written back by the risk model | — |

`CS_NEUTRAL_MODEL` needs columns that exist only on a panel with a risk model attached; production panels have none today, so it has no example.

### 4.8 Groups

A group can be given in three ways, all with the same membership rule:

| Way | Written | How the group is formed |
|---|---|---|
| Level | `GSUM(L, x)`: the first argument is **an integer without a unit** | L 0 everything, 1 industry, 2 market, 3 country, 4 global; a stock without a parent at level L is invalid |
| Group key | `GSUM(k, x)`: the first argument is anything else (a column, a formula, a constant with a unit such as `0:COUNT`); prints as `GSUM_BY(k, x)` | stocks with the same key on the same day form a group; key units CODE / COUNT / BOOL / DAYS |
| Whole-panel members | `GSUM_ALL(L, x)` | groups by level L, but membership **ignores the stock pool** |

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `GSUM(L, x)` | A | L 0..4 | X -> same; BOOL -> COUNT | sum leaves i64 -> the whole group invalid | sum over the group's members, given to every member | `RATIO(volume, GSUM(1, volume))` |
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
| `GMEAN_BY(k, x)` | A | — | (key, X) -> unit of x | a single member counts | mean among the members with the same key (128-bit sum, then divide: valid even when the sum leaves i64; the level form is `XL_PARENT`) | `SUB(RET(20, close), GMEAN_BY(PARENT_ID(1), RET(20, close)))` |
| `ORD(k, x)` | A | — | (key, X) -> COUNT | non-members invalid (members as in `GRANK_BY`); a one-member group gives 0 | the **exact position** of x among the members with the same key: from 0 = largest, every group exactly 0..n-1. Equal values are broken by a fixed shuffle (the same rule as the stock picking ranks, never by ticker order); k as in `GSUM`: a unitless integer = level (`0` whole market, `1` industry, ...), anything else = group key | `ORD(1, x) == 0` = each industry's top one; for the top N use the reserved functions `GTOP(N, first, x)`, `CS_TOPK(k, x)` (5.2) |
| `PARENT_ID(L)` | A | L 1..4 | -> CODE | no parent at level L -> invalid | the id of the level-L parent; a group key only | `RATIO(volume, GSUM(PARENT_ID(1), volume))` |
| `GSUM_ALL(L, x)` | A | L 0..4 | X -> same; BOOL -> COUNT | no member in the group that day -> the whole group invalid; sum leaves i64 -> the whole group invalid | members = every stock with x valid that day (the stock pool is not applied); the result is written to every cell of the group, including cells where x is invalid, outside the pool, or halted. With L = 0 every stock gets the same value | `SUB(RET(20, close), RATIO(GSUM_ALL(0, RET(20, close)), GCOUNT_ALL(0, RET(20, close))))` |
| `GCOUNT_ALL(L, x)` | A | L 0..4 | anything -> COUNT | no member -> invalid (not 0) | the member count of `GSUM_ALL` | `MASK(RATIO(GSUM_ALL(1, close > ROLL_MEAN(60, close)), GCOUNT_ALL(1, close)) > 0.5, RET(20, close))` |
| `CS_PEER_RANK(L, x)` | R | L 1..4 | X -> RATIO 0..1 | fewer than 2 members -> invalid | percentile inside the group | `CS_PEER_RANK(1, RET(20, close))` |
| `CS_PEER_Z(L, x)` | A | L 1..4 | X -> RATIO | | z-score inside the group | `CS_PEER_Z(1, FN_EP(eps, close))` |
| `CS_REL_PARENT(L, x)` | R | L 1..4 | X -> same | | x - the mean of the group's members | `CS_REL_PARENT(1, RET(20, close))` |
| `XL_PARENT(L, x)` | A | L 1..4 | X -> same | | the mean of the group's members; every member gets the same value | `RATIO(volume, XL_PARENT(1, volume))` |

Common combinations:

| Wanted | Written |
|---|---|
| percentile inside a group | `GPCT(k, x)` (= `RATIO(GRANK(k, x), ADD(SUB(GCOUNT(k, x), 1:COUNT), SUB(GCOUNT(k, x), 1:COUNT)))`, section 5.2) |
| mean inside a group | `GMEAN_BY(k, x)` (any unit; by level: `XL_PARENT(L, x)`) |
| z-score inside a group | `MASK(GSTD(k, x) > 0, RATIO(SUB(x, GMEAN_BY(k, x)), GSTD(k, x)))`; by level L: `XL_PARENT(L, x)`, `GSTD(L, x)` |
| how far above the group mean | `SUB(x, group mean)` |
| everything as one group | `GSUM(0, x)`, or a constant key `GSUM(0:COUNT, x)` |
| example keys | `PARENT_ID(1)` (industry), `CS_QCUT(5, ROLL_MEAN(20, volume))` (five volume buckets), `close > open` (up or down candle), `MONTH()`, `CORR_GROUP(...)` (chapter 6) |

- CODE (code columns, `PARENT_ID`) is only a group key: no arithmetic, and it cannot be a whole formula.
- In a condition the industry key can be written `industry`; in a score formula write `PARENT_ID(1)`.
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
| `FN_PS(a, b)` | R | — | (MONEY, QTY) -> MONEY | b <= 0 -> invalid | per-share value = total / shares | `FN_BP(FN_PS(equity, shares), close)` |
| `FN_EP(a, b)` | R | — | (MONEY, PRICE) -> RATIO | b <= 0 -> invalid; a must be a per-share value | 4 x earnings per share / price (the inverse of P/E, quarter to year) | `FN_EP(eps, close)` |
| `FN_BP(a, b)` | R | — | (MONEY, PRICE) -> RATIO | a <= 0 or b <= 0 -> invalid; a must be a per-share value | book value per share / price (the inverse of P/B) | `FN_BP(FN_PS(equity, shares), close)` |
| `FN_ROE(a, b)` | R | — | (MONEY, MONEY) or (AMOUNT, AMOUNT) -> RATIO | b <= 0 -> invalid | 4 x net income / equity | `FN_ROE(net_income, equity)` |
| `FN_YOY(N, x)` | R | N 1..1,260 | X -> RATIO | either <= 0 -> invalid; warm-up N | ln(x[t] / x[t-N]): log growth (0.1 is about 10%) | `FN_YOY(252, revenue_m)` |
| `FN_EGROW(N, x)` | R | N 1..252 | X -> RATIO | base 0 -> invalid; warm-up N | (x[t] - x[t-N]) / \|x[t-N]\|: works with a negative base | `CS_PCT(FN_EGROW(252, eps))` |
| `FN_GM_CHG(N, a, b)` | R | N 1..252 | (MONEY, MONEY) -> RATIO | b <= 0 -> invalid; warm-up N | a = gross profit, b = revenue: gross margin - the gross margin N days ago | `FN_GM_CHG(252, fund.gross, revenue)` |
| `FN_ACCRUAL(N, a, b)` | A | N 1..252 | (MONEY, MONEY) -> RATIO | b <= 0 -> invalid; warm-up N | a = accruals, b = assets: 4 x a / the mean of today's assets and the assets N days ago | `NEG(FN_ACCRUAL(252, SUB(net_income, fund.cfo), assets))` |
| `FN_DY(N, a, b)` | R | N 1..252 | (MONEY, PRICE) -> RATIO | b <= 0 -> invalid; a must be a per-share dividend column that has a value only on ex-dividend days; warm-up N-1 | dividends of the last N days / price = dividend yield | `FN_DY(250, dps, raw.close)` |

Per-share values and totals do not mix: the first input of `FN_EP`, `FN_BP`, `FN_DY` must be a per-share value (`eps`, `dps`, `FN_PS(...)`); two money inputs where one is per-share and the other a total are rejected as well. To compare a total with market cap write `RATIO(total, fund.mktcap)`.

### 4.11 Calendar

| Signature | Layer | Params | Units | Invalid and warm-up | Meaning | Example |
|---|---|---|---|---|---|---|
| `MONTH()` | A | — | -> COUNT 1..12 | panel without a calendar -> invalid | month | `MASK(MONTH() == 1, NEG(RET(20, close)))` |
| `WEEKDAY()` | A | — | -> COUNT 0..6 | as above | weekday, 0 = Monday, 4 = Friday | `MASK(WEEKDAY() == 0, NEG(RET(1, close)))` |
| `MDAY()` | A | — | -> COUNT 1..31 | as above | day of the month | `MASK(MDAY() <= 10, RET(5, close))` |
| `YEAR()` | A | — | -> COUNT | as above | calendar year | `MASK(YEAR() >= 2020, RET(60, close))` |
| `EPOCH_DAY()` | A | — | -> DAYS | as above | days since 1970-01-01 (calendar days, market holidays included); week number = `FLOOR((EPOCH_DAY() + 3) / 7)` (weeks start on Monday) | `EPOCH_DAY() - LAG(1, EPOCH_DAY()) >= 4` (first trading day after a long holiday) |
| `DAY_INDEX()` | A | — | -> DAYS | always valid | panel day number (0 on the first day) | `SCANSUM(DAY_INDEX() < 1, volume)` |

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

### 4.13 Atoms with three or four inputs

| Name | Inputs | Meaning | Status |
|---|---|---|---|
| `SELECT(c, a, b)` | (BOOL, X, X) -> X | c invalid -> invalid; a where c is true, else b | not accepted in formula text; write `c ? a : b` (= `FILL(MASK(c, a), MASK(NOT(c), b))`) |
| `REC2(K, x, c0, c1, c2)` | x: X; coefficients RATIO -> X; K 0..756 | second-order linear recursion y[t] = floor((c1 x y[t-1] + c2 x y[t-2] + c0 x x[t]) / 1e6); y = x on the two starting days; invalid for K steps after a start | not accepted in formula text |
| `STATESCAN(x, up, lo)` | all three the same unit -> X | conditional state scan: in the up state, x falling below the previous value switches to the down state, otherwise take max(previous, up); the down state mirrors it | not accepted in formula text |

A node of a formula tree has only two children, so these three do not fit; writing one returns `APX_UNSUPPORTED`.

---

## 5. Reserved functions as atoms

There are only two layers: an **atom** is a smallest operation, implemented directly by the engine; a **reserved function** is composed of atoms and always decomposes back into atoms. There are two kinds of reserved function:

- **Marked R in chapter 4**: their names are in the operator tables. 5.1 lists the atom definition of every one, bit-identical to writing the name.
- **Expanded at text level** (5.2): replaced by atoms when the formula is read, with no operator number of their own. The expanded nodes count toward the formula size.

### 5.1 Reserved functions marked R and their atom definitions (38)

The reserved function and the atom form on the right are bit-identical (value, valid bits, unit). An empty "Condition" = for all inputs; otherwise bit-identical within that range.

| Reserved function | Atom definition | Condition |
|---|---|---|
| `DIFF(N, x)` | `SUB(x, LAG(N, x))` | |
| `OR(a, b)` | `NOT(MUL(NOT(a), NOT(b)))` | |
| `MAX2(a, b)` | `FILL(MASK(GT(a, b), a), MASK(VALID(a), b))` | |
| `MIN2(a, b)` | `FILL(MASK(LT(a, b), a), MASK(VALID(a), b))` | |
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
| `FIRST_IN_WINDOW(N, b)` | `MUL(b, LT(WSUM(N-1, LAG(1, b)), 1:COUNT))` | b only 0/1 (comparisons, `NOT`, `OR`, `VALID`, `EV_PULSE`, `EV_CHANGE` and their `LAG`, `MASK`, `FILL`, BOOL products) |
| `COUNT(N, b)` | `WSUM(N, b)` | |
| `EV_POST_N(N, b)` | `LAG(N, b)` | |
| `EV_PULSE(c)` | `VALID(c)` | c is a column |
| `FN_PS(a, b)` | `MASK(GT(b, 0), RATIO(a, b))` |  |
| `FN_BP(a, b)` | `MASK(MUL(GT(a, 0), GT(b, 0)), RATIO(a, b))` |  |
| `FN_EP(a, b)` | `MASK(GT(b, 0), MULDIV(4, a, b))` |  |
| `FN_ROE(a, b)` | `MASK(GT(b, 0), MULDIV(4, a, b))` |  |
| `FN_DY(N, a, b)` | `MASK(GT(b, 0), RATIO(WSUM(N, FILL(a, 0:MONEY)), b))` | a >= 0 (dividends) |
| `FN_EGROW(N, x)` | `RATIO(SUB(x, LAG(N, x)), ABS(LAG(N, x)))` |  |
| `FN_GM_CHG(N, a, b)` | `DIFF(N, MASK(GT(b, 0), RATIO(a, b)))` |  |
| `CS_REL_RET(N, L, x)` | r = `RET(N, x)`: `MASK(GT(GCOUNT(L, r), 1:COUNT), SUB(r, RATIO(GSUM(L, r), GCOUNT(L, r))))` |  |
| `CS_RANK_MEAN(N, x)` | p = `CS_PCT(x)`: `MASK(MUL(VALID(p), NOT(LT(MUL(2:COUNT, WSUM(N, VALID(p))), N:COUNT))), RATIO(WSUM(N, FILL(p, 0:RATIO)), WSUM(N, VALID(p))))` |  |
| `RVOL(N, x)` | `ROLL_STD(N, LOGR(x, LAG(1, x)))` |  |
| `EV_DAYS_SINCE(N, b)` | s = `SUB(SCANSUM(FILL(b, 0:BOOL), 1:DAYS), 1:DAYS)`: `MASK(NOT(GT(s, N:DAYS)), s)` |  |
| `UP_STREAK(N, x)` | c = `FILL(GT(x, LAG(1, x)), 0:BOOL)`: `MASK(MUL(VALID(x), VALID(LAG(N, VALID(x)))), MIN2(SCANSUM(NOT(c), MUL(c, 1:COUNT)), N:COUNT))` |  |
| `DOWN_STREAK(N, x)` | as `UP_STREAK` with `LT` in place of `GT` |  |
| `EV_EARN_DRIFT(N, a, b)` | e = `EV_CHANGE(b)`, base = `LATCH(e, LAG(1, a))`, s = `SUB(SCANSUM(e, 1:DAYS), 1:DAYS)`: `MASK(MUL(GT(base, 0), LT(s, N:DAYS)), RATIO(SUB(a, base), base))` |  |
| `EV_IDX_CHG(N, b)` | e = `EV_CHANGE(b)`, s = `SUB(SCANSUM(e, 1:DAYS), 1:DAYS)`, m = `RATIO(SUB(N:DAYS, s), N:DAYS)`: `MASK(VALID(WSUM(N+1, b)), FILL(MASK(LT(s, N:DAYS), FILL(MASK(b, m), MASK(NOT(b), NEG(m)))), 0:RATIO))` | b only 0/1 |
| `CS_NEUTRAL_COUNTRY(a, b)` | `CS_NEUTRAL_IND(3, a, b)` |  |
| `VOL_RATIO(N, x)` | s = `WSUM(N, LAG(1, x))`: `MASK(GT(s, 0), MULDIV(N, x, s))` | x >= 0 (volume, turnover) |

- `2*(N-1)`, `N-1`, `N+1`, `k/10` in a definition are the numbers after the parameter is substituted (the divisor of `ROLL_RANK(60, x)` is `118:COUNT`).
- A leading "s = ...", "c = ..." in a long definition only writes a repeated sub-formula once; expanded, it is the same atom tree.
- N of `RET`, `RET_SKIP`, `DIST_HIGH`, `DIST_LOW`, `FN_YOY` goes up to 1,260: above 252 the formula is expanded into the atom definition of the table when it is read (`LAG`, `ROLL_MAX` / `ROLL_MIN`, `LOGR` already go to 1,260), and the nodes count as expanded; whatever N is, it is the same definition with the same result.
- The division in `FN_PS`, `FN_BP`, `FN_EP`, `FN_DY` uses the two unit rules of chapter 3: money / shares -> money per share (MONEY), money per share / price -> ratio. The reserved functions also check that the first input is a per-share value; the atom forms do not.

### 5.2 Reserved functions expanded at text level (29)

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
| `LISTED_TDAYS()` | `SUB(cal.tday_idx, list.tday_idx)` | COUNT | trading days since listing: counted from the actual listing date, halted days included; the listing day = 0; stocks listed before 2000-01-04 count from 2000-01-04; invalid before listing. Needs the panel columns `cal.tday_idx` and `list.tday_idx` | `MASK(LISTED_TDAYS() >= 500, RET(60, close))` |
| `MKT_MEAN(x)` | `RATIO(GSUM_ALL(0, x), GCOUNT_ALL(0, x))` | unit of x | the equal-weighted whole-market mean of x (floored), the same value for every stock. Members = every stock with x valid that day; the stock pool is not applied and there is no common-stock filter; no stock valid -> invalid | `RET(20, close) - MKT_MEAN(RET(20, close))` |
| `EW_INDEX(p)` | `EXP(SCANSUM(DAY_INDEX() < 1, FILL(LOG(1 + MKT_MEAN(MASK(common == 1, RATIO(p, LAG(1, p)) - 1))), 0)))` | RATIO | the equal-weighted market: common stocks rebalanced to equal weight daily, 1 on the panel's first day, the same value for every stock; a stock with p invalid today or yesterday is left out that day; write `close` for p. A panel that cannot tell common stocks -> error | `EW_INDEX(close) > ROLL_MEAN(250, EW_INDEX(close)) ? CS_PCT(RET(60, close)) : CS_PCT(NEG(RET(60, close)))` |
| `STREAK(b)` | `SCANSUM(NOT(b), MUL(b, 1:COUNT))` | COUNT | number of consecutive days b is true (false today = 0); invalid on a day b is invalid and until b is next false; invalid before the first false | `STREAK(foreign_net > 0) >= 3` |
| `SINCE(b)` | `SCANSUM(b, 1:DAYS) - 1:DAYS` | DAYS | trading days since b was last true (0 on a true day); invalid on a day b is invalid and until b is next true; invalid before the first true | `SINCE(EV_CHANGE(revenue_m) && revenue_m >= ROLL_MAX(250, revenue_m)) <= 20` |
| `STATE(on, off)` | `LATCH(on \|\| off, MUL(on, 1:COUNT)) > 0:COUNT` | BOOL | true from a day on is true, false from a day off is true, held otherwise; both true the same day = true; a day with on or off invalid keeps the previous state; invalid before the first on or off | `STATE(close > ROLL_MEAN(250, close), close < ROLL_MEAN(60, close))` |
| `GMEAN(first, x)` | `RATIO(GSUM(first, x), GCOUNT(first, x))` | unit of x | mean of x over the group's members (floored), written to every member; members = in the stock pool that day with x valid (and the key valid for a group key); first as in GSUM: a unitless integer = level (0 whole market), anything else = group key; COUNT x gives RATIO; non-members invalid | `GMEAN(CS_QCUT(10, fund.mktcap), RET(20, close))` |
| `GDEMEAN(first, x)` | `MASK(GCOUNT(first, x) > 1:COUNT, x - GMEAN(first, x))` | unit of x | x minus its group mean (GMEAN); a group of one -> invalid; non-members invalid. x needs a 1e6 fixed-point unit (PRICE, RATIO, MONEY, ...) | `GDEMEAN(CORR_GROUP(RATIO(close, LAG(1, close))), RET(20, close))` |
| `GTOP(N, first, x)` | `ORD(first, x) < N` | BOOL | x is among the N largest in its group (exact ranks: exactly N per group, all of a smaller group); first = a level number (0 whole market, 1 industry, ...) or a group key; equal values broken by a fixed shuffle, the same rule as the stock picking ranks; non-members invalid | `MASK(GTOP(3, 1, fund.mktcap), RET(20, close))` (the 3 largest in each industry) |
| `CS_TOPK(k, x)` | `ORD(0, x) < k` | BOOL | x is among the k largest in the stock pool that day (exactly k stocks); equal values broken by a fixed shuffle; x invalid or outside the pool -> invalid | `MASK(CS_TOPK(50, fund.mktcap), RET(20, close))` |
| `MKT_SHARE(b)` | `MKT_MEAN(MASK(common == 1, MUL(b, 1:RATIO)))` | RATIO | share of common stocks with b true, 0..1 (floored to 1e-6), one value for every stock; denominator = common stocks with b valid that day, stock pool not applied; none valid -> invalid; no common-stock flag -> error | `MKT_SHARE(close > ROLL_MEAN(60, close)) < 0.3 ? 0.5 : 1` |
| `MKT_COUNT(b)` | `GSUM_ALL(0, MASK(common == 1, b))` | COUNT | number of common stocks with b true, one value for every stock; counts common stocks with b valid, stock pool not applied; none valid -> invalid; no common-stock flag -> error | `MKT_COUNT(raw.close >= raw.limit_up)` |
| `BASKET_MEAN(sel, x)` | `MKT_MEAN(MASK(sel, x))` | unit of x | equal-weighted mean of x over the stocks with sel true (floored), written to every stock; stock pool not applied; none of them with x valid that day -> invalid | `CORR(250, RET(1, close), BASKET_MEAN(symbol == "2330", RET(1, close)))` |
| `CLIP(x, lo, hi)` | `MAX2(MIN2(x, hi), lo)` | unit of x | x limited to lo..hi (lo when lo > hi); any invalid -> invalid; all three in the same unit | `CLIP(RET(1, close), -0.1, 0.1)` |
| `BETWEEN(x, lo, hi)` | `x >= lo && x <= hi` | BOOL | lo <= x <= hi (inclusive); any invalid -> invalid; all three in the same unit | `BETWEEN(CS_PCT(fund.mktcap), 0.3, 0.7)` |
| `SIGN(x)` | `MUL(x > 0, 1:COUNT) - MUL(x < 0, 1:COUNT)` | COUNT | sign of x: -1 / 0 / 1; x invalid -> invalid | `GSUM(0, SIGN(RET(1, close)))` |
| `CROSS_UP(a, b)` | `a > b && !(LAG(1, a) > LAG(1, b))` | BOOL | the day a moves above b (a > b today, a <= b yesterday); any of today's or yesterday's values invalid -> invalid; same unit | `CROSS_UP(close, ROLL_MEAN(20, close))` |
| `CROSS_DOWN(a, b)` | `a < b && !(LAG(1, a) < LAG(1, b))` | BOOL | the day a moves below b (a < b today, a >= b yesterday); any of today's or yesterday's values invalid -> invalid; same unit | `CROSS_DOWN(foreign_net, 0)` |
| `MONTH_START()` | `MDAY() < LAG(1, MDAY())` | BOOL | 1 on the first trading day of a calendar month, else 0; invalid on the panel's first day | `MONTH_START()` |
| `QUARTER_START()` | `MONTH_START() && (MONTH() - 1) % 3 == 0` | BOOL | 1 on the first trading day of January, April, July and October, else 0; invalid on the panel's first day | `QUARTER_START()` |
| `YEAR_START()` | `MONTH() < LAG(1, MONTH())` | BOOL | 1 on the first trading day of a year, else 0; invalid on the panel's first day | `YEAR_START()` |
| `PREV_MONTH_RET(p)` | `RATIO(AT_MONTH_START(LAG(1, p)), AT_MONTH_START(LAG(1, AT_MONTH_START(LAG(1, p))))) - 1` | RATIO | last calendar month's return = p on last month's final trading day / p on the final trading day of the month before - 1, same value all month; either p invalid (or a zero denominator) -> invalid all month | `CS_PCT(PREV_MONTH_RET(close))` |
| `ROLL_MEAN_AVAIL(N, x)` | `RATIO(WSUM($N, FILL(x, 0)), WSUM($N, VALID(x)))` | unit of x | mean of x over the days it is valid within the last N days (floored; invalid days are left out of the denominator); invalid for the panel's first N - 1 days; all invalid in the window -> invalid; N 1..1,260 | `ROLL_MEAN_AVAIL(20, foreign_net)` |
| `VWAP(N)` | `RATIO(WSUM($N, px.turnover), WSUM($N, volume))` | MONEY | N-day volume-weighted average price = total turnover / total shares traded (TWD per share, unadjusted: compare with raw.close); any invalid cell in the window, zero shares, or a turnover total outside fixed point -> invalid; N 1..1,260. Needs the panel columns px.turnover and volume | `raw.close > VWAP(20)` |

- "Start of month" is `MDAY() < LAG(1, MDAY())` (today's day number is smaller than the previous trading day's). It assumes adjacent trading days are less than 28 calendar days apart.
- Return of the previous calendar month = `PREV_MONTH_RET(close)`; listed for 2 years = `LISTED_TDAYS() >= 500`; the highest price since listing = the `LISTED_DAYS` example.
- `MKT_MEAN`, `EW_INDEX`, `MKT_SHARE`, `MKT_COUNT`, `BASKET_MEAN`, `MONTH_START`, `QUARTER_START` and `YEAR_START` give the same value for every stock, so they can go straight into `exposure` and `rebalance_when` (chapter 9).
- Inputs marked X in one function are the same kind of quantity: a constant without a unit takes the unit of the one that has it (in `CLIP(close, 10, 100)`, 10 and 100 are PRICE). The first argument of `GMEAN` and `GDEMEAN` works as in `GSUM`: an integer without a unit = level (0 whole market, 1..4), anything else = group key.
- `EW_INDEX` measured on the Taiwan panel (4,127 days): maximum relative error against a double-precision reference 6.07e-3, and 3.86e-4 inside any 250 days.
- `SQRT` on integer units is also a text-level expansion: for COUNT / DAYS, `SQRT(n)` = `SQRT(RATIO(n, 1:same unit))`; for AMOUNT, `SQRT(a)` = `MUL(1000, SQRT(RATIO(a, 1000000:AMOUNT)))` (three decimals).

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
| `idx.` | market level: indices, the same value for every stock | `idx.taiex`, `idx.taiex_tr`, `idx.tpex`, `idx.tpex_tr` |
| `etf.` | market level: ETF closes, the same value for every stock | `etf.0050`, `etf.0050_adj` |
| `list.` | listing information | `list.date`, `list.tday_idx` |
| `cal.` | calendar, the same value for every stock | `cal.tday_idx` |
| `sym.` | the numeric value of the stock code | `sym.code_num` (2330 -> 2330) |
| `news.` | material announcements | `news.major_cnt` |
| `tdcc_` | shareholding distribution (weekly) | `tdcc_people`, `tdcc_big1000_pct` |
| `dt_`, `sbl_` | day trading, securities lending | `dt_vol`, `sbl_vol` |

Four code columns are not panel columns: `industry`, `market`, `common`, `symbol` (section 2.4).

### 7.2 How to find the columns

The column table changes with the panel, so it is not copied here.

| Looking for | How |
|---|---|
| the columns and units of a panel | `stock_formula_help topic=column` (optionally `panel=`) |
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
| quarterly statements (`revenue`, `eps`, `fund.*`...) | with a known filing date: the next trading day after it; without one: the next trading day after the statutory deadline (annual report 3/31, Q1 5/15, Q2 8/14, Q3 11/14) | carried from the previous quarter for at most 155 days from publication, then invalid |
| monthly revenue `revenue_m` | switches on each row's publication date (published by the 10th of each month) | carried until the next row |
| institutional net buying (`foreign_net`...) | that day's close | not carried; before 2020 a day without trades is missing, later it is 0 |
| margin columns (`margin_bal`...) | that day's close | carried; stocks without margin eligibility have no value |
| shareholding distribution (`tdcc_*`) | the next trading day after the reference date | carried until the next period |
| cash dividend per share `dps` | the next trading day after the ex-dividend date | an event column: a value on that day only, invalid on every other day |
| market level (`idx.*`, `etf.*`) | that day's close, the same value for every stock | holidays are invalid |
| exchange prices (`raw.*`) | **one row late**: `raw.close` on panel day t is the close of the previous trading day | |

Key points:

- `close`, `open`, `high`, `low` are adjusted prices (dividends and splits adjusted to the latest). Thresholds compare against the adjusted price, not the price quoted that day. For the actual quoted level use `raw.*`.
- The publication day of a carried column (statements, monthly revenue) = the day its value changes, `EV_CHANGE(column)`. For windows counted in periods use `RWIN` (section 4.6) rather than guessing a number of days.
- Sum event columns (`dps`) with `FN_DY` or `WSUM`; summing a carried column over N days counts the same figure many times.
- Market-level columns have the same value for every stock; random search does not pick them as leaves by itself, they are used only where a formula names them.
- `eps` is the average single-quarter EPS of the last four quarters, so `FN_EP(eps, close)` = the inverse of the trailing P/E.

---

## 8. Stock pool

### 8.1 Writing the condition

The stock pool is `universe=...` of `stock_backtest`: one BOOL formula per day and stock; cells where it is true stay. It is written in the same language as chapters 2-4.

| Plain words | Written |
|---|---|
| a close on each of the last 250 trading days | `WSUM(250, VALID(close)) >= 250` |
| listed for two years | `cal.tday_idx - list.tday_idx >= 500` (or `LISTED_TDAYS() >= 500`) |
| OTC only, excluding stocks listed for fewer than 60 trading days | `market == TWO && cal.tday_idx - list.tday_idx >= 60` |
| 20-day average volume above 500 lots, traded every day | `COUNT(20, volume > 0) >= 20 && ROLL_MEAN(20, volume) > 500000` |
| mid caps: market cap between the 30th and 70th percentile | `CS_PCT(fund.mktcap) >= 0.3 && CS_PCT(fund.mktcap) <= 0.7` |
| top 30% by market cap, without TSMC and Hon Hai | `CS_PCT(fund.mktcap) >= 0.7 && !(symbol in {"2330", "2317"})` |
| only among these names | `symbol in {"2330", "2454", "2317"}` |
| no financials, no construction | `!(industry in {17, 14})` or `!cls.FIN() && !cls.REALTY()` |
| largest one-year drop below 40% | `WDROP(250, LOG(close)) < 0.5108` |
| just above the 250-day average (crossed within 20 days) | `close > ROLL_MEAN(250, close) && COUNT(20, close > ROLL_MEAN(250, close)) < 20` |
| margin usage below 30% and short / margin ratio above 10% | `RATIO(margin_bal, margin_limit) < 0.3 && RATIO(short_bal, margin_bal) > 0.1` |
| no limit-up or limit-down close in the last 60 days | `COUNT(60, raw.close >= raw.limit_up \|\| raw.close <= raw.limit_down) == 0` |
| no full-delivery, disposition or attention stocks | `!full_delivery && !disposition && !attention` |
| behaving like these names | `GSUM(CORR_GROUP(RATIO(close, LAG(1, close))), symbol in {"2330", "2382", "3231"}) > 0` |
| the last digit of the code equals the last digit of today's date | `sym.code_num % 10 == MDAY() % 10` |

Rules:

- Cells where the formula is invalid (window not full and warm-up included) are outside the pool.
- Members of cross-section operators (`CS_PCT`, `GSUM`...) = the pool **before** the filter, computed once, not iterated. `CS_PCT(fund.mktcap) >= 0.7` is the top 30% of the whole market.
- One result library is bound to one stock pool. Another pool means another library.
- Fewer than 1 stock per day on average -> error `E_EMPTY_UNIVERSE`.
- At most 20 symbols may be listed one by one (counted over the whole condition). More than that is listing a whole kind of stock: use a class noun or a seed basket.
- The plan returns `effective.universe_stats`: mean stocks per day, stocks on the last day, sample codes. To preview only: `stock_formula_help topic=universe universe="..."`.

### 8.2 Default = all common stocks

Without `universe` the pool is every common stock of the market.

- `common_only` (default true): stocks that are certainly not common stocks (preferred shares, ETFs, beneficiary certificates, warrants, depositary receipts) are removed first, then `universe` applies. To include ETFs and the like: `common_only=false`.
- The code column `common`: 1 = common stock, 0 = not, invalid = cannot tell.
- Conditions the user did not state are not added. A liquidity threshold or a listing-age rule is not a default.

### 8.3 Class nouns

Official nouns `cls.NAME()`: one kind of stock, written the same way in every market. The definition = a condition on that market's own classification column (industry codes in Taiwan). Used only in conditions.

| Noun | Meaning | Taiwan definition |
|---|---|---|
| `cls.FIN()` | financials | `industry in {17}` |
| `cls.SEMI()` | semiconductors | `industry in {24}` |
| `cls.TECH()` | technology, electronics (semiconductors included) | `industry in {24, 25, 26, 28, 29, 30, 31, 36}` |
| `cls.COMM()` | communication | `industry in {27}` |
| `cls.HEALTH()` | health care, biotech | `industry in {22}` |
| `cls.STAPLES()` | consumer staples | `industry in {2, 18, 33}` |
| `cls.DISCR()` | consumer discretionary | `industry in {4, 11, 12, 16, 32, 37, 38}` |
| `cls.INDU()` | industrials | `industry in {5, 6, 15, 35}` |
| `cls.MATER()` | materials | `industry in {1, 3, 8, 9, 10, 21}` |
| `cls.ENERGY()`, `cls.UTIL()` | energy, utilities (one definition in Taiwan) | `industry in {23}` |
| `cls.REALTY()`, `cls.CONSTR()` | real estate, construction | `industry in {14}` |
| `cls.SHIPPING()` | shipping | `industry in {15}` |
| `cls.STEEL()` | steel | `industry in {10}` |
| `cls.PLASTIC()` | plastics | `industry in {3}` |
| `cls.ECOMP()` | electronic components | `industry in {28}` |
| `cls.PC()` | computers and peripherals | `industry in {25}` |
| `cls.OPTO()` | optoelectronics | `industry in {26}` |
| `cls.BIO()` | biotech | `industry in {22}` |
| `cls.GREEN()` | green energy | `industry in {35}` |

- Exclude with `!cls.FIN()`; keep only with `cls.SEMI()`. Classes may overlap (`TECH` contains `SEMI`).
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
| Functions | element-wise `abs sqrt log exp has pow min max where`; cross-section of the day `sum mean sd var count quantile min max` | chapter 4 |
| Example | `fund.mktcap >= quantile(fund.mktcap, 0.7)`, `industry == 24 && fund.mktcap > 10e9` | `CS_PCT(fund.mktcap) >= 0.7`, `industry in {24} && fund.mktcap > 10e9` |

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
| `max_positions` | 1..500 | omitted: tries 20 and 50, keeps one | hold at most N stocks | `{"max_positions":10}` |
| `weighting` | `equal` / `score` / `inv_vol` | `equal` | equal weight / by rank (higher weighs more) / by inverse volatility (steadier weighs more) | `{"weighting":"inv_vol","max_weight":15}` |
| `max_weight` | 0..100 (%) | 0 (no cap) | cap per stock; the excess stays in cash | `{"max_positions":10,"max_weight":15}` |
| `hold_days` | 2..60 | not used | pick every day; each day's purchase is one slice held N trading days and then sold (N staggered slices) | `{"hold_days":20}` |
| `hold` | BOOL formula | not used | a condition per day and stock: ranks stay on the whole market, names that fail are not bought and are sold on the rebalance day | `{"hold":"close > ROLL_MEAN(60, close)"}` |
| `rebalance_when` | BOOL formula (one value per day for the market) | not used | rebalance on the days it holds | `{"rebalance_when":"MDAY() < LAG(1, MDAY())"}` |
| `exposure` | RATIO formula 0..1 (one value per day for the market) | 1 (fully invested) | total exposure; the rest stays in cash | `{"exposure":"idx.taiex < ROLL_MEAN(200, idx.taiex) ? 0.5:RATIO : 1:RATIO"}` |
| `weight` | numeric formula (per day and stock) | not used | on a rebalance day each stock's weight is in proportion to this value | `{"max_positions":20,"weight":"fund.mktcap"}` |
| `tie_by` | numeric formula (per day and stock) | not used (ties follow the fixed shuffle) | when picking the top N, stocks with the same score are ordered by this value, larger first; stocks without a value come last among the tied; equal values then `mix64((entity_id<<32)^day)`, then the symbol index. Only orders ties: the score and the weights are unchanged and the warm-up is not raised; for smaller first write `NEG(...)` | `{"tie_by":"fund.mktcap"}` |
| `group_by` | formula (per day and stock; a number or a condition, no `pos.*`) | not used | what a group is: stocks with the same value on a day are one group; given together with `group_cap` (9.5a) | `{"group_by":"CORR_GROUP(RET(1, close))","group_cap":3}` |
| `group_cap` | 1..255 | not used | at most this many holdings per group: on a rebalance day names are taken in rank order, a name whose group is full is skipped (9.5a) | as above |
| `add_when` | BOOL formula, may use `pos.*` and `pos.size`, `pos.adds`, `pos.cuts`, `pos.avg_ret` | not used | add condition: at the close where it turns from false to true for a holding, add once at the next fill (9.5b) | `{"add_when":"pos.ret >= 0.1","size_init":0.5}` |
| `cut_when` | as `add_when` | not used | cut condition: when it triggers, sell `cut_size` of the current position (9.5b) | `{"cut_when":"pos.ret >= 0.2"}` |
| `add_size` | > 0 | 1 | units per add (multiples of the entry unit) | `{"add_when":"pos.ret >= 0.1","add_size":0.5}` |
| `cut_size` | 0 < x < 1 | 0.5 | share of the current position sold per cut; to sell all write `exit_when` | `{"cut_when":"pos.ret >= 0.2","cut_size":0.3}` |
| `size_init` | 0.01..1 | 1 | share of the target weight bought at entry (cash kept for adds) | `{"add_when":"pos.ret >= 0.1","size_init":0.5}` |
| `max_size` | >= 1 | no cap | cap per holding, in entry units | `{"add_when":"pos.ret >= 0.1 * (pos.adds + 1)","max_size":3}` |
| `gross_max` | 0.01..2 | 1 | gross cap: weights may sum to at most this share of the account; > 1 = margin (give `margin_rate`) (9.5b) | `{"gross_max":0.8}` |
| `cash_min` | 0..0.99 | 0 | minimum cash share; the stricter of it and `gross_max` applies | `{"cash_min":0.1}` |
| `add_mode` | `cash` / `scale` | `cash` | adds beyond the gross cap: only the room left / shrink the other holdings | `{"add_mode":"scale"}` |
| `max_new` | >= 1 | no limit | at most this many new names per rebalance (the best-ranked; the other slots stay cash) | `{"max_new":3}` |
| `cash_rate` | annual rate, e.g. 0.015 | 0 | annual interest on cash (long-only) | `{"cash_rate":0.015}` |
| `margin_rate` | annual rate, e.g. 0.05 | not used | annual margin rate (required when `gross_max` > 1) | `{"gross_max":1.5,"margin_rate":0.05}` |
| `exit_when` | BOOL formula, may use `pos.*` | not used | exit condition: checked at every close, sold when true without waiting for the rebalance day | `{"exit_when":"pos.ret <= -0.08 \|\| pos.ret >= 0.2"}` |
| `keep_when` | BOOL formula, may use `pos.*` and `pos.rank` | not used | keep condition: on a rebalance day, holdings for which it is true stay | `{"max_positions":10,"keep_when":"pos.rank <= 30"}` |
| `reentry_days` | 0..10,000 | 0 | after an `exit_when` exit the same stock is not bought back at rebalances within N trading days | `{"exit_when":"pos.ret <= -0.08","reentry_days":20}` |
| `exit_below` | PRICE formula, may use `pos.entry`, `pos.peak` | not used | intraday stop level: sold once the day's low touches it, without waiting for the close (9.6a) | `{"exit_below":"pos.entry * 0.92"}` |
| `exit_above` | PRICE formula, may use `pos.entry`, `pos.peak` | not used | intraday take-profit level: sold once the day's high touches it (9.6a) | `{"exit_above":"pos.entry * 1.2"}` |
| `lock` | `limit` / `none` | omitted: simulated when the panel has the exchange limit prices | rule exits on locked limit days: `limit` moves a blocked sale to the next trading day, `none` does not simulate locks (9.6a) | `{"exit_below":"pos.entry * 0.92","lock":"none"}` |
| `grid` | an object: name -> values to try | not used | threshold grid: numbers to try in `exit_when` / `keep_when` / `exit_below` / `exit_above` are written `$name` and each name's values are given here (9.11) | `{"exit_when":"pos.drawdown >= $dd","grid":{"dd":[0.06,0.1,0.14]}}` |
| `grid_by` | a screening expression (the ordering, 11.3) | with `grid`: `apx.SHARPE`; without: combined score | orders a formula's combinations (larger first); without `grid` it orders the portfolio settings | `{"exit_when":"pos.drawdown >= $dd","grid":{"dd":[0.06,0.1,0.14]},"grid_by":"apx.CALMAR"}` |
| `initial_capital` | 1..1e12 | 1,000,000 | starting capital; given = a share-level account is computed | `{"initial_capital":4000000}` |
| `lot` | `odd` / `board` | `odd` | odd lots (1 share) / board lots only (1,000 shares) | `{"lot":"board"}` |
| `min_fee` | 0..10,000 | 20 | minimum commission per order | `{"min_fee":20}` |
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

### 9.4 `hold`, `exposure`, `weight`

**`hold`** (BOOL per day and stock): ranks and factor metrics are still computed on the original stock pool; this strategy only buys and holds stocks for which the condition is true. An invalid cell counts as failing.

**`exposure`** (one RATIO per day for the whole market):

- The value used = floor(value x 100) clamped to 0..100: whole percentage points, and it may change every day.
- A change costs only the difference traded; the rest stays in cash. On a day when only the exposure changes there is no selection and no rebalance is counted.
- No stock valid that day -> the previous day's value. 100 throughout -> treated as not given.

| Plain words | Written |
|---|---|
| half invested while TAIEX is below its 200-day average | `idx.taiex < ROLL_MEAN(200, idx.taiex) ? 0.5:RATIO : 1:RATIO` |
| half invested while the equal-weighted market is below its 250-day average | `EW_INDEX(close) < ROLL_MEAN(250, EW_INDEX(close)) ? 0.5 : 1` |
| 30% invested when the equal-weighted market is more than 15% off its one-year high | `RATIO(EW_INDEX(close), ROLL_MAX(250, EW_INDEX(close))) < 0.85 ? 0.3 : 1` |
| 30% invested on a day the whole market falls more than 2% on average | `MKT_MEAN(RATIO(close, LAG(1, close)) - 1) < -0.02 ? 0.3 : 1` |
| out of the market on a Friday the 13th | `MDAY() == 13 && WEEKDAY() == 4 ? 0:RATIO : 1:RATIO` |
| inversely to the market's 20-day volatility, at most fully invested | `MIN2(RATIO(0.01:RATIO, ROLL_STD(20, FILL(RATIO(GSUM(0, LOGR(close, LAG(1, close))), GSUM(0, MASK(VALID(LOGR(close, LAG(1, close))), 1:RATIO))), 0:RATIO))), 1:RATIO)` |

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

Price x volume (`MUL(close, volume)`) fails the unit rules; for traded value use the column `px.turnover`.

**`weighting`** (when `weight` is not given):

| Value | Weights |
|---|---|
| `equal` | 1 / number of stocks each; with `max_weight` and (number of stocks x cap) below 100%, each = the cap and the rest is cash |
| `score` | by rank: higher weighs more; weights are rounded to basis points and the remainder goes to the top-ranked stock |
| `inv_vol` | by inverse volatility: steadier weighs more; stocks with zero volatility are not candidates |

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

Shape rules:

| Rule | Note |
|---|---|
| `pos.*` goes inside a comparison | `>`, `>=`, `<`, `<=` (or `GT`, `LT`) |
| one side is linear in the state leaves | times a constant, plus or minus a constant; coefficients within +-100 |
| the other side is a constant, or a whole market-only formula | for instance `-2 * ROLL_STD(20, RET(1, close))` |
| comparisons are joined with `&&`, `\|\|`, `!` | at most 6 comparisons per rule (equal ones count once); more -> `E_POS_ATOMS` |
| a sub-condition without `pos.*` | is one market condition, compiled like any condition (for instance `close < ROLL_MEAN(20, close)`) |

Behaviour:

- **`exit_when`**: checked at every close for every holding; when true the holding is sold at the next fill without waiting for the rebalance day. Its share stays in cash until the next rebalance and the other weights are untouched.
- **`keep_when`**: on a rebalance day, holdings for which it is true stay, and only the remaining slots are filled by rank. Names that fail `hold` or left the stock pool are still sold.
- **`reentry_days`**: after an `exit_when` exit the same stock is not bought back at rebalances within N trading days. 0 = only the rebalance on the exit day is blocked.
- When a comparison reads an invalid cell, the rule does not hold for that stock.
- `summary.rule_exits_by_rule` in the report counts the exits per rule (each part joined by `||`).
- Exit rules are a setting of the whole search: every strategy follows them, whether its formula was sampled, given, or enumerated from a user function.

### 9.5a Group cap: `group_by`, `group_cap`

"At most N holdings per group". The two arguments go together: `group_by` says what a group is (a formula per day and stock; **stocks with the same value on a day are one group**), `group_cap` is the number of holdings per group (1..255).

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

**Adding and cutting**: `add_when` / `cut_when` trigger once at the close where the condition turns from false to true (staying true acts once), adding or cutting at the next fill; a later reversal does not undo it. The other way is a `weight` that reads `pos.*` (e.g. `"weight": "fund.mktcap * (pos.peak_ret >= 0.1 ? 2 : 1)"`), setting a target weight by state; use one of the two.

New leaves (only in `add_when`, `cut_when` and exit rules): `pos.size` (current position / entry unit), `pos.adds`, `pos.cuts` (adds and cuts filled), `pos.avg_ret` (return against the average price after adds); intraday touch levels may also use `pos.avg` (the average price; the entry price when nothing was added), e.g. `"exit_below": "pos.avg * 0.92"`.

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

A level formula = the extreme of 1-3 terms (`MAX2` for `exit_below`, `MIN2` for `exit_above`); each term is `constant x pos.entry + constant x pos.peak + a market-only price formula`.

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
| Buy cost | commission 0.1425% | |
| Sell cost | commission 0.1425% + transaction tax 0.3% | the tax on ETFs is 0.1% |
| Minimum commission | 20 per order (`min_fee`) | counted only in the share-level account |
| Turnover cap | none | `max_turnover_bp` in `config` |

- Costs and trading rules follow the market table of the panel's market (the values above are Taiwan's; other markets use their own).
- They can be overridden through `config` of `stock_backtest`: `fee_bp_x100` (in 0.01 basis points, default 1425), `tax_sell_bp_x100` (default 3000), `tax_sell_etf_bp_x100` (default 1000); `cost_bp` >= 0 = the legacy model, N basis points per side and no separate tax.
- Strategy equity accumulates additively (no compounding); the annual return and maximum drawdown in reports use compounding (chapter 11).
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

Commission per order = max(`min_fee`, floor(traded amount x 0.1425%)).

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
- Limits: 4 names, 1..16 values each, at most 64 sets in all.
- A value must not change how many days the condition looks back: window lengths such as an average's days are not accepted and are searched separately.
- Every set runs. Per formula and portfolio setting the sets are compared by `grid_by` (default `apx.SHARPE`) and one is kept; the set it used is in `program.exit_when` of the report, and the numbers of every set are in `exit_grid` of the job summary.
- Every set is one trial in the multiple-testing count (10.3).

### 9.12 Backtest rules as formulas: `weight` + `rebalance_when`

Every argument above is shorthand for two formulas. You can write the formulas directly; plan maps them back to the same rules (the numbers are bit-identical):

- `weight`: each stock's target weight every day, written as a product of factors. It must have one factor `apx.SHARE(apx.PICK(N, keep), key)` (which names and how to split); the other factors are exposure, exits and sizing.
- `rebalance_when`: rebalance days. `apx.EVERY(21)` = every 21 trading days; or a condition with one value per day for the market (9.3).
- `hold` is still the candidate condition, with two more kinds of factor: `GTOP(k, group key, apx.SCORE())` (at most k per group) and `!FILL(pos.exit_days <= n, false)` (no buy-back within n trading days of an exit; n = 0 blocks the exit day only).
- Tie key: wrap the score formula in `apx.THEN(score, key)`.

| Argument | Formula |
|---|---|
| `max_positions=10` | `weight="apx.SHARE(apx.PICK(10, false), 1)"` (`1` = equal weight) |
| `weighting="score"` / `"inv_vol"` | key `apx.RANK_KEY()` / `apx.INV_VOL_KEY()` |
| `weight="SQRT(fund.mktcap)"` | `apx.SHARE(apx.PICK(10, false), SQRT(fund.mktcap))` |
| `max_weight=15` | `apx.CAP(0.15, apx.SHARE(...))` |
| `keep_when="pos.rank <= 30"` | `apx.PICK(10, FILL(pos.rank <= 30, false))` |
| `max_new=3` | `apx.PICK(10, false, 3)` |
| `exposure="..."` | times `(...)` (the market-wide formula in parentheses) |
| `gross_max=0.8`, `cash_min=0.1` | times `0.8`, times `(1 - 0.1)` |
| `gross_max=1.5` (margin) | times `MIN2(1, 1.5 / acct.gross_req)` (`acct.gross_req` = the sum of target weights at a rebalance; `margin_rate` still given) |
| `exit_when="cond"` | times `!(pos.held && FILL(cond, false))` |
| `exit_below`, `exit_above` | times `!(pos.held && FILL(TOUCH_BELOW(pos.entry * 0.92) \|\| TOUCH_ABOVE(pos.entry * 1.2), false))` |
| `add_when="A"`, `size_init=0.5` | times `FILL(pos.size + 1 * (A && !FILL(LAG(1, A), false) && !apx.EVERY(21)), 0.5)` (`!apx.EVERY(21)`: on rebalance days the trigger is recorded, not traded; wrapping in `MIN2(cap, ...)` = `max_size`) |
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
- The same thing written twice (e.g. `max_positions` and `apx.PICK`, `rebalance_days` and `apx.EVERY`) -> `APX_CONFLICT`; keep one. With a `weight` formula, `max_positions`, `weighting`, `max_weight`, `exposure`, `gross_max`, `cash_min`, `keep_when`, `exit_when`, `exit_below`, `exit_above`, `max_new`, the sizing arguments and `hold_days` go inside the formula.
- Environment facts stay arguments: `initial_capital`, `contribution`, `contribution_every`, `lot`, `min_fee`, `lock`, `cash_rate`, `margin_rate`, `add_mode`, `grid`, `grid_by`.
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
- The same tree (equal after expansion) counts once.

Every tree additionally tries a set of portfolio settings (a base grid of 16: rebalance every 5 / 21 days, at most 20 / 50 positions), and `screen_pick` keeps one. When `rebalance_days` or `max_positions` is given, that axis is fixed to the given value.

### 10.2 Admission thresholds

Thresholds are arguments of the search, not properties of a formula:

| Source | Thresholds not stated |
|---|---|
| random generation | defaults apply |
| given formulas (`formulas`), user-function grids (`nouns`) | none applied: every strategy is computed and recorded; nothing is filtered on the user's behalf |
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

- Scalars and series mix freely (scalars broadcast). Division by 0 = missing (`nan`); a comparison with a missing value is false.
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
| Scores | `cs_bp` (cross-section score), `ts_bp` (time score), `novel_bp` (novelty), `short_bp` (shortness), `total_bp` (combined score), `n_nodes` (nodes of the tree) |
| Strategy summary | `net_ret_1e6` (net return over the period), `max_dd_1e6` (maximum drawdown), `n_rebalances`, `avg_positions_x100`, `breadth_min` (fewest selectable stocks) |
| Portfolio arguments | `reb_days`, `p_long`, `p_short`, `max_positions`, `cost_bp`, `weight_mode` |
| Panel | `n_days`, `day_first`, `day_mid`, `n_symbols` |
| Factor metrics | `ic_x1e4` (mean rank IC), `ic_t_x100` (IC t-value), `ic_pos_x1e4`, `ic_days`, `spread_1e6` (annualised top 10% - bottom 10% spread), `spread_days`, `quint_1e6_1`...`quint_1e6_5` (annualised quintile returns), `quint_days`, `quint_rho_x1e4`, `quint_mono_x1e4`, `turnover_bp_avg`, `turnover_days`, `breadth_med`, `breadth_days`, `ic_decay_x1e4_1` / `_5` / `_21` / `_63`, `ic_decay_days_1` / `_5` / `_21` / `_63`, `ic_half_life_days`, `ic_h1_x1e4`, `ic_h2_x1e4` |
| Exposure | `size_rho` (rank correlation between ranking and market cap), `ind_hhi`, `ind_top`, `n_ind`, `adv_med`, `adv_min` |

The tail of a name states the scale: `_bp` = ten-thousandths, `_1e6` = millionths, `_x100`, `_x1e4`.

**Functions**:

| Kind | Functions |
|---|---|
| Element-wise | `ABS` `SQRT` `LOG` `EXP` `POW(x, y)` `MIN2(x, y)` `MAX2(x, y)` `WHERE(cond, a, b)` `HAS(x)` |
| Reductions (series -> scalar) | `SUM` `MEAN` `SD` `VAR` (sample, n - 1) `COUNT` (number of elements) `FIRST` `LAST` `MIN(x)` `MAX(x)` (missing values skipped; `AMIN` / `AMAX` are the same) `QUANTILE(x, q)` `STREAK(x)` (longest run of true) `LEN` |
| Series -> series | `CUM` `PEAK` `DRAWDOWN` (fall of additive equity) `NAV` (compounded net value) `DRAWDOWN_NAV` (compounded fall) `DIFF` `FILTER(x, cond)` |
| Windows (parameter first) | `ROLL_SUM(n, x)` `ROLL_MEAN(n, x)` `ROLL_STD(n, x)` `ROLL_MIN(n, x)` `ROLL_MAX(n, x)` (complete windows only, length N - n + 1), `SKIP(k, x)` (drop the first k cells), `LAG(k, x)` (shift back k cells, the first k missing, same length) |
| Grouping | `BY_SUM(key, x)` `BY_MEAN` `BY_SD` `BY_MIN` `BY_MAX` `BY_COUNT` `BY_PROD1` (compounded) `BY_KEY(key)` |
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
| `apx.MAX_DD` | `apx.MAX_DD_OF(r)` | maximum drawdown (closes, compounded; as in the report) |
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
| `apx.SORTINO` | `apx.ANN_RET / apx.DOWN_VOL` | Sortino |
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
| `apx.ROLL_SHARPE_MIN` | `MIN(ROLL_MEAN(252, r) / ROLL_STD(252, r)) * SQRT(252)` | minimum rolling one-year annualised Sharpe |
| `apx.ROLL_RET_MIN` | `MIN(EXP(ROLL_SUM(252, LOG(1 + r)))) - 1` | minimum rolling one-year return (compounded): the worst one-year holding period |

`m` (the pool's equal-weighted mean) and `b` (the market index) are two different yardsticks: `apx.IR`, `apx.BETA`, `apx.EXCESS_ANN` use `m`; the `*_B` metrics use `b`.

### 11.3 One threshold, one ordering

Screening has two expressions: the **threshold** (`screen_gate`, kept when true) and the **ordering** (`portfolio.grid_by`, larger first).

**Threshold `screen_gate`**: every candidate must pass. It can use atoms, `apx.*`, `user.*` and check outputs `check.<NAME>.<output>`. When it is evaluated follows from what it refers to: it is split at top-level `&&`; the parts with `check.*` are evaluated after the checks, the rest before the checks (a candidate failing them is not checked). With a top-level `||` and `check.*`, the whole expression is evaluated after the checks (same result, more checks run). Using `check.*` needs that check on (`checks`).

| Plain words | Written |
|---|---|
| at least 60% winning months | `screen_gate="apx.MONTH_WIN >= 0.6"` |
| beats the pool's equal-weighted mean every year | `screen_gate="MIN(BY_SUM(year, r - m)) > 0"` |
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
| Intervals | `full`, `window.is`, `window.oos`, `window.seg` (cut by `windows`: `rolling`, `split`), `{"begin","end"}` |
| Portfolio arguments | `cost_bp`, `p_long`, `p_short`, `rebalance_days`, `weight_mode`, `max_positions`, `max_weight_bp`, `max_turnover_bp` (expressions; `vary` expands several sets) |

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
 "windows": {"kind": "split", "n": 2},
 "runs": [{"id": "seg", "interval": "window.seg"}],
 "outputs": {"seg_ic": "seg.ic_days > 0 ? seg.ic_x1e4 : 0", "lo": "AMIN(seg_ic)", "hi": "AMAX(seg_ic)"},
 "pass": "lo > 0 || hi < 0"}
```

| Field | Meaning |
|---|---|
| `runs` | the re-runs and what each one changes (not written = the original); at most 8 |
| `windows` | how the interval is cut (`rolling`, `split`); at most 32 segments |
| `outputs` | integers computed from the metrics of the runs; at most 16. These are the "screening metrics after the check", read in the threshold `screen_gate` as `check.<NAME>.<output>` |
| `pass` | a condition over `outputs` (and `$params`, `env.*`) |
| `params` | parameters: `{"name","default","min","max"}`, written `$name` in expressions |

**Step 1: what each re-run changes (`runs`).** Each item is one re-run; `id` is required (expressions refer to it). What can change:

| Key | Value | Meaning |
|---|---|---|
| `with` | a comma-separated list of transforms, one transform per item (table below) | the stock pool and the returns are both changed here; not written = original |
| `interval` | `full` (default) | the whole period |
| | `window.is`, `window.oos` | the in-sample and out-of-sample parts of rolling windows (needs `windows.rolling`) |
| | `window.seg` | each equal segment (needs `windows.split`) |
| | `{"begin": "env.n_days - 252", "end": "env.n_days"}` | only some days (day indices, half-open; may use `env.first`, `env.n_days`) |
| portfolio arguments | `cost_bp`, `p_long`, `p_short`, `rebalance_days`, `weight_mode`, `max_positions`, `max_weight_bp`, `max_turnover_bp` | the value is an expression; `prog.FIELD` is the original value, e.g. `"cost_bp": "prog.cost_bp * 3"` |
| `vary` | `{"k": [1, 2, 3]}` or `{"k": {"from": 1, "to": "$mult"}}` | expands the re-run into several (one variable, 1..16 values); a variable named after a portfolio argument is applied directly |

Each item of `with` (commas inside parentheses do not separate items):

| Written as | Changes | Meaning |
|---|---|---|
| a BOOL expression, or `UNIVERSE(expression)` | the stock pool | re-rank and re-run only among the stocks where it is true. Same grammar and units as `universe=` in chapter 8; cross-sectional operators (`CS_PCT`...) see the original pool, computed once. Several items in one run intersect |
| `PEER_SIDE() == 0`, `PEER_SIDE() == 1` | the stock pool | similar stocks paired two by two, each pair split into groups A and B. `PEER_SIDE(ruler)` measures similarity with that expression (none = the `PEER_SWAP.ruler` one) |
| `RANDOM_HALF()`, `RANDOM_HALF(k)` | the stock pool | a random half of the stocks (fixed across periods); k numbers the draw, a different k is a different half |
| `RETURNS(expression)` | the returns | the expression must give a RATIO; `ret` is the original return. It is applied to the full-day, overnight and intraday return each; the intraday price path for touch exits stays as it was |
| `PEER_SWAP()`, `PEER_SWAP(ruler)` | the returns | when a stock is picked, the whole bar of its closest peer on that day is used instead; ranks and holdings unchanged. Same as `REMAP(PEER_IDX())` |
| `SHUFFLE()`, `SHUFFLE(k)`, `SHUFFLE(k, group key)` | the returns | returns shuffled across stocks each day (cross-sectional signal gone, market and distribution unchanged). k numbers the draw; with a group key only stocks with the same key are shuffled among themselves. Same as `REMAP(SHUFFLE_IDX(...))` |
| `REMAP(index expression)` | the returns | each cell takes the whole bar, on the same day, of the stock the expression points at. `PEER_IDX()` and `SHUFFLE_IDX()` give a stock index (an integer); cells where the expression is invalid keep their own. To change only some cells: `REMAP(MASK(condition, PEER_IDX()))` |

- One run may carry several stock-pool conditions and at most one return transform (two kinds: write two runs).
- Expressions may use `$param` and this run's `vary` variable (`RUN.var`); they are replaced by their integer values first. The `vary` values must then be numbers or `$param`.
- One round holds at most 2 different stock pools and 2 different return transforms (a transform used by several runs or checks counts once).
- The old keys `universe` and `ret` and the old names (`universe.peer_a`, `universe.peer_b`, `universe.random_half`, `ret.peer_swap`, `ret.shuffle`) are still accepted; they mean the same as writing them in `with`.

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

**Step 2: add `windows` to cut time.** `{"kind": "rolling", "is_days": 504, "oos_days": 126, "max": 32}`: from the last day backwards every `oos_days`, the in-sample part right before; `{"kind": "split", "n": 4}`: equal segments.

**Step 3: compute outputs (`outputs`).** `RUN.FIELD` reads a metric of one re-run; field names as the scalar atoms of 11.1 (`net_ret_1e6`, `max_dd_1e6`, `ic_x1e4`, `ic_t_x100`, `ic_days`, `spread_1e6`, `quint_1e6_1`...`5`, `turnover_bp_avg`...). A re-run with windows or `vary` is a series of values and needs an aggregate to become one number; an output may only refer to outputs written before it; names must not clash with field names, run ids, parameter names, `env` or `prog`.

**Step 4: write the condition (`pass`).**

**Step 5: validate, install, use.** `stock_formula_help topic=validate document={...}` validates (every error returned) -> `my_stock_strategies_change action=install document={...}` stores it -> a backtest or search with `checks=["user.local.HALVES"]`; parameters go in the brackets in the order of the definition's `params`: `checks=["user.local.HALVES(3000)"]`. At most 8 checks per strategy.

Check expressions are integers (i64): division by 0 = 0; comparisons and logic return 0 / 1.

| Kind | What can be written |
|---|---|
| Values | numbers, `$param`, `env.NAME`, `prog.FIELD`, `RUN.FIELD` (a metric of one run; field names as the scalar atoms of 11.1), `RUN.VAR` (a variable of `vary`) |
| Operators | `+ - * / %`, comparisons, `&& \|\| !`, `?:` |
| Functions | `ABS(x)`, `MIN2(a, b)`, `MAX2(a, b)` (same names as score formulas), `SIGN(x)` (-1 / 0 / 1), `KEEP_BP(variant, original)` (variant / original, in ten-thousandths, e.g. `KEEP_BP(swap.net_ret_1e6, base.net_ret_1e6)`) |
| Aggregates | `COUNT` `SUM` `ALL` `ANY` `AMIN` `AMAX` (the window dimension first, else the `vary` dimension), `FIRST` `LAST`, `NTH(expr, k)` (the k-th of `vary`, from 0), `WINNER(RUN, FIELD, VAR)` (the variable value of the member with the largest FIELD in that run; used in another run's portfolio argument = pick in-sample, use out-of-sample), `n_windows` |
| Old forms | lower-case names are accepted and identified by the upper-case form (same definition, same `content_hash`): `min(a, b)` = `MIN2(a, b)`, `min(x)` = `AMIN(x)`, `keep_bp(...)` = `KEEP_BP(...)`...; `MIN(a, b, c)` answers with the fix `MIN2(MIN2(a, b), c)` |
| `env` | `n_days` (panel days), `first` (first day after warm-up), `n_symbols`, `peer_nn_corr_bp` (average peer correlation), `peer_fallback_bp`, `peer_paired_bp`, `mt_dsr_bp`, `mt_ic_padj_bp`, `mt_n_eff_x100`, `ic_t_x100`, `net_ret_1e6` |

| Plain words | `runs` -> `outputs` -> `pass` |
|---|---|
| IC must also be positive over the last year | `[{"id":"late","interval":{"begin":"env.n_days - 252","end":"env.n_days"}}]` -> `{"late_ic":"late.ic_days > 0 ? late.ic_x1e4 : 0"}` -> `late_ic > 0` |
| profitable with both weekly and monthly rebalancing | `[{"id":"rb","vary":{"rebalance_days":[5,21]}}]` -> `{"worst":"AMIN(rb.net_ret_1e6)"}` -> `worst > 0` |
| profitable on a random half of the stocks with costs tripled | `[{"id":"h","with":"RANDOM_HALF()","cost_bp":"prog.cost_bp * 3"}]` -> `{"r":"h.net_ret_1e6"}` -> `r > 0` |
| profitable with only 10 holdings, max drawdown <= 40% | `[{"id":"n10","max_positions":"10"}]` -> `{"r":"n10.net_ret_1e6","dd_bp":"n10.max_dd_1e6 / 100"}` -> `r > 0 && dd_bp <= 4000` |
| pick the long threshold in-sample, confirm out-of-sample, both windows profitable | `windows {"kind":"rolling","is_days":504,"oos_days":252,"max":2}`; `[{"id":"is","interval":"window.is","vary":{"p_long":[170,180,190]}},{"id":"oos","interval":"window.oos","p_long":"WINNER(is, net_ret_1e6, p_long)"}]` -> `{"oos_ret":"oos.net_ret_1e6","ok":"ALL(oos_ret > 0)"}` -> `n_windows >= 2 && ok == 1` |

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
- The unit of an input must equal the declared one (an input declared MONEY also takes AMOUNT); a constant input is read in the declared unit.
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
| 252 (1 year) | `RET`, `RET_SKIP`, `RVOL`, `PARK`, `GK`, `SKEW`, `KURT`, `ACORR`, `SLOPE`, `R2`, `MAXDD`, `DIST_HIGH`, `DIST_LOW`, `UP_STREAK`, `DOWN_STREAK`, `CORR`, `VOL_RATIO`, `AMIHUD`, `BETA`, `RESID_MOM`, `CS_REL_RET`, `WARG`, `WQUANT`, `WMAD`, `FN_YOY`, `FN_EGROW`, `FN_GM_CHG`, `FN_ACCRUAL`, `FN_DY`, `EV_EARN_DRIFT`, `EV_IDX_CHG` |
| 250 | `CS_RANK_CHG`, `CS_RANK_MEAN`, `EV_POST_N`, `EV_DAYS_SINCE` |
| 84 | `EMA` (warm-up 3N-1) |
| 756 | the warm-up steps K of `SMOOTH` |
| 256 | the window W of algorithmic operators |
| 12 | the periods k of `RWIN`, `RWIN_PCT` |
| 20 | the periods k of `EWIN` |
| 100 | the buckets of `CS_QCUT`, the k of `CS_WINZ`, the q of `WQUANT` |

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
| MONEY | about +-9.22 trillion; total columns above that are read as AMOUNT (3.2) |

### 13.5 Others

| Item | Limit |
|---|---|
| points of `CORR_GROUP` / `CORR_SEED` (lookback / step) | 20..1,024 (quant 0: 20..500) |
| seeds of `CORR_SEED` | 1..64 |
| symbols listed one by one in a stock pool | 20 (over the whole condition) |
| comparisons in `exit_when` / `keep_when` | 6 each; state-leaf coefficients within +-100 |
| terms in `exit_below` / `exit_above` | 1-3 each |
| group cap `group_cap` | 1..255 names |
| user-function grid x bindings | 20,000 combinations |
| threshold grid `grid` | names in portfolio rules: 4, 1..16 values each, at most 64 sets; names in score formulas: 1..256 values each, at most 20,000 trees after expansion |
| checks | 8 checks per strategy; per check 8 runs, 16 outputs, 32 segments, 16 `vary` sets |
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
| `APX_UNIT_MISMATCH` | units disagree (PRICE + QTY, PRICE x QTY) | bring both sides to the same unit or to unitless numbers: `RATIO(a, b)`, `CS_PCT(x)`; for traded value use the column `px.turnover` |
| `APX_UNIT_MISMATCH` | one side MONEY, the other AMOUNT | write the total column on the MONEY side as `column:AMOUNT`; per-share values cannot be mixed with it |
| `APX_UNIT_MISMATCH` | `&&`, `\|\|`, `!` or the condition of `?:` is not BOOL | both sides must be the results of comparisons; multiply numbers with `*` |
| `APX_UNIT_MISMATCH` | the `:UNIT` after a column name differs from the column's unit | remove it or use the column's unit |
| `APX_UNIT_MISMATCH` | arithmetic on a code column | code columns support only `==`, `!=`, `in {...}`, or use as a group key |
| `APX_PARAM_RANGE` | a parameter outside the operator's range (for instance `ROLL_STD(300, x)`) | the ranges in the rows of chapter 4; the message states the legal range. N of `RET`, `DIST_HIGH`, `DIST_LOW`, `VOL_RATIO`, `FN_YOY` in 253..1,260 gets the same quantity with a long window (2.7) |
| `APX_PARAM_TYPE` | a parameter that is not an integer | parameters are integers; a fractional factor goes into an input (`0.5:RATIO`) |
| `APX_ARITY` | too few inputs, too many arguments, a missing parameter | parameters first, inputs after; for instance `ROLL_MEAN(20, close)` |
| `APX_ARITY` | wrong argument count for the accepted forms `LE`, `AND`, `IF`, `MIN`, `MAX`, `GPCT` (2.5) | the message states how many and gives the form |
| `APX_UNIT_MISMATCH` | a rank inside a group `GRANK` added to or compared with a ratio; a numeric column as the input of an event-days operator; `FILL(condition, 0:RATIO)` | the fix in the message: `GPCT(g, x)`, `EV_CHANGE(column)`, `0:BOOL` (2.7) |
| `APX_UNSUPPORTED` | an atom with three or four inputs (`SELECT`, `REC2`, `STATESCAN`) | write `SELECT` as `c ? a : b`; build the others from two-input forms |
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
| `E_UNKNOWN_CODE` | the code column has no such name | industries are code numbers, exchanges `TW` / `TWO`, stock codes in double quotes |
| `E_AMBIGUOUS_NAME` | a column name without its prefix matches two or more columns | write the full column key |
| `E_EMPTY_UNIVERSE` | the condition leaves no stock on any day, or fewer than 1 a day on average | relax the thresholds; check industry codes and column names first |
| `APX_INVALID_ARGUMENT` | more than 20 symbols listed one by one in the stock pool | use a class noun (`!cls.FIN()`) or a seed basket |
| `E_NOT_MARKET_WIDE` | `rebalance_when` / `exposure` has two or more values on the same day | use only things that are the same for every stock: the calendar, `idx.*`, `MKT_MEAN`, `EW_INDEX`; a per-stock condition belongs in `hold` |
| `E_NO_REBALANCE` | `rebalance_when` never holds over the period | check the condition |
| `E_NO_WEIGHT` | `weight` is never positive over the period | only stocks with a weight > 0 are held |
| `E_POS_RULE` | `pos.*` outside a comparison, not linear, or the other side neither a constant nor a market-only formula; `pos.rank` in `exit_when` | the shape rules of 9.5; for instance `pos.ret <= -0.08 \|\| pos.ret >= 0.2` |
| `E_POS_ATOMS` | a rule with more than 6 comparisons | merge, or move part of it to the other rule |
| `E_RANGE` | `reentry_days` outside 0..10,000; `group_cap` outside 1..255 | |
| `E_POS_GROUP` | only one of `group_by`, `group_cap` given; `group_by` uses `pos.*` | give both, e.g. `{"group_by":"CORR_GROUP(RET(1, close))","group_cap":3}`; the group key reads only that day's data |
| `E_POS_SIZE` | a sizing or account argument out of range or missing its partner (e.g. `size_init` without `add_when`, `gross_max` > 1 without `margin_rate`) | follow the ranges in 9.2 and the examples in 9.5b |
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
| `E_RESERVED`, `E_NAMESPACE`, `E_NAME` | the name of a definition file is not acceptable | `user.<author>.NAME`; `apx.*` is reserved |
| `E_DUP`, `E_HASH` | the same name and version with different content | `version` + 1 |
| `APX_CHECK_*` | an error in a check definition (`E_EXPR`, `E_UNIT`, `E_PARAM`, `E_INTERVAL`, `E_WINDOWS`, `E_VARY`, `E_RESOURCE`, `E_CHECK_ATOM_PLACE` (a transform in the wrong place), `E_CHECK_LANG` (`with` written in the lowercase screen language), `E_RET_MIX`, `E_WITH_TYPE` (an item of `with` is not a condition; returns go inside `RETURNS(...)`), `E_WITH_TWICE` (one run changes the returns twice), `E_WITH_UNKNOWN` (no such transform; the closest name is given), `E_RET_UNIT` (the `RETURNS` expression is not a RATIO), `E_UNIVERSE_EMPTY` (the pool averages under 1 stock a day), `E_RES_LIMIT` (more than 2 pools or 2 return transforms in a round; the ones in use are listed), `E_OUTPUT`, `E_PASS`... with the prefix `APX_CHECK_`) | the answer carries the path (for instance `runs[1].interval`), the position and a suggestion |

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

**U02  OTC only, excluding stocks listed for fewer than 60 trading days**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "universe": "market == TWO && cal.tday_idx - list.tday_idx >= 60"}
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

**W01  Only stocks whose code ends in the same digit as today's date**

```json
{"trees_from": "formulas", "formulas": ["RET(60, close)"],
 "universe": "sym.code_num % 10 == MDAY() % 10"}
```

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
