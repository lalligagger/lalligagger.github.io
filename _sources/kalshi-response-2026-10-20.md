# RE: Kalshi's "Who Calibrates Markets"

**What must a trading strategy prove before it bets real money, and how close is ours?**

**Summary.** On 19 Oct 2026 Kalshi's research team published ["Who Calibrates Markets? Evidence from Trader Micro-Data"](https://kalshi.com/research/publications/who-calibrates-markets) (Kagan and Baiocchi). Kalshi is a prediction market: people buy and sell contracts that pay $1 if an event happens, so a contract's price is the crowd's probability for it. The paper finds that about 0.15 % of Kalshi accounts forecast better than the prices they trade against, after correcting for luck, and that prices tend to move their way after they trade.

We build racinglines ([web demo](https://racinglines.bet/term), [code](https://github.com/racinglines/racinglines-public)), a model that prices motorsport markets (Formula 1, NASCAR, MotoGP and more) and strategies that trade when the model disagrees with the market. We turned the paper's tests into five rules a strategy must pass before it trades real money, and held our own model to them, using two seasons of real prices from Kalshi and from Polymarket, a second prediction market.

**The answer: not yet.**

- **Our first apparent edge was mostly a model flaw.** On the day of the last qualifying session before a race, our F1 model gave its own favourite about a 37 % chance to win; that driver won about two races in three. In 2026 favourites happened to win less often than the market expected, so a model that priced them too low looked like it had an edge ([section 4](#where-the-2026-edge-came-from)).
- **Calibration fixes the favourite, not the edge.** Recalibrating the model's win probabilities on earlier races lifts the favourite after qualifying to 56 to 67 % (it won 63 to 71 %), in both seasons. But the model still loses to the market on race winners in 2025 ([section 5](#before-and-after-calibration)).
- **Every test splits the same way.** With calibration on, 2026 looks better (our F1 trades on Kalshi beat the market price by 13.7¢ a contract), 2025 looks worse, and no strategy setting holds up in both seasons. So the first rule, skill, is **not met** ([section 6](#what-it-all-means)).
- **What counts now is forward:** strategies fixed and published before each race, starting with the US Grand Prix in Austin (23 to 25 Oct 2026).

**Edit note.** Earlier versions of this page (20 Oct) reported an F1 2026 edge. This version explains the flaw behind it and re-runs every test with the fix. Every version is linked in [Appendix A](#appendix-a-versions). We said we would publish these results whatever they showed, and we have.

*How to read this. All times are UTC. "The paper" is Kagan and Baiocchi (2026); we point to its sections rather than restating its methods, and we don't republish it. Every number here is a backtest on recorded market data, run on a read-only mirror or a copy of our production database; nothing touched the live site. Nothing here is betting advice.*

*Links marked #NNN go to our private source repository and open only for its collaborators. Our public code is [racinglines/racinglines-public](https://github.com/racinglines/racinglines-public) (MIT) and the `racinglines` package on PyPI. The code changes, notes and the query behind every number can be shared on request: hello@racinglines.bet.*

## 1. The paper

### What it asks

A prediction market price is a crowd's forecast. If a contract on "Verstappen wins" trades at 40¢, the crowd says 40 %. The paper asks who makes those forecasts accurate. Its answer is a small group of **takers**, traders who accept the prices others post rather than posting their own, and who are right against the price more often than chance allows.

The paper's tools, which we reuse throughout:

- **The mid:** the midpoint between the best price to buy and the best price to sell, weighted by how much is offered at each. It is the market's forecast at that moment.
- **The skill score** (Brier skill score, paper sec 3.2): how much more accurate a trader's prices are than the mid's, on the outcomes that actually happened. Above zero means better than the market.
- **The luck test** (sec 3.3): with millions of traders, some look skilled by chance. The test redraws every outcome from the market's own probabilities many times, and corrects for how many traders were tested at once (a false-discovery correction). Only results that survive count.
- **Price impact** (sec 3.4): after a skilled trader buys, does the mid move toward them and stay there? If it does, the trader was ahead of the market, not just lucky on the outcome.

### What it finds

| Finding | Number | Paper |
|---|---|---|
| Skilled accounts, and their share of taker volume | 14,694 accounts (0.15 %), 22.8 % of volume | Table 1, sec 4.2 |
| Accuracy on their own trades (Brier score, lower is better), skilled vs market | 0.13813 vs 0.14705; 64 % of the gain is better calibration | Table 3 |
| Median skill score: all skilled traders / those better on both calibration and discrimination | 0.0189 / 0.0382 | Table 3B |
| Skilled again in the other half of events | 44.5 % (all traders: 3.0 %) | Table 4 |
| Skilled taker: paid over the mid; mid move by the next order / by the close; outcome minus mid | 1.5–1.8¢; 1.7¢ / 3.3¢; 3.8¢ | Fig 9 |
| Other takers: paid over the mid; mid move | 1.2¢; 0.3¢ or less | sec 5.4 |
| Share of taker trades that buy YES, other vs skilled | 67 % vs 53 % | sec 4.3 |
| Sports markets with no skilled taker | 8.2 % (no split by sport; motorsport isn't named) | Table 5 |

The paper states its own limits (sec 6): the price-impact results are associations, the skilled group is chosen on outcomes, arbitrage counts as skill, and the data cover one year.

### Why it matters for a model like ours

Racinglines is an attempt to be one of those skilled takers in motorsport, with a model instead of a hunch. The model simulates each race from the drivers' recent results, practice and qualifying, and turns that into a probability for every market it can price: who wins, who makes the podium, who takes pole, which of two drivers finishes ahead. A strategy then decides when the gap between our probability and the market's price is worth trading.

The paper gives us a way to check ourselves that most backtests skip. A backtest can look profitable because one of many settings got lucky, because it assumes prices nobody could actually have traded at, or because the season it was tuned on was unusual. The paper's tests answer each of those. They are also the right tests for anyone who wants to trade on a model, including people who use ours.

## 2. Five rules before real money

Before any taker strategy trades real money on Kalshi or any prediction market, by us or by someone using racinglines, it must pass all five. Anyone else proves them on their own fees, speed and size. We adopted these rules on 19 Oct and check every strategy we build against them.

| # | Rule | Where we stand, 21 Oct | Section |
|---|---|---|---|
| 1 | **Skill, not luck or selection** | **Not met.** The 2026 edge was mostly a bias against favourites; no setting holds up in both seasons or passes the luck test | [4](#where-the-2026-edge-came-from), [5](#before-and-after-calibration) |
| 2 | **Out of sample** | **Starts now.** The first truly unseen test is Austin, 23 to 25 Oct | [3.5](#the-tests) |
| 3 | **The real order book** | **Met for F1 backtests.** Every trade is filled against a recorded order book; not met for our older NASCAR and MotoGP paper bets | [5.4](#calibrated-the-polymarket-candidates) |
| 4 | **The price comes our way** | **Partly met.** Yes on Kalshi, about break-even on Polymarket, no error bars yet | [5.5](#calibrated-does-the-price-come-our-way) |
| 5 | **Paper matches real** | **Not started.** No real money yet | — |

**1. Skill, not luck or selection.** A positive skill score against the mid, at prices we could really have traded at, that survives the luck test with a correction for *every* setting we have ever tried. The best of many settings, chosen after the fact, is never evidence. *(Paper sec 3.2–3.3, Table 1, App. A.1.)*

**2. Out of sample.** The result holds on races the strategy has never seen, not on the season it was tuned on. We commit each race's strategy to a public file before the race starts and score it exactly as committed, win or lose ([#213](https://github.com/racinglines/racinglines/pull/213)). *(Paper sec 4.4, Table 4.)*

**3. The real order book.** The edge survives the actual spread, depth and fees: each simulated trade fills at the best available price, or works through the recorded order book for larger sizes, never at the last traded price plus a guessed cost ([#202](https://github.com/racinglines/racinglines/pull/202), [#219](https://github.com/racinglines/racinglines/pull/219)). *(Paper sec 2.1, App. A.4.)*

**4. The price comes our way.** After we trade, the mid moves toward our side and stays there. We measure it as markouts (the mid's move after each trade) and as closing-line value (how far the price moved our way by the time the market closed) ([#223](https://github.com/racinglines/racinglines/pull/223)). *(Paper sec 3.4, 5.4, Fig 9.)*

**5. Paper matches real.** Simulated fills match real fills before any money goes in. Our bar: closing-line value above +2 % over 100 live bets, and six live race weekends where simulated and real fills agree. *(Our rule, not the paper's.)*

## 3. How we tested {#how-we-tested}

### 3.1 The model and the race weekend

An F1 weekend has practice sessions, qualifying (which sets the starting grid) and the race. Our model prices every market at several **stages** of the weekend: before any running, after each practice session and after qualifying. At each stage it uses only what was known then. Most stages come before qualifying, when the most is still uncertain.

### 3.2 The markets and the prices

- **Venues:** Kalshi and Polymarket, the two largest prediction markets that list motorsport.
- **Markets:** F1 race winner, podium, pole position, head-to-heads and top constructor; NASCAR finishing positions on Kalshi.
- **Seasons:** F1 2025 and 2026 (2026 up to round 16), NASCAR 2025 and 2026.
- **Prices:** two sources. Our own recorder takes a snapshot of each market's prices every 5 minutes ("our tape"). Probalytics, a market-data vendor, supplies full historical order books for both venues, with every price level and size ("Probalytics' books"). Its Polymarket books start on 20 Nov 2025.

### 3.3 Strategies and settings

A strategy decides what to trade when our probability and the market's price disagree. A **setting** is one choice of its knobs: how big a gap to require, how much to stake, which markets to trade, and so on. Over the past months we saved 20 settings for Kalshi and 106 for Polymarket. Each setting trades a race weekend in four ways, run side by side:

- **update:** at every stage, trade toward a position sized by the gap, and close it when the gap disappears;
- **hold:** buy at the first stage where there is a gap, and hold to the end;
- **last:** trade only at the last stage, after qualifying;
- **early:** like update, but no new trades after the last practice session or qualifying.

Unless a table says otherwise, the numbers are for **update**.

### 3.4 Scoring

- **Edge vs the mid, in cents per contract:** for each trade, the outcome minus the mid when we traded, from our side. Before costs.
- **After costs:** the same, from the price we actually paid, fees included.
- **Skill score:** the paper's Brier skill score of our forecasts against the mid's.
- **Error bars:** every interval is 95 %, and resamples whole race weekends, because outcomes in one race are linked (only one driver can win).
- **The luck test** redraws every outcome from the mid 500 times and corrects across every setting, season and trading mode together.

### 3.5 The tests {#the-tests}

Each test answers one rule:

1. **The model on every market** (rule 1): every market the model prices, at every stage, traded or not. Is the model's probability better than the market's, and when they disagree by more than 5¢, who is right?
2. **One setting on Kalshi, trade by trade** (rules 1 and 3): every trade written out with the order book at that moment.
3. **Every setting, with the luck test** (rule 1): all 20 Kalshi and 106 Polymarket settings, corrected for all of them.
4. **A pre-registered Polymarket test** (rules 2 and 3). Before running anything we committed two untuned candidates (the model's default strategy, filling at the best price or working through the book), a market filter (a Polymarket market counts only if its volume and its trade count both reach the Kalshi median for the same race and market type; 717 of 2,866 pass, all F1), a rule for picking one candidate, and the scorer ([#276](https://github.com/racinglines/racinglines/pull/276), 20 Oct 03:52). The pick is made on five races (Las Vegas, Qatar and Abu Dhabi 2025, 2026 rounds 3 to 6), scored once on 2026 rounds 7 to 16, then scored live from Austin with nothing changed. Only the live part is blind: our model had already been run on the earlier races.
5. **Does the price come our way** (rule 4): the mid at each trade, at the next stage of the weekend (a median 2.5 to 3.5 hours later) and at the last stage before the race (a median 23 to 29 hours later).
6. **The paper's own figures,** rebuilt for us: Fig 1 (where we sit on the skill distribution), Table 3 (where the accuracy comes from) and App. A.1–A.2 (why a skill score isn't profit).

### 3.6 Calibration: the before and after

A model can rank drivers well and still be too cautious, giving the favourite 37 % when 60 % is right. **Calibration** corrects that: it sharpens the model's win probabilities by a power fitted only on earlier races, so it never sees the race it is pricing. Section 4 explains why it matters. Section 5 runs everything twice:

- **Before:** the model as it ran on 20 Oct, calibration off.
- **After:** the same model, settings, markets and scorers on 21 Oct, calibration on, as our F1 model now runs. 262 runs, none failed ([the exact runbook and scripts](kalshi-response-2026-10-21-as-run-calib-rerun.html)).

MotoGP isn't in the comparison (it has no calibration yet), and neither are our measurements of the whole market's trading, which don't use our model ([v4](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md), A3.6 and A4.4–A4.5).

## 4. Where the 2026 edge came from {#where-the-2026-edge-came-from}

With calibration off, the tests on 20 Oct looked promising for 2026: every saved setting bought on the right side of the mid in F1 2026, on both venues, and about half of the gap the model predicted showed up in the results. The same model found nothing in 2025. That evening a live run for Austin priced the favourite far below the market, so we went back to the 2026 runs and split their edge by direction: bets **against the favourite** (the market says above 35¢ and the model says lower), and everything else.

| Run | Bets against the favourite: count, average result | Every other big disagreement | Share of the total edge against the favourite | Markets above 50¢: market / model / how often they won | Race-win favourite: market / model / won |
|---|---|---|---|---|---|
| F1 Kalshi, our tape, 2026 | 248: **+11.9¢** | 410: +2.7¢ | **73 %** | 0.65 / **0.37** / 0.47 | 0.42 / **0.17** / 0.30 |
| F1 Polymarket, Probalytics' books, 2026 | 186: **+13.7¢** | 364: +4.3¢ | **62 %** | 0.63 / **0.37** / 0.40 | 0.40 / **0.17** / 0.31 |
| F1 Polymarket, our tape, 2026 | 211: +11.2¢ | 478: +0.8¢ | 86 % | 0.64 / 0.39 / 0.47 | 0.42 / 0.17 / 0.34 |
| F1 Kalshi, our tape, 2025 | 236: +3.8¢ | 274: +4.5¢ | 42 % | 0.72 / **0.45** / 0.67 | 0.39 / **0.19** / 0.45 |
| F1 Polymarket, our tape, 2025 | 241: +2.9¢ | 507: −2.6¢ | — (total about 0) | 0.67 / **0.50** / 0.63 | 0.41 / **0.20** / 0.41 |

*Each row counts a market once per stage, so one race counts several times; the race-win columns cover 13 to 23 races per run.*

- **The model priced favourites far too low, in both seasons.** Where the market said 65 to 72 %, the model said 37 to 50 %. For the race favourite it said about 17 to 20 % where the market said about 40 %.
- **2026 rewarded that; 2025 didn't.** In 2026 race favourites won 30 to 34 % of the time against a price of 40 to 42 %, so betting against them made most of the 2026 edge. In 2025 they won about as often as priced.
- **Backing outsiders added little** (+2 to +3.5¢ on markets under 15¢). This isn't the classic favourite-longshot bias; it is a model that spreads the field too evenly, meeting a season of beaten favourites.
- **Not known:** whether 2026's favourites were truly overpriced, or unlucky over 13 to 16 races.

The same night we re-ran 18 season simulations on Polymarket's prices, with calibration off and on and different weights on recent form, and asked how often the model's own favourite won:

| Model's favourite to win | Calibration | 2025 (24 races): model / won | 2026 (17 races): model / won |
|---|---|---|---|
| After qualifying | off (5 settings) | **0.38–0.39** / 0.63 | **0.36** / 0.68–0.71 |
| After qualifying | on (4 settings) | 0.58–0.60 / 0.63 | 0.66–0.67 / 0.71 |
| Before qualifying | off | 0.19–0.22 / 0.27–0.49 | 0.18–0.19 / 0.15–0.34 |
| Before qualifying | on | 0.29–0.32 / 0.28–0.50 | 0.27–0.29 / 0.16–0.35 |

*Ranges are across settings; "won" changes with the setting because it can change who the favourite is. "Before qualifying" counts several stages per race.*

After qualifying, calibration fixes the favourite in both seasons. Before qualifying, where most of the 2026 edge came from, even the calibrated favourite (27 to 32 %) stays below the market's (about 40 %). No setting held up in both seasons. ([Scripts and runs](kalshi-response-2026-10-20-as-run-state-batch.html).)

## 5. Before and after calibration {#before-and-after-calibration}

Every table shows **after** (calibration on), with **before** (calibration off) beside it or in brackets.

### 5.1 The model {#before-and-after-the-model}

**The favourite.** The model's favourite to win, its probability / how often it won, on Polymarket's prices, every completed race:

| | 2025 after qualifying | 2026 after qualifying | 2025 before qualifying | 2026 before qualifying |
|---|---|---|---|---|
| Before | 0.38 / won 0.63 | 0.36 / won 0.68 | 0.22 / won 0.27 | 0.18 / won 0.15 |
| After | **0.56** / won 0.63 | **0.67** / won 0.71 | 0.32 / won 0.27 | 0.29 / won 0.21 |

*24 races in 2025, 17 in 2026.*

**Against the market on race winners.** Brier score, model minus market (below zero means the model was more accurate), and closing-line value for the update strategy:

| | 2025 before / after qualifying | 2026 before / after qualifying | Closing-line value 2025 | 2026 | 2026 rounds 13–17 |
|---|---|---|---|---|---|
| Before | +0.0068 / +0.0079 | −0.0006 / +0.0025 | +0.99 pp | +0.32 pp | −0.99 pp |
| After | +0.0077 / +0.0088 | −0.0018 / **−0.0065** | +1.30 pp | +0.58 pp | −0.19 pp |

*Rounds 13 to 17 of 2026 were held out of earlier tuning.*

**Every market it prices, at every stage**, traded or not. "Slope" asks how much of the model's predicted edge was real: 1 means all of it, 0 none.

| Model, venue, prices, season | Skill vs market: after (before) | Slope: after (before) | Result where the gap was over 5¢: after (before) |
|---|---|---|---|
| F1, Kalshi, our tape, 2026 R1–16 | −0.01 [−0.10, +0.09] (+0.00) | **0.48 [0.20, 0.85]** (0.52) | **+6.7¢ [+1.0, +13.4]** (+6.2¢) |
| F1, Kalshi, Probalytics' books, 2026 R5–16 | +0.02 [−0.07, +0.11] (+0.00) | **0.59 [0.18, 1.00]** (0.51) | +7.1¢ [−0.1, +14.0] (+5.8¢) |
| F1, Polymarket, Probalytics' books, 2026 R3–16 | +0.03 [−0.12, +0.12] (+0.01) | **0.59 [0.05, 1.01]** (0.53) | +8.4¢ [−0.8, +16.2] (+7.4¢) |
| F1, Polymarket, our tape, 2026 R1–16 | −0.04 [−0.17, +0.06] (−0.04) | 0.36 [−0.02, 0.76] (0.38) | +5.4¢ [−1.3, +12.4] (+4.0¢) |
| F1, Kalshi, our tape, 2025 | −0.10 [−0.24, +0.02] (−0.14) | 0.09 [−0.34, 0.58] (0.09) | +3.2¢ [−2.9, +10.1] (+4.2¢) |
| F1, Polymarket, our tape, 2025 | **−0.14 [−0.24, −0.05]** (−0.16) | −0.11 [−0.38, 0.22] (−0.10) | −1.0¢ [−5.9, +4.6] (−0.8¢) |
| NASCAR, Kalshi, 2026 | **−0.10 [−0.15, −0.05]** (−0.10) | −0.07 [−0.34, 0.18] (−0.06) | −1.3¢ [−5.0, +2.5] (−0.7¢) |
| NASCAR, Kalshi, 2025 | −0.19 (−0.19) | too few | too few |

| | Race favourite: model / market / won | Markets above 50¢: model / market | Share of the 2026 edge against the favourite |
|---|---|---|---|
| Before | 0.17–0.20 / 0.39–0.42 / 0.30–0.45 | 2026 0.37–0.39, 2025 0.45–0.50 / 0.63–0.72 | 62–86 % |
| After | **0.23–0.28** / 0.39–0.42 / 0.30–0.45 | 2026 0.40–0.42, 2025 0.52–0.54 / 0.63–0.72 | **52–74 %** |

**What it says:** calibration fixes the favourite after qualifying in both seasons, but doesn't make an edge. Before qualifying, which is most stages, the favourite stays well below the market's price, and the model still loses to the market in 2025.

### 5.2 One setting on Kalshi, trade by trade {#calibrated-one-setting-on-kalshi}

One saved setting, every trade, against Kalshi's mid.

| Sport, season | Trades (weekends): after (before) | Edge vs mid, ¢ per contract: after (before) | After costs: after (before) | Skill score vs mid: after (before) |
|---|---|---|---|---|
| F1 2025 | 131 (20) (151) | +1.0 [−8.7, +11.4] (+5.4) | −3.6 (+1.0) | **−0.28 [−0.59, −0.02]** (−0.19) |
| F1 2026 | 234 (15) (223) | **+13.7 [+7.4, +20.1]** (+12.0) | **+9.7** (+8.0) | +0.08 [−0.06, +0.19] (+0.05) |
| NASCAR 2025 | 23 (8) (24) | +11.9 [−6.9, +20.3] (+12.0) | +7.5 (+7.6) | −0.36 [−1.42, +0.88] (−0.33) |
| NASCAR 2026 | 575 (31) (584) | +0.1 [−4.1, +4.6] (−0.4) | **−5.3** (−5.8) | **−0.16 [−0.26, −0.07]** (−0.16) |

**What it says:** F1 2026 is the only row where the edge is clearly above zero; F1 2025 loses after costs, and the NASCAR model loses to Kalshi's price.

### 5.3 Every setting, with the luck test {#calibrated-every-setting-tried}

| Venue, season | Settings | Edge vs mid, median (range): after (before) | After costs, median: after (before) | Edge clearly above zero: after (before) | Skill clearly above zero: after (before) | Pass the luck test: after (before) |
|---|---|---|---|---|---|---|
| Kalshi, 2026 R1–16 | 20 | +10.9¢ (+4.9 to +24.2) (+9.2¢) | +7.0¢ (+5.1¢) | **18 of 20** (11) | 3 of 20 (3) | 20 of 20 (19) |
| Kalshi, 2025 | 20 | +3.3¢ (−0.7 to +6.3) (+4.7¢) | **−0.9¢** (+0.8¢) | 0 of 20 (4) | 0 of 20 (0) | 4 of 20 (4) |
| Polymarket, 2026 R3–16 | 106 | +12.2¢ (+5.5 to +20.0) (+8.8¢) | +9.1¢ (+5.7¢) | **40 of 106** (2) | **0 of 106** (0) | 301 of 773 (0 of 749) |
| Polymarket, 2025 R22–24 | 106 (101) | **−15.2¢ (−35.6 to +8.8)** (+4.5¢) | **−19.9¢** (+0.6¢) | 0 of 106 (15) | 0 of 106 (0) | 0 (0) |

*"Clearly above zero" means the 95 % interval is above zero. The luck-test column counts setting, season and trading-mode combinations, and treats every trade as independent, which flatters it; the skill intervals resample whole weekends and are the ones to trust. The 2025 Polymarket window is only three races, the first with full order books.*

**What it says:** no setting has an edge clearly above zero in both seasons on either venue, and none has a skill score clearly above zero on Polymarket. Across settings, the 2025 and 2026 results are negatively correlated (Kalshi −0.25, Polymarket −0.47): a setting that did well in one season tended to do worse in the other.

### 5.4 The pre-registered Polymarket test {#calibrated-the-polymarket-candidates}

The before row is the test exactly as registered and scored on 20 Oct. The after rows use the same candidates, races, filter, pick rule and scorer with calibration on; they weren't registered in advance.

| Window | Candidate | Trades (weekends) | After costs | Vs the mid | Skill vs mid |
|---|---|---|---|---|---|
| Pick races, after | **works through the book, update (picked)** | 103 (7) | −4.2¢ [−14.2, +5.0] | −1.6¢ [−11.6, +7.9] | −0.20 [−0.48, +0.02] |
| Scored races 7–16, after | **works through the book, update (picked)** | 198 (9) | **+8.8¢ [−1.4, +18.1]** | +10.8¢ [0.0, +19.6] | +0.05 [−0.21, +0.21] |
| Scored races 7–16, before | works through the book, update (picked) | 213 (9) | +6.8¢ [−4.7, +18.0] | +8.9¢ [−2.7, +20.0] | −0.01 [−0.32, +0.21] |

**What it says:** the same candidate is picked either way, its result on the scored races could still be zero, and its skill score is about zero. It doesn't pass. Every simulated trade had a real order book behind it, and working through the book cost under 1¢ more than the best price at these sizes.

### 5.5 Does the price come our way? {#calibrated-does-the-price-come-our-way}

| Trades | Trades: after (before) | Paid over the mid: after (before) | Mid move by the next stage: after (before) | By the last stage: after (before) |
|---|---|---|---|---|
| Kalshi, 20 settings, Probalytics' books, 2026 R5–16 | 4,799 (4,844) | 3.6¢ (3.6¢) | **+3.4¢** (+2.8¢) | **+6.7¢** (+4.5¢) |
| Kalshi, 20 settings, our tape, 2026 | 3,282 (3,084) | 1.5¢ (1.6¢) | **+4.8¢** (+3.4¢) | **+8.1¢** (+5.3¢) |
| Polymarket, the registered candidates | 641 (709) | 1.9¢ (1.8¢) | +1.7¢ (+1.6¢) | +1.9¢ (+1.4¢) |
| *Paper, skilled takers* | | *1.5–1.8¢* | *1.7¢ by the next order* | *3.3¢ by close* |

**What it says:** on Kalshi the price moves our way by about twice what we paid before the race starts, the same shape as the paper's skilled takers; on Polymarket it about breaks even. These are simulated trades that didn't move the price themselves, the same trade appears under several settings, and there are no error bars yet.

### 5.6 The paper's figures, rebuilt for us {#calibrated-the-papers-outputs}

**Fig 1: where we sit on the paper's skill scale.** The skill score of each setting's actual fills against the mid, with the luck test.

| Venue, season | Settings | Skill score of the fill, median (range): after (before) | Above the paper's median skilled trader (0.0189): after (before) | Luck test passed alone: after (before) | Passed after correcting for all settings |
|---|---|---|---|---|---|
| Polymarket F1 2026 | 106 | **+0.028 (+0.015 to +0.045)** (+0.016) | **95 of 106** (19) | 86 of 106 (3) | **0** (0) |
| Polymarket F1 2025 | 106 (101) | −0.081 (−0.124 to −0.030) (−0.021) | 0 (0) | 0 (0) | 0 (0) |
| Kalshi F1 2026, the 5.2 setting | 1 | +0.037 (+0.033) | yes (yes) | no, p = 0.07 (p = 0.06) | — |
| Kalshi F1 2025, the 5.2 setting | 1 | +0.016 (+0.011) | no (no) | no, p = 0.10 (p = 0.14) | — |

![Every saved setting on the paper's skill scale, before (grey) and after (blue) calibration](img/fig1-bss-placement-v4.png)

**Table 3: where the accuracy comes from.** The paper splits a Brier score into reliability (how well calibrated; lower is better) and resolution (how well it tells outcomes apart; higher is better).

| | F1 Kalshi 2026: model / mid, after | before | F1 Polymarket scored races: model / mid, after | before |
|---|---|---|---|---|
| Reliability (lower is better) | 0.0109 / 0.0200 | 0.0067 / 0.0189 | 0.0170 / 0.0158 | 0.0088 / 0.0135 |
| Resolution (higher is better) | **0.0289 / 0.0196** | 0.0218 / 0.0208 | **0.0471 / 0.0358** | 0.0333 / 0.0412 |

**App. A.1–A.2: a skill score is not profit.** The Kalshi F1 2026 trades of 5.2, each re-priced as if we had paid h cents over the mid:

| Paid over the mid | 1¢ | 2¢ | 3¢ | 4¢ |
|---|---|---|---|---|
| Skill score of the fill: after | 0.0114 | 0.0219 | 0.0316 | 0.0404 |
| before | 0.0104 | 0.0200 | 0.0286 | 0.0363 |
| Profit per contract (¢): after | 20.56 | 19.56 | 18.56 | 17.56 |
| before | 16.87 | 15.87 | 14.87 | 13.87 |

*Weighted by contracts, before fees.*

**What it says:** in 2026 almost every Polymarket setting now sits above the paper's median skilled trader, but none passes the luck test once all settings are counted, and every 2025 setting is below zero. Paying more over the mid raises the skill score while each cent paid comes straight off profit, as the paper warns, so a skill score is always read next to profit.

## 6. What it all means {#what-it-all-means}

### What the paper asks of a trader

Kalshi's paper says a market is kept accurate by a small group of takers who are right against the price, who can be told apart from the lucky by a strict test, and who leave a mark: the price moves their way and stays. That is a high bar, and it is the right one for anyone who wants to trade on a model. We turned it into five rules and held our own motorsport model to them.

### What our tests found

- **Skill (rule 1): not met.** In F1 2026 almost everything looks good: every saved setting beats the mid on both venues, our trades on Kalshi make 13.7¢ a contract before costs and 9.7¢ after, and on Polymarket most settings score above the paper's median skilled trader. But nothing passes the luck test once every setting we have tried is counted, and nothing holds up in 2025. A skilled trader in the paper is skilled again in the other half of events 44.5 % of the time; across our settings, a good 2026 tends to mean a bad 2025.
- **Out of sample (rule 2): starts now.** Every backtest window here had been seen, in some form, while the model was built. The pre-registered Polymarket test made money on the races it was scored on (+6.8¢ a contract after costs, +8.8¢ with calibration on), but not by enough to rule out zero.
- **The real order book (rule 3): met for F1 backtests.** Every simulated F1 trade fills against a recorded order book; 7 to 8 % of Kalshi F1 trades had no book at that moment and are counted as such, not priced off a last trade. Our older NASCAR and MotoGP paper bets don't meet it.
- **The price comes our way (rule 4): partly met.** On Kalshi the mid moves our way by 3 to 5¢ by the next stage and 7 to 8¢ by the last stage before the race, more than the 1.5 to 3.6¢ we paid over the mid. On Polymarket it about breaks even. Without error bars and with trades counted under several settings, it doesn't count yet.
- **Paper matches real (rule 5): not started.** No real money has gone in.

### Why the 2026 edge appeared

Most of it was one bet. The model priced favourites far too low, in both seasons, and in 2026 favourites won less often than the market priced them. Betting against them made 62 to 86 % of the 2026 edge. In 2025 favourites won about as priced, and the same model made nothing. The tests caught it the way the paper says they should: not by any single number, but because the result didn't survive a second sample.

### What calibration changed

Calibration is a real fix to the model. After qualifying, the favourite goes from about 37 % to 56 to 67 %, close to how often it won, in both seasons. It is now on in our F1 model, and the Austin strategy uses it. But it isn't an edge. Before qualifying, which is most of a race weekend, the favourite is still well below the market's price. Re-running every test with calibration on made 2026 look better and 2025 look worse, so the gap between the seasons got wider, not narrower. We don't yet know why.

### Is there room for a model in motorsport?

Possibly. On Kalshi, motorsport takers pay about 2.1 to 2.5¢ over the mid, about twice the paper's average of 1.2¢, and 62 % of Kalshi's motorsport volume trades in the last tenth of a market's life, mostly during the event ([v4](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md), A3.6 and A4.4). That looks like a thinner, more expensive market than the big leagues that dominate the paper's sports volume, which is where an informed model could matter. And in 2026 the market's favourites were overpriced. Whether that is a pattern or one season is exactly what two seasons of history can't tell us.

### What it means for anyone trading on a model

A backtest can look like skill for three reasons that aren't skill: one lucky setting among many, prices that couldn't really have been traded, or one unusual season. Our 2026 result passed the first two checks and failed the third. Anyone building on racinglines, or on any model, should ask the same three questions before trusting a result, and should expect the honest answer to be "not yet" more often than not.

### What's next

- **Forward, from Austin (23 to 25 Oct).** Each race's strategy is committed before the race and scored as committed; the pre-registered Polymarket pick is scored live with nothing changed. This is the only blind test, and rule 2's record starts here.
- **The favourite before qualifying.** The next model work is how the model spreads the front of the field before qualifying, judged on both seasons together.
- **Why calibration widens the gap between seasons.** Not yet checked.
- **Error bars for the markouts,** with each trade counted once, so rule 4 can count.
- **Skilled traders in F1.** Polymarket trades carry wallet addresses, so the paper's account-level tables can be built for F1 there.
- **Real money (rule 5),** only after the rest.

We said we would publish what the tests showed, whatever it was. Our first claim of an edge was mostly a model flaw; the flaw is partly fixed; the edge isn't proven.

## 7. Caveats

1. **Simulated, not executed.** A mid move after a simulated trade is the market coming round to our view, not our impact. Our own impact on thin order books is unmeasured.
2. **How fills are modelled changes the score** (paper sec 2.3). We report filling at the best price and working through the book side by side.
3. **The mid is estimated** wherever we have no recorded order book. 2025 Kalshi is estimates only.
4. **Timing.** Our order books are 5-minute snapshots and we trade at fixed stages, while the paper's sports volume sits in the last tenth of a market's life (Fig 7).
5. **Selection.** The luck test's correction runs over every combination tried. The best setting in hindsight is never evidence.
6. **Linked outcomes.** Our luck test draws each driver independently and ignores that only one can win, so its p-values are approximate.
7. **Sample size.** NASCAR has 40 races and MotoGP 11; the paper's bar is 100 events.
8. **Gaps in our data.** Our own Kalshi trade record has a gap from 29 Sep to 9 Oct.
9. **Fees.** The skill score ignores fees; profit and closing-line value include them. We report all three together.
10. **Scope.** We have no linked cycling markets on either venue, so cycling, which we also model, isn't tested here.
11. **Not blind.** Every backtest window here was seen in some form while the model was built. Only the forward tests from Austin are blind.
12. **Calibration was chosen after looking.** Its power is fitted only on earlier races, but we turned it on after seeing both seasons, so the "after" results are a re-measurement, not a test. Calibrated runs on our database copy also differ very slightly from the same settings on production (the 2026 favourite after qualifying: 0.668 against 0.670); the cause isn't yet checked.

## Appendix A: versions {#appendix-a-versions}

- **20 Oct, 00:54:** [first draft](https://github.com/lalligagger/lalligagger.github.io/blob/ffbb1e9c8e2512e76fefa816090172304f6cfb1a/_sources/kalshi-response-2026-10-20.md), from our stored paper bets.
- **20 Oct, 02:24 to 12:35 (v2, v3):** the simulations, trade by trade, and the 20 Oct tests with calibration off ([as published](https://github.com/lalligagger/lalligagger.github.io/blob/d215d10/_sources/kalshi-response-2026-10-20.md)).
- **21 Oct, about 01:58 (v4):** the favourite flaw explained and the first fixes tested; the 20 Oct results kept in its Appendix A ([as published](https://github.com/lalligagger/lalligagger.github.io/blob/33aaee3/_sources/kalshi-response-2026-10-20.md)).
- **21 Oct (this version):** every test re-run with calibration on; rewritten for first-time readers.

## Appendix B: how we built this

From the paper's release to this page. "Person" steps needed a decision, an upload or a publish; "agent" steps were done by AI coding agents (Claude Code) with no input.

| Time (UTC) | Step | Who |
|---|---|---|
| 19 Oct, about 19:20 | Kalshi publishes the paper | — |
| 23:43 | The paper is handed to an agent for research | Person |
| 23:46 to 23:56 | **Questions set:** the paper's findings, a simulation in its framework, our gaps, and the top one: what must a strategy prove before it trades real money? | Person |
| 23:50 to 23:59 | **First numbers:** our stored Kalshi paper bets on the paper's skill scale, with a luck test; the market's price impact rebuilt from our data | Agent |
| 23:57 to 00:01 | **Five rules adopted** and written into our development guide | Agent |
| 20 Oct 00:04 | **First draft** (7 pages), 21 minutes after the hand-off | Agent |
| by 00:54 | **First draft published** here | Person |
| 00:54 to 01:36 | Simulation code: every simulated trade written out with the order book at that moment ([#275](https://github.com/racinglines/racinglines/pull/275)) | Agent |
| 01:16 to 01:18 | **Publish-whatever-it-shows rule set**; a new blind test is always allowed | Person |
| 01:49 to 01:59 | **Simulation run**, read-only. [Runbook as run](kalshi-response-2026-10-20-as-run-resim.html) | Person |
| 02:36 to 02:56 | **Every Kalshi setting** re-run trade by trade. [As run](kalshi-response-2026-10-20-as-run-rule1.html) | Agent |
| 03:52 | **Polymarket test registered** before any of it ran ([#276](https://github.com/racinglines/racinglines/pull/276)) | Agent |
| 03:55 to 04:01 | **Polymarket test run** and scored once. [As run](kalshi-response-2026-10-20-as-run-pm-step3.html) | Agent |
| 05:23 to 11:03 | **Overnight batch:** the model on every market, Kalshi on full order books, markouts, the market's own trades, and every Polymarket setting. [As run](kalshi-response-2026-10-20-as-run-state-batch.html) | Agent |
| by 12:35 | **Results with calibration off published** here | Person |
| 22:56 | **The flaw found:** a live run for Austin prices the favourite far below the market | Person |
| 23:00 to 23:50 | **Traced:** 62 to 86 % of the 2026 edge is betting against favourites; 18 more model runs scored | Agent |
| 21 Oct, by 01:58 | **First fixes tested and published** (calibration, grid noise) | Person |
| 02:06 | **Decision:** re-run every test with calibration on | Person |
| 02:59 to 08:47 | **Calibrated re-run**, 262 runs, none failed. [As run](kalshi-response-2026-10-21-as-run-calib-rerun.html) | Person |
| 11:01 onward | **Scored and written:** the same scripts on the new runs; this version | Agent |
