# As run: every v3 run again, calibration on (21 Oct 2026)

*This is the runbook as written and run, with machine names, paths, database names and accounts redacted (shown as `<...>`). The commands are otherwise unchanged. The outcome of each run is recorded at the end, and the job script it started is in the appendix.*


**What this runs.** The 20 Oct runs behind v3's plots and tables, again with the same settings, windows and engine,
with one change: `calib_power = walk_forward` on every setting. Then the plots are redone for v4. One transient unit,
`rl-v4-calib`, runs all of it from staging's checkout:

| Part | What | Feeds |
|---|---|---|
| A | the model on every market, on Probalytics' books (F1 Polymarket 2025 r22-24 and 2026 r3-16, F1 Kalshi 2026 r5-16) | 3.1 |
| K | rule 1 on Kalshi: every saved Kalshi F1 setting, 2025 and 2026 r1-16 (includes the step 2 setting 11c96efd2255) | 3.2, 3.3, Fig 1, Fig 2, Table 3 |
| N | NASCAR on Kalshi, the baseline, 2025 and 2026 | 3.2 |
| C | Kalshi on Probalytics' books, the saved settings, 2026 r5-16 | 3.5 |
| S | the step 3 candidates (touch, book) | 3.4 |
| B | rule 1 on Polymarket: every saved Polymarket F1 setting, one process each (106 settings x 2 seasons) | 3.3, Fig 1 |
| E | markouts of our trades | 3.5, Fig 9 |

What isn't rerun:
- **MotoGP** has no fitted calibration table.
- **The crowd tables** (3.6, Fig 3, 4.4, 4.5) don't use our model, so v3's stand.
- **Our own tape's calibrated dumps** already exist from the 20 Oct batch (part G).
- **Part S isn't a pre-registered test with calibration.** It is a new entry dated 21 Oct, not a re-score of the
  commit 730644c test.

**Where it writes.** The demo database `<demo-db>`, not the mirror. A calibrated setting has a new model
key, so the F1 sweep prices and saves its stages (`model_runs` rows). On the read-only mirror that is the
`ReadOnlySqlTransaction` stop of 20 Oct. The demo database holds production's tape through Singapore. Step 4 backs it
up first. Nothing touches production or `<staging-db>`.

**How long.** Unknown. v3's Polymarket part alone was estimated at 3 to 6 hours, and calibration adds stage pricing per
model key. Step 6 times one round so you can judge before starting. The VM has 2 vCPUs: nothing else `rl-*` runs
meanwhile, and nothing merges into staging until it ends. It needs to end before Austin's FP1 (Fri 23 Oct 17:30Z).

**When to start.** Core's calibration-default change has to merge to staging and be promoted to main before the Austin
lock (Wed 21 Oct 20:00Z); the thread "PyPI release from racinglines" owns that sequence. Start this run only once that
sequence is done (or the release thread says no staging merge is due before Friday), and tell that thread when it
starts and ends, so no staging merge lands while it runs.

Run the steps in order, one block at a time. Files are in `<mac-folder>/v4-calib/` on your Mac.

## 1. Check staging and that the VM is free

```
# LOCAL (Mac), in <checkout>
conda deactivate 2>/dev/null; bash scripts/deploy/vm.sh staging status
```

**Check:** the first line reads `staging: d619426 ...` (or a later staging commit).

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "systemctl list-units 'rl-*' --state=running,activating --no-legend | grep -vc jev-monitor; ls -1dt <app>/data/runs/state-batch-* <app>/data/runs/kalshi-rule1-* | head -n 4; ls <app>/data/runs/kalshi-rule1-*/grids | head -n 3"
```

**Check:**
- The first line is `0`.
- The list shows `state-batch-20261020T052307Z` and a `kalshi-rule1-...` folder.
- Grid files named like `2025-<key>.json` are listed.

If the count isn't `0`, stop: another queue is running.

## 2. Copy the files to the VM

```
# LOCAL (Mac)
cp <mac-folder>/state-batch/calib_dump.py <mac-folder>/v4-calib/ && cd <mac-folder>/v4-calib && bash -n v4_calib_rerun.sh && md5 v4_calib_rerun.sh stats.py
```

**Check:** no error, and two checksums are printed:
- `v4_calib_rerun.sh`: 066e5ddbdbf20af339b7d25affb7f113
- `stats.py`: 6f54983392754cc3414167f0f413e26e

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "rm -rf <tmp>/v4-calib-in && mkdir -p <tmp>/v4-calib-in" && cd <mac-folder>/v4-calib && gcloud compute scp --recurse v4_calib_rerun.sh stats.py calib_dump.py grids <vm>:<tmp>/v4-calib-in/ --project <project> --zone <zone> --tunnel-through-iap
```

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "sudo install -d -m 755 <tmp>/v4-calib <tmp>/v4-calib/grids && sudo install -m 755 <tmp>/v4-calib-in/v4_calib_rerun.sh <tmp>/v4-calib/ && sudo install -m 644 <tmp>/v4-calib-in/stats.py <tmp>/v4-calib-in/calib_dump.py <tmp>/v4-calib/ && sudo install -m 644 <tmp>/v4-calib-in/grids/*.json <tmp>/v4-calib/grids/ && bash -n <tmp>/v4-calib/v4_calib_rerun.sh && ls <tmp>/v4-calib <tmp>/v4-calib/grids && echo copy-ok"
```

**Check:** the listing shows:
- in `<tmp>/v4-calib/`: `calib_dump.py`, `grids`, `stats.py`, `v4_calib_rerun.sh`;
- in `grids/`: `pm-book.json`, `pm-touch.json`, `smoke.json`.

The last line is `copy-ok`.

## 3. Check the demo database has the markets the runs need

Read-only. The demo should match the mirror for these seasons (it was copied from production through Singapore).

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "cd <prod-app> && for d in <staging-db> <demo-db>; do echo \"== \$d\"; sudo -u racinglines docker compose exec -T db psql -U racinglines -d \$d -tAc \"SELECT c.code, ml.exchange, s.year, count(*) FROM market_links ml JOIN races r ON r.id=ml.race_id JOIN events e ON e.id=r.event_id JOIN seasons s ON s.id=e.season_id JOIN competitions c ON c.id=s.competition_id WHERE c.code IN ('f1_wdc','nascar_cup') AND s.year IN (2025,2026) AND e.start_date < '2026-10-08' GROUP BY 1,2,3 ORDER BY 1,2,3\"; done"
```

**Check:** the rows under `<demo-db>` match the rows under `<staging-db>`. The demo may be lower
only where the mirror has links added after the demo was built. If any F1 Polymarket or Kalshi row is missing or much
lower in the demo, stop and paste both blocks into the thread.

## 4. Back up the demo database

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "cd <prod-app> && UTC=\$(date -u +%Y%m%dT%H%M%SZ) && sudo -u racinglines docker compose exec -T db pg_dump -U racinglines <demo-db> | gzip > <tmp>/rl-demo-before-v4-calib-\$UTC.sql.gz && gzip -t <tmp>/rl-demo-before-v4-calib-\$UTC.sql.gz && sudo install -o racinglines -m 640 <tmp>/rl-demo-before-v4-calib-\$UTC.sql.gz <app>/data/backups/db/<demo-db>-before-v4-calib-\$UTC.sql.gz && rm <tmp>/rl-demo-before-v4-calib-\$UTC.sql.gz && ls -la <app>/data/backups/db/ | tail -n 2"
```

**Check:** the last line lists `<demo-db>-before-v4-calib-<time>.sql.gz` with a size in the tens of MB
or more. Note its name for step 5.

## 5. Record the change in the demo's change log

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "B=\$(ls -1t <app>/data/backups/db/<demo-db>-before-v4-calib-* | head -1) && sudo -u racinglines bash -c 'set -a; . <env-file>; set +a; export DATABASE_URL=\"\${DATABASE_URL%/*}/<demo-db>\"; cd <app> && .venv/bin/racinglines db changes --sport f1 --add \"v4 calib rerun (Kalshi response v4): F1 sweeps with calib_power=walk_forward save stage pricings (model_runs) to the demo only. Backup: '\"\$B\"'. Rollback: restore that dump.\" && .venv/bin/racinglines db changes --sport f1 --limit 1'"
```

**Check:** the last line printed is the note you just added, naming the backup file.

## 6. Time one round (the smoke test)

One calibrated Polymarket setting, 2026 round 3 only, on the demo database. It saves that round's calibrated pricing,
which the full run reuses.

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "sudo -u racinglines bash -c 'set -a; . <env-file>; set +a; export DATABASE_URL=\"\${DATABASE_URL%/*}/<demo-db>\" PYTHONUNBUFFERED=1 PROBALYTICS_CACHE_EVICT=1; cd <app> && S=\$SECONDS && .venv/bin/racinglines f1 sweep --year 2026 --rounds 3 --no-fetch --grid <tmp>/v4-calib/grids/smoke.json --trades-out <tmp>/v4-calib-smoke > <tmp>/v4-calib-smoke.log 2>&1; echo exit=\$? seconds=\$((SECONDS - S)); tail -n 5 <tmp>/v4-calib-smoke.log; ls <tmp>/v4-calib-smoke'"
```

**Check:**
- `exit=0`.
- A `trades*.parquet` file is listed.
- The `seconds` value is the time for one round of one setting.

If the exit isn't 0, stop and paste the output into the thread. If `seconds` is above 600, paste it into the thread
before going on, because the full run would then likely not end before Friday.

## 7. Start the run

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "sudo systemctl reset-failed rl-v4-calib 2>/dev/null; sudo systemd-run --unit=rl-v4-calib --uid=<app-user> <tmp>/v4-calib/v4_calib_rerun.sh && sleep 60 && sudo journalctl -u rl-v4-calib --no-pager -o cat | grep -E 'v4 calib rerun|grids:|polymarket settings|STOP'"
```

**Check:**
- A line `v4 calib rerun from <commit> on <demo-db>, ...` appears.
- A line `... polymarket settings, ... kalshi setting-years` appears, with numbers above 0.

If you see `STOP`, paste the lines into the thread.

Then post "v4 calib rerun started" in the Claude thread "Kalshi market efficiency report".

## 8. Follow it

Check now and then; it is fine to leave it overnight.

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "systemctl is-active rl-v4-calib; df -h / | tail -n 1; d=\$(ls -1dt <app>/data/runs/v4-calib-* | head -1); tail -n 4 \$d/run.log"
```

**Check:** it is done when:
- the first line is `inactive` (or `failed`);
- `run.log` ends with `all done in M min: K ok, F failed`.

`FAILED ...` lines along the way are expected; note the count. The second line is the VM disk: about 4.5 GB was
free when the run started (4.9 GB at 02:01Z, less the step 4 dump). If it shows under 1 GB, paste it into the thread.

**8b. ETA inputs (read-only, optional).** How many distinct model keys the Polymarket settings have (each new key
prices every round once; settings that share a key reuse it), and when each 2026 setting finished:

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "sudo journalctl -u rl-v4-calib --no-pager -o cat | grep -m1 'grids:'; d=\$(ls -1dt <app>/data/runs/v4-calib-* | head -1); grep -m1 'start A' \$d/run.log; ls -ltr --time-style=+%H:%M \$d/B/2026/*.log \$d/B/2025/*.log | awk '{print \$6, \$7}' | sed 's#.*/B/##' | tail -n 30"
```

Paste the output in the thread; Claude turns it into an ETA.

## 9. Copy the outputs to the Mac

```
# LOCAL (Mac)
mkdir -p <mac-folder>/v4-calib/raw && cd <mac-folder>/v4-calib/raw && d=$(gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "ls -1dt <app>/data/runs/v4-calib-* | head -1" | tr -d '\r') && gcloud compute scp --recurse --project <project> --zone <zone> --tunnel-through-iap "<vm>:$d" . && find . -name '*.parquet' | wc -l && tail -n 3 v4-calib-*/run.log
```

**Check:** a parquet count is printed, then the last 3 lines of `run.log`.

Post "v4 calib rerun ended: K ok, F failed" in the Claude thread. Claude reads the folder from the Mac, scores every run
with the v3 scripts, redoes the plots and tables, and builds v4.1 for you to push.

## Rollback

The run only adds `model_runs` rows (and their predictions) to `<demo-db>`, and files under
`data/runs/v4-calib-*`. To undo the database side, restore the step 4 dump into the demo database:

```
# LOCAL (Mac), only if a rollback is needed
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "B=\$(ls -1t <app>/data/backups/db/<demo-db>-before-v4-calib-* | head -1) && echo \$B && cd <prod-app> && sudo -u racinglines docker compose exec -T db psql -U racinglines -d postgres -c 'DROP DATABASE <demo-db> WITH (FORCE)' -c 'CREATE DATABASE <demo-db>' && gunzip -c \$B | sudo -u racinglines docker compose exec -T db psql -U racinglines -d <demo-db> -q && echo restored"
```

Staging serves the demo database, so staging is down for the minutes the restore takes.


## As run (21 Oct 2026, UTC)

| Time (UTC) | Event |
|---|---|
| 02:06 | Rerun set: every run behind v3's tables and plots, again with `calib_power = walk_forward` |
| before 02:59 | Steps 1 to 3 passed: staging at d2674e7, no other queue running, the demo database matches the mirror on all nine link rows. Steps 4 and 5: demo database backed up and the change logged |
| 02:59:08 | Unit started on staging commit d2674e7, on the demo database: 106 Polymarket settings (47 model keys), 40 Kalshi setting-years |
| 02:59 to about 03:51 | Parts A, K, N, C and S done |
| 08:46:30 | Part B done: 106 Polymarket settings × 2 seasons, every 2026 setting on rounds 3 to 16 (no retries needed) |
| 08:47:01 | Markouts done: 20,736 trades. **All done in 347 min: 262 ok, 0 failed** |
| 11:01 | Outputs copied off the server (304 trade files plus the markouts), then scored with v3's scripts, unchanged |

## Appendix: scripts as run

### Job script: v4_calib_rerun.sh

```bash
#!/usr/bin/env bash
# v4 of the Kalshi response (21 Oct 2026): every run behind the paper's plots and tables, again with calibration on
# (calib_power = walk_forward: the win-price power fitted on earlier races only). Same settings, windows and engine as
# the 20 Oct runs (v3), only calib_power added. Runs on the season demo database (<demo-db>), not the
# read-only mirror: a calibrated setting has a new model key, so the F1 sweep has to price and save its stages
# (model_runs rows, demo database only; back it up first). Parts, in order:
#   A  the model on every market, Probalytics' books (calib_dump.py): F1 Polymarket 2025 r22-24 and 2026 r3-16,
#      F1 Kalshi 2026 r5-16 (our own tape's calibrated dumps already exist from the 20 Oct batch, part G)
#   K  rule 1 on Kalshi: every saved Kalshi F1 setting (the grids of the 20 Oct kalshi-rule1 run), 2025 and 2026 r1-16;
#      includes 11c96efd2255, the step 2 setting
#   N  NASCAR on Kalshi, the sport's baseline (step 2), 2025 and 2026. MotoGP has no fitted table, so it isn't rerun
#   C  Kalshi on Probalytics' books: the 20 saved Kalshi F1 settings, 2026 r5-16
#   S  the step 3 candidates (pm-touch, pm-book), 2025 r22-24 and 2026 r3-16. Not pre-registered with calibration:
#      a new entry, not a re-score of the commit 730644c test
#   B  rule 1 on Polymarket: every saved Polymarket F1 setting, one process per setting, 2025 r22-24 and 2026 r3-16
#      (retry r3-15 on a failure), the two seasons side by side
#   E  stats.py: markouts of K's 2026 trades, S and C (the crowd's fill tables don't depend on our model; v3's stand)
# A progress line every 5 minutes. Output: data/runs/v4-calib-<UTC>/
set -uo pipefail
set -a; . <env-file>; set +a
export DATABASE_URL="${DATABASE_URL%/*}/<demo-db>"
export PYTHONUNBUFFERED=1 RACINGLINES_PROGRESS_SEC=300 PROBALYTICS_CACHE_EVICT=1
cd <app>
R=.venv/bin/racinglines
PY=.venv/bin/python
S=<tmp>/v4-calib
SB=$(ls -1dt data/runs/state-batch-* | head -1)
K1=$(ls -1dt data/runs/kalshi-rule1-* | head -1)
OUT=data/runs/v4-calib-$(date -u +%Y%m%dT%H%M%SZ)
mkdir -p "$OUT/grids/kalshi" "$OUT/grids/pm-one"
say() { echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) $*" | tee -a "$OUT/run.log"; }
T0=$SECONDS
case "$DATABASE_URL" in */<demo-db>) ;; *) say "STOP: not the demo database"; exit 1 ;; esac
say "v4 calib rerun from $(git rev-parse --short HEAD) on ${DATABASE_URL##*/}, grids from $SB and $K1 -> $OUT"
$PY - "$SB/grids" "$K1/grids" "$S/grids" "$OUT/grids" <<'PY' || { say "STOP: could not build the grids"; exit 1; }
import glob, json, os, sys
from racinglines.pipelines import sweep_settings as SS
sb, k1, s3, out = sys.argv[1:5]
def cal(entries):
    got = [dict(e, settings=dict(e["settings"], calib_power="walk_forward")) for e in entries]
    for e in got:
        SS.Settings.from_dict(e["settings"])
    return got
pm = cal(json.load(open(f"{sb}/pm-rule1.json")))
for e in pm:
    json.dump([e], open(f"{out}/pm-one/{e['job']}.json", "w"))
json.dump(cal(json.load(open(f"{sb}/kalshi-pb.json"))), open(f"{out}/kalshi-pb.json", "w"))
ks = sorted(glob.glob(f"{k1}/*.json"))
for f in ks:
    json.dump(cal(json.load(open(f))), open(f"{out}/kalshi/{os.path.basename(f)}", "w"))
for k in ("pm-touch", "pm-book"):
    json.dump(cal(json.load(open(f"{s3}/{k}.json"))), open(f"{out}/{k}.json", "w"))
keys = {SS.Settings.from_dict(e["settings"]).model_key for e in pm}
print(f"grids: {len(pm)} polymarket ({len(keys)} model keys), {len(ks)} kalshi rule 1, kalshi-pb, step 3")
PY
say "$(ls "$OUT/grids/pm-one" | wc -l) polymarket settings, $(ls "$OUT/grids/kalshi" | wc -l) kalshi setting-years"
echo "part,job,year,rounds,status" > "$OUT/status.csv"
( while sleep 300; do say "progress: $(( (SECONDS - T0) / 60 )) min elapsed; $(( $(wc -l < "$OUT/status.csv") - 1 )) runs done, $(grep -c ',failed' "$OUT/status.csv") failed"; done ) &
BEAT=$!
trap 'kill $BEAT 2>/dev/null' EXIT
run() {   # part job year rounds cmd... (gets --trades-out <dir>)
  local part=$1 job=$2 y=$3 rounds=$4 d; shift 4; d="$OUT/$part/$job"; mkdir -p "$d"
  if "$@" --trades-out "$d" > "$d.log" 2>&1; then echo "$part,$job,$y,$rounds,ok" >> "$OUT/status.csv"; return 0; fi
  echo "$part,$job,$y,$rounds,failed" >> "$OUT/status.csv"
  say "FAILED $part $job: $(grep -m1 -oE '[A-Za-z]+Error: .{0,100}' "$d.log")"; return 1
}
calib() {   # name sport year rounds settings-json
  say "start A $1"
  if $PY "$S/calib_dump.py" "$2" "$3" "$4" "$5" "$OUT/calib/$1" > "$OUT/calib-$1.log" 2>&1; then
    echo "A,$1,$3,$4,ok" >> "$OUT/status.csv"; say "done A $1: $(tail -n 1 "$OUT/calib-$1.log")"
  else echo "A,$1,$3,$4,failed" >> "$OUT/status.csv"; say "FAILED A $1: $(grep -m1 -oE '[A-Za-z]+Error: .{0,100}' "$OUT/calib-$1.log")"; fi
}
mkdir -p "$OUT/calib"
calib f1-pm-pb-cal-2025 f1 2025 22-24 '{"tape_source": "probalytics", "taker_fill": "touch", "calib_power": "walk_forward"}'
calib f1-pm-pb-cal-2026 f1 2026 3-16 '{"tape_source": "probalytics", "taker_fill": "touch", "calib_power": "walk_forward"}'
calib f1-kalshi-pb-cal-2026 f1 2026 5-16 '{"venue": "kalshi", "tape_source": "probalytics", "calib_power": "walk_forward"}'
say "start K: kalshi rule 1"
for g in "$OUT"/grids/kalshi/*.json; do
  b=$(basename "$g" .json); y=${b%%-*}; k=${b#*-}
  if [ "$y" = 2026 ]; then run K "f1-$y-$k" "$y" 1-16 $R f1 sweep --year "$y" --rounds 1-16 --no-fetch --grid "$g"
  else run K "f1-$y-$k" "$y" all $R f1 sweep --year "$y" --no-fetch --grid "$g"; fi
done
say "done K: $(grep -c '^K,.*,ok' "$OUT/status.csv") ok ($(( (SECONDS - T0) / 60 )) min)"
for y in 2025 2026; do
  run N "nascar-$y" "$y" all $R strategy sweep nascar --year "$y" --venue kalshi --no-fetch --calib-power walk_forward
done
say "done N ($(( (SECONDS - T0) / 60 )) min)"
run C kalshi-pb-2026 2026 5-16 $R f1 sweep --year 2026 --rounds 5-16 --no-fetch --grid "$OUT/grids/kalshi-pb.json"
say "done C ($(( (SECONDS - T0) / 60 )) min)"
for k in pm-touch pm-book; do
  run S "f1-2025-$k" 2025 22-24 $R f1 sweep --year 2025 --rounds 22-24 --no-fetch --grid "$OUT/grids/$k.json"
  run S "f1-2026-$k" 2026 3-16 $R f1 sweep --year 2026 --rounds 3-16 --no-fetch --grid "$OUT/grids/$k.json"
done
say "done S ($(( (SECONDS - T0) / 60 )) min)"
season() {   # year rounds [retry-rounds]
  local y=$1 rounds=$2 retry=${3:-} g job
  for g in "$OUT"/grids/pm-one/*.json; do
    job=$(basename "$g" .json)
    if run B "$y/$job" "$y" "$rounds" $R f1 sweep --year "$y" --rounds "$rounds" --no-fetch --grid "$g"; then :
    elif [ -n "$retry" ]; then
      mv "$OUT/B/$y/$job.log" "$OUT/B/$y/$job.first.log"
      run B "$y/$job" "$y" "$retry" $R f1 sweep --year "$y" --rounds "$retry" --no-fetch --grid "$g" && say "$job $y: ok on $retry"
    fi
  done
  say "season $y done: $(grep -c "^B,$y/.*,ok" "$OUT/status.csv") ok, $(grep -c "^B,$y/.*,failed" "$OUT/status.csv") failed ($(( (SECONDS - T0) / 60 )) min)"
}
mkdir -p "$OUT/B/2025" "$OUT/B/2026"
season 2025 22-24 &
P1=$!
season 2026 3-16 3-15 &
P2=$!
wait $P1; wait $P2
mkdir -p "$OUT/kalshi-pb-2026" && cp "$OUT"/C/kalshi-pb-2026/*.parquet "$OUT/kalshi-pb-2026/" 2>/dev/null
say "stats: start"
if RL_RULE1="$OUT/K" RL_STEP3="$OUT/S" RL_MARKOUTS_ONLY=1 $PY "$S/stats.py" "$OUT" > "$OUT/stats.log" 2>&1; then
  say "stats: done ($(tail -n 1 "$OUT/stats.log"))"; else say "stats FAILED: $(tail -n 1 "$OUT/stats.log")"; fi
say "all done in $(( (SECONDS - T0) / 60 )) min: $(grep -c ',ok$' "$OUT/status.csv") ok, $(grep -c ',failed$' "$OUT/status.csv") failed"
```

### Statistics: stats.py (markouts part)

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
v4 rerun (21 Oct): RL_RULE1 / RL_STEP3 name those folders instead, and RL_MARKOUTS_ONLY=1 skips the fills tables
(the crowd's fills don't depend on our model, so v3's stand).
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
    r1 = os.environ.get("RL_RULE1") or latest("data/runs/kalshi-rule1-*")
    if r1:
        out += [("rule1-kalshi-ours", f) for f in sorted(glob.glob(f"{r1}/f1-2026-*/*.parquet"))]
    s3 = os.environ.get("RL_STEP3") or latest("data/runs/pm-step3-*")
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
    if os.environ.get("RL_MARKOUTS_ONLY"):
        sys.exit(0)
    nf = fills()
    print(f"stats: {n} trade markouts, {nf} resolved fills binned")
```
