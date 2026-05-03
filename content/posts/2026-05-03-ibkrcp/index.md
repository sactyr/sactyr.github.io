---
title: "Building ibkrcp: An R Package for the IBKR Client Portal API"
author: ''
date: "2026-05-03"
slug: []
categories:
  - Deep Dives
tags:
  - R
  - CRAN
  - IBKR
  - httr2
  - httptest2
  - testthat
  - roxygen2
---

## Background

As part of building a quantitative algorithmic trading system in R, I needed to communicate with Interactive Brokers' Client Portal REST API - a lightweight HTTP interface for account management, market data, and order placement that runs as a Java process on localhost.

The problem: no R package existed for it. The established packages (`IBrokers`, `rib`) target the older TWS socket-based API, which requires a full TWS or IB Gateway installation. The Client Portal API is much better suited to cloud deployments - no GUI, no forced daily restart, roughly 50MB of memory. So I ended up writing the integration myself in a single `ibkr_api.R` file inside my trading project.

Once that file stabilised, the natural next step was to extract it into a proper R package - to clean up the trading project, and to give something back to the small community of R developers working with IBKR. The result is `ibkrcp`, which I've just submitted to CRAN.

This post covers the full packaging journey from scaffold to submission.

---

## Package scope

`ibkrcp` is deliberately narrow. It wraps the Client Portal API and nothing else - no trading logic, no signal generation, no portfolio management. The functions cover four areas:

- **Session management** - `ibkr_tickle()`, `ibkr_auth_status()`, `ibkr_reauthenticate()`
- **Account and portfolio** - `ibkr_get_accounts()`, `ibkr_get_summary()`, `ibkr_get_positions()`
- **Market data** - `ibkr_search_contracts()`, `ibkr_get_price_history()`, `ibkr_get_trading_schedule()`
- **Orders** - `ibkr_place_order()`, `ibkr_get_orders()`, `ibkr_cancel_order()`

Each function is a thin wrapper around an `httr2` request, returning either a data frame or a named list. The only dependency is `httr2` - `jsonlite` was initially included but turned out to be unused, since `httr2::resp_body_json()` handles all JSON parsing internally.

---

## Setting up the package structure

Starting from scratch with `usethis`:

```r
usethis::create_package("ibkrcp")
usethis::use_mit_license()
usethis::use_testthat()
usethis::use_roxygen_md()
```

The R files are split by domain - `session.r`, `account.r`, `market_data.r`, `orders.r`, and `utils.r` for the shared HTTP helpers (`ibkr_get()`, `ibkr_post()`, `ibkr_delete()`). Every request goes through one of these helpers, which handle SSL configuration, headers, and error responses in a single place:

```r
ibkr_get <- function(endpoint, params = NULL) {
  req <- request(paste0(IBKR_BASE_URL, endpoint)) |>
    req_options(ssl_verifypeer = FALSE, ssl_verifyhost = FALSE) |>
    req_headers("User-Agent" = "R/ibkrcp", "Accept" = "*/*")

  if (!is.null(params)) req <- req |> req_url_query(!!!params)

  resp <- req |> req_perform()

  if (resp_status(resp) != 200) {
    stop(sprintf("IBKR API GET %s failed with status %d: %s",
      endpoint, resp_status(resp), resp_body_string(resp)))
  }

  resp |> resp_body_json()
}
```

The SSL flags deserve a note: the Client Portal Gateway runs on localhost with a self-signed certificate. Verifying SSL against localhost is not meaningful, and IBKR's own documentation acknowledges this. It's flagged explicitly in the CRAN submission notes so reviewers aren't surprised by it.

---

## Documentation with roxygen2

Every exported function has a full roxygen2 block - `@param`, `@return`, `@export`, and a description. The internal HTTP helpers are marked `@noRd` to keep them out of the generated manual.

Always verify the `NAMESPACE` after documenting:

```r
devtools::document()
readLines("NAMESPACE")
```

---

## Unit tests with httptest2

Since `ibkrcp` makes HTTP calls to a locally running gateway, tests can't hit a real server in CI. `httptest2` solves this with file-based mock fixtures - pre-recorded JSON responses stored on disk, served in place of real HTTP calls during tests.

The fixture directory structure mirrors the URL path under `tests/testthat/`:

```
localhost-5000/v1/api/
  tickle.json
  iserver/
    auth/status.json
    orders.json
    account/U1234567/orders-50e22c-POST.json
    marketdata/history-45530b.json
    secdef/search-5b4185.json
  portfolio/
    accounts.json
    U1234567/summary.json
    U1234567/positions/0.json
  trsrv/secdef/schedule-c1b48e.json
```

A few things that tripped me up:

**Query parameter hashing.** For GET requests with query parameters, `httptest2` appends a hash of the query string to the fixture filename. So `ibkr_search_contracts("VGS")` doesn't look for `search.json` - it looks for `search-5b4185.json`. The error message tells you exactly what hash to use:

```
Expected mock file: localhost-5000/v1/api/iserver/secdef/search-5b4185.*
```

**POST requests** follow the same pattern, with `-POST` appended after the hash:

```
Expected mock file: localhost-5000/v1/api/iserver/account/U1234567/orders-50e22c-POST.*
```

The approach for alternate scenarios - empty positions, unauthenticated session, no price history - is `with_mock_dir("scenario_name", {...})`, which looks for fixtures under a subdirectory of that name. For example, the unauthenticated tickle test:

```r
with_mock_dir("unauthenticated", {
  test_that("ibkr_tickle() stops when not authenticated", {
    expect_error(ibkr_tickle(), "not authenticated")
  })
})
```

The final test suite: **42 tests, 0 failures, 0 warnings**.

---

## R CMD check

```r
devtools::check()
```

Result:

```
0 errors | 0 warnings | 1 note
```

The single note:

```
Non-standard file/directory found at top level: 'CRAN-comments.md'
```

This is expected - `CRAN-comments.md` is a submission-only file that lives at the package root during development but isn't part of the R package standard. It's universally accepted by CRAN reviewers and disappears once the package is published.

---

## Submission

```r
devtools::submit_cran()
```

The package is now with CRAN. Once accepted, the trading project can swap:

```r
source("R/live_trading/ibkr_api.R")
```

for:

```r
library(ibkrcp)
```

Until then, you can install the development version directly from GitHub:

```r
pak::pak("sactyr/ibkrcp")
```

The Client Portal Gateway must be running and authenticated before any function calls will work - see the [package vignette](https://github.com/sactyr/ibkrcp) for the full setup walkthrough.

---

*The full source code is available on [GitHub](https://github.com/sactyr/ibkrcp).*
