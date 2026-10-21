# RE: Kalshi's "Who Calibrates Markets"

**What must a taker strategy prove before it trades real money, and how close are we?**

**Short summary.** On 19 Oct Kalshi's research team published ["Who Calibrates Markets? Evidence from Trader Micro-Data"](https://kalshi.com/research/publications/who-calibrates-markets) (Kagan and Baiocchi). They find that about 0.15 % of Kalshi accounts forecast better than the price they trade against, after correcting for luck, and that the price moves their way after they trade. We build racinglines ([web demo](https://racinglines.bet/term), [repo](https://github.com/racinglines/racinglines-public)), a model that prices motorsport markets (Formula 1, NASCAR, MotoGP and more) and strategies that trade on it. We turned the paper into five rules a taker strategy must pass before it trades real money, and tested ourselves against them.

**The answer: not yet.**

- **Our first edge was mostly a model flaw.** After qualifying, our F1 model gave its own top pick about 37 % to win; that driver won about two races in three. In 2026 favourites lost more often than priced, so a model that priced them too low looked like it had an edge ([section 3](#where-the-2026-edge-came-from)).
- **Calibration fixes the favourite, not the edge.** Calibrating win prices on earlier races lifts the favourite after qualifying to 56 to 67 % (it won 63 to 71 %), in both seasons. The model still loses to the market on race winners in 2025 ([section 4](#before-and-after-the-model)).
- **Every test, re-run with calibration on, splits the same way.** 2026 looks better (Kalshi trades +13.7¢ a share against the mid), 2025 worse, and no setting holds in both seasons ([section 5](#before-and-after-every-test)). So rule 1, skill, is **not met**.
- **What counts now is forward:** strategies locked before each race, from the US Grand Prix in Austin (23 to 25 Oct).

**Edit note.** v3 (20 Oct) reported an F1 2026 edge; v4 and v4.1 (21 Oct) found the flaw behind it, fixed it and re-ran every test. Each version is linked in [Appendix A](#appendix-a-revisions). We said we would publish these results whatever they showed, and we have.

*How to read this. All times are UTC. "The paper" is Kagan and Baiocchi (2026); we point to its sections rather than restating its methods, and we don't republish it. Every number here is a backtest: v3's ran read-only on a mirror of our production database (19 Oct 23:45 to 20 Oct 11:03), v4's and v4.1's on a copy of it (20 Oct 23:42 to 21 Oct 08:47). Nothing touched production, and nothing here is betting advice.*

*Access on request: PR links (#NNN) go to our private source repository, so they open only for its collaborators; our public code is [racinglines/racinglines-public](https://github.com/racinglines/racinglines-public) (MIT) and the `racinglines` package on PyPI. Internal-book links are for admins only. The PRs, process notes and the query behind every number can be shared on request: hello@racinglines.bet.*

## Why this paper matters to us

A prediction market price is a crowd's forecast. The paper asks who makes that forecast accurate, and finds a small group of traders who buy and sell against the price and are right more often than it is. Racinglines is an attempt to be one of those traders, in motorsport, with a bottoms-up model and a lot of historical race data. Our model simulates each race from the drivers' recent results, practice and qualifying, and turns that into a fair price for every market it can: who wins, who makes the podium, head-to-heads and more. A strategy then decides when a gap between our fair price and the market's is worth trading.

The paper gives us a way to check ourselves that most backtests skip. A backtest can look profitable because one of many settings got lucky, because it fills at prices nobody could have traded at, or because the season it was tuned on was unusual. The paper's tests (skill against the mid, a luck test over every account, what happens to the price after a trade) answer each of those. They are also the right tests for anyone, a customer included, who wants to trade on a model.

## The answer: five golden rules

Before any taker strategy is deployed with real money on Kalshi or any prediction market, by us or by a customer, it must prove all five. A customer proves them on their own fees, latency and size. We adopted these as our own rules on 19 Oct, and every taker strategy we build is now checked against them. They open our development guide ([#274](https://github.com/racinglines/racinglines/pull/274)) and the trading chapter of our internal book ([3.4 Trading](https://racinglines.bet/docs/internal/system/trading.html)).

| # | Rule | Status, 21 Oct | Evidence |
|---|---|---|---|
| 1 | **Skill, not luck or selection** | **Not met:** v3's 2026 edge was mostly a bias against favourites; the fixed model doesn't beat the market in both seasons, and with calibration on no setting holds in both seasons or passes the luck test | [3](#where-the-2026-edge-came-from), [4](#before-and-after-the-model), [5.2](#calibrated-every-setting-tried) |
| 2 | **Out of sample** | In force; forward record starts at Austin | [A3.4 in v4](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md) |
| 3 | **The real book** | **Met for F1 backtests** (full order books, every fill had a book); not met for NASCAR/MotoGP paper bets | [5.3](#calibrated-the-polymarket-candidates), [A3.3 in v4](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md) |
| 4 | **The price comes our way** | **Partly met,** re-measured with calibration on: yes on Kalshi in backtests (by more than in v3), about break-even on Polymarket; no intervals yet | [5.4](#calibrated-does-the-price-come-our-way), [A3.5 in v4](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md) |
| 5 | **Paper reconciles with real** | Not started (no real money yet) | — |

**1. Skill, not luck or selection.** A positive Brier skill score against the size-weighted mid at realistic fills, surviving a luck test with a false-discovery correction across *every* setting tried.

- *Paper:* sec 3.2–3.3, Table 1, App. A.1
- *Where we are:* in F1 2026 every saved setting beats the mid on both venues, but none survives the correction over all settings, and none holds in 2025 as well. Most of the 2026 edge was the model pricing favourites too low (section 3), and the fixed model doesn't beat the market in both seasons (section 4). With calibration on, every setting's 2026 edge grows and its 2025 loss deepens (section 5).

**2. Out of sample.** It holds forward on pre-registered locks, not on the season it was tuned on.

- *Paper:* sec 4.4, Table 4
- *Where we are:* per-event strategy locks, committed before each race and scored as bet (since 15 Oct, [#213](https://github.com/racinglines/racinglines/pull/213)), and a pre-registered Polymarket test ([#276](https://github.com/racinglines/racinglines/pull/276)) whose blind half starts at Austin.

**3. The real book.** The edge clears the actual spread, depth and fees: fills at the touch or walking the recorded book, never a last-trade price plus a fixed cost.

- *Paper:* sec 2.1, App. A.4
- *Where we are:* our F1 backtests now fill against Probalytics' full order books on both venues ([#202](https://github.com/racinglines/racinglines/pull/202), [#203](https://github.com/racinglines/racinglines/pull/203)–[#205](https://github.com/racinglines/racinglines/pull/205), [#219](https://github.com/racinglines/racinglines/pull/219)), with every trade written out ([#275](https://github.com/racinglines/racinglines/pull/275)). Our stored NASCAR and MotoGP paper bets still used a one-price fill.

**4. The price comes our way.** After entry the mid moves toward our side and doesn't reverse.

- *Paper:* sec 3.4, 5.4, Fig 9
- *Where we are:* markouts of every backtest trade, at the next stage of the race weekend and at the last one before the race, plus closing-line value ([#223](https://github.com/racinglines/racinglines/pull/223)). Re-measured with calibration on in section 5.4.

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

## 3. Where the 2026 edge came from {#where-the-2026-edge-came-from}

On 20 Oct v3 reported that every strategy setting we had saved bought on the right side of the market in F1 2026, and that about half of the model's predicted edge showed up in the results ([as published in v4](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md)). That evening a live run for Austin priced the favourite far below the market, so we went back to v3's own runs and split their edge by direction: bets against the favourite, and everything else. Then we scored the model's top pick to win in 18 sweeps our core team ran that night, in both seasons. "Section 3.1" below means [A3.1 in v4](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md).

**What this tests.** Section 3.1 counts every market where the model and the price disagree by more than 5¢. Here we split those by direction. "Against the favourite" means the market prices the outcome above 35¢ and the model is lower. Same runs as 3.1; calibration off unless named.

| Run | Rows against the favourite: avg result | Every other big disagreement | Share of the total edge against the favourite | Markets above 50¢: market / model / how often they won | Race-win favourite: market / model / won |
|---|---|---|---|---|---|
| F1 Kalshi, our tape, 2026 | 248: **+11.9¢** | 410: +2.7¢ | **73 %** | 0.65 / **0.37** / 0.47 | 0.42 / **0.17** / 0.30 |
| F1 Polymarket, Probalytics' books, 2026 | 186: **+13.7¢** | 364: +4.3¢ | **62 %** | 0.63 / **0.37** / 0.40 | 0.40 / **0.17** / 0.31 |
| F1 Polymarket, our tape, 2026 | 211: +11.2¢ | 478: +0.8¢ | 86 % | 0.64 / 0.39 / 0.47 | 0.42 / 0.17 / 0.34 |
| F1 Kalshi, calibrated (walk-forward), 2026 | 233: +13.3¢ | 433: +3.2¢ | 69 % | 0.65 / 0.40 / 0.47 | 0.42 / 0.23 / 0.30 |
| F1 Kalshi, our tape, 2025 | 236: +3.8¢ | 274: +4.5¢ | 42 % | 0.72 / **0.45** / 0.67 | 0.39 / **0.19** / 0.45 |
| F1 Polymarket, our tape, 2025 | 241: +2.9¢ | 507: −2.6¢ | — (total ≈ 0) | 0.67 / **0.50** / 0.63 | 0.41 / **0.20** / 0.41 |

*Rows are market-stages, so one race counts several times; the race-win favourite rows cover 13 to 23 races per run.*

**What it says:**

- **The model prices favourites far too low, in both seasons.** Where the market says 65 to 72 %, the model says 37 to 50 %. For the race-win favourite it says about 17 to 20 % where the market says about 40 %. Walk-forward calibration of win prices moves that only to 23 %.
- **2026 rewarded that bias; 2025 didn't.** 2026 favourites won less often than priced (race-win favourite 30 to 34 % against 40 to 42 %), so betting against them made most of the 2026 edge. 2025 favourites won about as priced, so the same model had nothing.
- **Buying longshots added little** (+2 to +3.5¢ on markets under 15¢). The edge isn't the classic favourite-longshot story; it is the model's flat field meeting a season of beaten favourites.
- **Inferred, not shown:** whether 2026's favourites were truly overpriced or just unlucky over 13 to 16 races. A model that priced favourites closer to the market would have given up most of the 2026 edge and lost less in 2025.

**Update, 20 Oct about 23:45 UTC: 18 new model runs.** Our core team re-ran the season sweeps on Polymarket's tapes with the recent-form half-life at 30, 60, 120 and 240 days, calibration off and on, and a grid variant: 9 settings × 2 seasons, every completed race. For each race and stage we took the model's top pick to win and asked how often it won.

| Model's top pick to win | Calibration | 2025 (24 races): model / won | 2026 (17 races): model / won |
|---|---|---|---|
| After qualifying | off (5 settings) | **0.38–0.39** / 0.63 | **0.36** / 0.68–0.71 |
| After qualifying | on, walk-forward (4 settings) | 0.58–0.60 / 0.63 | 0.66–0.67 / 0.71 |
| Before qualifying | off | 0.19–0.22 / 0.27–0.49 | 0.18–0.19 / 0.15–0.34 |
| Before qualifying | on | 0.29–0.32 / 0.28–0.50 | 0.27–0.29 / 0.16–0.35 |

*Ranges are across the settings in each row; "won" moves with the half-life because it changes who the top pick is. Rows before qualifying count several stages per race.*

- **After qualifying, calibration off makes the model far too flat, in both seasons, and calibration fixes it** (measured). Uncalibrated, the pole-day favourite gets about 37 %; it wins about two times in three. Almost every run in this report had calibration off.
- **Before qualifying, where most of the 2026 edge came from, even the calibrated model's favourite (27 to 32 %) stays below the market's (about 40 %).** So the bet against favourites remains, and how it pays depends on the season: in 2026 the model's own top pick before qualifying won only 15 to 35 % of the time (measured).
- **No new setting holds in both seasons.** The half-life that scores best on 2025 scores worse on 2026, and none has positive closing-line value on the held-out 2026 rounds 13 to 17 (core team's sweep note, measured). A fix to how the model spreads the front of the field before qualifying is in progress and not in these numbers.
- The correction stands: rule 1 is not met.

Script: `reconcile_favs.py` in our state batch folder ([as run](kalshi-response-2026-10-20-as-run-state-batch.html)).


## 4. Before and after calibration: the model {#before-and-after-the-model}

Sections 4 and 5 compare two versions of the same model:

- **Before** is v3 (20 Oct), with calibration off, as the model ran then.
- **After** is v4.1 (21 Oct), with calibration on (`walk_forward`: the win prices sharpened by a power fitted only on earlier races), as our F1 model now runs.

Section 4 scores the model's own predictions. Section 5 re-runs every v3 test.

**The favourite.** The model's top pick to win, its probability / how often it won, on Polymarket's tapes, every completed race:

| | 2025 after qualifying | 2026 after qualifying | 2025 before qualifying | 2026 before qualifying |
|---|---|---|---|---|
| Before | 0.38 / won 0.63 | 0.36 / won 0.68 | 0.22 / won 0.27 | 0.18 / won 0.15 |
| After | **0.56** / won 0.63 | **0.67** / won 0.71 | 0.32 / won 0.27 | 0.29 / won 0.21 |

*24 races in 2025, 17 in 2026; "before qualifying" counts every earlier stage, so each race counts several times.*

**Against the market on race winners.** Brier score, model minus market (below zero means the model was more accurate), and closing-line value (CLV) for the "update" strategy:

| | 2025 before / after qualifying | 2026 before / after qualifying | CLV 2025 | CLV 2026 | CLV 2026 rounds 13–17 |
|---|---|---|---|---|---|
| Before | +0.0068 / +0.0079 | −0.0006 / +0.0025 | +0.99 pp | +0.32 pp | −0.99 pp |
| After | +0.0077 / +0.0088 | −0.0018 / **−0.0065** | +1.30 pp | +0.58 pp | −0.19 pp |

**Every market it prices, every stage**, traded or not:

| Model, venue, tape, season | Skill vs market: after (before) | Edge slope: after (before) | Realized where \|edge\| > 5¢: after (before) |
|---|---|---|---|
| F1, Kalshi, our tape, 2026 R1–16 | −0.01 [−0.10, +0.09] (+0.00) | **0.48 [0.20, 0.85]** (0.52) | **+6.7¢ [+1.0, +13.4]** (+6.2¢) |
| F1, Kalshi, Probalytics' books, 2026 R5–16 | +0.02 [−0.07, +0.11] (+0.00) | **0.59 [0.18, 1.00]** (0.51) | +7.1¢ [−0.1, +14.0] (+5.8¢) |
| F1, Polymarket, Probalytics' books, 2026 R3–16 | +0.03 [−0.12, +0.12] (+0.01) | **0.59 [0.05, 1.01]** (0.53) | +8.4¢ [−0.8, +16.2] (+7.4¢) |
| F1, Polymarket, our tape, 2026 R1–16 | −0.04 [−0.17, +0.06] (−0.04) | 0.36 [−0.02, 0.76] (0.38) | +5.4¢ [−1.3, +12.4] (+4.0¢) |
| F1, Kalshi, our tape, 2025 | −0.10 [−0.24, +0.02] (−0.14) | 0.09 [−0.34, 0.58] (0.09) | +3.2¢ [−2.9, +10.1] (+4.2¢) |
| F1, Polymarket, our tape, 2025 | **−0.14 [−0.24, −0.05]** (−0.16) | −0.11 [−0.38, 0.22] (−0.10) | −1.0¢ [−5.9, +4.6] (−0.8¢) |
| NASCAR, Kalshi, 2026 | **−0.10 [−0.15, −0.05]** (−0.10) | −0.07 [−0.34, 0.18] (−0.06) | −1.3¢ [−5.0, +2.5] (−0.7¢) |
| NASCAR, Kalshi, 2025 | −0.19 (−0.19) | too few | too few |

| | Race-win favourite: model / market / won | Markets above 50¢: model / market | Share of the 2026 edge against the favourite |
|---|---|---|---|
| Before | 0.17–0.20 / 0.39–0.42 / 0.30–0.45 | 2026 0.37–0.39, 2025 0.45–0.50 / 0.63–0.72 | 62–86 % |
| After | **0.23–0.28** / 0.39–0.42 / 0.30–0.45 | 2026 0.40–0.42, 2025 0.52–0.54 / 0.63–0.72 | **52–74 %** |

**What it says:** calibration fixes the favourite after qualifying in both seasons, but it doesn't make an edge. Before qualifying, which is most stages, the favourite is still well below the market, and the model still loses to the market in 2025.

*v4 also tried adding noise to the starting grid; it made no difference, so it is left out here ([v4 as published](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md)).*

## 5. Before and after calibration: every test {#before-and-after-every-test}

Every run behind v3's tables and plots, again with calibration on: the same settings, windows, tapes and scorers. That is 262 runs (21 Oct 02:59 to 08:47), none failed ([as run](kalshi-response-2026-10-21-as-run-calib-rerun.html)). MotoGP (no calibration yet) and the crowd tables (they don't use our model) aren't rerun. Each table shows after, with before in brackets.

### 5.1 One setting on Kalshi, trade by trade {#calibrated-one-setting-on-kalshi}

One saved setting, every opening taker trade, against Kalshi's plain mid ([v3's method](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md)).

| Sport, season | Trades (weekends): after (before) | Edge vs mid, ¢/share: after (before) | After costs: after (before) | Skill score vs mid: after (before) |
|---|---|---|---|---|
| F1 2025 | 131 (20) (151) | +1.0 [−8.7, +11.4] (+5.4) | −3.6 (+1.0) | **−0.28 [−0.59, −0.02]** (−0.19) |
| F1 2026 | 234 (15) (223) | **+13.7 [+7.4, +20.1]** (+12.0) | **+9.7** (+8.0) | +0.08 [−0.06, +0.19] (+0.05) |
| NASCAR 2025 | 23 (8) (24) | +11.9 [−6.9, +20.3] (+12.0) | +7.5 (+7.6) | −0.36 [−1.42, +0.88] (−0.33) |
| NASCAR 2026 | 575 (31) (584) | +0.1 [−4.1, +4.6] (−0.4) | **−5.3** (−5.8) | **−0.16 [−0.26, −0.07]** (−0.16) |

### 5.2 Every setting tried: the luck test {#calibrated-every-setting-tried}

Every setting we ever saved, 20 on Kalshi and 106 on Polymarket, with a false-discovery correction over every setting, season and taker mode ([v3's method](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md)).

| Venue, season | Settings | Edge vs mid, median (range): after (before) | After costs, median: after (before) | CI above zero: after (before) | Skill CI above zero: after (before) | Pass after correction: after (before) |
|---|---|---|---|---|---|---|
| Kalshi, 2026 R1–16 | 20 | +10.9¢ (+4.9 to +24.2) (+9.2¢) | +7.0¢ (+5.1¢) | **18 of 20** (11) | 3 of 20 (3) | 20 of 20 (19) |
| Kalshi, 2025 | 20 | +3.3¢ (−0.7 to +6.3) (+4.7¢) | **−0.9¢** (+0.8¢) | 0 of 20 (4) | 0 of 20 (0) | 4 of 20 (4) |
| Polymarket, 2026 R3–16 | 106 | +12.2¢ (+5.5 to +20.0) (+8.8¢) | +9.1¢ (+5.7¢) | **40 of 106** (2) | **0 of 106** (0) | 301 of 773 cells (0 of 749) |
| Polymarket, 2025 R22–24 | 106 (101) | **−15.2¢ (−35.6 to +8.8)** (+4.5¢) | **−19.9¢** (+0.6¢) | 0 of 106 (15) | 0 of 106 (0) | 0 (0) |

*Update mode except the last column. The correction's q-values treat every trade as independent; the skill intervals resample whole weekends and are the ones to trust.*

**What it says:** no setting has an interval above zero in both seasons on either venue, and none has a skill interval above zero on Polymarket. Rule 1 is not met.

### 5.3 The Polymarket candidates {#calibrated-the-polymarket-candidates}

The two un-tuned candidates and pick rule of our pre-registered test ([v3](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md)). The pre-registered result is the before row; the after rows are a new entry, not registered before they ran.

| Window | Candidate | Trades (weekends) | After costs | Vs size-weighted mid | Skill vs mid |
|---|---|---|---|---|---|
| Pick window, after | **book-walk, update (the pick)** | 103 (7) | −4.2¢ [−14.2, +5.0] | −1.6¢ [−11.6, +7.9] | −0.20 [−0.48, +0.02] |
| Holdout R7–16, after | **book-walk, update (the pick)** | 198 (9) | **+8.8¢ [−1.4, +18.1]** | +10.8¢ [0.0, +19.6] | +0.05 [−0.21, +0.21] |
| Holdout R7–16, before | book-walk, update (the pick) | 213 (9) | +6.8¢ [−4.7, +18.0] | +8.9¢ [−2.7, +20.0] | −0.01 [−0.32, +0.21] |

**What it says:** the same candidate is picked, its holdout interval still includes zero, and its skill is about zero. No candidate passes. Every fill had a real book.

### 5.4 Does the price come our way? {#calibrated-does-the-price-come-our-way}

The mid after each backtest trade, against Probalytics' size-weighted mid ([v3's method](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md)).

| Trades | Mode | Trades: after (before) | Paid over the mid: after (before) | Mid move by the next stage: after (before) | By the last stage: after (before) |
|---|---|---|---|---|---|
| Kalshi, 20 settings, Probalytics' books, 2026 R5–16 | update | 4,799 (4,844) | 3.6¢ (3.6¢) | **+3.4¢** (+2.8¢) | **+6.7¢** (+4.5¢) |
| Kalshi, 20 settings, our tape, 2026 | update | 3,282 (3,084) | 1.5¢ (1.6¢) | **+4.8¢** (+3.4¢) | **+8.1¢** (+5.3¢) |
| Polymarket, the candidates (5.3) | update | 641 (709) | 1.9¢ (1.8¢) | +1.7¢ (+1.6¢) | +1.9¢ (+1.4¢) |
| *Paper, skilled takers* | | | *1.5–1.8¢* | *1.7¢ by the next order* | *3.3¢ by close* |

**What it says:** on Kalshi the price comes our way by about twice what we paid; on Polymarket it about breaks even. These are backtest trades with no intervals yet, so rule 4 is partly met and doesn't count yet.

### 5.5 The paper's outputs: Fig 1, Table 3, App. A.1–A.2 {#calibrated-the-papers-outputs}

**Fig 1:** the skill score of each setting's fills against the mid, with the luck test.

| Venue, season | Settings | Skill score of the fill, median (range): after (before) | Above the paper's median skilled trader (0.0189): after (before) | Luck test p < 0.05: after (before) | After correction |
|---|---|---|---|---|---|
| Polymarket F1 2026 | 106 | **+0.028 (+0.015 to +0.045)** (+0.016) | **95 of 106** (19) | 86 of 106 (3) | **0** (0) |
| Polymarket F1 2025 | 106 (101) | −0.081 (−0.124 to −0.030) (−0.021) | 0 (0) | 0 (0) | 0 (0) |
| Kalshi F1 2026, the 5.1 setting | 1 | +0.037 (+0.033) | yes (yes) | no, p = 0.07 (p = 0.06) | — |
| Kalshi F1 2025, the 5.1 setting | 1 | +0.016 (+0.011) | no (no) | no, p = 0.10 (p = 0.14) | — |

![Every saved setting on the paper's skill scale, before (grey) and after (blue) calibration](img/fig1-bss-placement-v4.png)

**Table 3**, the Murphy split on the same trades:

| | F1 Kalshi 2026: model / mid, after | before | F1 Polymarket holdout: model / size-weighted mid, after | before |
|---|---|---|---|---|
| Reliability (lower is better) | 0.0109 / 0.0200 | 0.0067 / 0.0189 | 0.0170 / 0.0158 | 0.0088 / 0.0135 |
| Resolution (higher is better) | **0.0289 / 0.0196** | 0.0218 / 0.0208 | **0.0471 / 0.0358** | 0.0333 / 0.0412 |

**App. A.1–A.2**, the Kalshi F1 2026 trades of 5.1, each re-priced at mid + h:

| Paid over mid | 1¢ | 2¢ | 3¢ | 4¢ |
|---|---|---|---|---|
| Skill score of the fill: after | 0.0114 | 0.0219 | 0.0316 | 0.0404 |
| before | 0.0104 | 0.0200 | 0.0286 | 0.0363 |
| Profit per contract (¢): after | 20.56 | 19.56 | 18.56 | 17.56 |
| before | 16.87 | 15.87 | 14.87 | 13.87 |

*Contract-weighted, before fees.*

**What it says:** in 2026 almost every Polymarket setting now sits above the paper's median skilled trader, but none passes the luck test once every setting is counted, and every 2025 setting is below zero.

### 5.6 Where the rules stand {#calibrated-what-it-adds-up-to}

- **Rule 1: not met.** Calibration lifts every 2026 test and lowers every 2025 one; nothing holds in both seasons.
- **Rule 3: holds for F1 backtests.** Every Polymarket fill had a real book; 7 to 8 % of Kalshi F1 trades had no book at the fill and are counted, not priced off a last trade.
- **Rule 4: partly met.** Yes on Kalshi, break-even on Polymarket, no intervals yet.

## 6. What we take from this

Kalshi's paper says a market is made accurate by a small group of traders who are right against the price, can be told apart from the lucky by a strict test, and leave a mark: the price moves their way and stays. We held racinglines to those tests, and the first thing they caught was us.

**The tests worked on us.** v3 passed most of what we could measure in 2026: positive scores against the mid, every saved setting on the right side, the price moving our way afterwards. The check the paper insists on, whether a result survives across samples, is what failed: the same model lost in 2025. Splitting the edge by direction showed why. Most of it was one bet, against favourites, made by a model that priced them far too low.

**Calibration is the fix, and it isn't an edge.** Fitted on earlier races only, it brings the favourite's price after qualifying close to how often favourites won, in both seasons. It doesn't make the model beat the market in both seasons, and before qualifying the model still prices favourites below the market. A better model is necessary; it isn't the same thing as skill.

**Motorsport still looks like room for a model.** Taking a price costs about twice the paper's average, volume arrives late on Kalshi (A3.6 in [v4](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md)), and in 2026 the market's favourites were overpriced. Whether that last part is a pattern or one season is exactly what we can't tell from history.

So rule 1 is not met, rule 3 holds for F1 backtests, rule 4 is partly met (re-measured with calibration on: yes on Kalshi in backtests, break-even on Polymarket), and rules 2 and 5 are where we start.

### What's next

- **Forward, from Austin (23 to 25 Oct).** Each race's locked strategy scores as bet, and the frozen Polymarket pick scores live. This is the only blind test, and rule 2's record starts here.
- **Calibration is on.** It fixes the favourite after qualifying in both seasons, so we made it the default for our hosted F1 model on 21 Oct, logged before Austin, and the Austin lock uses it.
- **Why calibration widens the season split.** It helps every 2026 test and hurts every 2025 one (section 5); we haven't checked why.
- **The favourite before qualifying.** Even calibrated, the model is below the market there. The next model work is how the model spreads the front of the field before qualifying, judged on both seasons together.
- **Intervals and de-duplication for the markouts,** so rule 4 can count. The luck test and markouts are re-measured on the calibrated model (section 5).
- **Real fills (rule 5).** Only live trading answers it.
- **Skilled takers in F1.** Polymarket trades carry wallets, so the paper's account-level tables (Table 1, Table 5 for F1) can be built there.

We said we would publish what the tests showed, whatever it was. This is what they showed: our first claim of an edge was mostly a model flaw, the flaw is partly fixed, and the edge isn't proven.

## 7. Caveats

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
12. **Calibration was chosen after looking.** Its power is fitted on earlier races only, but we turned it on after seeing both seasons, so the after side of sections 4 and 5 is a re-measurement, not a test. Calibrated runs on our database copy also differ very slightly from the same settings on production (the 2026 favourite after qualifying: 0.668 against 0.670); the cause is not yet checked.


## Appendix A: revisions

- **v1, 20 Oct 00:54:** [first draft](https://github.com/lalligagger/lalligagger.github.io/blob/ffbb1e9c8e2512e76fefa816090172304f6cfb1a/_sources/kalshi-response-2026-10-20.md), from our stored paper bets.
- **v2, 20 Oct 02:24:** the first simulation, trade by trade on Kalshi.
- **v3, 20 Oct 12:30:** rewritten as a state of racinglines, with the overnight runs. **v3b** (same day): the paper's outputs rebuilt and a conclusion ([as published](https://github.com/lalligagger/lalligagger.github.io/blob/d215d10/_sources/kalshi-response-2026-10-20.md)).
- **v3c, 20 Oct evening (prepared, folded into v4):** the correction below.

> **Correction, 20 Oct (evening, v3c).** Most of the F1 2026 "edge" in this report is one bet: against the favourite. Our model prices favourites far too low, in both seasons (markets above 50¢ in 2026: the market says about 65 %, our model about 37 %). In 2026 favourites won less often than their prices said, so fading them paid (they won 40 to 47 % of the time); in 2025 they won about as priced, so it didn't. That explains the 2025/2026 split, and it means the 2026 result is mostly a model bias meeting one season, not skill. Every run here had calibration off unless it says otherwise. We've added [3.7](#where-the-2026-edge-came-from) (with 18 new model runs, added the same night) and corrected 3.1, 4.2 and the conclusion; the earlier wording is in [v3b](https://github.com/lalligagger/lalligagger.github.io/blob/d215d10/_sources/kalshi-response-2026-10-20.md).

- **v4, 21 Oct:** rebuilt on the fixed model; v3's results moved to an Appendix A ([as published](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md)).
- **v4.1, 21 Oct:** every v3 test re-run with calibration on (section 5); rule 4 re-measured; v3's results appendix removed in favour of the v4 link above (so revisions are now Appendix A and the build log Appendix B); appendices collapsed by default; bullet lists that had run into the paragraph before them now show as lists.

## Appendix B: how we built this


From the paper's release to this page. The big steps are on their own rows; the editorial passes are grouped into three rounds.
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
| 02:15 to 02:40 | **Editorial work, round 2** (v2): simulation results, as-run runbook page, PR tooltips, access note | High-touch |
| 02:24 | **v2 published**, with the simulation results | High-touch |
| 02:36 to 02:56 | **Rule 1 on Kalshi:** all 20 saved F1 settings re-run trade by trade, 40 runs, read-only. [As run](kalshi-response-2026-10-20-as-run-rule1.html) | Streamlined |
| 03:52 | **Polymarket test locked:** settings, windows, floor, pick rule and scorer committed before any of it ran (racinglines PR [#276](https://github.com/racinglines/racinglines/pull/276)) | Streamlined |
| 03:55 to 04:01 | **Polymarket test run**, read-only, then scored once. [As run](kalshi-response-2026-10-20-as-run-pm-step3.html) | Streamlined |
| 05:02 to 05:10 | **"State of racinglines" set:** run every historical test our data allows, and say what decides "a fair price" vs "an edge worth trading" | High-touch |
| 05:23 to 11:03 | **Overnight batch** on our server, read-only, in two parts: the model on every market it prices, Kalshi on full order books, markouts, every resolved fill, and rule 1 over all 106 saved Polymarket settings (212 season runs, 207 completed). [As run](kalshi-response-2026-10-20-as-run-state-batch.html) | Streamlined |
| 06:45 to 12:22 | **Overnight results scored:** the model on every market, all 126 saved settings with the luck test, markouts, crowd fills | Streamlined |
| 12:13 to 12:40 | **Editorial work, round 3** (v3): the whole page rewritten as a state of racinglines, with context before each test, every as-run runbook linked, demo and repo links | High-touch |
| by 12:35 | **v3 published** here | High-touch |
| 13:00 (6:00 Pacific) | **Published on LinkedIn** | High-touch |
| 22:56 | **The flaw found:** a live run for Austin prices the favourite far below the market; the owner asks us to reconcile it with this page | High-touch |
| 23:00 to 23:25 | **Reconciled:** 62 to 86 % of v3's 2026 edge is betting against favourites; correction drafted (v3c) | Streamlined |
| 23:25 | **Correction held** until new model runs land | High-touch |
| 23:42 to 23:50 | 18 earlier model runs scored on the favourite, both seasons | Streamlined |
| 21 Oct 00:08 | **v4 set:** rebuild on the fixed model, v3 to an appendix | High-touch |
| by 01:21 | **Eight new sweeps** (grid noise, calibration off and on) on a copy of our database | High-touch |
| 01:49 onward | **Scored and written:** section 4, then v4 | Streamlined |
| about 01:58 | **v4 published** here | High-touch |
| 02:06 | **v4.1 set:** re-run every run behind v3's tables and plots with calibration on, and redo every plot | High-touch |
| 02:59 to 08:47 | **Calibrated rerun** on the copy of our database, after a backup: 262 runs, 0 failures. [As run](kalshi-response-2026-10-21-as-run-calib-rerun.html) | High-touch |
| 11:01 onward | **Scored and written:** every v3 script reused unchanged on the new runs; section 5 | Streamlined |

*Internal book pages behind this report: [3.4 Trading (golden rules)](https://racinglines.bet/docs/internal/system/trading.html), [3.5 Evaluation and sweeps](https://racinglines.bet/docs/internal/system/evaluation.html), [4.5 DITL reports](https://racinglines.bet/docs/internal/ops/ditl.html), [Appendix A: every report and its sources](https://racinglines.bet/docs/internal/appendix/reports.html).*
