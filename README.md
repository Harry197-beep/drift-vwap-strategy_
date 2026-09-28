

# Drift VWAP Pullback: backtests on IDX stocks and gold

![Python](https://img.shields.io/badge/python-3-blue) ![Status](https://img.shields.io/badge/status-research%20log-lightgrey) ![Not financial advice](https://img.shields.io/badge/not%20financial%20advice-red)

**English** | [Bahasa Indonesia](README.id.md)

An independent attempt to rebuild the *Drift VWAP Pullback* day-trading strategy (presented by
Matteo Conti for NASDAQ 100 futures) and test it on the Indonesian market (the IHSG index,
BBCA, and 55 LQ45 stocks) and on gold.

**Result: no edge survived trading costs.** This repo is a record of what was tried, not a
strategy to trade.

> [!WARNING]
> Educational research only, not financial advice. The rules were reconstructed from a text
> summary of the video, so they may not match what the original author trades. This project is
> not affiliated with Matteo Conti.

## Results at a glance

Five-minute bars from Yahoo Finance, about 60 days, before trading costs unless stated.
Snapshot taken 28 Sep 2026; Yahoo only serves a rolling window, so a re-run will differ.

| Market | Window | Trades | Win rate | Result |
| --- | --- | --- | --- | --- |
| IHSG index (`^JKSE`, TWAP) | 2026-07-06 to 09-25 | 11 | 45.5% | -1.25% total, max drawdown -2.23% |
| BBCA (`BBCA.JK`, real volume) | same | 14 | 85.7% | +2.12% total, max drawdown -0.45% |
| 55 LQ45 stocks, pooled | same | 618 | 73.5% | +0.066% per trade |
| Gold futures (`GC=F`) | 2026-07-19 to 09-28 | 36 | 47.2% | -0.050% per trade (t = -1.5) |

## The strategy

Rules as summarized from the video (built for NQ futures on 5-minute and 15-minute charts):

- Ignore the first hour of the session (9:30-10:30 ET) so VWAP can settle.
- Three conditions must hold together:
  1. Price stays above VWAP for longs (below it for shorts).
  2. VWAP slopes the same way over the past 15 minutes.
  3. Price moved at least 0.1% in that direction over the past hour.
- **Trigger:** the first pullback to VWAP. For a long, enter at the open of the next candle
  after a red candle touches near VWAP. For a short, after a green candle does.
- **Risk:** about 80 points at risk to make 40-50 points (a negative reward-to-risk ratio).
- **Guardrails:** one position at a time, at most 4 trades a day, stop after 2 consecutive
  losses in a day, no new trades after 15:30 ET, everything closed by 15:55 ET.

## What the strategy is for

According to the summary, the goal is **passing prop-firm challenges**, not growing a personal
account. The stated numbers: 64% win rate; 49.8% chance to pass one challenge (20,000
simulations); 74.8% / 87.3% / 93.6% chance to pass at least one in 2 / 3 / 4 attempts; about 3.4
trading days to pass; developed on 2020-2024 data and checked on 2024-August 2026.

Two things stand out from the arithmetic (our reading of a secondhand summary):

- **Break-even win rate = risk / (risk + reward).** Risking 80 to make 40-50 needs 61.5% to
  66.7%. At 45 points it is exactly 64%, so the claimed win rate sits at break-even before costs.
- **The multi-attempt figures are just compounding.** `1 - 0.502^n` gives 74.8%, 87.3% and 93.6%.
  That is repeated independent tries at a roughly coin-flip challenge, and says nothing about
  profit per trade. Prop challenges usually charge an evaluation fee, so check the firm's terms.

## How the backtests work

- 5-minute OHLCV from Yahoo's public chart endpoint, using only `requests` (no `yfinance`).
- **Percent, not points.** NQ's 80/40-50 points do not transfer to a different price level, so
  stop, target and drift thresholds are percentages (0.45% stop, 0.25% target for IDX).
- **Jakarta hours.** 09:00-12:00 and 13:30-15:49 (Friday 09:00-11:30 and 14:00-15:49). VWAP resets
  each half-session and the first 30 minutes of each are ignored.
- **Real VWAP when volume exists.** Yahoo reports zero volume for the `^JKSE` index (an index is
  computed, not traded), so that run falls back to an unweighted average price (TWAP).
- **Fills.** Entry at the next bar's open, exit at the exact stop or target, stop checked first
  if both are hit in one bar.
- **Gold version.** COMEX session 08:20-13:30 ET with VWAP anchored at the open, both
  directions, a maximum-hold time stop, and a 27-setting grid (stop, target, hold time) ranked on
  the first half of the days and scored on the second half.

## Results in detail

**IDX, 618 trades:** average win +0.250%, average loss -0.448%. Exits: 452 target, 159 stop, 7
end of day. 39 of 55 stocks had a positive average. Longs: 154 trades, 70.1% win, +0.04%. Shorts:
464 trades, 74.6% win, +0.07%. 40 of 57 trading days were positive.

**After costs (IDX):** break-even round-trip cost is 0.066%. At 0.1% the average trade is
-0.034%, at 0.2% it is -0.134%, at 0.4% it is -0.334%. Retail round trips on IDX are commonly
around 0.25-0.4% (brokerage plus the sell-side tax; check your own broker).

**Holding longer (IDX long entries, 154 trades):** the signal's return over a random entry in the
same stock was -0.04% same day, +0.38% at +1 day (standard error 0.22), +0.27% at +3 days
(0.33), -0.22% at +5 days (0.41). That is noise. Random entries gained 0.09%, 0.42% and 0.70%
over 1, 3 and 5 days, so the window was a rising market and any buy-and-hold looks good in it.

**Gold:** stop 0.25% / target 0.15% / 60-minute hold. Break-even win rate 62.5%, actual 47.2%.
Longs 47.1%, shorts 47.4%, both about -0.05% per trade. First half -0.065%, second half
-0.040%. Costs of 0.01%-0.03% make it -0.060% to -0.080%. **0 of 27 grid settings were positive
in both halves**; the best first-half setting was -0.017% there and -0.054% on the second half.

## What we tried

1. Converted point thresholds to percentages and re-mapped the session to Jakarta hours.
2. Tried the iTick data API for real index data. Its free plan has no stock or index data, so
   it was dropped (the code path is still in `idx_drift_vwap.py`, unused).
3. Replaced the volume-less index with a real, liquid stock (BBCA) so VWAP is real.
4. Widened to the 55 LQ45 stocks to check whether BBCA was luck (it was not, but the edge is tiny).
5. Split results by direction and subtracted costs.
6. Tested holding for 1, 3 and 5 days instead of scalping.
7. Moved to gold with both directions, a time stop, and a first-half/second-half check.

## Why it did not help

- The average IDX trade earns about 0.07%; realistic costs are several times that.
- Gold shows no edge in either direction or half.
- The shape of the strategy (small wins, large losses, high win rate) suits a challenge with a
  profit target and a drawdown limit, not a per-trade edge.

## Limitations

- **Small samples.** About 60 days of 5-minute data. The 618 IDX trades come from one window and
  move together, so the real sample is much smaller.
- **Two engine versions.** The IDX runs used an earlier, looser trigger (previous candle's colour
  plus a touch, and a 0.05% minimum VWAP slope). The gold run follows the summary more literally
  (the touching candle itself must be red or green, and VWAP only needs the right sign). Results
  are not perfectly comparable.
- **Simplified fills.** Tick size, bid-ask spread, slippage and IDX limits on retail short selling
  are not modelled. Costs are assumptions, not your broker's numbers.
- **Data.** `GC=F` futures stand in for spot XAUUSD (spot has no central volume). A 1.68% single
  5-minute move appeared in the gold data and was not investigated.
- **Secondhand rules.** The tests may not match what the original author actually trades.

To test the original claim properly, run it on NQ itself over years of 5-minute data with real
volume and realistic commissions and slippage. Free sources do not offer that.

## Repo contents

| File | What it does |
| --- | --- |
| `idx_drift_vwap.py` | Backtest for IDX data: Yahoo bars, WIB sessions, midday break. Edit `TICKER` at the top |
| `run_universe.py` | Runs that backtest over the LQ45 list and pools the trades (skips stocks with too little or zero-volume data) |
| `analyze.py` | Direction split, cost scenarios, winning days |
| `hold_test.py` | Compares holding IDX long entries for days against random entries |
| `gold_vwap.py` | Gold version with time stop and walk-forward grid |
| `fetch_idx_repo.py` | Inspects the `Dataset-Saham-IDX` GitHub repo |
| `drift_vwap_story.py` | Manim animation summarizing the findings |

## Quick start

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
pip install -r requirements.txt

python idx_drift_vwap.py     # one ticker: set TICKER = "BBCA.JK" (or "^JKSE") at the top
python run_universe.py       # LQ45 universe, takes a few minutes
python analyze.py            # reads universe_trades.csv from the previous step
python hold_test.py          # multi-day holding test
python gold_vwap.py          # gold
```

Keep `DATA_SOURCE = "yahoo"` in `idx_drift_vwap.py`. Everything runs on plain Python without a C
compiler (useful on Android terminals such as VSCodroid, where `yfinance` fails to install
because of `lxml`); `tzdata` is needed for the Jakarta and New York timezones.

The animation needs Manim, which requires cairo, pango and ffmpeg, so render it on a desktop or
Google Colab: `manim -pql drift_vwap_story.py DriftVWAPStory`.

> [!CAUTION]
> Never commit API keys. If you enable a keyed data source, keep the key out of the repo.

## Data sources and credit

- Price data: Yahoo Finance's unofficial chart endpoint, for personal and educational use.
  Check their terms. No price data is stored or redistributed in this repo.
- [`wildangunawan/Dataset-Saham-IDX`](https://github.com/wildangunawan/Dataset-Saham-IDX) was
  used only to get the LQ45 stock list. Its data has its own terms.
- Strategy presented by **Matteo Conti**. This is an independent, unofficial replication attempt.
-
