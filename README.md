# Crypto Statistical Arbitrage: Momentum, Reversal, and Carry

A short-horizon (4-hour) cross-sectional study across 35 large-cap cryptocurrency perpetual futures. The goal is to explore a tradable statistical-arbitrage edge in liquid crypto. The analysis builds clean price and funding panels; tests momentum, reversal, and carry signals; and combines the survivors into a market-neutral book under train, validation, and holdout splits.

The Jupyter notebook (`crypto_statarb.ipynb`) is kept concise and is meant to be read top to bottom. This README explains the decisions behind it.

> 📓 **[View the rendered notebook on nbviewer](https://nbviewer.org/github/dcroessler/crypto_qr/blob/main/crypto_statarb.ipynb)** — GitHub's in-page notebook preview is occasionally flaky on figure-heavy notebooks; nbviewer always renders it.

## Final Result

A single fast signal does not survive costs. A diversified, slow-traded book of momentum and carry, with the momentum speed and the blend weights both re-chosen each quarter from past data only, is net-positive across all three periods at every cost level:


| net Sharpe | train (20-22) | validation (23) | holdout (24+) |
| ---------- | ------------- | --------------- | ------------- |
| 7 bps      | +2.68         | +1.06           | +1.89         |
| 13 bps     | +2.49         | +0.75           | +1.63         |
| 20 bps     | +2.26         | +0.37           | +1.33         |


Full sample, scaled to 10% annual volatility, benchmark BTC: Sharpe 1.62, annualized alpha about 13% (t-stat 3.9), max drawdown -7%, beta about zero. The book earns a real, market-neutral return.

## Data

- **Source:** Binance USDT-M perpetual 4h klines and 8h funding rates, downloaded as monthly files from the public `data.binance.vision` CDN and cached to `data/` (parquet).
- **Why perpetuals, not spot:** the strategies are long/short and market-neutral, so they need an instrument that can actually be shorted, and the carry signal is defined on perpetual funding. Spot would make both incoherent.
- **Universe:** 35 of the largest coins with sufficient perp history (from 2020-2021). A larger cross-section makes the ranking signals more robust. Coins enter the panel when their perp lists.
- **Cleaning:** the kline file format gained a header row in 2024, and funding timestamps jitter by a few milliseconds; both are handled. Funding stamps are rounded to the hour so they land on the grid (without this, roughly half of all funding events were silently dropped). Days missing from occasionally-short monthly files are backfilled from the daily archive, and 1:1 rebrands (MATIC to POL, EOS to A) are stitched onto the original series so the asset stays in the panel instead of vanishing.

## Methodology decisions

**Train / validation / holdout.** History is split into train (2020-2022), validation (2023), and a holdout (2024 onward). All signal and parameter choices are made on train, validation is the honest check, and the holdout is supposedly* scored once. This is the main guard against overfitting, which is the failure mode this project is built to avoid.

*see Honest Limitations

**Costs from the start.** Trades are charged 20 bps (market orders); 7 and 13 bps are also reported for limit-order execution, which is realistic for these patient, liquidity-providing strategies. Costs are applied from the first backtest because at a 4h horizon they dominate and decide everything.

**Signal construction.** Cross-sectional signals rank coins, demean, and normalize to a dollar-neutral book with unit gross exposure. Time-series signals are transformed through tanh. Carry ranks coins by perpetual funding (short the high-funding names) and the carry P&L includes the funding cashflow, capped at 0.5% per 8h so a single dislocation (for example SOL during the 2022 FTX collapse) cannot dominate.

**Why a published trend signal.** Hand-built signals only marginally beat costs, and tuning them further by hand is how overfitting starts. The momentum sleeve therefore uses the Baz / Rohrbach trend construction (the average of three EWMA crossovers at different speeds, volatility-normalized, passed through a response that down-weights extreme trends). Its parameters come from the literature rather than from fitting our data, and its multi-speed design keeps turnover low enough to clear costs.

**Walk-forward selection (no in-sample weights).** Two things are chosen quarterly using only the prior year of data and then held forward: the momentum speed, and the momentum/carry blend weights. Standing at the end of each quarter, weights are estimated from the trailing year and applied to the next quarter, then re-estimated. This avoids the mistake of computing optimal weights and scoring them on the same period. Inverse-volatility (risk-parity) weights are used because a mean-variance optimizer is unstable with so few sleeves.

**Diversification.** Momentum and carry are roughly uncorrelated and strong in different regimes (momentum held up in 2023, carry in 2024-2026), so an equal-risk blend has a smaller drawdown and is positive in every period even when each sleeve alone is not. This is the core of the result.

**Performance metrics.** Returns, volatility, Sharpe, max drawdown, and alpha and beta against BTC, plus the alpha t-stat (the t-statistic on the intercept of a regression of the strategy's returns on BTC). A t-stat above about 2, together with a near-zero beta, indicates a real and independent edge.

## Honest limitations

- ++The holdout is not a clean one-shot test.++ It was evaluated three times as the approach evolved: first on a hand-built strategy (no literature, no rebalancing), then with the literature trend signal, and finally with that signal plus quarterly rebalancing (this book). Each look erodes its independence, so the result is robust and reasoned, not a pristine out-of-sample proof. A genuinely clean test needs future data.
- The 35-coin universe is fixed rather than point-in-time, so there is mild survivorship bias.
- 7 bps assumes limit-order fills, which suffer adverse selection; realistic all-in cost is 7-20 bps, and the result is reported across that range.
- Natural next steps: a broader point-in-time universe and live forward testing.

## References

- Hubrich (2017), *Know When to Hodl 'Em: Factor-Based Investing in the Cryptocurrency Space* (equal-weight factor combination).
- Liu, Tsyvinski & Wu (2022), *Common Risk Factors in Cryptocurrency*, Journal of Finance.
- Rohrbach, Suremann & Osterrieder (2017), *Momentum and Trend Following Trading Strategies for Currencies Revisited* (trend signal; builds on Baz et al. 2015).

## Running it

```
pip install -r requirements.txt
jupyter notebook crypto_statarb.ipynb
```

The first run downloads and caches the data (about 30 minutes, latency-bound); later runs load the cached parquet files instantly.

## Acknowledgments

Claude Code (Anthropic's agentic coding assistant) was used throughout the implementation, for the data pipeline, backtest and plotting code, and iterative debugging. The research direction, design decisions, and final interpretation are my own.