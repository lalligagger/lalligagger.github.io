# DITL, 15 to 25 Oct 2026, Part 2 (draft): first real bets, closing-line value, and NASCAR

**Short summary.** Part 1 ended with nothing proven. Part 2 is where the work meets real books and an honest yardstick. On
16 Oct we bet a sportsbook's US Grand Prix race-winner board on the calibrated F1 model (6 of its 7 lines, $9.80). We
bet six cycling races (28 bets, $169.20). We also measured, for the first time, whether our backtest trades beat the
market's closing price (closing-line value, CLV), and whether our model beats Kalshi on NASCAR. So far, with one
misclick and two off-model tickets taken out, three cycling stages settled for +$58.69 on $90.00, and the model's own
lines made +$32.69 of that on $80.00. Three stages say little either way. F1 calibration raises backtest profit in
every strategy-season but not on the three-weekend holdout. On NASCAR, Kalshi prices race wins more sharply than we do.
**This is a working draft:** from here it adds only the work that locks in our Austin strategy and bets, and it will
be finished after the race on Sun 25 Oct. The remaining cycling and Phoenix results are follow-ups for a later part.

*All times are UTC. Sportsbooks are named only by letter (A, B, C). Stakes on the ₮ (USDT) book are shown as $.
Nothing here is betting advice.*

## Where we started

Part 1 (13 to 14 Oct) left us with:

- **A calibration fix for the flat F1 model.** It was tested on two seasons of recorded prices but never used on a live
  board.
- **No closing-line value anywhere.** Our go/no-go goal for October (G1) is CLV above +2% over 100 live bets, and nothing
  measured it yet.
- **Evidence that our backtest's one-price fills flatter thin markets.** Probalytics' full-depth order books showed it.
- **A rule from 15 Oct:** every remaining F1 round is paper-traded on one strategy locked before the first session, and
  scored as bet.

## Stats

Every count below leaves out three tickets that were not the model's: one misclick (Eneco stage 3, Van Dijke, $2.00 at
67, which won $134.00) and two "vibed" Austin tickets the owner added on their own call (Russell $1.00, a second
Hamilton $1.00). A second misclick (Japan Cup, Garibbo over Prades $9.40, placed in place of Matthews over Vacek) is
counted as the intended bet. As placed, misclicks included, the settled total is +$190.69 on $92.00.

| | This part (to 17 Oct 14:00) |
|---|---|
| Bets placed | **34**: F1 6 (Sportsbook B); cycling 28 (Sportsbooks B and C) |
| Staked | **$179.00**: F1 $9.80; cycling $169.20 |
| Settled | 16 bets on $90.00: **5 won, 11 lost, net +$58.69**: Tour of Guangxi stage 5 +$47.19 (5 bets), stage 4 +$20.50 (6), Eneco stage 3 −$9.00 (5) |
| Model lines only | 15 settled bets on $80.00, **+$32.69** (stage 4's Magnier ticket was outside the model slips) |
| Every recommendation at its own size, placed or not | 48 settled lines on $100.50, **+$48.26**; 3 Guangxi stage 5 lines still unknown |
| Open | 18: Austin (6, settles 25 Oct), Japan Cup (7), Chrono des Nations (4) and a Guangxi overall-winner future (1), settling 18 Oct |
| Paper (picked, not placed) | Guangxi stage 4 analysis slips: 12 lines, $15.14, all lost |
| NASCAR Phoenix (18 Oct) | No bet yet: one paper candidate (Chevrolet manufacturer, $16 at +335), owner's call |
| Live CLV | *Pending*: the six Austin bets get a CLV against Polymarket's close after the race. Cycling has none: no exchange tape, no captured closing odds |

Stakes on the ₮ book (Eneco, Guangxi stage 5, Japan Cup, Chrono) and the $ book (Guangxi stage 4, Austin) are added as
dollars. Split by currency, the settled net is +$20.50 on $15.50 and +₮38.19 on ₮74.50.

## What ran

- **The Austin board (16 Oct, 04:00 to 04:40).** Sportsbook B's 22-line race-winner board came in nine days before the
  race. We priced it on production data with the walk-forward calibration power applied (a = 2.1). Once that setting
  reached production (04:36), production priced the board itself, and the two agreed within 3.2 points on every driver.
  The slip bets every line with at least 5% expected value against the calibrated model, at half Kelly on a $100 bank,
  rounded up to the nearest 10 cents with a $1 minimum.
- **CLV in every backtest (16 Oct, 05:06 to 05:45).** Every saved season sweep now records CLV per strategy. We ran the
  calibrated model off and on for 2025 and 2026 on Polymarket's recorded prices: 44,000 backtest bets, every one with a
  closing price.
- **Cycling (15 to 17 Oct).** Eneco Tour stage 3, Tour of Guangxi stages 4 and 5 and its overall winner, the Japan
  Cup and the Chrono des Nations, priced on our road model. An experimental stage-aware model was tested and kept off.
- **Fill fidelity (14 to 16 Oct).** The F1 backtest re-run with fills at the ask on Probalytics' order books, on both
  Polymarket and Kalshi.
- **NASCAR (16 Oct).** The same calibration fitted for NASCAR on 158 races, swept against Kalshi's recorded prices, and
  a Sportsbook A board for Sunday's Phoenix race priced against both.

## F1: the Austin slip

![Our prices on Sportsbook B's US Grand Prix board](img/austin-hud.png)

| Bet | Odds (decimal) | Stake | Calibrated model | Book |
|---|---|---|---|---|
| Leclerc to win | 13 | $3.50 | 14.1% | 7.7% |
| Russell to win | 11 | *not placed* (rec $2.60) | 13.8% | 9.1% |
| Hamilton to win | 13 | $2.10 | 11.5% | 7.7% |
| Hadjar to win | 21 | $1.20 | 7.0% | 4.8% |
| Piastri to win | 19 | $1.00 (not yet on the book's history) | 5.9% | 5.3% |
| Gasly to win | 501 | $1.00 | 0.3% | 0.2% |
| Lindblad to win | 501 | $1.00 | 0.3% | 0.2% |

We passed on Verstappen and Antonelli. The book (44% and 29%) and Polymarket (44% and 25%) agree, and our model is far
lower (23% and 15%). That gap comes from how the model ranks drivers, which calibration doesn't touch. We also passed
on Norris, whose 3% edge is under the 5% bar. Expected profit at our prices was +$7.38 on the full $12.40 slip. The Russell line was not placed at its size: the
only Russell ticket is a $1.00 the owner added on their own call, left out above along with a second $1.00 Hamilton
ticket.

## Closing-line value: does the calibrated model beat the close?

**How CLV is measured.** Entry is the price we paid, as the YES price, with costs and fees. Close is the exchange's price
at the start of the race that decides the market (qualifying for pole). Polymarket uses the last trade within 6 hours,
and Kalshi the book mid within 10 minutes. CLV in points is close minus entry for a YES position, the other way round
for NO. CLV in percent divides that by the price paid. Means are per bet, not weighted by stake. Points is the steadier
read, because percent over-weights cheap contracts.

![CLV by strategy, calibration off and on](img/f1-clv-by-strategy.png)

The two pre-race strategies, on Polymarket's tape. CLV is shown as mean points · mean % · share of bets beating the
close · bets.

| | early, off | early, on | hold, off | hold, on |
|---|---|---|---|---|
| 2025 CLV | +0.0118 · +3.2% · 46% · 480 | +0.0160 · +4.3% · 47% · 428 | +0.0139 · +5.3% · 46% · 211 | +0.0211 · +8.4% · 46% · 192 |
| 2025 P&L | −$322.85 | −$238.75 | −$737.48 | −$496.67 |
| 2026 CLV | +0.0060 · −0.4% · 43% · 475 | +0.0067 · +0.6% · 45% · 458 | +0.0089 · +1.2% · 43% · 228 | +0.0063 · +2.8% · 42% · 206 |
| 2026 P&L | +$2,008.12 | +$2,915.66 | +$553.61 | +$1,187.77 |
| Holdout CLV (2025 Las Vegas, Qatar, Abu Dhabi) | −0.0054 · −15.4% · 48% · 33 | +0.0115 · +3.6% · 55% · 29 | +0.0050 · −16.5% · 44% · 18 | +0.0339 · +12.4% · 50% · 16 |
| Holdout P&L | −$126.50 | −$169.26 | −$239.61 | −$303.95 |

**Log loss against the market** (race-win prices, lower is better), from the calibration study's own runs:

| Season | Model, off | Model, on | Polymarket |
|---|---|---|---|
| 2025 | 0.1694 | 0.1627 | 0.1439 |
| 2026 | 0.2244 | 0.2194 | 0.2222 |

What this says:
- **Profit.** In-sample, calibration raises backtest profit for every strategy in both seasons: 18 of 18
  strategy-seasons.
- **CLV.** It moves less than profit: up for every 2025 taker strategy, mixed in 2026.
- **Share beating the close.** Fewer than half of any taker strategy's bets beat the close (42 to 51%). The positive
  means rest on a minority of larger moves.
- **Holdout.** On the three weekends the tuning never saw, calibration loses more money but has the better CLV. At 16 to
  33 bets per strategy, that is noise either way.
- **Log loss.** The model gets closer to Polymarket with calibration but stays behind it in 2025; in 2026 it edges
  ahead.

## Fill fidelity: what survives real order books

Backtest profit for the `update` taker strategy on F1 2026, in-sample, on Probalytics' order books:

| | Polymarket | Kalshi |
|---|---|---|
| Our recorded prices, one price per order | +$1,096 | +$1,096 |
| Probalytics' order books, fills at the ask | +$1,415 | +$25 |
| … plus a sale floor (never sell below our fair price minus the edge) | +$2,031 | +$1,470 |
| … walking the book instead, with the floor | +$1,918 | +$1,607 |

At the ask, Polymarket's in-sample edge survives and Kalshi's almost disappears. Most of Kalshi's loss was the strategy
selling into wide bids whenever our fair price dipped: a sale floor alone brings it back. Walking the book adds about
nothing once the floor is on. All of this is in-sample and averaged over three seeds. On three held-out 2025 weekends the
floor made things worse on Polymarket (−$37 to −$72 at the ask), so it is not adopted: it stays an option, off by
default, and our live signals don't use it. Kalshi has no held-out test yet.

The verdict: the new pipeline (Probalytics' tape, fills at the ask) is more faithful because it removes the optimism
of filling at the mid price, but none of the strategy changes we tried (spread limit, calibration, book walk, sale
floor) shows an edge out of sample. The full report is on [its own page](ditl-2026-10-16-part-2-fill-fidelity.html). On three held-out 2025 weekends, fills at the ask
showed no edge on either tape.

## Cycling: three stages settled, and a model gap

![Eneco Tour stage 3 slip and result](img/eneco-stage3-slip.png)

- **Eneco Tour stage 3** (Sportsbook C, 5 model bets, $9.00): all five lost, −$9.00. Van Dijke won at 67 to 1, and our
  model had him at 1%; the one ticket on him was a misclick and is left out. The two sprinter bets show a known gap: the
  road model barely separates hilly stages from flat ones.
- **Tour of Guangxi stage 4** (Sportsbook B, 6 tickets, $15.50): +$20.50. Magnier won at 3.60 (+$26.00), a ticket the
  owner placed outside both model slips; the five model-slip tickets lost (−$5.50).
- **Tour of Guangxi stage 5** (Sportsbook C, 5 head-to-head matchups, $65.50): 4 won, 1 lost, **+$47.19**. Four came
  from the baseline road model's slip, one (Blikra over Taminiaux) from a second recommendation table the same hour.
  The experimental stage-aware model backed only one of the five.
- **Open** (settle 18 Oct): the Japan Cup (7 bets, $41.70), the Chrono des Nations time trial (4 matchups, $32.50) and
  a Guangxi overall-winner future ($5.00 at 21).
- **CLV is not available for cycling:** there is no exchange and no captured closing odds. We report log loss instead.
  A stage-aware model beat our baseline on 2026 stage races, but its prices run too close to 50%. The one-day version was
  no better, so neither is in use.

![Cycling stage races: log loss before and after (not CLV)](img/cycling-logloss.png)

## NASCAR: Kalshi is sharper than our model

The same walk-forward calibration, fitted on 158 NASCAR races 2023 to 2026, comes out at a power of 1.2. Swept on
Kalshi's recorded prices for 2026 (34 races):

| | Calibration off | Calibration on | Kalshi |
|---|---|---|---|
| Race-win log loss | 0.2358 | 0.2370 | **0.2089** |
| Race-win Brier | 0.0590 | 0.0590 | **0.0551** |
| Race-win calibration error (ECE) | 0.0212 | **0.0113** | 0.0178 |
| `update` taker CLV (points · % · positive · bets) | −0.0475 · −5.72% · 35% · 689 | −0.0467 · −5.72% · 34% · 676 | |

Calibration fixes the shape of the prices but not their accuracy, and every taker strategy has negative CLV.
Polymarket's NASCAR books are too thin to trade or score.

![NASCAR CLV by strategy](img/nascar-clv-by-strategy.png)

Against Sportsbook A's board for Sunday's race at Phoenix (18 Oct, 19:00 UTC), one pick survives requiring the book to
sit below both our model and Kalshi: Chevrolet as winning manufacturer at +335, $16 on paper. It is fragile, because
Kalshi's no-bid drivers decide it. The whole NASCAR thread is on [its own page](ditl-2026-10-16-part-2-nascar-chat.html).

![Our prices and Kalshi's on Sportsbook A's Phoenix board](img/nascar-hud.png)

Why so few picks: the model is much flatter than both the book and Kalshi on the favourites, so the driver market has
none.

| Driver | Odds | Book | Model, raw → calibrated | Kalshi |
|---|---|---|---|---|
| Ryan Blaney | +350 | 22.2% | 9.3% → 10.9% | 20.5% |
| Christopher Bell | +350 | 22.2% | 7.8% → 8.9% | 18.5% |
| Denny Hamlin | +503 | 16.6% | 8.9% → 10.4% | 12.0% |
| Joey Logano | +570 | 14.9% | 4.5% → 4.5% | 10.5% |
| Kyle Larson | +660 | 13.2% | 5.6% → 5.9% | 9.5% |

The Chevrolet pick clears the bar against Kalshi only because 12 Chevrolet drivers with no bid count at half their 1¢
ask, about 6 points. Without them Kalshi has Chevrolet at 19.5%, below the book's 23.0%. A model-only set would add
RFK Racing at +2500 ($21), resting on the model alone. Locking nothing is also defensible.

## What's next

- **Settle Austin** (25 Oct) and add the six bets' live CLV against Polymarket's close and the winner's log loss at
  the calibrated and uncalibrated price. The Japan Cup, the Chrono, the Guangxi overall winner and Phoenix settle outside this part (follow-up below).
- **Lock** the round 18 paper strategy before US GP practice one (Fri 23 Oct, 17:30 UTC), and score it as bet.
- **Fix two open problems** before any 2026 number is final. A Lab diagnostic job fails on the Austin event. And the
  2026 sweep profits don't match the calibration study's own (2025 matches to the cent).
- **Decide the open rules:** a minimum bid before an exchange price counts on long shots, a bank-relative minimum stake,
  and whether calibration becomes the F1 default.

## Follow-up: unsettled bets and open picks

| Item | Bets | Staked | Settles | Where it's reported |
|---|---|---|---|---|
| F1 Austin race winner (Sportsbook B) | 6 | $9.80 | Sun 25 Oct | This page: result, live CLV, winner's log loss |
| Japan Cup | 7 | $41.70 | 18 Oct | A later part |
| Chrono des Nations | 4 | $32.50 | 18 Oct | A later part |
| Tour of Guangxi overall winner | 1 | $5.00 | 18 Oct | A later part |
| NASCAR Phoenix (paper) | 0 | $16 on paper | Sun 18 Oct | A later part (lock or pass before the green flag) |
| F1 round 18 paper strategy | | | Sun 25 Oct | This page: lock before practice one, Fri 23 Oct 17:30 UTC |

## What this does and doesn't show

- **It shows** our first real bets placed on a calibrated model, and the first CLV measurements on our backtests.
- **It shows** calibration helps F1 backtest profit in-sample, and that Kalshi out-prices our NASCAR model.
- **It doesn't show** that any strategy makes money. Without the misclick, the settled profit is +$58.69 over three
  stages, most of it from one stage's matchups. Every 2026 backtest number is in-sample, and the holdout is three weekends.
- **It doesn't show** live CLV yet. That arrives with Austin, and it is what the +2%-over-100-bets goal is measured on.
