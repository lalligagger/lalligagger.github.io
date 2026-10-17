# DITL Part 2 (draft): fill fidelity, our tape against Probalytics' order books

*The full report behind the fill-fidelity section of [DITL Part 2](ditl-2026-10-16-part-2-draft.html#fill-fidelity-what-survives-real-order-books). Original title: "Old vs new: our tape and one-price fills against Probalytics' books (F1 2026)".*

*Fill-fidelity thread, 14 to 16 October 2026. All runs on staging (`racinglines_staging_demo`). Golden profile A
(min edge 0.10), mean of 3 seeds unless stated. Every 2026 number is in-sample: profile A was tuned on 2026.*

The owner's plan (14 Oct, 22:03Z) had three steps: measure how wrong our fill prices are against Probalytics' order
books, iterate the fill logic on their data, and then compare the old pipeline (our 5-minute tape, one price per
order) with the new one (Probalytics' tape, fills at the ask, then walked through the book). This report covers that
comparison. It also covers the two changes tested alongside it, a 10¢ spread limit and the C48 walk-forward
calibration, and adds a 2025 holdout (Las Vegas, Qatar, Abu Dhabi) as the only out-of-sample check.

## Top line

| | Polymarket (R1–17) | Kalshi (R5–17) |
|---|---|---|
| **Old: our tape, one price** | **+1,096** | **+1,096** |
| New tape, Probalytics at mid | +3,014 | +25 |
| **New tape, Probalytics at the ask** | **+1,415** | **+25** |
| … + spread limit 10¢ | +1,595 | +151 |
| … + calibration (`calib_power = walk_forward`) | +1,670 | +207 |
| … book walk, no limit price (bug, before #219) | +629 | −2,026 |
| … book walk + sale floor `edge` | +1,918 | +1,607 |
| … touch + sale floor `edge` (no walk) | +2,031 | +1,470 |
| *Old tape + spread limit* | (not run) | +1,350 |
| *Old tape + calibration* | +1,737 | +1,430 |
| **2025 holdout (out of sample), our tape** | **−71** | n/a |
| **2025 holdout, Probalytics at the ask** | **−37** | n/a |

These are taker `update` P&L in $, from a $1k bankroll with a $50 maximum stake.

**Verdict.**
- **Polymarket:** the edge survives realistic fills in-sample (+1,415 at the ask).
- **Kalshi:** the edge almost disappears at the ask (+1,096 to +25). Our 5-minute snapshots flattered it.
- **Calibration and the spread limit** both help on both tapes.
- **Out of sample** (three 2025 weekends), profile A shows no edge on either tape.
- **The sale floor** (no taker sale more than min edge under fair) is the largest single in-sample gain: +616 on Polymarket and +1,445 on Kalshi. Walking the book adds about nothing on top (section 4). On the 2025 holdout (Polymarket only), the floor is slightly worse (−$19 / −$35), so it stays off by default.

## 1. How wrong were our fill prices?

The probe `scripts/probes/backtest_fill_error.py` (#202) priced each $50 taker order at every 2026 stage cutoff two
ways: our one-price fill, and a walk through Probalytics' book at that moment. Median error by market kind, all
priced markets, run `backtest-fill-error/20261014T221720Z`:

| Kind | Polymarket | Kalshi |
|---|---|---|
| race_win | −0.6% | ~0% |
| pole | −6% | ~0% |
| podium | −18% | ~0% |
| constructor_top | −47% | ~0% |
| race_h2h | −42% | ~0% |

- **Polymarket:** we filled at the 5-minute price history, close to the mid. Thin markets sit far from their book: a head-to-head at ~0.50 on our tape sat at about 0.10 / 0.89 in the book.
- **Kalshi:** we already bought at our snapshot's ask, so the price error is near zero. The P&L gap there comes from snapshot timing, not the fill rule (section 2).

## 2. The tape: ours vs Probalytics (same strategy, same fills)

| Strategy | PM ours | PM Probalytics ask | Kalshi ours | Kalshi Probalytics ask |
|---|---|---|---|---|
| early | +2,437 | +1,573 | +1,728 | +1,369 |
| hold | +1,640 | +1,304 | +1,835 | +1,760 |
| last | +709 | +1,712 | +871 | +532 |
| **update** | **+1,096** | **+1,415** | **+1,096** | **+25** |

- **Polymarket:** the Probalytics tape is better for `update` and `last` and worse for `early` and `hold`. Mid fills (+3,014) are an upper bound only; every real taker pays the ask.
- **Kalshi:** `update` collapses on Probalytics' tape. It rebalances at every stage, and their exact-second books show the edge we saw at our 5-minute snapshot was often gone at the cutoff. `hold`, which trades once, barely moves (+1,835 to +1,760).
- **Maker P&L is identical** on both tapes for every job. The maker replays our own tape whatever `tape_source` says, so this report says nothing about maker fills.

## 3. Spread limit and calibration (on both tapes)

| `update` | PM ours | PM Probalytics | Kalshi ours | Kalshi Probalytics |
|---|---|---|---|---|
| base | +1,096 | +1,415 | +1,096 | +25 |
| spread ≤ 10¢ | (not run) | +1,595 | +1,350 | +151 |
| calibration | +1,737 | +1,670 | +1,430 | +207 |

- **The 10¢ spread limit** adds $125 to $255 wherever it ran. Skipping books with no real market is worth it.
- **Calibration** gives the largest in-sample gain on both tapes. It also lifts the Polymarket maker from +586 to +1,385 (unseeded job). But the holdout (section 5) goes the other way.

**Walk-forward (one seed, Probalytics at the ask).** Choosing min edge and spread limit as you go, using only weekends already raced:

| | Walk-forward | Best in hindsight |
|---|---|---|
| Polymarket | +960 | +1,684 (min edge 0.10 + spread limit) |
| Kalshi | +122 | +379 (min edge 0.15 + spread limit) |

- Choosing as you go costs about half the hindsight best.
- Min edge 0.05 loses on Kalshi (−516 / −560).
- No setting is clearly better than profile A beyond the noise floor of about $400 a season.

## 4. The book walk

| | PM `update` | PM `hold` | Kalshi `update` | Kalshi `hold` |
|---|---|---|---|---|
| at the ask | +1,415 | +1,304 | +25 | +1,760 |
| walk, no limit (old #205 code) | +629 | +505 | −2,026 | +845 |
| walk with limit (#219) | +1,918 | +1,018 | +1,607 | +1,508 |

**The first walk had a bug.** An order was sized at the touch, then walked through every level until it filled. On a thin Kalshi book (a few contracts at the touch, then 0.99 asks) most of a $50 order filled at negative edge, and `update` sold down into 0.01 bids.

#219 made the walk a limit order:
- a buy takes only levels where fair − price − fee ≥ min edge;
- a sale takes only levels where price − fee ≥ fair − min edge;
- the rest of the order is not traded.

**`hold` never sells, so it shows the depth cost alone:** walking costs a little against the touch (Kalshi +1,760 to +1,508, Polymarket +1,304 to +1,018).

**The sale floor, separated (16 Oct, `golden-probalytics-sellfloor-2026`).** #219's sale limit was ad hoc, so #225 made it an explicit setting, `sell_floor`, for both fill modes (`none` / `fair` / `edge`, where `edge` = no sale below fair − min edge). Each change then ran on its own:

| `update`, mean of 3 seeds | Polymarket | Kalshi |
|---|---|---|
| touch, no floor | +1,415 | +25 |
| **touch + floor `edge`** | **+2,031** | **+1,470** |
| book, no floor (sells the touch level only) | +1,795 | +806 |
| book + floor `edge` (#219) | +1,918 | +1,607 |

- **The exit rule is the big effect:** the floor alone adds +616 on Polymarket and +1,445 on Kalshi. Without it, `update` sells into wide bids whenever fair dips at a stage.
- **Depth, with the floor on both sides,** is about a wash: −113 on Polymarket and +137 on Kalshi, within the seed noise.
- **Other strategies:**
  - `hold` and `last` are unchanged by the floor (Polymarket +1,304 / +1,712, Kalshi +1,760 / +532).
  - `early` rises a little: Polymarket +1,573 to +1,642, Kalshi +1,369 to +1,602.
- **This is in-sample, and the holdout didn't confirm it** (section 5): on 2025 Polymarket, the floor made `update` $19 to $35 worse.

## 5. 2025 holdout (out of sample)

Las Vegas, Qatar and Abu Dhabi 2025 (rounds 22–24), Polymarket only. Probalytics' stage files for these weekends were pulled on 15 Oct. The first run had none, so every Probalytics job traded nothing.

| | `update` | `hold` | Maker | Trades |
|---|---|---|---|---|
| Our tape | −71 | −171 | −228 | 27 |
| Probalytics at the ask | −37 | −187 | −228 | 27 |
| Probalytics + calibration | −95 | −184 | −516 | 23 |
| Our tape + sale floor `edge` | −90 | −171 | −228 | 21–22 |
| Probalytics ask + sale floor `edge` | −72 | −188 | −228 | 21–22 |

The floor rows are from the 17 Oct run `holdout-2025-sellfloor` (#228). Trade counts are per seed, read from the run's results.json. The floor blocks the sales that cut losing positions, so there are fewer trades.

| `update` by weekend | Ours | Probalytics ask | + calibration |
|---|---|---|---|
| Las Vegas (R22) | +5 | +12 | −43 |
| Qatar (R23) | +73 | +49 | −16 |
| Abu Dhabi (R24) | −149 | −99 | −36 |

- **No edge out of sample on either tape.** With three weekends this is a direction, not a verdict.
- **The two tapes agree here** (the same 27 trades, within $34). The 2026 tape gap is mostly a Kalshi effect, and this holdout has no Kalshi.
- **Calibration is worse here on both taker and maker.** Its in-sample gain should not be treated as settled.
- **The sale floor doesn't help here either:**
  - `update` is $19 worse on our tape (−71 to −90) and $35 worse on Probalytics (−37 to −72).
  - `hold` is unchanged, as it should be.
  - The amounts are small, but the sign is the wrong way.
  - Most of the floor's in-sample gain was on Kalshi (+1,445), and this holdout has no Kalshi, so the test can't confirm or rule that part out.

## What this does and doesn't show

- **In-sample.** Profile A, the spread limit and the calibration table were all chosen after seeing 2026. Only the 2025 holdout is clean, and it is three weekends of Polymarket.
- **Session overruns can leak results.** A stage cutoff is the scheduled session end + 30 minutes (`racinglines/core/stages.py:43`), so a red-flagged session that overruns can leak its result into "after Quali" trades. This is the P0 in `docs/todo.md` (#208). It applies equally to every row here, so it shifts levels more than comparisons.
- **The maker is not tested.** It replays our tape on every run.
- **The sale floor is not confirmed.**
  - It wins in-sample.
  - It loses slightly on the 2025 holdout, which is Polymarket only.
  - Nothing out of sample tests it on Kalshi.
- **Fill-error probe not re-run.** The `backtest_fill_error.py` probe has not been re-run with the walk's new limit.
- **Few seeds.** Three seeds agree within about $100 to $300 per row. The walk-forward is one seed.

## What's next

1. **Forward test from the US GP** (round 18, FP1 23 Oct 17:30Z): one strategy locked in `paper/locks/2026-18.toml` before FP1 and scored as bet. This is the only test that answers the in-sample question.
2. **The sale floor stays off by default.**
   - The holdout didn't confirm it: Polymarket was slightly worse, and Kalshi wasn't tested.
   - Its decision-log row (#237) should record "not adopted".
   - The next real test is the forward test: score the locked strategy both with and without the floor. That needs only a replay, so it doesn't change the lock.
   - `fair` (no sale below fair) has not been run.
3. **Calibration** stays off by default until the forward test. Promotion is an owner decision.
4. **Fix the overrun P0**, then re-run the golden queues once.
5. **Re-run the fill-error probe** with the limit walk.
