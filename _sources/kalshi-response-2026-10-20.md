# RE: Kalshi's "Who Calibrates Markets"

**What must a taker strategy prove before it trades real money, and how close are we?**

**Short summary.** Kalshi's research team published ["Who Calibrates Markets? Evidence from Trader Micro-Data"](https://kalshi.com/research/publications/who-calibrates-markets) (Kagan and Baiocchi, October 2026) on 19 Oct. Their headline is that about 0.15 % of Kalshi accounts forecast better than the price they trade against, after correcting for luck, and that the price moves their way after they trade. We build racinglines ([web demo](https://racinglines.bet/term), [repo](https://github.com/racinglines/racinglines-public)), a model that prices motorsport markets (Formula 1, NASCAR, MotoGP and more) and strategies that trade on it. So we turned their paper into five rules a taker strategy must pass before it trades real money, and then ran every historical test our data allows to see where we stand.

The short answer: **our model is a useful second opinion in F1 2026 and not yet more than that.**
- Scored on every F1 market it priced this season, traded or not, the model is as accurate as the market overall. Where it disagrees with the price by more than 5¢, it was right more often than wrong: about half of its predicted edge showed up in the results, on both Kalshi and Polymarket.
- Every strategy setting we have saved, 20 on Kalshi and 106 on Polymarket, bought on the right side of the market's mid in 2026 backtests. None passes the luck test once every setting tried is counted, and none holds in both 2025 and 2026. In 2025, and in NASCAR, the model loses to the market.
- After our Kalshi trades, the price kept moving our way until the race. That is the paper's mark of an informed trader, but it is measured on backtest trades, not real ones.
- So rule 1 (skill, not luck) is **partly met for F1 2026 only**, and the deciding tests are now forward: our locked per-race strategies and a pre-registered Polymarket holdout, both from the US Grand Prix in Austin (23 to 25 Oct).

**v3, 20 Oct 12:30.** This version rewrites the whole page as a state of racinglines, with the overnight results. Earlier versions: [v1, first draft](https://github.com/lalligagger/lalligagger.github.io/blob/ffbb1e9c8e2512e76fefa816090172304f6cfb1a/_sources/kalshi-response-2026-10-20.md), and v2 (02:24, the first simulation), both in this site's commit history. **No cheating:** we said before running anything that we would publish these results whatever they showed, and we have.

*All times are UTC. "The paper" is Kagan and Baiocchi (2026), [kalshi.com/research](https://kalshi.com/research/publications/who-calibrates-markets); we point to its sections rather than restating its methods, and we don't republish it. Our numbers come from read-only runs on a mirror of our production database, 19 Oct 23:45 to 20 Oct 11:03. Nothing here is betting advice.*

*Access on request: the PR links (#NNN) point to github.com/racinglines/racinglines, our private source repository, so they only open for its collaborators. Our public code is [racinglines/racinglines-public](https://github.com/racinglines/racinglines-public) (MIT) and the `racinglines` package on PyPI. Internal-book links (racinglines.bet/docs/internal) are for admins only. The PRs, our process notes, the internal draft and the query list behind every number can be shared on request: hello@racinglines.bet.*

## Why this paper matters to us

A prediction market price is a crowd's forecast. The paper asks who makes that forecast accurate, and finds a small group of traders who buy and sell against the price and are right more often than it is. Racinglines is an attempt to be one of those traders, in motorsport, with a model instead of intuition. Our model simulates each race from the drivers' recent results, practice and qualifying, and turns that into a fair price for every market it can: who wins, who makes the podium, head-to-heads and more. A strategy then decides when a gap between our fair price and the market's is worth trading.

The paper gives us a way to check ourselves that most backtests skip. A backtest can look profitable because one of many settings got lucky, because it fills at prices nobody could have traded at, or because the season it was tuned on was unusual. The paper's tests (skill against the mid, a luck test over every account, what happens to the price after a trade) answer each of those. They are also the right tests for anyone, a customer included, who wants to trade on a model.

## The answer: five golden rules

Before any taker strategy is deployed with real money on Kalshi or any prediction market, by us or by a customer, it must prove all five. A customer proves them on their own fees, latency and size. We adopted these as our own rules on 19 Oct, and every taker strategy we build is now checked against them. They open our development guide ([#274](https://github.com/racinglines/racinglines/pull/274)) and the trading chapter of our internal book ([3.4 Trading](https://racinglines.bet/docs/internal/system/trading.html)).

| # | Rule | Status, 20 Oct | Evidence |
|---|---|---|---|
| 1 | **Skill, not luck or selection** | **Partly met: F1 2026 only** | [3.1](#the-model-on-every-market-it-prices), [3.3](#every-setting-tried-the-luck-test) |
| 2 | **Out of sample** | In force; forward record starts at Austin | [3.4](#a-pre-registered-polymarket-test) |
| 3 | **The real book** | **Met for F1 backtests** (full order books, every fill had a book); not met for NASCAR/MotoGP paper bets | [3.3](#every-setting-tried-the-luck-test), [4.1](#fig-1-where-our-strategies-sit-on-the-skill-distribution) |
| 4 | **The price comes our way** | **Partly met:** yes on Kalshi in backtests, about break-even on Polymarket | [3.5](#does-the-price-come-our-way) |
| 5 | **Paper reconciles with real** | Not started (no real money yet) | — |

**1. Skill, not luck or selection.** A positive Brier skill score against the size-weighted mid at realistic fills, surviving a luck test with a false-discovery correction across *every* setting tried.
- *Paper:* sec 3.2–3.3, Table 1, App. A.1
- *Where we are:* in F1 2026 every saved setting beats the mid on both venues, but none survives the correction over all settings, and none holds in 2025 as well.

**2. Out of sample.** It holds forward on pre-registered locks, not on the season it was tuned on.
- *Paper:* sec 4.4, Table 4
- *Where we are:* per-event strategy locks, committed before each race and scored as bet (since 15 Oct, [#213](https://github.com/racinglines/racinglines/pull/213)), and a pre-registered Polymarket test ([#276](https://github.com/racinglines/racinglines/pull/276)) whose blind half starts at Austin.

**3. The real book.** The edge clears the actual spread, depth and fees: fills at the touch or walking the recorded book, never a last-trade price plus a fixed cost.
- *Paper:* sec 2.1, App. A.4
- *Where we are:* our F1 backtests now fill against Probalytics' full order books on both venues ([#202](https://github.com/racinglines/racinglines/pull/202), [#203](https://github.com/racinglines/racinglines/pull/203)–[#205](https://github.com/racinglines/racinglines/pull/205), [#219](https://github.com/racinglines/racinglines/pull/219)), with every trade written out ([#275](https://github.com/racinglines/racinglines/pull/275)). Our stored NASCAR and MotoGP paper bets still used a one-price fill.

**4. The price comes our way.** After entry the mid moves toward our side and doesn't reverse.
- *Paper:* sec 3.4, 5.4, Fig 9
- *Where we are:* markouts of every backtest trade, at the next stage of the race weekend and at the last one before the race, plus closing-line value ([#223](https://github.com/racinglines/racinglines/pull/223)).

**5. Paper reconciles with real.** Simulated fills match real fills before any money goes in.
- *Paper:* (ours, not the paper's)
- *Where we are:* our go/no-go goal is CLV above +2 % over 100 live bets and six live race weekends that reconcile. No real money yet.

## 2. The paper in one table

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

## 3. Where racinglines stands: every historical test we can run (20 Oct)

Two questions run through this section. First, **is the model's price better than the market's?** That is a question about the model alone, so we score it on every market it prices. Second, **is a gap between our price and the market's an edge worth trading?** That is a question about strategies, so we score every strategy setting we have ever saved, with the fills a real trader would have got. The paper answers both for Kalshi's traders; we answer them for ourselves.

Everything here is a backtest on a mirror of our production database, read-only. Prices and fills come from two tapes: our own recorder's ("our tape") and Probalytics' full order books, which carry every level and size ("Probalytics' books"). All intervals are 95 % and resample whole race weekends, because outcomes within one race are linked.

### 3.1 The model on every market it prices

**What this tests.** Here the model is the strategy: no tuned thresholds, no selection. At every stage of every race weekend (before the weekend, after each practice, after qualifying), we take every market the model prices and the venue's price at that moment, whether or not we traded. Two numbers come out:
- **Skill score against the market** (the paper's BSS): above zero means the model's probabilities beat the market's on the outcomes.
- **The edge curve.** For each market, the predicted edge is our price minus the market's, and the realized edge is the outcome minus the market's price. If the slope of realized on predicted is 1, the model's edge is all real; if it is 0, none of it is. This is what decides "a fair price" vs "an edge worth trading": a strategy should trade only where the part of the edge that is real clears the costs.

| Model, venue, tape, season | Race weekends | Market-stages | Skill vs market | Edge slope [95 % CI] | Realized where \|edge\| > 5¢ |
|---|---|---|---|---|---|
| F1, Kalshi, our tape, 2026 R1–16 | 15 | 1,013 | +0.00 [−0.10, +0.09] | **0.52 [0.23, 0.83]** | **+6.2¢ [+0.8, +12.2]** |
| F1, Kalshi, Probalytics' books, 2026 R5–16 | 11 | 1,084 | +0.00 [−0.11, +0.10] | **0.51 [0.13, 0.89]** | +5.8¢ [−0.6, +12.8] |
| F1, Polymarket, Probalytics' books, 2026 R3–16 | 13 | 963 | +0.01 [−0.13, +0.11] | **0.53 [0.08, 0.93]** | +7.4¢ [0.0, +14.4] |
| F1, Polymarket, our tape, 2026 R1–16 | 16 | 1,774 | −0.04 [−0.18, +0.07] | 0.38 [−0.04, 0.74] | +4.0¢ [−2.1, +9.9] |
| F1, Kalshi, our tape, 2025 | 22 | 844 | −0.14 [−0.31, +0.01] | 0.09 [−0.31, 0.54] | +4.2¢ [−1.3, +10.4] |
| F1, Polymarket, our tape, 2025 | 23 | 2,662 | **−0.16 [−0.28, −0.08]** | −0.10 [−0.33, 0.14] | −0.8¢ [−4.8, +3.3] |
| NASCAR, Kalshi, 2026 | 34 | 2,086 | **−0.10 [−0.15, −0.05]** | −0.06 [−0.35, 0.19] | −0.7¢ [−4.4, +2.8] |
| NASCAR, Kalshi, 2025 | 8 | 104 | −0.19 | too few | too few |
| MotoGP, Kalshi, 2026 | 11 | 46 | −0.20 | too few | too few |

*"Realized where |edge| > 5¢" is the average result of siding with the model on those markets, before costs. A market appears once per stage, so the rows are not independent; the intervals resample whole weekends.*

**What it says:**
- **F1 2026: about half the model's edge is real.** The slope is close to 0.5 on both venues and both tapes, with three of the four intervals above zero. Overall the model is no better than the market (skill ≈ 0), but where they disagree by more than 5¢ the model was right more often than wrong. That answers the fair-price question: our price is a second opinion worth about half its face value, so a trade needs a predicted edge of roughly twice its cost.
- **F1 2025 and NASCAR: no edge.** The slope is about zero, and the model loses to the market on accuracy. The worst case is the 2025 Polymarket constructor markets (skill −1.6 over 16 weekends).
- **The 2025 Polymarket row is our "ready at launch" case.** It asks what today's model would have done from F1's first Polymarket markets: it would have lost to them. Our tape starts with those markets, and the answer doesn't need a decay curve, because there was no edge to decay.
- **Calibrating doesn't change it.** The same runs with the model's win prices recalibrated on earlier races only (`calib_power = walk_forward`) move every number by a few hundredths at most (F1 Kalshi 2026 slope 0.48, F1 Polymarket 2025 skill −0.14, NASCAR 2026 skill −0.10). Calibrating per sport and per market kind, rather than one power on win prices, isn't built yet.

The paper's Table 3 makes the same split for Kalshi's skilled traders; here it is for the model, and it is 2026-specific.

### 3.2 One setting on Kalshi, trade by trade (20 Oct, 01:49Z)

We re-ran the published Kalshi backtests with every trade written out: F1 2025 and 2026 (one fixed settings set) and
each sport's baseline for NASCAR 2025–26 and MotoGP 2026 ([runbook as run](kalshi-response-2026-10-20-as-run-resim.html)). Each opening taker trade is one
forecast. We score our fair f against Kalshi's mid m at that moment, with outcome y. The edge is y − m for a YES buy
and m − y for a NO buy. All intervals are 95% and resample whole race weekends, because outcomes in one race are linked.

| Sport, season | Trades (weekends) | Edge vs mid, ¢/share | After costs | Skill score vs mid | No book at the fill |
|---|---|---|---|---|---|
| F1 2025 | 151 (20) | +5.4 [−2.9, +15.0] | +1.0 | −0.19 [−0.46, +0.03] | 5 % |
| F1 2026 | 223 (15) | **+12.0 [+5.5, +18.4]** | **+8.0** | +0.05 [−0.14, +0.18] | 9 % |
| NASCAR 2025 | 24 (8) | +12.0 [−5.2, +19.8] | +7.6 | −0.33 [−1.36, +0.89] | 4 % |
| NASCAR 2026 | 584 (31) | −0.4 [−4.7, +3.8] | **−5.8** | **−0.16 [−0.25, −0.08]** | 1 % |
| MotoGP 2026 | 30 (11) | +2.0 [−11.4, +17.4] | −0.1 | −0.26 [−1.00, +0.22] | **100 %** |

*Update mode shown; the hold, last and early modes agree. The skill score is the paper's 1 − Brier(ours) / Brier(mid),
against the plain mid: Kalshi's tape in this run has no touch sizes, so there is no size-weighted mid.*

- **It shows** that F1 2026's trades landed on the right side of Kalshi's mid, by 12¢ a share before costs and 8¢
  after, in all four taker modes (each with a false-discovery-corrected q of 0.013 or less).
- **It shows** that NASCAR's model loses to Kalshi's price (skill score −0.16, entirely below zero), and that MotoGP
  fails rule 3 before skill can be judged: none of its fills had a recorded book.
- **It doesn't show** skill (rule 1). The same F1 settings find no edge in 2025, and they are one of 20 settings tried
  on these seasons (6th of 20 by profit in 2026). We label the F1 result 2026-specific until a forward test agrees.
- **It doesn't show** price impact or markouts (rule 4). The trade file carries the mid at entry only. The close-point
  CLV from the same runs is +7.3 % for F1 2026 and −0.1 % for F1 2025.
- **It doesn't show** a full rule 1 test. That needs every setting tried, which is section 3.3.


### 3.3 Every setting tried: the luck test

**What this tests.** A strategy setting is our choice of thresholds, sizing and timing on top of the model. We have tried many over the past months, and the paper's rule is that a setting's result only counts after correcting for every setting tried. So we re-ran **every setting we ever saved**: 20 on Kalshi (F1 2025 and 2026, our tape) and 106 on Polymarket (F1, at the touch on Probalytics' books, on the markets that pass the test floor in 3.4). Each trade is scored against the venue's mid, and the false-discovery correction (Benjamini–Hochberg) covers every setting, season and taker mode together.

| Venue, season | Settings | Weekends | Edge vs mid, median (range) | After costs, median | CI above zero (vs mid) | Skill CI above zero | Pass after correction |
|---|---|---|---|---|---|---|---|
| Kalshi, 2026 R1–16 | 20 | 15 | +9.2¢ (+4.3 to +25.2) | +5.1¢ | 11 of 20 | 3 of 20 | 19 of 20 (q < 0.05) |
| Kalshi, 2025 | 20 | 20–22 | +4.7¢ (+2.4 to +7.3) | +0.8¢ | 4 of 20 | 0 of 20 | 4 of 20 |
| Polymarket, 2026 R3–16 | 106 | 13 | +8.8¢ (+4.3 to +13.5) | +5.7¢ | 2 of 106 | 0 of 106 | **0 of 749 cells** |
| Polymarket, 2025 R22–24 | 101 | 3 | +4.5¢ (−14.5 to +10.4) | +0.6¢ | 15 of 101 | 0 of 101 | **0** |

*Update mode shown; the correction runs over all four taker modes. On Kalshi our tape has no order sizes, so the mid is plain; on Polymarket it is size-weighted. The 2025 Polymarket window is only Las Vegas, Qatar and Abu Dhabi, the first races with Probalytics' books. Five 2025 Polymarket settings couldn't run read-only, and 53 of the 2026 settings ran on rounds 3 to 15 because round 16 had no stored pricing for them. Polymarket intervals use 500 resamples.*

**What it says:**
- **In 2026, every saved setting bought on the right side of the mid, on both venues,** and every Polymarket setting stayed positive after costs. That is a property of the model, not of one lucky setting: selection alone would not make all 126 positive.
- **None passes the full test.** On Polymarket no setting survives the correction across all 749 cells (the smallest q is 0.054), and none has a skill score interval above zero. On Kalshi 19 of 20 pass the correction in 2026, but those q-values treat trades as independent, so they're optimistic; the weekend-resampled intervals are the ones to trust, and they clear zero for 11 of 20.
- **Nothing holds across seasons.** No setting has an interval above zero in both 2025 and 2026 on either venue, and the ranking reverses: across settings, 2025's edge and 2026's are negatively correlated (Kalshi −0.51, Polymarket −0.33). Picking a setting by last season's result would have hurt.
- So rule 1 is **partly met**: the 2026 edge is real in the sense that it doesn't depend on the setting, but it is 2026-specific until a forward test agrees. A 2026-only effect, such as a pricing bias in this season's newer F1 markets, fits the data as well as skill does.

[Kalshi rule 1, as run](kalshi-response-2026-10-20-as-run-rule1.html) · [Polymarket rule 1, as run (part 2 of the overnight batch)](kalshi-response-2026-10-20-as-run-state-batch.html)

### 3.4 A pre-registered Polymarket test

**What this tests.** Section 3.3 scores settings we tuned in the past. To get a clean read, we dropped every tuned setting and wrote down two un-tuned candidates (the model's defaults, filling at the touch or walking the book), the windows, a market floor, the rule for picking one candidate, and the scorer, and committed all of it before anything ran (racinglines commit 730644c, 20 Oct 03:52, [#276](https://github.com/racinglines/racinglines/pull/276)).
- **Window:** Probalytics' Polymarket books begin 20 Nov 2025, so the test covers Las Vegas 2025 to 2026 round 16.
- **Market floor:** a Polymarket market counts only if its volume and its trade count both reach the Kalshi median for the same race and market kind. 717 of 2,866 Polymarket markets pass. NASCAR and MotoGP fail it, so the test is F1 only.
- **Split, forward in time:** pick on Las Vegas, Qatar, Abu Dhabi and 2026 rounds 3 to 6; score the pick once on rounds 7 to 16; then score the same frozen pick live from Austin (round 18).

| Window | Candidate | Trades (weekends) | After costs | Vs size-weighted mid | Skill vs mid |
|---|---|---|---|---|---|
| Pick window | **book-walk, update (the pick)** | 124 (7) | −3.2¢ [−10.0, +1.8] | −1.2¢ [−7.9, +4.1] | −0.21 [−0.39, −0.07] |
| Holdout R7–16 | **book-walk, update (the pick)** | 213 (9) | **+6.8¢ [−4.7, +18.0]** | +8.9¢ [−2.7, +20.0] | −0.01 [−0.32, +0.21] |
| Holdout R7–16 | book-walk, hold (not picked) | 93 (9) | +8.6¢ [+0.1, +17.0] | +10.6¢ [+2.8, +19.3] | +0.03 [−0.20, +0.25] |

**What it says:**
- **No candidate passes.** The pick made money in the holdout, but its interval includes zero and its skill score is zero. Every candidate lost in the pick window.
- **Every fill had a real book,** and walking the book cost under 1¢ more than the touch at these sizes. Rule 3 holds for this test.
- **The holdout isn't blind.** Earlier work had already run our model on those races, so the live holdout from Austin is the only truly blind test. It is frozen and scores as each race settles.

[As run](kalshi-response-2026-10-20-as-run-pm-step3.html)

### 3.5 Does the price come our way?

**What this tests.** The paper's Fig 9 follows the mid after a skilled trader's order: it moves their way and stays there, while other takers' trades leave little trace. We do the same for every backtest trade, against Probalytics' size-weighted mid: at the trade, at the next stage of the weekend (a median 2.5 to 3.5 hours later), and at the last stage before the race (a median 23 to 29 hours later).

| Trades | Mode | Trades | Paid over the mid | Mid move by the next stage | By the last stage |
|---|---|---|---|---|---|
| Kalshi, 20 settings, Probalytics' books, 2026 R5–16 | update | 4,844 | 3.6¢ | **+2.8¢** | **+4.5¢** |
| same | early | 3,268 | 3.6¢ | +2.9¢ | +4.7¢ |
| Kalshi, 20 settings, our tape, 2026 | update | 3,084 | 1.6¢ | +3.4¢ | +5.3¢ |
| Polymarket, pre-registered test | update | 709 | 1.8¢ | +1.6¢ | +1.4¢ |
| *Paper, skilled takers* | | | *1.5–1.8¢* | *1.7¢ by the next order* | *3.3¢ by close* |

**What it says:**
- **On Kalshi the price keeps moving our way until the race, by more than we paid.** That is the paper's skilled-trader shape, and it is rule 4. On Polymarket the move is about what we paid, so it breaks even.
- **These are backtest trades.** They didn't move the price; the price came round to the model's view on its own. Our own impact on thin books is unmeasured until we trade.
- No intervals yet, and the same trade appears under many settings, so the rows overlap. These numbers don't count towards rule 4 until both are fixed.

### 3.6 The crowd: is motorsport priced fairly?

Over 3.19 million resolved fills on the F1 markets we link (both venues, every month in our Probalytics pull), the price paid tracks the settled frequency at the ends: a Kalshi YES bought at 5¢ settled 6.7 % of the time, and at 95¢, 97.3 %. The middle (35 to 65¢) is noisy because a few big markets dominate it. A per-market weighting is still to do, so we draw no conclusion on the paper's Fig 6 yet.

## 4. First pass (19 Oct): their outputs, rebuilt from our stored paper bets

This was our first evening's work, before any simulation ran. It uses the paper bets already in our database, and it is kept here as it was published. Section 3 supersedes it wherever the two overlap.

### 4.1 Fig 1: where our strategies sit on the skill distribution

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

### 4.2 Table 3: where the edge comes from

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

### 4.3 App. A.1–A.2: a skill score is not profit

The same 815 bets, re-priced at mid + h. The paper's App. A.2 explains why the score can rise while profit falls.

| Paid over mid | 1¢ | 2¢ | 3¢ | 4¢ |
|---|---|---|---|---|
| Skill score | 0.0090 | 0.0168 | 0.0234 | 0.0287 |
| Profit per contract (¢, contract-weighted) | 6.85 | 5.86 | 4.86 | 3.87 |

![Skill score and profit as the fill gets worse](img/fig2-bss-vs-cost.png)

So rule 1 is checked at realistic fills, and always next to profit and CLV.

### 4.4 Fig 9: price impact, for the whole motorsport crowd

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

### 4.5 Before eq. 4: how far the last trade is from the mid

**Paper method:** sec 3.4 "Order Flow and the Last Traded Price". The regression itself comes with the simulation; this is the step
before it. All Kalshi motorsport trades in the last 3 days were matched to our last book at most 6 minutes earlier.

| Taker side | Trades | Avg spread | Trade − plain mid | Trade − weighted mid | At or through the touch |
|---|---|---|---|---|---|
| YES | 23,459 | 4.8¢ | +4.2¢ | +4.7¢ | 63.7 % |
| NO | 8,764 | 8.0¢ | −4.2¢ | −3.8¢ | 50.5 % |

A last-trade price is the mid plus about 4¢ in the direction of whoever traded last. That is why rule 3 bans
"last trade plus a fixed cost" fills, and why the simulation estimates the mid from the book wherever we have one.

### 4.6 What we can't rebuild yet

| Paper output | Why not | Path |
|---|---|---|
| Table 1, Fig 2–4, Table 4 (skilled accounts, their traits, persistence) | Kalshi's public tape has no account ids | Polymarket trades carry wallets (our tape: 1,282 wallets on 7,139 F1 trades) |
| Table 5 for F1 (is F1 among the 8.2 % of Sports markets with no skilled taker?) | The paper gives no per-sport split | Per-sport skilled-taker coverage for F1 on Polymarket wallets |
| Table 6 (eq. 4) | Needs a last-price series with net order flow | With the simulation |
| Fig 6 by price bin | One resolved event per sport since our tape restarted | Probalytics fills, May to Sep 2026 |

## 5. Caveats

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
10. **Scope.** We have no linked cycling markets on either venue, so cycling, which we model, is not tested here.
11. **Not blind.** Every backtest window here, holdouts included, was seen in some form while the model was built. Only the forward tests (the per-event locks and the live Polymarket holdout from Austin) are blind.

## How we built this: ops timeline

From the paper's release to this page. The big steps are on their own rows; all the editorial passes share one row.
**High-touch** steps needed the owner (a decision, an upload, a publish); **streamlined** steps ran in our project
threads with no input.

| Time (UTC) | Step | Touch |
|---|---|---|
| 19 Oct, about 19:20 | Kalshi publishes the paper | — |
| 23:43 | The paper is handed to our research thread | High-touch |
| 23:46 to 23:56 | **Key questions set:** the paper's findings, a simulation in its framework, our gaps, the mid from last trades, our place on Fig 1, and then the top-line one: what must a strategy prove before it trades real money? | High-touch |
| 23:50 to 23:59 | **First numbers:** our stored Kalshi paper bets on their skill scale with a 1,000-draw luck test; crowd price impact rebuilt from our tape (9 min) | Streamlined |
| 23:57 to 00:01 | **Five golden rules adopted** and written into our development guide and internal book (4 min) | Streamlined |
| 20 Oct 00:04 | **First draft** (internal, 7 pages), 21 minutes after the hand-off | Streamlined |
| 00:12 to 00:54 | **Editorial work, round 1** (v1): public edition in the site's layout, title, PR and source links, preview card, landing-page card | High-touch |
| by 00:54 | **Live draft published** here | High-touch |
| 00:54 to 01:36 | Simulation code: every backtest trade written out with the book's bid, ask and sizes (racinglines PR #275); one CI fix, so the backtest's own results stay byte-identical | Streamlined |
| 01:16 to 01:18 | **No-cheating rule set:** steps 2 and 3 get published whatever they show; a new blind test is always allowed | High-touch |
| 01:49 to 01:59 | **Simulation run** on our server, read-only: five sport-seasons in two runs, about 4.5 minutes in all. Every F1 total matches the published backtest to the cent. [Runbook as run](kalshi-response-2026-10-20-as-run-resim.html) | High-touch |
| 02:07 to 02:12 | **Trades scored** in the paper's framework (step 2 below) | Streamlined |
| 02:15 | **Polymarket test split chosen**, before any of it runs (step 3) | High-touch |

*Internal book pages behind this report: [3.4 Trading (golden rules)](https://racinglines.bet/docs/internal/system/trading.html), [3.5 Evaluation and sweeps](https://racinglines.bet/docs/internal/system/evaluation.html), [4.5 DITL reports](https://racinglines.bet/docs/internal/ops/ditl.html), [Appendix A: every report and its sources](https://racinglines.bet/docs/internal/appendix/reports.html).*
| 02:15 to 02:40 | **Editorial work, round 2** (v2): simulation results, as-run runbook page, PR tooltips, access note | High-touch |
| 02:24 | **v2 published**, with the simulation results | High-touch |
| 02:36 to 02:56 | **Rule 1 on Kalshi:** all 20 saved F1 settings re-run trade by trade, 40 runs, read-only. [As run](kalshi-response-2026-10-20-as-run-rule1.html) | Streamlined |
| 03:52 | **Polymarket test locked:** settings, windows, floor, pick rule and scorer committed before any of it ran (racinglines PR [#276](https://github.com/racinglines/racinglines/pull/276)) | Streamlined |
| 03:55 to 04:01 | **Polymarket test run**, read-only, then scored once. [As run](kalshi-response-2026-10-20-as-run-pm-step3.html) | Streamlined |
| 05:02 to 05:10 | **"State of racinglines" set:** run every historical test our data allows, and say what decides "a fair price" vs "an edge worth trading" | High-touch |
| 05:23 to 11:03 | **Overnight batch** on our server, read-only, in two parts: the model on every market it prices, Kalshi on full order books, markouts, every resolved fill, and rule 1 over all 106 saved Polymarket settings (212 season runs, 207 completed). [As run](kalshi-response-2026-10-20-as-run-state-batch.html) | Streamlined |
| 06:45 to 12:22 | **Overnight results scored:** the model on every market, all 126 saved settings with the luck test, markouts, crowd fills | Streamlined |
| 12:13 to 12:35 | **Editorial work, round 3** (v3): the whole page rewritten as a state of racinglines, with context before each test, and every as-run runbook linked | High-touch |

## What's next

- **Forward, from Austin (23 to 25 Oct).** Each race's locked strategy scores as bet, and the frozen Polymarket pick scores live. This is the only blind test, and rule 2's record starts here.
- **The trade rule from 3.1.** Trade only where half the predicted edge clears the spread and fees. It replaces tuned thresholds with one number measured on every market, but it was found on 2026 data, so it is pre-registered before it counts.
- **Finish rule 4.** Intervals and de-duplication for the markouts, and the crowd's price impact by order size (paper sec 5.4), which needs Probalytics' day books.
- **Calibration per sport and kind.** One win-price power doesn't fix 2025 or NASCAR; a fit per market kind and field might.
- **Real fills (rule 5).** Only live trading answers it.

Also on the list: F1 skilled-taker coverage on Polymarket, where trades carry wallets, and the market-life split of the paper's Fig 7.
