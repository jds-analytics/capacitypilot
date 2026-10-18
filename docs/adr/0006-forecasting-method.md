# ADR 0006: Forecasting method

- Status: Proposed (method family decided by the director; specifics need approval)
- Date: 2026-10-17
- Related: docs/spec.md "Forecast targets", docs/metrics.md

## Decision in one line

Every night, for each site, we forecast **daily peak power (kW)** and **rack units in use (RU)** for the
next **90 days** using a classical **exponential smoothing model with a damped trend and weekly
seasonality** (statsmodels `ETSModel`), adding yearly seasonality when a site has at least 2 years of
history. Each forecast gives a best guess plus **80% and 95% ranges**. A model only ships if it beats a
simple "same as last week" baseline in a backtest.

Plain-English terms:
- **Exponential smoothing (ETS)**: a well-understood statistical method that weights recent history more
  than old history and models level, trend and repeating patterns.
- **Damped trend**: growth is assumed to slow over time rather than continue forever, which avoids wild
  90-day projections.
- **Seasonality**: patterns that repeat (weekday vs weekend; summer vs winter).
- **Prediction interval**: a range we expect the real value to fall inside, e.g. 80% of the time.
- **Backtest**: pretend it is a past date, forecast, then compare with what actually happened.
- **Seasonal naive baseline**: "the forecast for next Monday is last Monday's value". If our model
  cannot beat this, it is not worth shipping.

## Context

- Director decision: per-site kW and RU, 90 days ahead, classical statistics (seasonal trend with
  prediction interval), nightly.
- Analytics rules: documented baseline (seasonal naive), backtest over at least 90 days with rolling
  origins, report MAPE, WAPE and interval coverage against targets in the spec.
- Data is synthetic, generated with trend, weekly and yearly seasonality, step changes (new customers
  moving in), gaps and bad rows.

## Options considered

| Option | Pros | Cons |
|---|---|---|
| **ETS damped + weekly seasonal (MSTL + ETS when >= 730 days)** (chosen) | Classical, fast (well under **1 s** per series), honest intervals, small dependency (statsmodels) | Handles step changes only by adapting over time |
| SARIMA | Classical, flexible | Slow to fit with weekly seasonality on daily data; fragile parameter search |
| Prophet | Handles holidays and changepoints | Heavy dependency (Stan toolchain), arm64 build friction; not what the director chose |
| Machine-learning models (gradient boosting, neural) | Can use many features | Not classical; needs more data; harder to explain intervals |

## Decision

1. **Input series**: per site, per day: `peak_kw` = maximum hourly average kW across the site;
   `ru_used` = maximum RU in use that day. Built from the daily rollup table.
2. **Minimum history**: **56 days**. Sites with less get status `insufficient_history` and no forecast.
3. **Model**: ETS with additive damped trend and additive weekly seasonality (period 7). When at
   least **730 days** exist, use MSTL (periods 7 and 365) with ETS on the seasonally adjusted series.
4. **Intervals**: 80% and 95% prediction intervals; RU forecasts are clipped to `[0, site RU capacity]`
   and kW to `>= 0`.
5. **Organic demand only**: the forecast models observed usage. Approved and allocated reservations are
   layered on top as "committed demand" in the capacity view, so they are not double-counted.
6. **Backtest**: rolling origins every **14 days** across the most recent **180 days** (at least
   **6 origins**), each forecasting **90 days**; report WAPE and MAPE by horizon bucket (1-30, 31-90
   days) and interval coverage, versus the seasonal naive baseline. Proposed targets are in the spec.
7. **Fallback**: if the model fails to fit or its own backtest WAPE is worse than the baseline for a
   site, publish the seasonal naive forecast for that site and flag `model = seasonal_naive`.
8. **Versioning**: each run stores `model_version` and parameters so results are reproducible.

## Consequences

- Good: fast, explainable and cheap; a full nightly run of **500 sites x 2 metrics** is expected to
  finish in under **10 minutes** on a laptop.
- Cost: sudden step changes (a large tenant moving in) are learned only after they appear; this is why
  committed reservations are shown separately.
- Accuracy targets are **proposed** until the director approves them (open question).
