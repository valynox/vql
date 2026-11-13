# VQL — Valynox Quant Language

**AlphaPrime Stocks: <https://alphaprime.trading>**

VQL is the stock-selection language of [AlphaPrime Stocks](https://alphaprime.trading). One language writes score formulas, stock pools, backtest rules, screening and checks. The author is usually an AI: a person describes a strategy in plain words, and the AI writes it in VQL through MCP.

This repository publishes the language specification and the example library so that any AI model (and any person) can learn to write VQL.

## Files

| File | For | Content |
|---|---|---|
| `VQL.en.md` | people (English) | the full language reference |
| `VQL.zh-TW.md` | people (Traditional Chinese) | the same reference, section by section |
| `vql.txt` | AI | compact plain-text edition of the whole language, one file to fetch |
| `EXAMPLES.en.md`, `EXAMPLES.zh-TW.md` | people | the example library: plain words → complete arguments that produce a plan as is |
| `examples.json` | machines | the same example library (`vql.examples.v1`: id, zh, en, args, definitions) |

This repository: <https://github.com/valynox/vql>. The AI edition as one plain-text file: <https://raw.githubusercontent.com/valynox/vql/main/vql.txt>.

## Using VQL

- Start at <https://alphaprime.trading>: connect AlphaPrime Stocks to your AI, then describe the strategy in plain words.
- To teach any AI the language itself, point it at `vql.txt` (or this repository).
- In AlphaPrime Stocks, the MCP server is `https://alphaprime.trading/stocks/mcp`. Score formulas go in `formulas` of `stock_backtest`, the stock pool in `universe`; `stock_formula_help` looks up forms and validates definitions.
- Every example in the library passes plan: the form is checked, nothing is run.

## What is not here

This repository contains the language specification and examples only. It does not contain the engine, its source code or binaries, market data, or backtest results. Releases are generated from the maintained reference; changes are made upstream and published here as tagged versions.

## License

Copyright (c) 2026 VALYNOX CO., LTD. The specification and the examples are licensed under the Creative Commons Attribution 4.0 International License (CC BY 4.0); see `LICENSE`. The license covers the documents and examples in this repository only.

---

# VQL（Valynox Quant Language）

**AlphaPrime Stocks 官網：<https://alphaprime.trading>**

VQL 是 [AlphaPrime Stocks](https://alphaprime.trading) 的選股策略語言：一門語言寫分數公式、選股範圍、回測規則、篩選與檢驗。作者通常是 AI——人用白話講策略，AI 經 MCP 寫成 VQL。

這個 repo 公開語言規格與範例庫，讓任何 AI 模型（和任何人）都學得會寫 VQL。

## 檔案

| 檔 | 給誰 | 內容 |
|---|---|---|
| `VQL.zh-TW.md` | 人（繁中） | 完整語言說明 |
| `VQL.en.md` | 人（英文） | 同上，逐節對應 |
| `vql.txt` | AI | 整個語言的純文字精簡版，抓一個檔就夠 |
| `EXAMPLES.zh-TW.md`、`EXAMPLES.en.md` | 人 | 範例庫：白話 → 照抄就能出計畫的完整引數 |
| `examples.json` | 機器 | 同一份範例庫（`vql.examples.v1`：id、zh、en、args、definitions） |

這個 repo：<https://github.com/valynox/vql>。給 AI 的純文字版（一個檔）：<https://raw.githubusercontent.com/valynox/vql/main/vql.txt>。

## 怎麼用

- 從官網 <https://alphaprime.trading> 開始：把 AlphaPrime Stocks 接到你的 AI，再用白話講策略。
- 想讓任何 AI 學會這門語言本身，讓它讀 `vql.txt`（或這個 repo）。
- 在 AlphaPrime Stocks 裡，MCP 網址是 `https://alphaprime.trading/stocks/mcp`。分數公式放 `stock_backtest` 的 `formulas`，選股範圍放 `universe`；查寫法、驗自訂定義用 `stock_formula_help`。
- 範例庫每一題都過了 plan（只驗寫法、不開跑）。

## 這裡沒有的

這個 repo 只有語言規格與範例，不含引擎、引擎的原始碼或執行檔、市場資料、回測結果。每一版由維護中的說明產生，改動在上游做，再以標了版號的版本發佈到這裡。

## 授權

Copyright (c) 2026 VALYNOX CO., LTD.。規格與範例採用創用 CC 姓名標示 4.0 國際授權（CC BY 4.0），見 `LICENSE`。授權只涵蓋這個 repo 裡的文件與範例。
