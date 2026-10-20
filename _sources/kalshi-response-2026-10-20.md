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
twice the paper's average, and a last Kalshi trade prints about 4¢ from the mid.

**Update, 20 Oct 02:20 (v2): the Kalshi simulation has run.** Re-scored trade by trade, our F1 2026 trades beat
Kalshi's mid by 12¢ a share (8¢ after costs), and the edge holds across race weekends. But the same settings find no
edge in 2025, and they are one of 20 tried on these seasons, so it is not yet skill. On NASCAR, Kalshi's price beats our
model. Results are in [step 2](#step-2-kalshi-re-simulated-trade-by-trade-done-20-oct); the Polymarket test (step 3) is now
pre-registered. **This is a live draft, tracked in our commit history** ([first draft](https://github.com/lalligagger/lalligagger.github.io/commit/ffbb1e9c8e2512e76fefa816090172304f6cfb1a)).

*All times are UTC. "The paper" is Kagan and Baiocchi (2026), [kalshi.com/research](https://kalshi.com/research/publications/who-calibrates-markets); we point to its sections rather than restating its
methods, and we don't republish it. Our numbers come from read-only queries on our production database, run 19 Oct
23:45 to 20 Oct 00:00. Nothing here is betting advice.*

*Access on request: PR links go to our private repository and internal-book links (racinglines.bet/docs/internal) are admin-only. Those, our process notes, the internal draft and the query list behind every number can be shared on request: hello@racinglines.bet.*

## The answer: five golden rules

Before any taker strategy is deployed with real money on Kalshi or any prediction market, by us or by a customer, it
must prove all five. A customer proves them on their own fees, latency and size. We adopted these as our own rules on
19 Oct; every taker strategy we build is now checked against them. They open our development guide ([#274](https://github.com/racinglines/racinglines/pull/274)) and the trading chapter of our internal book ([3.4 Trading](https://racinglines.bet/docs/internal/system/trading.html)).

| # | Rule | Status today |
|---|---|---|
| 1 | **Skill, not luck or selection** | **Not met** |
| 2 | **Out of sample** | In force, short record |
| 3 | **The real book** | **Not met** for NASCAR/MotoGP paper bets |
| 4 | **The price comes our way** | Partial: close point only |
| 5 | **Paper reconciles with real** | Not started |

**1. Skill, not luck or selection.** A positive Brier skill score against the size-weighted mid at realistic fills, surviving a luck test with a false-discovery correction across *every* setting tried

- *Paper:* sec 3.2–3.3, Table 1, App. A.1
- *What we have:* Model log loss, Brier and calibration against the market in every sweep ([#221](https://github.com/racinglines/racinglines/pull/221), [#227](https://github.com/racinglines/racinglines/pull/227)); a stage-by-stage walk-forward pick ([#206](https://github.com/racinglines/racinglines/pull/206))
- *Missing:* Skill score per trade against the weighted mid; luck test; correction across the whole grid
- *Today:* **Not met.** NASCAR scores +0.009, but in-sample and with no grid correction

**2. Out of sample.** It holds forward on pre-registered locks, not on the season it was tuned on

- *Paper:* sec 4.4, Table 4
- *What we have:* Per-event strategy locks, committed before each event and scored as bet (since 15 Oct, [#213](https://github.com/racinglines/racinglines/pull/213)); season holdouts ([#212](https://github.com/racinglines/racinglines/pull/212), [#228](https://github.com/racinglines/racinglines/pull/228), [#237](https://github.com/racinglines/racinglines/pull/237))
- *Missing:* A long enough forward record
- *Today:* In force; the record is still short

**3. The real book.** The edge clears the actual spread, depth and fees: fills at the touch or walking the recorded book, never a last-trade price plus a fixed cost

- *Paper:* sec 2.1, App. A.4
- *What we have:* Kalshi touch fills ([#55](https://github.com/racinglines/racinglines/pull/55), on candle bid/ask from [#52](https://github.com/racinglines/racinglines/pull/52), [#53](https://github.com/racinglines/racinglines/pull/53)); touch or book-walk fill options ([#203](https://github.com/racinglines/racinglines/pull/203), [#204](https://github.com/racinglines/racinglines/pull/204), [#205](https://github.com/racinglines/racinglines/pull/205)) with a depth limit ([#219](https://github.com/racinglines/racinglines/pull/219)); a fill-error check against Probalytics' full order books ([#202](https://github.com/racinglines/racinglines/pull/202); tape from [#181](https://github.com/racinglines/racinglines/pull/181), [#200](https://github.com/racinglines/racinglines/pull/200))
- *Missing:* Our stored NASCAR/MotoGP Kalshi paper bets used a one-price fill (last price ± 1¢)
- *Today:* **Not met** for those paper bets; our F1 backtests fill at the touch

**4. The price comes our way.** After entry the mid moves toward our side and doesn't reverse

- *Paper:* sec 3.4, 5.4, Fig 9
- *What we have:* Closing-line value (CLV) for every backtest ([#223](https://github.com/racinglines/racinglines/pull/223)) and paper bet ([#81](https://github.com/racinglines/racinglines/pull/81)), which is the close-time point of that path
- *Missing:* Markouts at 5 min, 1 h and the next stage; the outcome gap
- *Today:* Partial: close point only

**5. Paper reconciles with real.** Simulated fills match real fills before any money goes in

- *Paper:* (ours, not the paper's)
- *What we have:* Probalytics' books as the truth for fills ([#202](https://github.com/racinglines/racinglines/pull/202), [#212](https://github.com/racinglines/racinglines/pull/212)); our go/no-go goal of CLV above +2 % over 100 live bets
- *Missing:* Real fills
- *Today:* Not started (no real money yet)


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
| 00:12 to 00:54 | Editorial work, several passes: public edition in the site's layout, title, PR and source links, preview card, landing-page card | High-touch |
| by 00:54 | **Live draft published** here | High-touch |
| 00:54 to 01:36 | Simulation code: every backtest trade written out with the book's bid, ask and sizes (racinglines PR #275); one CI fix, so the backtest's own results stay byte-identical | Streamlined |
| 01:16 to 01:18 | **No-cheating rule set:** steps 2 and 3 get published whatever they show; a new blind test is always allowed | High-touch |
| 01:49 to 01:59 | **Simulation run** on our server, read-only: five sport-seasons in two runs, about 4.5 minutes in all. Every F1 total matches the published backtest to the cent. [Runbook as run](kalshi-response-2026-10-20-as-run-resim.html) | High-touch |
| 02:07 to 02:12 | **Trades scored** in the paper's framework (step 2 below) | Streamlined |
| 02:15 | **Polymarket test split chosen**, before any of it runs (step 3) | High-touch |

*Internal book pages behind this report: [3.4 Trading (golden rules)](https://racinglines.bet/docs/internal/system/trading.html), [3.5 Evaluation and sweeps](https://racinglines.bet/docs/internal/system/evaluation.html), [4.5 DITL reports](https://racinglines.bet/docs/internal/ops/ditl.html), [Appendix A: every report and its sources](https://racinglines.bet/docs/internal/appendix/reports.html).*

## What's next: three steps

Each step gets its own "what this does and doesn't show".

**No cheating:** we will publish the results of steps 2 and 3 whatever they say about our early models and strategies.

### Step 1: Kalshi, from the paper bets already in our database (first draft)

- **It shows** the five rules we now hold every taker strategy to, and where each stands for us today.
- **It shows** our stored NASCAR Kalshi paper bets score +0.0090 on the paper's scale, beyond a 1,000-draw luck null,
  all of it from calibration.
- **It shows** motorsport takers on Kalshi pay 2.1–2.5¢ over the weighted mid, and a last trade sits about 4¢ from it.
- **It doesn't show** that any of our strategies is skilled: the bets are in-sample, the fills synthetic, and there are
  fewer than 100 events.
- **It doesn't show** anything about F1 on Kalshi from our own paper bets, any estimator's accuracy, or Kalshi's
  market-level calibration in motorsport.

### Step 2: Kalshi, re-simulated trade by trade (done, 20 Oct)

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
- **It doesn't show** a full rule 1 test, which needs every one of the 20 settings re-run trade by trade so that the
  correction covers them all. That run is next.

### Step 3: Polymarket on Probalytics, a fresh, pre-registered test (split chosen 20 Oct 02:15)

Probalytics' full order books give the best fill fidelity and the most volume. We drop every tuned or "best" setting
and start from un-tuned candidates. Committed before anything runs:

- **Window:** Probalytics' Polymarket books begin 20 Nov 2025. That makes the window Las Vegas 2025 to 2026 round 17.
  Rounds 1 and 2 of 2026 are left out because they had almost no Polymarket volume in our links.
- **Market floor:** a market is tested only if its Polymarket volume and trade count both reach the Kalshi median for
  the same round and market kind. That keeps about 95 % of Polymarket F1 volume. NASCAR and MotoGP fail the floor
  on Polymarket, so this test is F1 only.
- **Split, forward in time:** tune on Las Vegas, Qatar and Abu Dhabi 2025 plus 2026 rounds 3 to 6 (296 markets,
  $10.8M). Hold out rounds 7 to 17 (474 markets, $13.1M), scored once. The holdout is the bigger half on purpose: the
  model is fit on race data and needs no prices, so the tuning window only picks among a few un-tuned strategy
  candidates.
- **A second, live holdout:** from Austin (round 18) on, the same frozen settings are scored as each race settles.
- **Scoring:** skill score against the size-weighted mid, Murphy split, the luck test with false-discovery
  correction over every candidate, markouts, and CLV.

We also add one hypothetical: **a model ready the day F1 first traded on Polymarket.** It asks what edge that model
would have had, and how long the edge lasted.

- **It will show** whether any candidate is skilled out of sample, on the best book data we have. The hypothetical
  adds a decay curve: edge per race over time, with a half-life.
- **It won't show** real fills: that is rule 5, and only live trading answers it.
- **It won't show** the hypothetical against a real book. Before 20 Nov 2025 there are only trades, no books, so it
  is an upper bound. Today's model was also designed after those seasons were seen, and a real early trader would
  have moved the prices.

Also on the list: F1 skilled-taker coverage on Polymarket, where trades carry wallets.
