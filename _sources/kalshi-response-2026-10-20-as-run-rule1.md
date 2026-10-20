# As run: Kalshi rule 1, all 20 saved F1 settings (20 Oct 2026)

*This is the runbook as written and run, with machine names, paths, database names and accounts redacted (shown as `<...>`). The commands are otherwise unchanged. The outcome of each run is recorded at the end, and the job script it started is in the appendix.*


**For:** a Haiku-class agent in VS Code on the owner's Mac (terminal access). Every step below is mechanical: run
the command, compare the output with the **Check**, and go on or stop. Don't change any command, setting or file
except where a step says so. If a check fails, stop, and give the owner the step number and the last 20 lines of
output.

## Already done (don't redo)

Step 2 of the re-sim ran at 01:49 and 01:58Z. That was our own run, triggered by the owner. Its outputs are in two
folders, `data/runs/kalshi-resim-20261020T014914Z/` (F1 2025, NASCAR 2025 and 2026, MotoGP 2026) and
`data/runs/kalshi-resim-f1-2026-r1-16-20261020T015845Z/` (F1 2026). Leave both alone. "0 min" in those logs means
under a minute, which is expected because the stage pricings were already stored.

## Why

Golden rule 1 says a strategy's edge must survive a luck test corrected for **every setting tried**. Step 2 of our
Kalshi response re-ran one F1 setting (11c96efd2255). Its F1 2026 edge was +12¢/share against the mid, but that
setting is one of 20 saved Kalshi F1 sweep settings per season. This run re-runs all of them with `--trades-out`, so
the correction covers them all.

- 2025: every round.
- 2026: rounds 1 to 16. Round 17 has no stored stage pricings, and this run is read-only.

The results get published whatever they show (the owner's no-cheating rule).

## Ground rules

- **Read-only:** the job sets `PGOPTIONS=default_transaction_read_only=on`, so it cannot write to any database. A
  setting that needs a write fails on its own, gets logged, and the loop goes on.
- **One queue at a time:** the VM has 2 vCPUs. Step 2 checks that no `rl-*` unit is running.
- **The coordinator holds the staging merges** while this runs. Step 3 has the owner tell the Claude thread before
  starting, and step 6 has them tell it when the run ends.
- **No secrets:** none are needed. Never print `/etc/*.env` files.
- **Mac shell:** run `conda deactivate` first if a conda env is active, and use `/usr/bin/git`.

## Files

| Path on the Mac | What |
|---|---|
| `<mac-folder>/kalshi-rule1/rule1_job.sh` | The VM job (built and tested by Claude; the grid builder was tested on a local Postgres) |
| `<mac-folder>/kalshi-rule1/score.py` | The scorer used for step 2 |

## Step 1: confirm the files are on the Mac

```
# LOCAL (Mac)
ls -la <mac-folder>/kalshi-rule1/ && bash -n <mac-folder>/kalshi-rule1/rule1_job.sh && echo script-ok
```

**Check:** both files are listed and the last line is `script-ok`.

## Step 2: check the VM is free and on the right code

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "cd <app> && /usr/bin/git log --oneline -1 && systemctl list-units 'rl-*' --state=running,activating --no-legend | wc -l"
```

**Check:**
- The first line starts with `34b18f5` or a later staging commit.
- The last line is `0`, meaning no `rl-` unit is running. Old `failed` units and timers don't count. If it is not `0`, stop: another queue is running.

## Step 3: owner go

Ask the owner to post "rule 1 run starting" in the Claude thread "Kalshi Market Efficiency Report", then wait for
their OK to continue. That lets the coordinator hold the staging merges.

## Step 4: copy the job to the VM and start it

```
# LOCAL (Mac)
gcloud compute scp <mac-folder>/kalshi-rule1/rule1_job.sh <vm>:<tmp>/rule1_job.sh --project <project> --zone <zone> --tunnel-through-iap && gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "sudo install -m 755 <tmp>/rule1_job.sh <tmp>/rule1_job.sh && sudo systemctl reset-failed rl-kalshi-rule1 2>/dev/null; sudo systemd-run --unit=rl-kalshi-rule1 --uid=<app-user> <tmp>/rule1_job.sh && sleep 30 && sudo journalctl -u rl-kalshi-rule1 --no-pager | tail -n 5"
```

**Check:** the output shows these three lines:
- `kalshi rule-1 run from 34b18f5 on <staging-db> (read-only)`
- `N settings x years to run`, where N should be 40 (20 settings × 2 years)
- `start 1/N ...`

If you see `STOP: could not build the grids`, stop.

## Step 5: wait for it to finish

Each setting takes about 20 to 60 seconds, so expect about 15 to 40 minutes in all. Check every 5 minutes:

```
# LOCAL (Mac)
gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "systemctl is-active rl-kalshi-rule1; d=\$(ls -1dt <app>/data/runs/kalshi-rule1-* | head -1); tail -n 3 \$d/run.log"
```

**Check:** it's done when the first line is `inactive` (or `failed`) and run.log ends with
`all done in M min, N runs, failures: F`. A few `FAILED ... ReadOnlySqlTransaction` lines are expected and fine:
those settings need stage pricings that aren't stored. Note F.

## Step 6: owner note

Ask the owner to post this in the Claude thread: "rule 1 run ended: N runs, F failures".

## Step 7: copy the trade files to the Mac

```
# LOCAL (Mac)
mkdir -p <mac-folder>/kalshi-rule1/raw && cd <mac-folder>/kalshi-rule1/raw && d=$(gcloud compute ssh <vm> --project <project> --zone <zone> --tunnel-through-iap --command "ls -1dt <app>/data/runs/kalshi-rule1-* | head -1" | tr -d '\r') && gcloud compute scp --recurse --project <project> --zone <zone> --tunnel-through-iap "<vm>:$d" . && find . -name '*.parquet' | wc -l
```

**Check:** the count is N − F.

## Step 8: arrange the files for the scorer and score them

```
# LOCAL (Mac)
cd <mac-folder>/kalshi-rule1 && mkdir -p data && for f in raw/kalshi-rule1-*/f1-*/trades-*.parquet; do cp "$f" "data/$(basename "$(dirname "$f")").parquet"; done && ls data | wc -l && ~/py_dev/racinglines/.venv/bin/python -I score.py > scores.txt && wc -l scores.csv && tail -n 5 scores.txt
```

**Check:** the data count is N − F, scores.csv has more than 100 lines, and the tail shows a table. Each setting and
year is one block of 4 rows: update, hold, last and early. If the import fails, report the error and stop. Don't
install anything.

## Step 9: report back

Tell the owner the run is done, and give them two things to paste into the Claude thread:
1. The last 5 lines of `raw/kalshi-rule1-*/run.log`.
2. The path `<mac-folder>/kalshi-rule1/scores.csv`. Claude reads it from the Mac.

Don't interpret the results. Claude does the selection-corrected verdict, the report text and the publish.

## Footer (fill in)

- What ran: the unit name, the start and end time, N, F.
- How it was verified: the step 4 and step 5 check lines.
- Paths: the VM output folder name, `<mac-folder>/kalshi-rule1/scores.csv`.
- What was not checked: anything that was skipped.


## As run (20 Oct 2026, UTC)

| Time (UTC) | Event |
|---|---|
| 02:36:05 | Unit started on staging commit 34b18f5, read-only on the mirror |
| 02:56 | 40 runs done (20 settings × F1 2025 and 2026 rounds 1 to 16), **0 failures** |

Output folder `kalshi-rule1-20261020T023605Z`, scored on the Mac with the step 2 scorer, unchanged.

## Appendix: scripts as run

### Job script: rule1_job.sh

```bash
#!/usr/bin/env bash
# Rule 1 run: every saved Kalshi F1 sweep setting (model_runs kind=sweep, venue kalshi), 2025 and 2026 rounds 1-16,
# re-run with --trades-out, one process per setting so one failure doesn't stop the rest. Read-only on staging's mirror.
# A progress line every 5 minutes. Output: data/runs/kalshi-rule1-<UTC>/ (run.log, f1-<year>-<key>.log, f1-<year>-<key>/trades-<key>.parquet)
set -uo pipefail
set -a; . <env-file>; set +a
export DATABASE_URL="${DATABASE_URL%/*}/<staging-db>"
export PYTHONUNBUFFERED=1 RACINGLINES_PROGRESS_SEC=300 PGOPTIONS="-c default_transaction_read_only=on"
cd <app>
R=.venv/bin/racinglines
OUT=data/runs/kalshi-rule1-$(date -u +%Y%m%dT%H%M%SZ)
mkdir -p "$OUT/grids"
say() { echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) $*" | tee -a "$OUT/run.log"; }
T0=$SECONDS
( while sleep 300; do say "progress: $(( (SECONDS - T0) / 60 )) min elapsed"; done ) &
BEAT=$!
trap 'kill $BEAT 2>/dev/null' EXIT
say "kalshi rule-1 run from $(git rev-parse --short HEAD) on ${DATABASE_URL##*/} (read-only) -> $OUT"
.venv/bin/python - "$OUT/grids" <<'PY' || { say "STOP: could not build the grids"; exit 1; }
import json, os, sys
from sqlalchemy import create_engine, text
rows = create_engine(os.environ["DATABASE_URL"]).connect().execute(text(
    "SELECT DISTINCT ON (params->>'year', params->>'settings_key') params->>'year', params->>'settings_key', "
    "params->'settings' FROM model_runs WHERE kind = 'sweep' AND params->'settings'->>'venue' = 'kalshi' "
    "ORDER BY params->>'year', params->>'settings_key', id DESC")).fetchall()
for y, k, s in rows:
    json.dump([{"job": k, "settings": s}], open(os.path.join(sys.argv[1], f"{y}-{k}.json"), "w"))
print(f"{len(rows)} grids")
PY
N=$(ls "$OUT/grids" | wc -l); I=0; FAIL=0
say "$N settings x years to run"
for g in "$OUT/grids"/*.json; do
  I=$((I + 1)); b=$(basename "$g" .json); y=${b%%-*}; k=${b#*-}
  rounds=""; [ "$y" = 2026 ] && rounds="--rounds 1-16"
  say "start $I/$N f1-$y-$k"
  # shellcheck disable=SC2086
  if $R f1 sweep --year "$y" $rounds --no-fetch --grid "$g" --trades-out "$OUT/f1-$y-$k" > "$OUT/f1-$y-$k.log" 2>&1; then
    say "done $I/$N f1-$y-$k ($(( (SECONDS - T0) / 60 )) min)"
  else
    say "FAILED $I/$N f1-$y-$k: $(grep -m1 -oE 'ReadOnlySqlTransaction|Error: .{0,80}' "$OUT/f1-$y-$k.log")"; FAIL=$((FAIL + 1))
  fi
done
say "all done in $(( (SECONDS - T0) / 60 )) min, $N runs, failures: $FAIL"
```

