# RE: Kalshi's "Who Calibrates Markets"

**What must a taker strategy prove before it trades real money?**

**Short summary.** Kalshi's research team published ["Who Calibrates Markets? Evidence from Trader Micro-Data"](https://kalshi.com/research/publications/who-calibrates-markets) (Kagan
and Baiocchi, October 2026) on 19 Oct. Their headline is that about 0.15 % of Kalshi accounts forecast better than the
price they trade against, after correcting for luck, and that the price moves their way after they trade. We build
model-driven trading strategies for motorsport markets (racinglines), so we turned their paper into a checklist: five
things any taker strategy, ours or a customer's, must prove before it trades real money on Kalshi or any prediction
market. We then rebuilt the parts of their analysis our own data allows. On their skill scale, our stored NASCAR Kalshi
paper bets score +0.0090 and beat a 1,000-draw luck test (p < 0.001), but that is not yet skill: the bets are in-sample,
their fills are synthetic, and 40 races are too few. Motorsport takers on Kalshi pay 2.1 to 2.5¢ over the mid, about
twice the paper's average, and a last Kalshi trade prints about 4¢ from the mid. **This is a live draft, before our simulation runs and tracked in our commit history** (link TBA after the
first commit). As the simulation runs, it will score our strategies trade by trade in the paper's framework.

*All times are UTC. "The paper" is Kagan and Baiocchi (2026), [kalshi.com/research](https://kalshi.com/research/publications/who-calibrates-markets); we point to its sections rather than restating its
methods, and we don't republish it. Our numbers come from read-only queries on our production database, run 19 Oct
23:45 to 20 Oct 00:00. Nothing here is betting advice.*

*Access on request: PR links go to our private repository and internal-book links (racinglines.bet/docs/internal) are admin-only. Those, our process notes, the internal draft and the query list behind every number can be shared on request: hello@racinglines.bet.*

## The answer: five golden rules

Before any taker strategy is deployed with real money on Kalshi or any prediction market, by us or by a customer, it
must prove all five. A customer proves them on their own fees, latency and size. We adopted these as our own rules on
19 Oct; every taker strategy we build is now checked against them. They open our development guide ([#274](https://github.com/racinglines/racinglines/pull/274)) and the trading chapter of our internal book ([3.4 Trading](https://racinglines.bet/docs/internal/system/trading.html)).

| # | Rule | Paper | What we already have | What's missing | Status today |
|---|---|---|---|---|---|
| 1 | **Skill, not luck or selection.** A positive Brier skill score against the size-weighted mid at realistic fills, surviving a luck test with a false-discovery correction across *every* setting tried | sec 3.2–3.3, Table 1, App. A.1 | Model log loss, Brier and calibration against the market in every sweep ([#221](https://github.com/racinglines/racinglines/pull/221), [#227](https://github.com/racinglines/racinglines/pull/227)); a stage-by-stage walk-forward pick ([#206](https://github.com/racinglines/racinglines/pull/206)) | Skill score per trade against the weighted mid; luck test; correction across the whole grid | **Not met.** NASCAR scores +0.009, but in-sample and with no grid correction |
| 2 | **Out of sample.** It holds forward on pre-registered locks, not on the season it was tuned on | sec 4.4, Table 4 | Per-event strategy locks, committed before each event and scored as bet (since 15 Oct, [#213](https://github.com/racinglines/racinglines/pull/213)); season holdouts ([#212](https://github.com/racinglines/racinglines/pull/212), [#228](https://github.com/racinglines/racinglines/pull/228), [#237](https://github.com/racinglines/racinglines/pull/237)) | A long enough forward record | In force; the record is still short |
| 3 | **The real book.** The edge clears the actual spread, depth and fees: fills at the touch or walking the recorded book, never a last-trade price plus a fixed cost | sec 2.1, App. A.4 | Kalshi touch fills ([#55](https://github.com/racinglines/racinglines/pull/55), on candle bid/ask from [#52](https://github.com/racinglines/racinglines/pull/52), [#53](https://github.com/racinglines/racinglines/pull/53)); touch or book-walk fill options ([#203](https://github.com/racinglines/racinglines/pull/203), [#204](https://github.com/racinglines/racinglines/pull/204), [#205](https://github.com/racinglines/racinglines/pull/205)) with a depth limit ([#219](https://github.com/racinglines/racinglines/pull/219)); a fill-error check against Probalytics' full order books ([#202](https://github.com/racinglines/racinglines/pull/202); tape from [#181](https://github.com/racinglines/racinglines/pull/181), [#200](https://github.com/racinglines/racinglines/pull/200)) | Our stored NASCAR/MotoGP Kalshi paper bets used a one-price fill (last price ± 1¢) | **Not met** for those paper bets; our F1 backtests fill at the touch |
| 4 | **The price comes our way.** After entry the mid moves toward our side and doesn't reverse | sec 3.4, 5.4, Fig 9 | Closing-line value (CLV) for every backtest ([#223](https://github.com/racinglines/racinglines/pull/223)) and paper bet ([#81](https://github.com/racinglines/racinglines/pull/81)), which is the close-time point of that path | Markouts at 5 min, 1 h and the next stage; the outcome gap | Partial: close point only |
| 5 | **Paper reconciles with real.** Simulated fills match real fills before any money goes in | (ours, not the paper's) | Probalytics' books as the truth for fills ([#202](https://github.com/racinglines/racinglines/pull/202), [#212](https://github.com/racinglines/racinglines/pull/212)); our go/no-go goal of CLV above +2 % over 100 live bets | Real fills | Not started (no real money yet) |

## 1. The paper in one table

| Finding | Number | Paper |
|---|---|---|
| Skilled accounts / share of taker volume | 14,694 (0.15 %) / 22.8 % | Table 1, sec 4.2 |
| Brier on their own trades, skilled vs market | 0.13813 vs 0.14705; 64 % of the gain is calibration | Table 3 |
| Median skill score: all skilled / better on both parts | 0.0189 / 0.0382 | Table 3B |
| Skilled again in the other half of events | 44.5 % (all traders 3.0 %) | Table 4 |
| Skilled taker: paid over mid; mid move by next order / by close; outcome gap | 1.5–1.8¢; 1.7¢ / 3.3¢; 3.8¢ | Fig 9 |
| Other takers: paid over mid; mid move | 1.2¢; ≤ 0.3¢ | sec 5.4 |
| YES share of taker trades, other vs skilled | 67 % vs 53 % | sec 4.3 |
| Sports markets with no skilled taker | 8.2 % (no per-sport split; F1 and motorsport are not named) | Table 5 |

Their own limits are in sec 6: the impact results are associations, the skilled set is chosen on outcomes, arbitrage
scores as skill, and the sample covers one year.

## 2. Their outputs, rebuilt on our data

### 2.1 Fig 1: where our strategies sit on the skill distribution

**Paper method:** sec 3.2, eq. 2–3; luck test in sec 3.3.

**What we scored:** our 850 stored Kalshi taker paper bets (followed, market resolved, de-duplicated). These are paper
bets from demo profiles, not real money.
- NASCAR: 815 bets over 40 races (race win 2025–26; top 5/10/20, podium and head-to-head in 2026).
- MotoGP: 35 race-win bets over 11 races.
- F1: our F1 strategies run on Kalshi in backtests, but no F1 Kalshi paper bets are stored yet.

**The assumption we break:** our benchmark is each bet's stored price, not the size-weighted mid. For these bets that
price is last-trade-type (sec 2.5).

| Sport | Bets | Races | Skill score | Luck null: mean / 95th pct | p |
|---|---|---|---|---|---|
| NASCAR | 815 | 40 | **+0.0090** | −0.0008 / 0.0036 | < 0.001 |
| MotoGP | 35 | 11 | +0.0034 | −0.0035 / 0.0141 | 0.33 |

![Our two strategies on the skill scale](img/fig1-bss-placement.png)

On the paper's Fig 1, NASCAR would sit just right of the peak, below the paper's median skilled trader.

**Why this is not yet skill:**
- **Selection.** The "best effort" profile picked each market kind's best setting on these same races, with no
  false-discovery correction over the grid it came from.
- **Too few events.** 40 races is under the paper's 100-event bar.
- **A synthetic fill.** Every stored Kalshi taker bet filled at exactly its stored price ± 1¢.

### 2.2 Table 3: where the edge comes from

**Paper method:** sec 4.3, eq. 5 (the Murphy split). Same 815 NASCAR bets, 20 bins, contract-weighted; our split leaves
out the within-bin residual.

| | Our fill | Market (stored price) | Difference |
|---|---|---|---|
| Reliability (lower is better) | 0.0087 | 0.0098 | −0.0011 |
| Resolution (higher is better) | 0.0809 | 0.0816 | −0.0007 |
| Uncertainty | 0.2344 | 0.2344 | 0 |
| Brier | 0.1615 | 0.1630 | −0.0015 |

The whole gain is calibration: the bets pick sides the stored price under-prices, and resolution is slightly worse.
That fits the paper's majority pattern (Table 3B). Because the fill is fixed at price + 1¢, "our fill" here is the
market price shifted 1¢ toward the sides we chose.

### 2.3 App. A.1–A.2: a skill score is not profit

The same 815 bets, re-priced at mid + h. The paper's App. A.2 explains why the score can rise while profit falls.

| Paid over mid | 1¢ | 2¢ | 3¢ | 4¢ |
|---|---|---|---|---|
| Skill score | 0.0090 | 0.0168 | 0.0234 | 0.0287 |
| Profit per contract (¢, contract-weighted) | 6.85 | 5.86 | 4.86 | 3.87 |

![Skill score and profit as the fill gets worse](img/fig2-bss-vs-cost.png)

So rule 1 is checked at realistic fills, and always next to profit and CLV.

### 2.4 Fig 9: price impact, for the whole motorsport crowd

**Paper method:** sec 3.4 "Price Impact of Individual Orders", sec 5.4.

**What we used:** every Kalshi trade in our tape on markets that have resolved since 9 Oct: 34,327 F1 trades from one
race weekend and 30,350 NASCAR trades from one race. m0 is the size-weighted mid of our last order-book snapshot at most
6 minutes before the trade; later mids are the first snapshot in each window.

**The assumptions we break:**
- No skill split: Kalshi's public tape has no account ids, so this is the paper's "all takers" in aggregate.
- Our books are 5-minute snapshots, so there is no "next order" or 1-minute point.
- One event per sport, so the outcome gap is a single draw of luck.

| All takers | Paid over mid | Mid move 5–11 min | Mid move 60–66 min | Outcome − mid | YES share |
|---|---|---|---|---|---|
| F1 | 2.45¢ | 1.90¢ | 1.25¢ | 5.04¢ | 68.8 % |
| NASCAR | 2.08¢ | 0.82¢ | 0.95¢ | 0.88¢ | 76.0 % |
| Paper, other takers | 1.2¢ | ≤ 0.3¢ | — | −0.1¢ | 67 % |

![Crowd markouts on Kalshi motorsport](img/fig3-crowd-markout.png)

**What it says:**
- Motorsport takers on Kalshi pay about twice the paper's average spread (2.1–2.5¢ against 1.2¢).
- They lean YES more than the paper's takers (69–76 % against 67 %).
- The mid keeps part of the move (1–2¢ at 5 minutes). Some of that is likely a trade clearing a level in a thin book;
  the paper's sec 5.4 shows how to separate it by order size, which we haven't done yet.

We left out a "close" point: our last recorded book for a market isn't Kalshi's close.

### 2.5 Before eq. 4: how far the last trade is from the mid

**Paper method:** sec 3.4 "Order Flow and the Last Traded Price". The regression itself comes with the simulation; this is the step
before it. All Kalshi motorsport trades in the last 3 days were matched to our last book at most 6 minutes earlier.

| Taker side | Trades | Avg spread | Trade − plain mid | Trade − weighted mid | At or through the touch |
|---|---|---|---|---|---|
| YES | 23,459 | 4.8¢ | +4.2¢ | +4.7¢ | 63.7 % |
| NO | 8,764 | 8.0¢ | −4.2¢ | −3.8¢ | 50.5 % |

A last-trade price is the mid plus about 4¢ in the direction of whoever traded last. That is why rule 3 bans
"last trade plus a fixed cost" fills, and why the simulation estimates the mid from the book wherever we have one.

### 2.6 What we can't rebuild yet

| Paper output | Why not | Path |
|---|---|---|
| Table 1, Fig 2–4, Table 4 (skilled accounts, their traits, persistence) | Kalshi's public tape has no account ids | Polymarket trades carry wallets (our tape: 1,282 wallets on 7,139 F1 trades) |
| Table 5 for F1 (is F1 among the 8.2 % of Sports markets with no skilled taker?) | The paper gives no per-sport split | Per-sport skilled-taker coverage for F1 on Polymarket wallets |
| Table 6 (eq. 4) | Needs a last-price series with net order flow | With the simulation |
| Fig 6 by price bin | One resolved event per sport since our tape restarted | Probalytics fills, May to Sep 2026 |

## 3. Caveats for the simulation

1. **Simulated, not executed.** A mid move after a backtest trade is the market coming round to our view, not our
   impact. Our own impact on thin books is unmeasured.
2. **The fill model changes the score itself** (sec 2.3). We report touch and book-walk fills side by side.
3. **The benchmark mid is estimated** wherever we have no recorded book. 2025 Kalshi is estimate-only.
4. **Horizons.** Our books are 5-minute snapshots, and we trade at stage cutoffs, while the paper's Sports volume sits
   in the last tenth of a market's life (Fig 7).
5. **Selection.** The false-discovery correction runs over every combination tried. In-sample profiles are never
   evidence.
6. **Joint outcomes.** Our luck test draws each driver independently and ignores "one winner per race", so its
   p-values are approximate.
7. **Sample size.** NASCAR has 40 races and MotoGP 11; the paper's bar is 100 events.
8. **Tape gaps.** Our own Kalshi trade tape has a gap from 29 Sep to 9 Oct.
9. **Fees.** The skill score ignores fees; profit and CLV include them. We report all three together.
10. **Scope.** No cycling markets are listed on Kalshi in our data.

## How we built this: ops timeline

From the paper's release to this page, at the level of steps. **High-touch** steps needed the owner (a decision, an
upload, a publish); **streamlined** steps ran in our project threads with no input. **Quick** is under 20 minutes end
to end.

| Time (UTC) | Event | Touch | Speed |
|---|---|---|---|
| 19 Oct, about 19:20 | Kalshi publishes the paper | — | — |
| 23:43 | Owner reads it and hands it to our research thread | High-touch | Quick |
| 23:45 to 23:46 | Paper read; our Kalshi data and taker backtests inventoried; price impact (sec 3.4) mapped onto our closing-line value | Streamlined | Quick (2 min) |
| 23:46 to 23:50 | Owner sets the brief: findings, a simulation design in their framework, our gaps, the mid from last trades, our place on Fig 1 | High-touch | Quick |
| 23:50 to 23:59 | Stored Kalshi paper bets scored on their skill scale with a 1,000-draw luck test; crowd price impact rebuilt from our tape | Streamlined | Quick (9 min) |
| 23:54 to 23:56 | By-account analysis checked: possible on Polymarket only, parked | High-touch | Quick |
| 23:56 | **The question for this report is set: what must a strategy prove before it trades real money?** | High-touch | Quick |
| 23:57 to 00:01 | **Five golden rules adopted** and written into our development guide and internal book | Streamlined | Quick (4 min) |
| 20 Oct 00:04 | Internal draft (7 pages) delivered, 21 minutes after the hand-off | Streamlined | Quick |
| 00:12 to 00:20 | Owner decides to publish; this public edition built | High-touch | Quick |

*Internal book pages behind this report: [3.4 Trading (golden rules)](https://racinglines.bet/docs/internal/system/trading.html), [3.5 Evaluation and sweeps](https://racinglines.bet/docs/internal/system/evaluation.html), [4.5 DITL reports](https://racinglines.bet/docs/internal/ops/ditl.html), [Appendix A: every report and its sources](https://racinglines.bet/docs/internal/appendix/reports.html).*

## What's next

- **Estimate the mid from last trades** (paper sec 3.4) on Probalytics' order books and our own, scoring four simple
  estimators against the true weighted mid, then run eq. 4 on the crowd.
- **Dump every backtest trade** (ticker, time, side, fill, size, our fair, book with sizes, weighted mid, outcome) for
  F1, NASCAR and MotoGP on Kalshi, at touch and book-walk fills.
- **Score them in the paper's framework:** skill score for our fills and our fair, the Murphy split, a luck test with a
  false-discovery correction over the whole grid, markouts, random event halves, and a crowd baseline.
- **F1 skilled-taker coverage on Polymarket**, where trades carry wallets.

## What this does and doesn't show

- **It shows** the five rules we now hold every taker strategy to, and where each stands for us today.
- **It shows** our stored NASCAR Kalshi paper bets score +0.0090 on the paper's scale, beyond a 1,000-draw luck null,
  all of it from calibration.
- **It shows** motorsport takers on Kalshi pay 2.1–2.5¢ over the weighted mid, and a last trade sits about 4¢ from it.
- **It doesn't show** that any of our strategies is skilled: the bets are in-sample, the fills synthetic, and there are
  fewer than 100 events.
- **It doesn't show** anything about F1 on Kalshi from our own paper bets, any estimator's accuracy, or Kalshi's
  market-level calibration in motorsport.
