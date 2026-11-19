<div align="center">

**Multi-model confluence trading system for XAUUSDT and BTCUSDT**
Pine Script v6 · TradingView · 5m / 15m

![Pine Script](https://img.shields.io/badge/Pine%20Script-v6-1f6feb?style=flat-square)
![Platform](https://img.shields.io/badge/platform-TradingView-2962ff?style=flat-square)
![Markets](https://img.shields.io/badge/markets-XAUUSDT%20%7C%20BTCUSDT-f2a900?style=flat-square)
![Timeframes](https://img.shields.io/badge/timeframes-5m%20%7C%2015m-555?style=flat-square)
![Repaint](https://img.shields.io/badge/repainting-none%20(bar--close)-2ea043?style=flat-square)
![Status](https://img.shields.io/badge/status-in%20development-d29922?style=flat-square)

</div>

---

## Contents

- [Overview](#overview)
- [Project Goals and Targets](#project-goals-and-targets)
- [Current Status](#current-status)
- [System Design](#system-design)
- [Strategy Models](#strategy-models)
- [Confluence Scoring](#confluence-scoring)
- [Risk Management](#risk-management)
- [Non-Repainting Guarantees](#non-repainting-guarantees)
- [Installation](#installation)
- [Usage](#usage)
- [Alerts](#alerts)
- [Backtesting Methodology](#backtesting-methodology)
- [Results](#results)
- [Roadmap](#roadmap)
- [Known Limitations](#known-limitations)
- [Repository Layout](#repository-layout)
- [Contributing](#contributing)
- [Disclaimer](#disclaimer)

---

## Overview

Stark is a Pine Script v6 signal and execution system built for **XAUUSDT** and **BTCUSDT** on the **5-minute** and **15-minute** charts.

It does not rely on a single setup. Several independent strategy models each look for an entry, and a **confluence score (0–100)** decides whether a signal is strong enough to take. Every accepted signal comes with a fixed plan: entry, stop loss, three take-profit levels, and a trailing stop. Position size is calculated from a fixed **0.5% account risk** per trade.

The project has two deliverables:

| Deliverable | Purpose |
|---|---|
| **Indicator** | Chart UI, signals, trade boxes, dashboard, alerts. |
| **Strategy** | Same signal logic wired to `strategy.*` so it can be backtested with commission and slippage. |

Both are meant to run from one shared signal core, so the chart and the backtest cannot drift apart.

> **On the 70% target.** 70%+ win rate is the **goal this project is tuned toward**, not a promise and not a claim. Whether it is reachable, and at what cost to profit factor and trade count, is exactly what the backtests are for. All numbers in this repo come from logged runs, never from estimates. See [Results](#results).

---

## Project Goals and Targets

These are acceptance targets, used to decide whether a version is good enough to keep. They are editable, and a version that misses them is documented, not hidden.

| Metric | Target | Why it matters |
|---|---|---|
| Win rate (out-of-sample) | **≥ 70%** | Primary objective. |
| Profit factor (net of costs) | **≥ 1.30** | Win rate alone says nothing about edge. |
| Expectancy per trade | **> 0 R** after costs | The real pass/fail test. |
| Max drawdown | **≤ 10% of equity** | Keeps 0.5% risk sizing survivable. |
| Trades per instrument/timeframe | **≥ 200** in the test window | Fewer trades means the stats are noise. |
| In-sample vs out-of-sample gap | **Small and same sign** | Guards against curve-fitting. |

**Definition of a win.** A trade counts as a **win** only if net P&L after fees and slippage is positive. Trades closed at break-even are counted separately as `BE` and are not hidden inside the win rate.

**Why win rate needs context.** The break-even win rate depends on the reward-to-risk ratio:

```
break-even win rate = 1 / (1 + R)
```

At R = 1.0 you need over 50% just to cover costs. A high win rate built on small targets and wide stops can still lose money in a bad week. That is why profit factor, expectancy and drawdown are tracked next to win rate, and why none of them is reported alone.

---

## Current Status

The work builds on the existing `devVaibhav | FRVP + HTF Bias Pro` indicator.

| Area | State |
|---|---|
| HTF major trend filter (non-repainting) | Done |
| FRVP levels: POC, VAH, VAL, LVN | Done |
| AOI zones, 3-candle FVG, BoS / CHoCH | Done |
| RVOL + candle body signal gate | Done |
| Long/Short position boxes, SL / ENTRY / TP1–TP3 / TRAIL markers | Done |
| In-script W / L / BE and net R tracking | Done |
| Multi-model signal engine (beyond current confluence gate) | In progress |
| 0–100 confluence score | In progress |
| 0.5% risk position sizing | In progress |
| Strategy (`strategy.*`) version with costs | Not started |
| Session / killzone filter, liquidity sweep, unmitigated OB | Not started |
| Walk-forward + out-of-sample validation logs | Not started |
| Published backtest results | Not started |

---

## System Design

```
                ┌────────────────────────────────────┐
  Market data → │  Context layer                     │
                │  HTF bias · volatility regime ·    │
                │  session · FRVP levels · AOI       │
                └──────────────────┬─────────────────┘
                                   │
                ┌──────────────────▼─────────────────┐
                │  Strategy models (independent)     │
                │  trend pullback · range breakout / │
                │  retest · liquidity sweep ·        │
                │  FVG / OB mean reversion           │
                └──────────────────┬─────────────────┘
                                   │ candidate signals
                ┌──────────────────▼─────────────────┐
                │  Confluence scorer (0–100)         │
                │  threshold gate + regime filter    │
                └──────────────────┬─────────────────┘
                                   │ accepted signal
                ┌──────────────────▼─────────────────┐
                │  Trade manager                     │
                │  entry · SL · TP1–TP3 · trail ·    │
                │  0.5% risk sizing · W/L/BE stats   │
                └──────────────────┬─────────────────┘
                                   │
                  Chart UI · Dashboard · Alerts · Strategy orders
```

**Design rules**

1. Signals use closed-bar data only.
2. Each strategy model is switchable on/off, so each can be backtested alone and in combination.
3. Signal logic lives in one place (a shared Pine library). The indicator and the strategy only call it.
4. No parameter is tuned on the same data it is judged on.

---

## Strategy Models

The system is built to compare models, not to assume one is best. Each model is tested alone first, then combined.

| Model | Idea | Status |
|---|---|---|
| **HTF trend + pullback** | Trade with the HTF bias after a pullback into an AOI or FRVP level. | Base exists |
| **Range breakout + retest** | Break of a defined range, entry on the retest. | Planned |
| **Liquidity sweep reversal** | Stop-run beyond a recent high/low, then rejection back inside. | Planned |
| **FVG / order block reaction** | Entry on first mitigation of an unfilled gap or order block. | FVG done, OB planned |
| **POC mean reversion** | Fade stretched price back toward the POC in low-trend regimes. | Planned |

A model is dropped from the final build if it does not add value in out-of-sample testing. That result gets documented too.

---

## Confluence Scoring

Each candidate signal gets a score from 0 to 100. Only signals above a set threshold are traded. Weights below are the **starting point**, and they get tested, not trusted.

| Factor | Starting weight |
|---|---|
| HTF bias aligned | 20 |
| Price at AOI / FRVP level (POC, VAH, VAL, LVN) | 20 |
| Structure confirmation (BoS / CHoCH) | 15 |
| FVG or order block present | 10 |
| Liquidity sweep before entry | 10 |
| RVOL above threshold | 10 |
| Candle body / close strength | 5 |
| Session quality (killzone) | 5 |
| Volatility regime suitable (ATR band) | 5 |

The score threshold is the main lever for the win-rate vs trade-count trade-off. Raising it should raise win rate and cut frequency. The backtests will show how steep that curve actually is for each market.

---

## Risk Management

| Item | Rule |
|---|---|
| Risk per trade | **0.5% of equity** |
| Position size | `equity × 0.5% ÷ (entry − SL distance × contract value)` |
| Stop loss | Structure-based (beyond the swing / zone) with an ATR floor |
| TP1 / TP2 / TP3 | Snapped to FRVP levels where possible, with minimum R:R checks |
| Trailing stop | Step trail: BE after TP1, then TP1 after TP2 |
| Trade timeout | Closed after `maxTradeBars` |
| Daily loss cap | Planned |

Risk is always measured against the **distance to the stop**, so a wider stop means a smaller position, never a bigger loss.

---

## Non-Repainting Guarantees

- Signals fire on **bar close** (`confirmClose = true`).
- Higher-timeframe data is read from the **previous closed HTF bar** (`[1]` with `lookahead_on`).
- AOI bands and the FRVP window are built from **prior-bar** data.
- No future bars are referenced anywhere in signal logic.
- Alerts are meant to run on **bar close** to match what the chart shows.

Anyone can check this: load the indicator, let it print signals live for a session, reload, and confirm historical signals did not move.

---

## Installation

1. Open [TradingView](https://www.tradingview.com) and load **XAUUSDT** or **BTCUSDT**.
2. Open **Pine Editor** (bottom panel).
3. **Open → New indicator**, then clear the default code.
4. Paste the script from `pine/` in this repo. The first line must be `//@version=6`.
5. **Save**, then **Add to chart**.
6. For backtesting, repeat with the strategy script and open **Strategy Tester**.

If a shared library is used, publish it first (**Publish script → Library**) and update the `import` line to your own username.

---

## Usage

1. Set the chart to **5m** or **15m**.
2. Leave inputs on defaults for the first run.
3. Read the chart:

| On chart | Meaning |
|---|---|
| EMA ribbon + HTF dots | Short-term and higher-timeframe trend. |
| Green / Red AOI | Buy-interest / sell-interest zones. |
| POC, VAH, VAL, LVN | Volume-profile levels. |
| FVG boxes | Unfilled 3-candle gaps, removed on mitigation. |
| BoS / CHoCH | Structure breaks. |
| Position box | Risk side and profit side of the live trade. |
| `SL`, `ENTRY`, `TP1–TP3`, `TRAIL` | Right-edge trade levels. `✓` marks a hit target. |

Data Window output: `1` long, `-1` short, `0` none.

Change one input at a time. Compare the result against the logged baseline before keeping a change.

---

## Alerts

The script sends alerts through `alert()` calls with a JSON payload (direction, entry, SL, TP1–TP3, score).

1. Open **Create Alert** (`Alt + A`).
2. Condition: `devVaibhav`.
3. Choose **Any alert() function call**.
4. Trigger: **Once per bar close**.
5. Pick notification type (app, email, popup, webhook).

---

## Backtesting Methodology

This section is the contract for every number published in this repo.

### Test setup

| Setting | Value |
|---|---|
| Strategy type | `strategy(...)` with `process_orders_on_close = true` |
| Position sizing | 0.5% equity risk per trade |
| Initial capital | Fixed and stated in each log |
| Commission | Set per market to the real venue fee, stated in each log |
| Slippage | Non-zero, set per market, stated in each log |
| Data | Same exchange feed for all runs on one symbol |
| Bar magnifier | Off unless stated |

Fee and slippage values must match the broker or exchange actually used. Results with zero costs are not published.

### Validation protocol

1. **Split by time, not at random.** Example: first 70% of history is in-sample, last 30% is out-of-sample. The split is fixed before tuning starts.
2. **Tune only in-sample.** The out-of-sample window is touched once per version.
3. **Walk-forward.** Re-run across rolling windows (tune on window N, test on window N+1) to see whether the edge holds across market conditions.
4. **Parameter robustness.** Nudge key inputs by roughly ±20%. If results collapse, the setting is overfit and is rejected.
5. **Minimum sample.** Under 200 trades in a window, the result is marked "inconclusive".
6. **Cost stress test.** Re-run with higher fees and slippage. A system that only works at zero cost is not a system.
7. **Compare models.** Every strategy model runs alone and combined, on each market and timeframe.

### Metrics recorded per run

Total trades · Win rate · BE count · Profit factor · Net profit · Expectancy (R) · Average win / average loss · Max drawdown · Longest losing streak · Trades per week

Raw data comes from **Strategy Tester → List of trades → Export**, and the CSV is saved next to each log so any number can be re-checked.

### TradingView limits to be aware of

- Lower timeframes have limited history on lower-tier plans. The test window can be short, which caps the trade count.
- Backtests are simulations. Live fills, spread changes and latency differ.

---

## Results

**No backtest results are published yet.** Tables below stay empty until real runs are logged with the settings above. Nothing here is estimated or projected.

### Per market and timeframe (full system)

| Market | TF | Window | Trades | Win % | PF | Net profit | Max DD | Expectancy (R) | Sample |
|---|---|---|---|---|---|---|---|---|---|
| XAUUSDT | 5m | — | — | — | — | — | — | — | Not run |
| XAUUSDT | 15m | — | — | — | — | — | — | — | Not run |
| BTCUSDT | 5m | — | — | — | — | — | — | — | Not run |
| BTCUSDT | 15m | — | — | — | — | — | — | — | Not run |

### In-sample vs out-of-sample

| Market | TF | Set | Trades | Win % | PF | Max DD |
|---|---|---|---|---|---|---|
| XAUUSDT | 15m | In-sample | — | — | — | — |
| XAUUSDT | 15m | Out-of-sample | — | — | — | — |
| BTCUSDT | 15m | In-sample | — | — | — | — |
| BTCUSDT | 15m | Out-of-sample | — | — | — | — |

### Strategy model comparison

| Model | Market | TF | Trades | Win % | PF | Max DD |
|---|---|---|---|---|---|---|
| HTF trend + pullback | — | — | — | — | — | — |
| Range breakout + retest | — | — | — | — | — | — |
| Liquidity sweep reversal | — | — | — | — | — | — |
| FVG / OB reaction | — | — | — | — | — | — |
| POC mean reversion | — | — | — | — | — | — |
| **Combined (score-gated)** | — | — | — | — | — | — |

### Score threshold sweep

| Score threshold | Trades | Win % | PF |
|---|---|---|---|
| — | — | — | — |

### Run log

Each run gets one entry: date, script version, commit, symbol, timeframe, date range, fee, slippage, inputs changed, result, link to exported CSV.

---

## Roadmap

**Phase 1 — Signal core**
- [ ] Move signal logic into a shared Pine library
- [ ] Add range breakout/retest, liquidity sweep, POC reversion models
- [ ] Add unmitigated order block tracking
- [ ] Add NY session / killzone filter and pre-session gap anchor
- [ ] Build the 0–100 confluence scorer with a threshold input

**Phase 2 — Execution and risk**
- [ ] 0.5% risk-based position sizing
- [ ] Daily loss cap
- [ ] Strategy (`strategy.*`) version with commission and slippage inputs
- [ ] Dashboard: score, bias, live trade, W / L / BE, net R

**Phase 3 — Validation**
- [ ] Fix the in-sample / out-of-sample split
- [ ] Run each model alone on all four market/timeframe pairs
- [ ] Walk-forward and parameter robustness runs
- [ ] Cost stress tests
- [ ] Publish logged results

**Phase 4 — Iterate**
- [ ] Drop models that fail out-of-sample
- [ ] Re-tune threshold and weights against the targets
- [ ] Document what worked and what did not

---

## Known Limitations

- The 70% win-rate target may not be reachable at acceptable profit factor and trade count. If it is not, the results will say so.
- Backtests on TradingView do not model partial fills, order book depth or exchange outages.
- Limited intraday history can cap the sample size for 5m tests.
- Right-edge trade markers currently show only the latest trade. Older trades are kept as position boxes.
- Gold and crypto behave differently across sessions. A setting that works on one may not transfer.

---

## Repository Layout

Planned structure:

```
devVaibhav/
├── pine/
│   ├── devVaibhav_indicator.pine
│   ├── devVaibhav_strategy.pine
│   └── lib/devVaibhavCore.pine
├── backtests/
│   ├── XAUUSDT_5m/
│   ├── XAUUSDT_15m/
│   ├── BTCUSDT_5m/
│   └── BTCUSDT_15m/        # exported trade CSVs + run logs
├── docs/
│   └── methodology.md
└── README.md
```

---

## Contributing

Issues and pull requests are welcome. For any change to signal logic:

1. State what changed and why.
2. Re-run the affected backtests with the logged settings.
3. Include before/after metrics and the exported CSV.
4. Do not change the out-of-sample window to make a result look better.

---

## Disclaimer

This project is for education and research. It is not financial advice. Backtested and indicator-tracked results do not guarantee future performance. Leveraged trading on gold and crypto can lose more than you expect. Test on a demo account first and only risk money you can afford to lose.

---

<div align="center">

Built by [DeveloperVaibhav1](https://github.com/DeveloperVaibhav1)

</div>
