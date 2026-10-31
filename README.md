# Bank Spread Mean Reversion: TCB & STB vs. Financial Sector Benchmark

A statistical arbitrage backtest on two Vietnamese bank stocks, **Techcombank (TCB)** and **Sacombank (STB)**, traded against a sector benchmark. The idea is to open a position when the price spread between the banks and the benchmark moves unusually far from normal, then close it when the spread returns toward its average. Built with [Backtrader](https://www.backtrader.com/).

## Results (main notebook)

| Metric | Value |
|---|---|
| Period | 2020-03-19 to 2025-07-21 |
| Starting capital | 1,000,000,000 VND |
| Ending value | 1,053,701,187 VND |
| **Net return** | **+5.37%** (no leverage) |
| Round trips | 22 (average hold about 1.5 days) |
| Commission | 0.1% per leg, about one-third of gross profit |

Positions are open in fewer than 3% of trading sessions, so most of the capital sits unused. This is a proof of concept, not a production strategy.

## Strategies

### 1. `vn30_bank_etf_mean_reversion.ipynb` (main)
- **Benchmark:** SSIAM VNFIN LEAD ETF (FUESSVFL)
- **Spread:** `0.5 × (TCB + STB) − FUESSVFL`
- **Signal:** Bollinger Bands on the spread (50-period window, 2.8 standard deviations)
  - Spread below the lower band → buy TCB and STB, short FUESSVFL
  - Spread above the upper band → short TCB and STB, buy FUESSVFL
  - Close all legs when the spread moves back inside the bands
- **Sizing:** positions are bought in "bundles" of 1 TCB : 1 STB : 2 FUESSVFL. This keeps the trade matched to the spread formula. Each trade uses 60% of available cash.
- **Risk controls:**
  - Stop-loss at a 5% adverse change in the spread
  - Take-profit at a 10% favorable change
  - Emergency exit closes all legs if any order is rejected or canceled

### 2. `vn30_meanreversion_stb_tcb_flexible.py`
- **Benchmark:** VN30 index
- **Spread weights:** estimated with a Johansen cointegration test on log prices, scaled so the VN30 weight is −1
- **Lookback window:** 5x the spread's half-life (how quickly it reverts to its average), limited to 20–120 bars
- **Entry threshold:** the 90th percentile of the absolute z-score (how many standard deviations the spread is from its average)
- **Exit:** when the z-score is within ±0.1
- **Stop:** one standard deviation beyond the entry threshold
- **Output:** daily and per-trade Sharpe ratios

### 3. `vn30_meanreversion_stb_tcb_hardbounds.py`
Same setup as the flexible version, but the thresholds are fixed by hand:

| Parameter | Value |
|---|---|
| Long entry | z-score below −2.5 |
| Short entry | z-score above +1.0 |
| Exit | z-score within ±0.1 |
| Stop | 2.0 |

## Data

| File | Contents |
|---|---|
| `tcb.csv`, `stb.csv` | Raw daily prices (Vietnamese column headers) |
| `tcb_final.csv`, `stb_final.csv`, `fuess_final.csv` | Cleaned daily prices with columns `Day, Close, Open, High, Low, Volume`, used by the notebook |
| `vn30.csv` | VN30 index daily prices with columns `tradingdate, open, high, low, close` |
| `trade_log.csv` | Results for each leg of each trade (gross and net PnL, bars held), exported by the notebook |
| `trade_records.csv`, `trade_summary.csv` | Earlier trade outputs |

> The `.py` scripts read from `data/tcb_ver2.csv`, `data/stb_ver2.csv` and `data/vn30_ver2.csv`. Each file needs a date in the first column and a `Close` column. These files are not included in this repo.

## Setup

```bash
pip install backtrader pandas numpy statsmodels matplotlib
python vn30_meanreversion_stb_tcb_flexible.py
```

The notebook was written for Google Colab and reads from a Google Drive folder. Update the file paths in the run cell before using it anywhere else. The first cell patches an outdated Matplotlib function name that Backtrader's plotting still calls.

## Limitations and next steps
- **In-sample fitting:** the cointegration weights, half-life and z-score thresholds are all calculated from the full dataset, so the backtest results are optimistic.
- **Same-bar fills:** the `.py` scripts use `set_coc(True)`, which fills trades at the closing price of the same bar that produced the signal.
- **Untested spread:** the notebook's fixed 1:1:2 spread has not yet been checked for cointegration (ADF or Engle-Granger test).
- **Planned:**
  - Widen entry thresholds to filter out weak signals
  - Validate out of sample on data after July 2025
  - Add leverage or margin modeling to reduce idle cash
