# As run: the overnight batch (20 Oct 2026)

*This is the runbook as written and run, with machine names, paths, database names and accounts redacted (shown as `<...>`). The commands are otherwise unchanged. The outcome of each run is recorded at the end, and the job script it started is in the appendix.*

## Part 1: the state batch


**For:** the same Haiku-class agent in VS Code on the owner's Mac. It works the same way as before: run each command,
compare its output with the **Check**, then go on or stop. Don't change any command or file. If a check fails, stop
and give the owner the step number and the last 20 lines of output. **Read this file fresh from disk.** It is new, so
don't go by any earlier copy.

## Already done (don't redo)

- Rule 1 Kalshi run: `kalshi-rule1-20261020T023605Z`.
- Step 3 Polymarket run: `pm-step3-20261020T035547Z`. Its trade files and census are on the VM, and this batch reads
  them there.
- Claude has scored both.

## What this batch does

It is one unit, `rl-state-batch`, read-only on staging's mirror of prod. It writes no database rows. The parts run in
this order:

1. **A and F, the model on every market.** The un-tuned model, run as a season sweep that keeps every market it priced
   at every stage, traded or not. This covers F1 on Polymarket (our tape from 2025, which is "ready at launch", and
   Probalytics' books), F1 on Kalshi (ours, and Probalytics' books from round 5), NASCAR 2025 and 2026, and MotoGP
   2026. Ten runs.
2. **G, the same model calibrated.** As part 1, with the win-price calibration fitted on earlier races only
   (`calib_power = walk_forward`), for F1 on both venues and NASCAR on Kalshi. Six runs.
3. **C, Kalshi on Probalytics' books.** The 20 saved Kalshi F1 settings, 2026 rounds 5 to 16. One run.
4. **E, statistics.** Markouts of our trades, every F1 fill by price and outcome, and fills by stage of market life.
   This takes minutes.
5. **B, rule 1 on Polymarket.** Every saved Polymarket F1 setting, at the touch on Probalytics' books, 2025 rounds 22
   to 24 and 2026 rounds 3 to 16. The two seasons run side by side. This is the long part: roughly 1 to 3 hours, a
   guess.

All of it is one start command (step 4), so it can run unattended overnight. Parts 1 to 4 are a guess of 1 to 2
hours more; the whole batch, roughly 2 to 5 hours.

## Files on the Mac

| Path | What |
|---|---|
| `<mac-folder>/state-batch/state_job.sh` | The VM job, written by Claude |
| `<mac-folder>/state-batch/stats.py` | Part E, tested on sample files |
| `<mac-folder>/state-batch/calib_dump.py` | Parts A and F |

## Step 1: confirm the files

```
# LOCAL (Mac)
ls -la <mac-folder>/state-batch/ && bash -n <mac-folder>/state-batch/state_job.sh && echo script-ok
```

**Check:** the three files are listed, and the last line is `script-ok`.

## Step 2: check the VM is free, and what tape it has

Use the same one-time `safe.directory` override as before if git refuses.

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "cd <app> && /usr/bin/git log --oneline -1 && systemctl list-units 'rl-*' --state=running,activating --no-legend | grep -vc jev-monitor; ls data/probalytics/stages | grep -c KALSHI; ls -d data/runs/kalshi-rule1-* data/runs/pm-step3-*"
```

**Check:**
- The first line starts with `34b18f5` or a later staging commit.
- The second line is `0`. If not, stop: another queue is running. The one exception is `rl-jev-monitor` (the
  Jev review timer, every 15 minutes): it doesn't count, so if it is the only unit listed, go on.
- The third line is a number: the Kalshi stage files. If it's `0`, note it and go on. Part C will then have no books.
- Both run folders are listed.

## Step 3: owner go

Ask the owner to post "state batch starting" in the Claude thread "Kalshi Market Efficiency Report", then wait for
their OK. That lets the coordinator hold the staging merges.

## Step 4: copy the files to the VM and start the batch

```
# LOCAL (Mac)
gcloud compute scp <mac-folder>/state-batch/state_job.sh <mac-folder>/state-batch/stats.py <mac-folder>/state-batch/calib_dump.py <vm>:<tmp>/ --project <project> --zone <zone> --tunnel-through-iap && gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "sudo install -d -m 755 <tmp>/state-batch && sudo install -m 755 <tmp>/state_job.sh <tmp>/state-batch/state_job.sh && sudo install -m 644 <tmp>/stats.py <tmp>/calib_dump.py <tmp>/state-batch/ && sudo systemctl reset-failed rl-state-batch 2>/dev/null; sudo systemd-run --unit=rl-state-batch --uid=<app-user> <tmp>/state-batch/state_job.sh && sleep 60 && sudo journalctl -u rl-state-batch --no-pager | grep -E 'state batch from|grids:|dropped|start calib'"
```

**Check:** you should see:
- `state batch from 34b18f5 on <staging-db> (read-only)`, or a later commit;
- `grids: N polymarket, 20 kalshi, D dropped`. Note N and D. N should be around 100;
- `start calib f1-pm-ours-2025`.

If you see `STOP: could not build the grids`, stop.

## Step 5: wait for it to finish

Check about every 10 minutes:

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "systemctl is-active rl-state-batch; d=\$(ls -1dt <app>/data/runs/state-batch-* | head -1); tail -n 5 \$d/run.log"
```

**Check:** it's done when the first line is `inactive` (or `failed`) and run.log ends with
`all done in M min, failures: F`. If a `FAILED` line appears, note it and keep waiting: the parts after it still run.

## Step 6: owner note

Ask the owner to post this in the Claude thread: "state batch ended: F failures".

## Step 7: copy the outputs to the Mac

```
# LOCAL (Mac)
mkdir -p <mac-folder>/state-batch/raw && cd <mac-folder>/state-batch/raw && d=$(gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "ls -1dt <app>/data/runs/state-batch-* | head -1" | tr -d '\r') && gcloud compute scp --recurse --project <project> --zone <zone> --tunnel-through-iap "<vm>:$d" . && find . -name '*.parquet' | wc -l && ls */*.csv
```

**Check:** the parquet count is printed, and `markouts.csv`, `fills-bins.csv`, `fills-life.csv` and `fills-sample.csv`
are listed.

## Step 8: report back

Give the owner three things to paste into the Claude thread:
1. The whole `raw/state-batch-*/run.log`.
2. The last 3 lines of `raw/state-batch-*/stats.log`.
3. The path `<mac-folder>/state-batch/raw/`. Claude reads and scores it from the Mac.

Don't score or interpret anything.

## Footer (fill in)

- What ran: the unit name, the start and end time, N, D and F.
- How it was verified: the step 4 and step 5 check lines.
- Paths: the VM output folder name, the Mac `raw/` folder.
- What was not checked: anything skipped.


## Part 2: rule 1 on Polymarket, one process per setting


**For:** the same agent. It works the same way: run each command, compare with the **Check**, go on or stop. Don't
change any command or file. If a check fails, stop and give the owner the step number and the last 20 lines of output.
Read this file fresh from disk.

## Why

In the first batch (`state-batch-20261020T052307Z`), part B ran every setting in one process per season. A setting whose
model pricings weren't already stored for every round tried to save them, the read-only database refused
(`ReadOnlySqlTransaction`), and that stopped the whole season. As a result, 2025 got 42 of 106 settings and 2026 got 1.
This run starts one process per setting, so a refused setting drops only itself. A 2026 setting that fails on rounds 3
to 16 is retried once on rounds 3 to 15. Everything else in the first batch finished and is kept.

It is still read-only: it writes no database rows. It takes roughly 3 to 6 hours (a guess).

## Step 1: confirm the file

```
# LOCAL (Mac)
ls -la <mac-folder>/state-batch/pm_rule1_rerun.sh && bash -n <mac-folder>/state-batch/pm_rule1_rerun.sh && echo script-ok
```

**Check:** the file is listed, and the last line is `script-ok`.

## Step 2: check the VM is free

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "systemctl list-units 'rl-*' --state=running,activating --no-legend | grep -vc jev-monitor; ls <app>/data/runs/state-batch-20261020T052307Z/grids/pm-rule1.json"
```

**Check:** the first line is `0`, and the grid file is listed. If the count isn't `0`, stop: another queue is running.

## Step 3: owner go

Ask the owner to post "pm rule 1 rerun starting" in the Claude thread "Kalshi Market Efficiency Report", then wait for
their OK.

## Step 4: copy and start

```
# LOCAL (Mac)
gcloud compute scp <mac-folder>/state-batch/pm_rule1_rerun.sh <vm>:<tmp>/ --project <project> --zone <zone> --tunnel-through-iap && gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "sudo install -d -m 755 <tmp>/state-batch && sudo install -m 755 <tmp>/pm_rule1_rerun.sh <tmp>/state-batch/pm_rule1_rerun.sh && sudo systemctl reset-failed rl-pm-rule1 2>/dev/null; sudo systemd-run --unit=rl-pm-rule1 --uid=<app-user> <tmp>/state-batch/pm_rule1_rerun.sh && sleep 30 && sudo journalctl -u rl-pm-rule1 --no-pager -o cat | grep 'pm rule 1 rerun'"
```

**Check:** one line, `... pm rule 1 rerun from 34b18f5 ...: 106 settings, one process each -> data/runs/pm-rule1-<time>`.

## Step 5: wait for it to finish

Check about every 30 minutes. It is fine to leave it overnight.

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "systemctl is-active rl-pm-rule1; d=\$(ls -1dt <app>/data/runs/pm-rule1-* | head -1); tail -n 4 \$d/run.log"
```

**Check:** it's done when the first line is `inactive` (or `failed`) and run.log ends with
`all done in M min: K ok, F failed`. `FAILED <job>` lines along the way are expected, so note the count and keep
waiting.

## Step 6: owner note

Ask the owner to post this in the Claude thread: "pm rule 1 rerun ended: K ok, F failed".

## Step 7: copy the outputs to the Mac

```
# LOCAL (Mac)
mkdir -p <mac-folder>/state-batch/raw && cd <mac-folder>/state-batch/raw && d=$(gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "ls -1dt <app>/data/runs/pm-rule1-* | head -1" | tr -d '\r') && gcloud compute scp --recurse --project <project> --zone <zone> --tunnel-through-iap "<vm>:$d" . && find . -path './pm-rule1-*' -name '*.parquet' | wc -l && wc -l pm-rule1-*/status.csv
```

**Check:** a parquet count is printed, and `status.csv` has 213 lines (a header plus 2 × 106).

## Step 8: report back

Give the owner the folder path `<mac-folder>/state-batch/raw/pm-rule1-<time>/` and the last 3 lines of its
`run.log`. Claude reads the rest from the Mac. Don't score or interpret anything.


## As run (20 Oct 2026, UTC)

| Time (UTC) | Event |
|---|---|
| 05:23:07 | **Part 1** started on staging commit 34b18f5, read-only on the mirror. 0 saved settings dropped |
| 05:23 to 05:42 | 16 model-on-every-market runs (F1 both venues and tapes, NASCAR, MotoGP; un-tuned and calibrated), all done |
| 05:41:58 to 05:47:23 | Kalshi on Probalytics' books, 20 settings: 20 trade files |
| 05:47:56 | Statistics done: 20,594 trade markouts, 3,185,559 resolved fills binned |
| 05:52:21 | Polymarket rule 1, 2026: **failed** after 2 settings. A setting without stored stage pricings tried to save them, and the read-only guard refused |
| 06:01:40 | Polymarket rule 1, 2025: **failed** after 43 settings, same cause. Part 1 ended: 38 min, 2 failures |
| 06:41:40 | **Part 2** started: the same 106 Polymarket settings, one process per setting and season |
| 11:02:35 | Part 2 done in 260 min: **207 of 212 ok**. 5 failed, all 2025 settings refused for the same read-only reason. 53 of the 106 2026 settings ran on rounds 3 to 15 after failing on 3 to 16 (round 16 had no stored pricing for them) |

## Appendix: scripts as run

### Job script: state_job.sh

```bash
#!/usr/bin/env bash
# "State of racinglines" batch (20 Oct 2026): the historical tests still missing from the Kalshi response. Read-only on
# staging's mirror of prod (one queue; nothing writes to a database). Parts, in order:
#   A+F the un-tuned model on every market it prices, kept row by row (calib_dump.py): F1 on Polymarket (our tape, from
#      F1's first 2025 markets = "ready at launch"; and Probalytics' books), F1 on Kalshi (ours; Probalytics' books from
#      round 5), NASCAR 2025-26 and MotoGP 2026 on Kalshi
#   G  the same model calibrated (F1 and NASCAR, the two sports with a fitted table) (calib_power = walk_forward: the win-price power fitted on earlier races only), on our tape - does calibration close the gap between a fair price and a tradeable edge
#   C  Kalshi on Probalytics' books (sizes -> a size-weighted mid): the 20 saved Kalshi F1 settings, 2026 rounds 5-16
#   E  stats.py: markouts of our trades from the stage files; every F1 fill by price bin and outcome; fills by decile
#      of market life (reads the rule 1 and step 3 trade files already on the VM)
#   B  rule 1 on Polymarket: every saved Polymarket F1 sweep setting at the touch on Probalytics' books,
#      2025 rounds 22-24 and 2026 rounds 3-16 (the two seasons run side by side, 2 vCPU)
# Each sweep is one process per grid (entries share caches); a setting whose saved dict no longer parses is dropped and
# logged. A progress line every 5 minutes. Output: data/runs/state-batch-<UTC>/
set -uo pipefail
set -a; . <env-file>; set +a
export DATABASE_URL="${DATABASE_URL%/*}/<staging-db>"
export PYTHONUNBUFFERED=1 RACINGLINES_PROGRESS_SEC=300 PGOPTIONS="-c default_transaction_read_only=on"
cd <app>
R=.venv/bin/racinglines
PY=.venv/bin/python
S=<tmp>/state-batch
OUT=data/runs/state-batch-$(date -u +%Y%m%dT%H%M%SZ)
mkdir -p "$OUT/grids"
say() { echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) $*" | tee -a "$OUT/run.log"; }
T0=$SECONDS
( while sleep 300; do say "progress: $(( (SECONDS - T0) / 60 )) min elapsed; $(cat "$OUT"/*.log 2>/dev/null | grep -c '^grid ') grid entries started"; done ) &
BEAT=$!
trap 'kill $BEAT 2>/dev/null' EXIT
FAIL=0
sweep() {   # name year rounds grid
  local name=$1 y=$2 rounds=$3 g=$4 n
  n=$($PY -c "import json,sys; print(len(json.load(open(sys.argv[1]))))" "$g")
  say "start $name (year $y, rounds ${rounds:-all}, $n settings)"
  if $R f1 sweep --year "$y" ${rounds:+--rounds "$rounds"} --no-fetch --grid "$g" --trades-out "$OUT/$name" > "$OUT/$name.log" 2>&1; then
    say "done $name: $(ls "$OUT/$name"/*.parquet 2>/dev/null | wc -l) trade files ($(( (SECONDS - T0) / 60 )) min)"
  else
    say "FAILED $name: $(grep -m1 -oE 'ReadOnlySqlTransaction|Error: .{0,100}' "$OUT/$name.log") ($(grep -c '^grid ' "$OUT/$name.log") entries started)"
    FAIL=$((FAIL + 1))
  fi
}
say "state batch from $(git rev-parse --short HEAD) on ${DATABASE_URL##*/} (read-only) -> $OUT"
$PY - "$OUT/grids" <<'PY' || { say "STOP: could not build the grids"; exit 1; }
import json, os, sys
from sqlalchemy import create_engine, text
from racinglines.pipelines import sweep_settings as SS
out = sys.argv[1]
rows = create_engine(os.environ["DATABASE_URL"]).connect().execute(text(
    "SELECT DISTINCT ON (params->>'settings_key') params->>'settings_key', params->'settings', "
    "coalesce(params->'settings'->>'venue', 'polymarket') FROM model_runs WHERE kind = 'sweep' "
    "ORDER BY params->>'settings_key', id DESC")).fetchall()
pm, ka, bad = [], [], []
for key, st, venue in rows:
    st = dict(st or {})
    try:
        if venue == "kalshi":
            s = dict(st, tape_source="probalytics")
            SS.Settings.from_dict(s)
            ka.append({"job": f"k-{key}", "settings": s})
        elif venue == "polymarket":
            s = dict(st, venue="polymarket", tape_source="probalytics", taker_fill="touch")
            SS.Settings.from_dict(s)
            pm.append({"job": f"pm-{key}", "settings": s})
    except Exception as e:                                   # an old saved dict the schema no longer takes
        bad.append(f"{key}: {type(e).__name__}: {e}"[:200])
json.dump(pm, open(os.path.join(out, "pm-rule1.json"), "w"))
json.dump(ka, open(os.path.join(out, "kalshi-pb.json"), "w"))
open(os.path.join(out, "dropped.txt"), "w").write("\n".join(bad) + ("\n" if bad else ""))
print(f"grids: {len(pm)} polymarket, {len(ka)} kalshi, {len(bad)} dropped")
PY
say "$(cat "$OUT"/grids/dropped.txt | wc -l) saved settings dropped (see grids/dropped.txt)"
# A + F: the un-tuned model on every market it prices (calib.parquet: every tradeable market at every stage, traded
# or not) and its trades. A = the first two (ready at launch: our own Polymarket tape from F1's first 2025 markets).
calib() {   # name sport year rounds settings-json
  say "start calib $1 ($2 $3, rounds $4, settings $5)"
  if $PY "$S/calib_dump.py" "$2" "$3" "$4" "$5" "$OUT/calib-$1" > "$OUT/calib-$1.log" 2>&1; then
    say "done calib $1: $(tail -n 1 "$OUT/calib-$1.log") ($(( (SECONDS - T0) / 60 )) min)"
  else
    say "FAILED calib $1: $(grep -m1 -oE '[A-Za-z]+Error: .{0,100}' "$OUT/calib-$1.log")"; FAIL=$((FAIL + 1))
  fi
}
calib f1-pm-ours-2025 f1 2025 all '{}'
calib f1-pm-ours-2026 f1 2026 1-16 '{}'
calib f1-kalshi-ours-2025 f1 2025 all '{"venue": "kalshi"}'
calib f1-kalshi-ours-2026 f1 2026 1-16 '{"venue": "kalshi"}'
calib f1-pm-pb-2025 f1 2025 22-24 '{"tape_source": "probalytics", "taker_fill": "touch"}'
calib f1-pm-pb-2026 f1 2026 3-16 '{"tape_source": "probalytics", "taker_fill": "touch"}'
calib f1-kalshi-pb-2026 f1 2026 5-16 '{"venue": "kalshi", "tape_source": "probalytics"}'
calib nascar-kalshi-2025 nascar 2025 all '{"venue": "kalshi"}'
calib nascar-kalshi-2026 nascar 2026 all '{"venue": "kalshi"}'
calib motogp-kalshi-2026 motogp 2026 all '{"venue": "kalshi"}'
# G: the same model, calibrated on earlier races only
calib f1-pm-cal-2025 f1 2025 all '{"calib_power": "walk_forward"}'
calib f1-pm-cal-2026 f1 2026 1-16 '{"calib_power": "walk_forward"}'
calib f1-kalshi-cal-2025 f1 2025 all '{"venue": "kalshi", "calib_power": "walk_forward"}'
calib f1-kalshi-cal-2026 f1 2026 1-16 '{"venue": "kalshi", "calib_power": "walk_forward"}'
calib nascar-kalshi-cal-2025 nascar 2025 all '{"venue": "kalshi", "calib_power": "walk_forward"}'
calib nascar-kalshi-cal-2026 nascar 2026 all '{"venue": "kalshi", "calib_power": "walk_forward"}'
# C: Kalshi on Probalytics' books
sweep kalshi-pb-2026 2026 "5-16" "$OUT/grids/kalshi-pb.json"
# E: markouts, crowd calibration, market-life deciles
say "stats: start"
if $PY "$S/stats.py" "$OUT" > "$OUT/stats.log" 2>&1; then say "stats: done ($(tail -n 1 "$OUT/stats.log"))"; else say "stats FAILED: $(tail -n 1 "$OUT/stats.log")"; FAIL=$((FAIL + 1)); fi
# B: rule 1 on Polymarket, the two seasons side by side
sweep pm-rule1-2025 2025 "22-24" "$OUT/grids/pm-rule1.json" &
P1=$!
sweep pm-rule1-2026 2026 "3-16" "$OUT/grids/pm-rule1.json" &
P2=$!
wait $P1; wait $P2
FAIL=$((FAIL + $(grep -c '^.* FAILED pm-rule1' "$OUT/run.log")))
say "all done in $(( (SECONDS - T0) / 60 )) min, failures: $FAIL"
```

### Model on every market: calib_dump.py

```python
"""State-of-racinglines batch, part F (read-only): one season sweep, run as `racinglines strategy sweep` runs it
(pipelines/season_sweep.run), that also keeps every (fair, price, outcome) row the sweep scores for calibration -
every tradeable market at every stage, traded or not - instead of only the summary tables. It wraps
core/calibration.table to keep its input; nothing in the sweep changes.

Usage: calib_dump.py <sport> <year> <rounds|all> '<settings json>' <out dir>
Writes <out dir>/calib.parquet (event_key, stage, kind, model, market, y, ...), <out dir>/trades.parquet (--trades-out)
and prints the sweep's totals.
"""
import json
import os
import sys

import pandas as pd

from racinglines.core import calibration as CAL
from racinglines.db.config import get_engine
from racinglines.pipelines import season_sweep as S
from racinglines.pipelines import weekend_sweep as SW

sport, year, rounds, settings, out = sys.argv[1], int(sys.argv[2]), sys.argv[3], json.loads(sys.argv[4]), sys.argv[5]
rounds = None if rounds == "all" else [i for a, b in [(p.split("-") + p.split("-"))[:2] for p in rounds.split(",")]
                                       for i in range(int(a), int(b) + 1)]
kept = []
_table = CAL.table


def table(rows, *a, **k):
    if not kept:                                   # the first call gets every row; later ones are the same rows
        kept.append(pd.DataFrame(rows).copy())
    return _table(rows, *a, **k)


CAL.table = table
cls = S.settings_class(sport)
st = cls.from_dict(settings) if hasattr(cls, "from_dict") else cls(**settings)
res = S.run(get_engine(None), None, sport, year, rounds, settings=st, echo=lambda m: print(m, flush=True))
os.makedirs(out, exist_ok=True)
if kept:
    kept[0].to_parquet(os.path.join(out, "calib.parquet"), index=False)
SW.write_trades(res, out)
for k, v in (res.get("totals") or {}).items():
    print(f"{k:7s} P&L {v.get('pnl', 0):+9.2f} on ${v.get('bought', 0):,.0f}", flush=True)
print(f"calib rows: {len(kept[0]) if kept else 0}", flush=True)
```

### Statistics: stats.py

```python
"""State-of-racinglines batch, part E (read-only): three tables from Probalytics' files and our own trade files.

  markouts.csv   every opening taker trade of the rule 1 run (Kalshi), the step 3 run (Polymarket) and this batch's
                 Kalshi-on-Probalytics run: the size-weighted mid at its stage, at the next stage and at the last stage
                 before the race, from Probalytics' stage files (data/probalytics/stages/<event>_<VENUE>.parquet)
  fills-bins.csv every F1 fill on both venues (F1 tag), de-duplicated by fill id: contracts and fills per venue, raw
                 outcome side, taker side, 5-cent price bin and the market's resolution (market_links) - the raw counts
                 for the paper's Fig 6, interpreted offline
  fills-life.csv the same fills by decile of the market's traded life (first to last fill in fills-census.csv), plus
                 our own trades by the same deciles - the paper's Fig 7
  fills-sample.csv 5 rows per venue, ids dropped, to check the columns' meaning before reading the bins
Usage: stats.py <batch output dir>   (finds the rule 1 and step 3 folders under data/runs/ itself)
"""
import glob
import os
import sys
from pathlib import Path

import numpy as np
import pandas as pd
import pyarrow.parquet as pq
from sqlalchemy import create_engine, text

from racinglines.markets import probalytics as P

OUT = Path(sys.argv[1])
BASE = Path(P.root())
TAKER = ["update", "hold", "last", "early"]


def latest(pattern):
    got = sorted(glob.glob(pattern))
    return got[-1] if got else None


def trade_files():
    out = []
    r1 = latest("data/runs/kalshi-rule1-*")
    if r1:
        out += [("rule1-kalshi-ours", f) for f in sorted(glob.glob(f"{r1}/f1-2026-*/*.parquet"))]
    s3 = latest("data/runs/pm-step3-*")
    if s3:
        out += [("step3-pm", f) for f in sorted(glob.glob(f"{s3}/f1-*/*.parquet"))]
    out += [("kalshi-pb", f) for f in sorted(glob.glob(str(OUT / "kalshi-pb-2026" / "*.parquet")))]
    return out


def stage_books(event_key, venue):
    f = BASE / "stages" / f"{event_key}_{venue.upper()}.parquet"
    if not f.exists():
        return None
    t = pq.read_table(f)
    if venue == "kalshi":
        import pyarrow.compute as pc
        t = t.filter(pc.equal(t["outcome_pid"], "yes"))
        key = t["market_platform_id"]
    else:
        key = t["outcome_pid"]
    bid, bsz = P.best_levels(t["bid_px"], t["bid_sz"], highest=True)
    ask, asz = P.best_levels(t["ask_px"], t["ask_sz"], highest=False)
    df = pd.DataFrame(dict(key=key.to_pandas().astype(str), cutoff=pd.to_datetime(t["cutoff"].to_pandas(), utc=True)
                           .dt.tz_localize(None), bid=bid, ask=ask, bsz=bsz, asz=asz))
    ok = (df["bsz"] + df["asz"]) > 0
    df["smid"] = np.where(ok & df["bid"].notna() & df["ask"].notna(),
                          (df["bid"] * df["asz"] + df["ask"] * df["bsz"]) / (df["bsz"] + df["asz"]), np.nan)
    return df.dropna(subset=["smid"]).sort_values(["key", "cutoff"])


def markouts():
    rows, books = [], {}
    for src, f in trade_files():
        t = pd.read_parquet(f)
        t = t[t["mode"].isin(TAKER) & (t["shares"] > 0)]
        setting = os.path.basename(os.path.dirname(f))
        for (ev, venue), g in t.groupby(["event_key", "venue"]):
            k = (ev, venue)
            if k not in books:
                books[k] = stage_books(ev, venue)
            b = books[k]
            if b is None or not len(b):
                continue
            by_key = {kk: kg for kk, kg in b.groupby("key")}
            for r in g.itertuples(index=False):
                kb = by_key.get(str(r.key))
                if kb is None:
                    continue
                ts = pd.Timestamp(r.t).tz_localize(None) if pd.Timestamp(r.t).tzinfo else pd.Timestamp(r.t)
                at = kb[kb["cutoff"] <= ts + pd.Timedelta(minutes=1)]
                if not len(at):
                    continue
                e = at.iloc[-1]
                nxt = kb[kb["cutoff"] > e["cutoff"]]
                last = kb.iloc[-1]
                d = 1.0 if r.side == "YES" else -1.0
                m0 = e["smid"]
                paid = (r.price - m0) if d > 0 else (r.price - (1 - m0))
                rows.append(dict(source=src, setting=setting, mode=r.mode, event_key=ev, venue=venue, key=r.key,
                                 kind=getattr(r, "kind", None), side=r.side, shares=r.shares, m0=m0, paid=paid,
                                 next_min=(nxt.iloc[0]["cutoff"] - e["cutoff"]).total_seconds() / 60 if len(nxt) else np.nan,
                                 mo_next=d * (nxt.iloc[0]["smid"] - m0) if len(nxt) else np.nan,
                                 mo_last=d * (last["smid"] - m0) if last["cutoff"] > e["cutoff"] else np.nan,
                                 last_min=(last["cutoff"] - e["cutoff"]).total_seconds() / 60,
                                 gap=d * (float(r.outcome) - m0) if pd.notna(r.outcome) else np.nan))
    df = pd.DataFrame(rows)
    df.to_csv(OUT / "markouts.csv", index=False)
    return len(df), (df["source"].value_counts().to_dict() if len(df) else {})


def links():
    q = text("SELECT ml.exchange, ml.token_id, ml.condition_id, ml.outcome, ml.resolved_yes, ml.prediction AS kind "
             "FROM market_links ml JOIN competitions c ON c.id = ml.competition_id "
             "WHERE c.code LIKE 'f1%' AND ml.exchange IN ('polymarket', 'kalshi')")
    return pd.read_sql(q, create_engine(os.environ["DATABASE_URL"]))


def fills():
    L = links()
    pm_tok = {r.token_id: r for r in L[L.exchange == "polymarket"].itertuples(index=False)}
    pm_cond = {r.condition_id: r for r in L[L.exchange == "polymarket"].itertuples(index=False)}
    ka = {r.token_id: r for r in L[L.exchange == "kalshi"].itertuples(index=False)}
    census = pd.read_csv(latest("data/runs/pm-step3-*") + "/fills-census.csv", dtype={"market_platform_id": str})
    life = {(v, m): (pd.Timestamp(a), pd.Timestamp(b)) for v, m, a, b in
            census[["venue", "market_platform_id", "first", "last"]].itertuples(index=False)}
    bins, lifes, samples = [], [], []
    months = pd.period_range("2022-11", pd.Timestamp.utcnow().strftime("%Y-%m"), freq="M")
    for venue in ("POLYMARKET", "KALSHI"):
        v = venue.lower()
        for m in months:
            f = BASE / "fills" / f"F1_{venue}_{m}.parquet"
            if not f.exists():
                continue
            cols = [c for c in ("fill_id", "market_platform_id", "outcome_pid", "outcome_name", "outcome_index",
                                "taker_side", "price", "normalized_price", "size", "timestamp")
                    if c in set(pq.read_schema(f).names)]
            df = pq.read_table(f, columns=cols).to_pandas()
            if "fill_id" in df:
                df = df.drop_duplicates("fill_id")
            if len(samples) < (5 if v == "polymarket" else 10):
                samples.append(df.drop(columns=[c for c in ("fill_id",) if c in df]).head(5).assign(venue=v))
            df["mpid"] = df["market_platform_id"].astype(str)
            if v == "polymarket":
                link = [pm_tok.get(str(o)) or pm_cond.get(c) for o, c in zip(df["outcome_pid"].astype(str), df["mpid"])]
                df["is_link_token"] = [lk is not None and str(o) == lk.token_id
                                       for lk, o in zip(link, df["outcome_pid"].astype(str))]
            else:
                link = [ka.get(c) for c in df["mpid"]]
                df["is_link_token"] = True
            df["resolved_yes"] = [lk.resolved_yes if lk is not None else None for lk in link]
            df["kind"] = [lk.kind if lk is not None else None for lk in link]
            df = df[df["resolved_yes"].notna()]
            if not len(df):
                continue
            df["pbin"] = (np.clip(df["price"].astype(float), 0, 0.9999) // 0.05 * 5).astype(int)
            df["contracts"] = df["size"].astype(float)
            g = df.groupby(["outcome_name", "is_link_token", "taker_side", "pbin", "resolved_yes", "kind"],
                           dropna=False).agg(contracts=("contracts", "sum"), fills=("contracts", "size")).reset_index()
            bins.append(g.assign(venue=v, month=str(m)))
            sec = lambda x: pd.to_datetime(pd.Series(x), utc=True).astype("int64").to_numpy() / 1e9
            lo = [life.get((v, k), (pd.NaT, pd.NaT)) for k in df["mpid"]]
            ts, a, b = sec(df["timestamp"].to_numpy()), sec([x[0] for x in lo]), sec([x[1] for x in lo])
            known = np.array([x[0] is not pd.NaT and pd.notna(x[0]) for x in lo])
            span = np.where(known, b - a, 0)
            frac = np.where(span > 0, (ts - a) / np.where(span > 0, span, 1), np.nan)
            df["decile"] = np.clip(np.floor(frac * 10), 0, 9)
            lifes.append(df.groupby(["decile"], dropna=True).agg(contracts=("contracts", "sum"),
                                                                 fills=("contracts", "size")).reset_index().assign(venue=v))
    B = pd.concat(bins, ignore_index=True)
    B.groupby([c for c in B.columns if c not in ("contracts", "fills", "month")], dropna=False)[["contracts", "fills"]] \
        .sum().reset_index().to_csv(OUT / "fills-bins.csv", index=False)
    Lf = pd.concat(lifes, ignore_index=True).groupby(["venue", "decile"])[["contracts", "fills"]].sum().reset_index()
    ours = []
    for src, f in trade_files():
        t = pd.read_parquet(f)
        t = t[t["mode"].isin(TAKER) & (t["shares"] > 0)]
        v = t["venue"].iloc[0] if len(t) else None
        mk = t["cond"].astype(str) if v == "polymarket" and "cond" in t else t["key"].astype(str)
        for k, ts, sh in zip(mk, pd.to_datetime(t["t"], utc=True), t["shares"]):
            a, b = life.get((v, k), (None, None))
            if a is not None and pd.Timestamp(b) > pd.Timestamp(a):
                fr = (ts - pd.Timestamp(a)).total_seconds() / (pd.Timestamp(b) - pd.Timestamp(a)).total_seconds()
                ours.append(dict(source=src, venue=v, decile=int(min(max(np.floor(fr * 10), 0), 9)), contracts=sh))
    if ours:
        O = pd.DataFrame(ours).groupby(["source", "venue", "decile"]).agg(contracts=("contracts", "sum"),
                                                                        fills=("contracts", "size")).reset_index()
        Lf = pd.concat([Lf.assign(source="all fills"), O], ignore_index=True)
    Lf.to_csv(OUT / "fills-life.csv", index=False)
    pd.concat(samples, ignore_index=True).to_csv(OUT / "fills-sample.csv", index=False)
    return int(B["fills"].sum())


if __name__ == "__main__":
    n, by = markouts()
    print(f"markouts: {n} trades {by}")
    nf = fills()
    print(f"stats: {n} trade markouts, {nf} resolved fills binned")
```

### Part 2 job script: pm_rule1_rerun.sh

```bash
#!/usr/bin/env bash
# Part B again (20 Oct 2026): rule 1 on Polymarket, every saved Polymarket F1 setting at the touch on Probalytics'
# books, ONE PROCESS PER SETTING, so a setting that can't run read-only drops only itself. Read-only on staging's
# mirror of prod. In the first batch (state-batch-20261020T052307Z) a setting whose stage pricings aren't stored for
# every round tried to save them (INSERT INTO model_runs) and the read-only error stopped the whole season.
# 2026: rounds 3-16; a setting that fails there is retried once on rounds 3-15 (logged as "on 3-15").
# 2025: rounds 22-24, no retry. The two seasons run side by side (2 vCPU).
# Output: data/runs/pm-rule1-<UTC>/{2025,2026}/<job>/trades-<job>.parquet, status.csv (job, year, rounds, ok|failed)
set -uo pipefail
set -a; . <env-file>; set +a
export DATABASE_URL="${DATABASE_URL%/*}/<staging-db>"
export PYTHONUNBUFFERED=1 RACINGLINES_PROGRESS_SEC=300 PGOPTIONS="-c default_transaction_read_only=on"
cd <app>
R=.venv/bin/racinglines
PY=.venv/bin/python
SRC=$(ls -1dt data/runs/state-batch-* | head -1)/grids/pm-rule1.json
OUT=data/runs/pm-rule1-$(date -u +%Y%m%dT%H%M%SZ)
mkdir -p "$OUT/one"
say() { echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) $*" | tee -a "$OUT/run.log"; }
T0=$SECONDS
N=$($PY - "$SRC" "$OUT/one" <<'PY'
import json, sys
g = json.load(open(sys.argv[1]))
for e in g:
    json.dump([e], open(f"{sys.argv[2]}/{e['job']}.json", "w"))
print(len(g))
PY
)
say "pm rule 1 rerun from $(git rev-parse --short HEAD), grid $SRC: $N settings, one process each -> $OUT"
echo "job,year,rounds,status" > "$OUT/status.csv"
( while sleep 300; do say "progress: $(( (SECONDS - T0) / 60 )) min elapsed; $(( $(wc -l < "$OUT/status.csv") - 1 )) of $((2 * N)) season runs done, $(grep -c ',failed' "$OUT/status.csv") failed"; done ) &
BEAT=$!
trap 'kill $BEAT 2>/dev/null' EXIT
one() {   # year rounds grid-file -> 0 ok / 1 failed
  local y=$1 rounds=$2 g=$3 job; job=$(basename "$g" .json)
  $R f1 sweep --year "$y" --rounds "$rounds" --no-fetch --grid "$g" --trades-out "$OUT/$y/$job" > "$OUT/$y/$job.log" 2>&1
}
season() {   # year rounds [retry-rounds]
  local y=$1 rounds=$2 retry=${3:-} g job
  mkdir -p "$OUT/$y"
  for g in "$OUT"/one/*.json; do
    job=$(basename "$g" .json)
    if one "$y" "$rounds" "$g"; then echo "$job,$y,$rounds,ok" >> "$OUT/status.csv"
    elif [ -n "$retry" ] && mv "$OUT/$y/$job.log" "$OUT/$y/$job.first.log" && one "$y" "$retry" "$g"; then
      echo "$job,$y,$retry,ok" >> "$OUT/status.csv"; say "$job $y: ok on $retry"
    else echo "$job,$y,${retry:-$rounds},failed" >> "$OUT/status.csv"
      say "FAILED $job $y: $(grep -m1 -oE 'ReadOnlySqlTransaction|[A-Za-z]+Error: .{0,80}' "$OUT/$y/$job.log")"
    fi
  done
  say "season $y done: $(grep -c ",$y,.*,ok" "$OUT/status.csv") ok, $(grep -c ",$y,.*,failed" "$OUT/status.csv") failed ($(( (SECONDS - T0) / 60 )) min)"
}
season 2025 22-24 &
P1=$!
season 2026 3-16 3-15 &
P2=$!
wait $P1; wait $P2
say "all done in $(( (SECONDS - T0) / 60 )) min: $(grep -c ',ok' "$OUT/status.csv") ok, $(grep -c ',failed' "$OUT/status.csv") failed"
```

