# As run: Polymarket step 3, pre-registered (20 Oct 2026)

*This is the runbook as written and run, with machine names, paths, database names and accounts redacted (shown as `<...>`). The commands are otherwise unchanged. The outcome of each run is recorded at the end, and the job script it started is in the appendix.*


**For:** the same Haiku-class agent in VS Code on the owner's Mac that ran the rule 1 handoff. It works the same
way: run each command, compare its output with the **Check**, then go on or stop. Don't change any command, setting or
file. If a check fails, stop and give the owner the step number and the last 20 lines of output.

## Already done (don't redo)

- **Rule 1 run:** finished 02:56Z, 40 runs, 0 failures. Folder `kalshi-rule1-20261020T023605Z`. Claude has
  scored it.
- **The pre-registration is locked:** commit 730644c on `racinglines/racinglines`, pushed 03:52Z (draft PR #276). It
  fixes the two settings, the rounds and the scoring before anything runs, so nothing in this handoff may change them.

## What this run does

Read-only on staging's mirror of the prod database, like the rule 1 run. It has two parts.

1. **Fills census.** For every Probalytics F1 fill file, Polymarket and Kalshi, it records per market the number of
   fills, the $ volume, and the first and last fill. Claude uses this for the trade floor and the first Polymarket F1
   date.
2. **Four sweeps** with `--trades-out`:
   - two settings: `pm-touch` and `pm-book`, the model's defaults on Probalytics' Polymarket tape;
   - two seasons each: 2025 rounds 22–24 and 2026 rounds 3–16.

`pm-book` walks the full book at every order, so it may be slower than the rule 1 runs. How long it takes is unknown.

## Files on the Mac

| Path | What |
|---|---|
| `<mac-folder>/pm-step3/pm_job.sh` | The VM job, written by Claude. Its census part was tested on sample files. |
| `<mac-folder>/pm-step3/grids/pm-touch.json`, `pm-book.json` | The two settings, identical to the copies in commit 730644c |

## Step 1: confirm the files

```
# LOCAL (Mac)
ls -la <mac-folder>/pm-step3/ <mac-folder>/pm-step3/grids/ && bash -n <mac-folder>/pm-step3/pm_job.sh && echo script-ok
```

**Check:** `pm_job.sh` and both JSON files are listed, and the last line is `script-ok`.

## Step 2: check the VM is free, on the right code, and has the tape

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "cd <app> && /usr/bin/git log --oneline -1 && systemctl list-units 'rl-*' --state=running,activating --no-legend | wc -l && sudo grep -c '^PROBALYTICS_BUCKET=' <env-file>; ls data/probalytics/stages 2>/dev/null | grep -c POLYMARKET"
```

This prints counts only. Never print the env file itself.

**Check:** four lines.
1. Starts with `34b18f5` or a later staging commit.
2. `0`: no `rl-` unit is running. If not `0`, stop.
3. `1`: the bucket is set.
4. A number: the Polymarket stage files already on disk.

If line 3 is `0` and line 4 is `0`, stop: the VM has no Probalytics tape to read.

## Step 3: owner go

Ask the owner to post "pm step 3 run starting" in the Claude thread "Kalshi Market Efficiency Report", then wait for
their OK. That lets the coordinator hold the staging merges.

## Step 4: copy the job and the grids to the VM, then start it

```
# LOCAL (Mac)
gcloud compute scp --recurse <mac-folder>/pm-step3/pm_job.sh <mac-folder>/pm-step3/grids <vm>:<tmp>/ --project <project> --zone <zone> --tunnel-through-iap && gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "sudo install -m 755 <tmp>/pm_job.sh <tmp>/pm_job.sh && sudo rm -rf <tmp>/pm-step3-grids && sudo install -d -m 755 <tmp>/pm-step3-grids && sudo install -m 644 <tmp>/grids/pm-touch.json <tmp>/grids/pm-book.json <tmp>/pm-step3-grids/ && sudo systemctl reset-failed rl-pm-step3 2>/dev/null; sudo systemd-run --unit=rl-pm-step3 --uid=<app-user> <tmp>/pm_job.sh && sleep 45 && sudo journalctl -u rl-pm-step3 --no-pager | grep -E 'pm step-3 run from|census|start 1/'"
```

**Check:** all three of these show up.
- `pm step-3 run from 34b18f5 on <staging-db> (read-only)`, or a later commit.
- A `census:` line. If it hasn't finished yet, only `census: start` shows. That's fine.
- If the census has finished, `start 1/4 f1-2025-pm-touch (rounds 22-24)`.

If you see `census FAILED`, note it and go on: the sweeps still run.

## Step 5: wait for it to finish

Check about every 5 minutes:

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "systemctl is-active rl-pm-step3; d=\$(ls -1dt <app>/data/runs/pm-step3-* | head -1); tail -n 4 \$d/run.log"
```

**Check:** it's done when the first line is `inactive` (or `failed`) and run.log ends with
`all done in M min, 4 runs, failures: F`. Note F, and any `FAILED` line with its error text.

## Step 6: owner note

Ask the owner to post this in the Claude thread: "pm step 3 run ended: 4 runs, F failures".

## Step 7: copy the outputs to the Mac

```
# LOCAL (Mac)
mkdir -p <mac-folder>/pm-step3/raw && cd <mac-folder>/pm-step3/raw && d=$(gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "ls -1dt <app>/data/runs/pm-step3-* | head -1" | tr -d '\r') && gcloud compute scp --recurse --project <project> --zone <zone> --tunnel-through-iap "<vm>:$d" . && find . -name '*.parquet' | wc -l && find . -name fills-census.csv -exec wc -l {} \;
```

**Check:**
- The parquet count is 4 − F.
- `fills-census.csv` has more than 2 lines. If the census failed, it may be missing; say so.

## Step 8: report back

Give the owner three things to paste into the Claude thread, then go straight on to steps 9 and 10 (Claude has
added the scorer):
1. The last 6 lines of `raw/pm-step3-*/run.log`.
2. The census line from run.log: `census: N fill files (...), N markets, first Polymarket fill ...`. It is in the
   journal if not in run.log:
   `sudo journalctl -u rl-pm-step3 --no-pager | grep 'census:'`, over the same gcloud ssh as step 5.
3. The path `<mac-folder>/pm-step3/raw/`. Claude reads it from the Mac.

## Step 9 (added 04:05Z): dump the market links for the floor

Read-only. This writes one CSV on the VM and copies it to the Mac.

```
# LOCAL (Mac)
gcloud compute scp <mac-folder>/pm-step3/pm_links.sh <vm>:<tmp>/pm_links.sh --project <project> --zone <zone> --tunnel-through-iap && gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "sudo install -m 755 <tmp>/pm_links.sh <tmp>/pm_links.sh && sudo -u racinglines <tmp>/pm_links.sh" && gcloud compute scp <vm>:<tmp>/pm-links.csv <mac-folder>/pm-step3/links.csv --project <project> --zone <zone> --tunnel-through-iap && wc -l <mac-folder>/pm-step3/links.csv
```

**Check:** it prints `links: N rows, {'polymarket': ..., 'kalshi': ...}`, and links.csv has more than 1,000 lines.

## Step 10: score

```
# LOCAL (Mac)
cd <mac-folder>/pm-step3 && ~/py_dev/racinglines/.venv/bin/python -I pm_score.py > pm-scores.txt; tail -n 20 pm-scores.txt
```

**Check:** the first line starts with `trades in windows:`, then there is a 16-row table (tune and holdout, 2
settings × 4 modes), and the last line starts with `PICK`. If it errors, paste the error and stop. Don't install
anything.

Paste `pm-scores.txt` into the Claude thread. Don't interpret it: Claude writes the verdict.

## Footer (fill in)

- What ran: the unit name, the start and end time, F.
- How it was verified: the step 4 and step 5 check lines.
- Paths: the VM output folder name, the Mac `raw/` folder.
- What was not checked: anything skipped.


## As run (20 Oct 2026, UTC)

| Time (UTC) | Event |
|---|---|
| 03:52:33 | Pre-registration committed (racinglines commit 730644c, PR #276), before anything ran |
| 03:55:47 | Unit started on staging commit 34b18f5, read-only: fills census plus 4 sweeps |
| 04:00:40 | Done: 4 runs, **0 failures**. Census: 34 fill files, 8,785 markets, first fill 2025-06-01 |
| after | Links dump: 6,216 rows (Kalshi 3,350, Polymarket 2,866); scored once with the pre-registered scorer |

## Appendix: scripts as run

### Job script: pm_job.sh

```bash
#!/usr/bin/env bash
# Step 3 of the Kalshi response: the pre-registered Polymarket test on Probalytics' tape (prereg:
# paper/prereg/2026-10-20-polymarket-step3.md). Read-only on staging's mirror. Two parts:
#   1. fills census: every Probalytics F1 fill file (Polymarket and Kalshi), per market: fills, $ volume, first and
#      last fill -> fills-census.csv (the trade floor and the first Polymarket F1 listing date)
#   2. the un-tuned settings (grids/pm-touch.json, grids/pm-book.json) on 2025 rounds 22-24 and 2026 rounds 3-16,
#      with --trades-out, one process per setting and season so one failure doesn't stop the rest
# A progress line every 5 minutes. Output: data/runs/pm-step3-<UTC>/
set -uo pipefail
set -a; . <env-file>; set +a
export DATABASE_URL="${DATABASE_URL%/*}/<staging-db>"
export PYTHONUNBUFFERED=1 RACINGLINES_PROGRESS_SEC=300 PGOPTIONS="-c default_transaction_read_only=on" PROBALYTICS_CACHE_EVICT=1
cd <app>
R=.venv/bin/racinglines
G=<tmp>/pm-step3-grids
OUT=data/runs/pm-step3-$(date -u +%Y%m%dT%H%M%SZ)
mkdir -p "$OUT"
say() { echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) $*" | tee -a "$OUT/run.log"; }
T0=$SECONDS
( while sleep 300; do say "progress: $(( (SECONDS - T0) / 60 )) min elapsed"; done ) &
BEAT=$!
trap 'kill $BEAT 2>/dev/null' EXIT
say "pm step-3 run from $(git rev-parse --short HEAD) on ${DATABASE_URL##*/} (read-only) -> $OUT"
say "census: start"
.venv/bin/python - "$OUT/fills-census.csv" <<'PY' || say "census FAILED (the sweeps still run)"
import sys
from pathlib import Path
import pandas as pd
import pyarrow.parquet as pq
from racinglines.markets import probalytics as p
base = Path(p.root())
months = pd.period_range("2022-11", pd.Timestamp.utcnow().strftime("%Y-%m"), freq="M")
rows, seen = [], []
for venue in ("POLYMARKET", "KALSHI"):
    for m in months:
        f = base / "fills" / f"F1_{venue}_{m}.parquet"
        got = p.fetch([f], base)
        if not f.exists():
            continue
        seen.append(f.name)
        have = set(pq.read_schema(f).names)
        cols = [c for c in ("fill_id", "market_platform_id", "timestamp", "price", "size") if c in have]
        df = pq.read_table(f, columns=cols).to_pandas()
        p._evict(got)
        if "fill_id" in df:
            df = df.drop_duplicates("fill_id")
        df["usd"] = df["price"] * df["size"]
        g = df.groupby("market_platform_id").agg(fills=("usd", "size"), usd=("usd", "sum"),
                                                 first=("timestamp", "min"), last=("timestamp", "max"))
        rows.append(g.reset_index().assign(venue=venue.lower(), month=str(m)))
out = pd.concat(rows, ignore_index=True)
out = out.groupby(["venue", "market_platform_id"]).agg(fills=("fills", "sum"), usd=("usd", "sum"),
                                                       first=("first", "min"), last=("last", "max")).reset_index()
out.to_csv(sys.argv[1], index=False)
print(f"census: {len(seen)} fill files ({seen[0] if seen else '-'} to {seen[-1] if seen else '-'}), "
      f"{len(out)} markets, first Polymarket fill {out[out.venue == 'polymarket']['first'].min()}")
PY
[ -s "$OUT/fills-census.csv" ] && say "census: done, $(($(wc -l < "$OUT/fills-census.csv") - 1)) markets" || say "census: no file"
I=0; FAIL=0; N=4
for k in pm-touch pm-book; do
  for y in 2025 2026; do
    I=$((I + 1)); rounds="22-24"; [ "$y" = 2026 ] && rounds="3-16"
    say "start $I/$N f1-$y-$k (rounds $rounds)"
    if $R f1 sweep --year "$y" --rounds "$rounds" --no-fetch --grid "$G/$k.json" --trades-out "$OUT/f1-$y-$k" > "$OUT/f1-$y-$k.log" 2>&1; then
      say "done $I/$N f1-$y-$k ($(( (SECONDS - T0) / 60 )) min)"
    else
      say "FAILED $I/$N f1-$y-$k: $(grep -m1 -oE 'ReadOnlySqlTransaction|Error: .{0,80}' "$OUT/f1-$y-$k.log")"; FAIL=$((FAIL + 1))
    fi
  done
done
say "all done in $(( (SECONDS - T0) / 60 )) min, $N runs, failures: $FAIL"
```

### Links dump: pm_links.sh

```bash
#!/usr/bin/env bash
# Read-only: F1 market links (Polymarket and Kalshi) for the step 3 rounds -> <tmp>/pm-links.csv, for the market
# floor in pm_score.py. Staging's mirror of prod, read-only transaction.
set -euo pipefail
set -a; . <env-file>; set +a
export DATABASE_URL="${DATABASE_URL%/*}/<staging-db>" PGOPTIONS="-c default_transaction_read_only=on"
cd <app>
.venv/bin/python - <tmp>/pm-links.csv <<'PY'
import os, sys
import pandas as pd
from sqlalchemy import create_engine, text
q = text("""SELECT ml.exchange, ml.token_id, ml.condition_id, ml.prediction AS kind, ml.volume,
                   extract(year FROM e.start_date)::int AS year, e.series_round AS round
            FROM market_links ml JOIN races r ON r.id = ml.race_id JOIN events e ON e.id = r.event_id
                 JOIN competitions c ON c.id = ml.competition_id
            WHERE ml.exchange IN ('polymarket', 'kalshi') AND c.code LIKE 'f1%'
              AND ((extract(year FROM e.start_date) = 2025 AND e.series_round BETWEEN 22 AND 24)
                OR (extract(year FROM e.start_date) = 2026 AND e.series_round BETWEEN 1 AND 17))""")
df = pd.read_sql(q, create_engine(os.environ["DATABASE_URL"]))
df.to_csv(sys.argv[1], index=False)
print(f"links: {len(df)} rows, {df.exchange.value_counts().to_dict()}")
PY
```

