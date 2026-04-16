---
title: 'Building an Automated Crypto Trader Part 3: From Signals to Live Orders - Completing the Loop'
author: ''
date: '2026-04-10'
slug: []
categories:
  - Deep Dives
tags:
  - Algo-Trading
  - R
  - Crypto
  - Azure
  - Cloud
  - Independent-Reserve
  - Automation
  - Data Pipeline
  - Testthat
description: "Completing the crypto trading system - live order execution via Independent Reserve's private API, state persistence on Azure, unit testing with testthat, and lessons learned from running a live algorithmic trader"
draft: false
---

## Introduction

In [Part 1](https://sactyr.github.io/posts/2025-12-19-building-an-automated-crypto-trader-part-1-survival-of-the-fittest-backtesting/) of this series, we subjected eight trading strategies to a Monte Carlo gauntlet of 1,000 randomised time windows, measuring each against the CAPS (Calmar-adjusted Probability Score) to find a strategy robust enough for live deployment. In [Part 2](https://sactyr.github.io/posts/2026-02-11-automating-crypto-price-collection-azure/), we built the cloud-based data pipeline - a fully automated hourly price collector running on Azure Container Instances, feeding into Blob Storage.

Those two parts answered the two most important questions before going live: *does the strategy work?* and *do we have reliable data?* Part 3 answers the final question: *can we actually execute it?*

This post walks through the live trading engine - how a signal becomes an order, how the system remembers what it holds, how we tested it, and what we found when we finally pointed it at a real exchange.

---

## The Trading Engine: Architecture Overview

The live trading system is a single R script - `crypto_trader.R` - that runs as a daily scheduled job on Azure. Each run follows a strict sequence:

```
1. Connect to Azure Blob Storage → load bot state + price history
2. Generate trading signal (SMA crossover) from loaded price history
3. Check stop-loss shield
4. Execute order if required (private API)
5. Update and persist state back to Azure
```

Each step is wrapped in `tryCatch` with structured logging via the `logger` package. If anything fails - an API timeout, a corrupted state file, a network blip - the script logs the error and halts cleanly rather than firing a half-baked trade.

The script runs in two modes controlled by a single variable:

```r
trading_mode <- "TEST"   # Simulates orders without executing them
trading_mode <- "LIVE"   # Executes real market orders
```

This made it safe to iterate on the logic extensively before touching real money.

---

## Signal Generation in Production

In backtesting, signal generation is straightforward - you pass a full `xts` object of historical prices into the strategy function and get a vector of signals back. In production it's slightly more nuanced, because you're appending one new data point each day to a rolling history.

The strategy used was SMA crossover - buy when the fast SMA (26 periods) crosses above the slow SMA (50 periods), sell when it crosses below:

```r
# Generate signal from rolling price history
fast_sma <- SMA(Ad(prices_xts), n = fast_period)   # 26
slow_sma <- SMA(Ad(prices_xts), n = slow_period)   # 50

signal <- case_when(
  fast_sma > slow_sma & lag(fast_sma) <= lag(slow_sma) ~  1L,  # BUY
  fast_sma < slow_sma & lag(fast_sma) >= lag(slow_sma) ~ -1L,  # SELL
  TRUE                                                  ~  0L   # HOLD
)
```

The key production consideration: the slow SMA needs at least 50 daily bars before it produces a valid signal. On day one, the script does nothing and waits until sufficient history accumulates. This is the same "warmup period" behaviour from backtesting - the code handles both contexts identically, which is exactly what you want.

---

## Stop-Loss: The Shield

The SMA crossover strategy is a trend-follower - it stays in a position until the trend reverses, which can take a long time during a prolonged bear market. Without a stop-loss, a severe enough crash could wipe out the entire position before a sell signal fires.

The stop-loss "shield" is checked on every run, before signal generation:

```r
if (bot_state$in_position) {
  pnl_pct <- (current_price - bot_state$purchase_price) / bot_state$purchase_price

  if (pnl_pct <= -stop_loss_threshold) {
    log_warn(paste0(
      "SHIELD TRIGGERED: Price dropped ", round(pnl_pct * 100, 2),
      "% below entry. Forcing SELL."
    ))
    final_decision <- "SELL"
  }
}
```

The threshold was set at **-10%** - if BTC drops more than 10% below the entry price, the position is force-closed regardless of what the SMA crossover says. During the end-to-end test run, the stop-loss fired correctly - BTC was sitting ~20% below the test entry price and the shield triggered before the signal logic even ran.

---

## Connecting to Independent Reserve's Private API

Part 2 used Independent Reserve's **public** API - no authentication required, just GET requests for price history. Placing orders requires the **private** API, which uses HMAC-SHA256 request signing.

The signing process works as follows: every private request includes an API key, a nonce (Unix timestamp in milliseconds for uniqueness), and a signature. The signature is computed by:

1. Building a string from the endpoint URL and all parameters in a specific order
2. Hashing that string with SHA-256 using the API secret as the key
3. Converting the hash to uppercase hex

```r
ir_private_auth <- function(req, api_key, api_secret, extra_params = list()) {

  nonce      <- as.integer(Sys.time() * 1000)
  url        <- req$url

  # Parameters must be in this exact order for valid signature
  all_params <- c(
    list(apiKey = api_key, nonce = nonce),
    extra_params
  )

  sig_string <- paste0(
    url, ",",
    paste(
      paste0(names(all_params), "=", unlist(all_params)),
      collapse = ","
    )
  )

  signature <- toupper(as.character(
    openssl::sha256(chartr("", "", sig_string), key = api_secret)
  ))

  all_params$signature <- signature

  req |>
    req_method("POST") |>
    req_body_json(all_params)
}
```

One quirk worth documenting: Independent Reserve's private API uses **different endpoint names and parameter names** depending on the order direction.

| Direction | Endpoint | Volume Parameter |
|-----------|----------|-----------------|
| Buy | `PlaceMarketBuyOrder` | `volumeInSecondaryCurrencyAmount` (AUD) |
| Sell | `PlaceMarketSellOrder` | `primaryCurrencyAmount` (BTC) |

This asymmetry - buying in AUD, selling in BTC - is not immediately obvious from the documentation and took some trial and error to get right. The `place_ir_order()` function handles this branching internally:

```r
place_ir_order <- function(type, amount, key, secret) {

  endpoint     <- if (type == "MarketBuy") "PlaceMarketBuyOrder" else "PlaceMarketSellOrder"
  volume_param <- if (type == "MarketBuy") "volumeInSecondaryCurrencyAmount" else "primaryCurrencyAmount"

  extra_params <- list(
    primaryCurrencyCode   = "xbt",
    secondaryCurrencyCode = "aud"
  )
  extra_params[[volume_param]] <- amount

  request(file.path(ir_private_api_url, endpoint)) |>
    ir_private_auth(api_key = key, api_secret = secret, extra_params = extra_params) |>
    req_retry(max_tries = 3) |>
    req_perform() |>
    resp_body_json()
}
```

---

## State Management: The Bot's Memory

A stateless trading bot is dangerous - if the Azure container restarts mid-run, or the script crashes between placing an order and recording it, the bot loses track of what it holds.

The solution is a persistent `bot_state` object stored in Azure Blob Storage, updated at the end of every run:

```r
bot_state <- list(
  in_position             = FALSE,   # Are we currently holding BTC?
  purchase_price          = 0,       # Price at which we bought
  last_trade_time         = NA,      # Timestamp of last executed trade
  current_cash_holdings   = 1000,    # AUD available
  current_crypto_holdings = 0        # BTC held
)
```

On startup, the script downloads the latest state from Azure. At the end of every run - whether a trade fired or not - it uploads the updated state back. This means the bot always starts each day from a known, persisted position.

The state also includes a **desync detection** step. Before acting on the signal, the script cross-checks `bot_state$in_position` against the actual exchange balance:

```r
ir_btc_balance <- get_ir_accounts(key, secret) |>
  filter(CurrencyCode == "XBT") |>
  pull(AvailableBalance)

if (bot_state$in_position && ir_btc_balance < min_btc_threshold) {
  log_warn("DESYNC: State says IN position but exchange shows no BTC. Correcting state.")
  bot_state$in_position <- FALSE
}
```

This guards against the scenario where a SELL order was placed but the state file wasn't updated (e.g. due to a crash), preventing the bot from trying to sell BTC it no longer holds.

---

## Testing: Building Confidence Before Going Live

A trading bot that hasn't been tested is a donation to the market. Before running `crypto_trader.R` anywhere near real funds, two rounds of testing were completed.

### Phase 1: Unit Tests with `testthat`

The pure functions - those that take inputs and return outputs without side effects - were tested with `testthat`. Three test suites covered the most critical logic:

**`test_strat_sma_cross.R` (9 tests):** Verified that the strategy correctly identifies crossover events, handles edge cases like flat price data, and produces the expected signal vector structure.

**`test_place_ir_order.R` (8 tests):** Used `httptest2` to mock the Independent Reserve API responses. Tests verified that correct endpoints were called for buys vs sells, that invalid order types were rejected, and that 4xx responses were handled gracefully.

**`test_parse_azure_connection.R` (4 tests):** Verified the connection string parser correctly extracted the account name and key, including strings with `==` padding in the base64 key.

```
══ Results ════════════════════════════════════
[ FAIL 0 | WARN 0 | SKIP 0 | PASS 21 ]
```

21 tests, all green.

### Phase 2: End-to-End Local Run in TEST Mode

With unit tests passing, the full `crypto_trader.R` script was run locally with `trading_mode <- "TEST"`. This pointed at real Azure Blob Storage and the real Independent Reserve API for price data - but any orders were simulated rather than executed.

The end-to-end run confirmed:
- Azure connection and blob read/write working correctly
- Price history loading and daily downsampling to midnight-to-midnight bars
- SMA crossover signal generating correctly from live data
- Stop-loss shield triggering (BTC was ~20% below the test entry price at the time)
- State persisting correctly back to Azure after each run

```
INFO  | Daily bars available: 144 | From: 2025-10-01 | To: 2026-02-21
INFO  | Current Price: 96259.9 | Signal: 0
INFO  | Current P/L: -20.56%
WARN  | SHIELD TRIGGERED: Price dropped 10% below entry. Forcing SELL.
INFO  | FINAL DECISION: SELL
INFO  | TEST MODE: Market Sell simulated | BTC: 0.0066 | Price: 96259.9
SUCCESS | Bot memory successfully synced to Azure.
```

The full loop - from Azure download to signal generation to simulated execution to state upload - ran cleanly.

---

## The Reality of Crypto Markets

The system worked. All the pieces connected. And then came the part that backtests can never fully prepare you for: watching it run in the wild.

A few things became apparent quickly.

**24/7 markets don't respect daily bars.** The SMA crossover strategy was designed around daily closing prices - a concept that doesn't cleanly exist in crypto. Bitcoin trades continuously, which means "today's close" is an arbitrary snapshot at midnight. A significant move at 11:59 PM and a reversal at 12:01 AM effectively constitutes two different "days" for the signal, creating a kind of temporal noise that the backtests on historical daily data didn't fully capture.

**Volatility regimes shift dramatically.** BTC in a bull market and BTC in a bear market behave like different instruments. The SMA crossover - and most trend-following strategies - performs best during sustained directional moves. Sideways chop with high intraday volatility generates whipsaws: multiple false signals in quick succession, each one costing a transaction fee. In the backtests, these periods were averaged out across 1,000 windows. In live trading, you experience one regime at a time, and if you happen to start during a choppy period, the early results are dispiriting.

**The psychological dimension is real.** A simulated SELL at a 20% loss is a number on a screen. A live SELL at a 20% loss is a different experience entirely. Building a system that you can trust enough to let run without second-guessing every decision requires a level of conviction in the strategy that is hard to maintain through drawdowns - even when the drawdown is well within what the backtests predicted.

None of these are criticisms of the approach. They are honest realities of live algorithmic trading in any asset class, and crypto amplifies all of them.

---

## Closing the Chapter: What Carried Over

The decision to pivot away from crypto wasn't a failure of the system - it was a deliberate choice to apply the same framework to a more suitable market. ASX-listed ETFs offered lower volatility, regular dividends, and market hours that align naturally with a daily bar strategy.

Everything built across this series carried over directly:

- The **Monte Carlo backtesting framework** and CAPS scoring methodology - rerun verbatim on VGS, VAS and GOLD
- The **state management pattern** - the same concept of persisting `bot_state` to cloud storage, now using Google Cloud instead of Azure
- The **price pipeline architecture** - the same MD5 deduplication and incremental merge approach, now against IBKR's API instead of Independent Reserve's
- The **structured logging and error handling** patterns from `logger`
- The discipline of **unit testing before going live**

The tools changed. The thinking didn't.

If you've followed this series from the beginning - from the first Monte Carlo simulation through to a fully deployed cloud trading system - the most important takeaway is this: the value of building things properly compounds. The hours spent on unit tests, state management and error handling might seem disproportionate when you're eager to go live. But they're exactly what allows you to iterate quickly, debug confidently, and ultimately trust what you've built.

A new series covering the ASX ETF trading system - from backtesting through to live execution via Interactive Brokers - is coming. The story continues.

---

*The full source code for the crypto trading system is available on [GitHub](https://github.com/sactyr/crypto_trading).*
