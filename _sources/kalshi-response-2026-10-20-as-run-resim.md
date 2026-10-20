# As run: Kalshi re-sim, trade by trade (step 2), 20 Oct 2026

*This is the runbook as written and run, with machine names, paths, database names and accounts redacted (shown as
`<...>`). The commands are otherwise unchanged. The outcome of each run is recorded at the end.*


**What this runs.** These are the published Kalshi sweeps, re-run with `--trades-out` (#275, on staging at 34b18f5),
so every trade is saved with the touch bid, ask, sizes and outcome:

- **F1 2025 and 2026:** the settings of sweep runs 9636 and 9637 (`settings_key` 11c96efd2255, checked in the cloud:
  the grid entry below hashes to that key).
- **NASCAR 2025 and 2026, MotoGP 2026:** each sport's baseline settings on Kalshi.

**Database.** Every connection is read-only (`PGOPTIONS=default_transaction_read_only=on`), the same way
`scripts/vm/nascar_calibration.sh` reads the staging mirror.

This job is an exception to the rule that staging VM jobs set `STAGING_DB=<demo-db>`. That rule exists
because writes break the mirror, and this job cannot write: the read-only guard refuses any write. The mirror holds
the same Kalshi data as the published runs 9636 and 9637, which is why the job reads from it. Nothing is saved (no `--save`) and nothing is downloaded
(`--no-fetch`). If the F1 sweep needs to price a stage that isn't stored, it stops with a read-only error rather than
writing to the mirror. In that case, send me the log line and I'll switch that part to `<demo-db>`.

**Output.** `<app>/data/runs/kalshi-resim-<UTC>/`, holding `run.log`, `<sport>-<year>.log`, and
`<sport>-<year>/trades*.parquet`.

Run the steps in order. Each check says what you should see.

**Step 1: open a shell on the VM.**

```
# LOCAL (Mac)
conda deactivate 2>/dev/null; cd <checkout> && bash scripts/deploy/vm.sh ssh
```

**Step 2: confirm the 34b18f5 staging deploy finished, so the `--trades-out` code is what runs, and that no other
sweep is running.**

```
# VM (production)
cd <app> && /usr/bin/git log --oneline -1 && systemctl list-units 'rl-*' --no-legend
```

Check: the first line starts with `34b18f5` (or a later staging commit), and no `rl-` unit is listed. If one is
listed, stop here: the VM runs one queue at a time.

**Step 3: confirm the mirror has the Kalshi links.**

```
# VM (production)
cd <prod-app> && sudo -u <app-user> <psql> -U <db-user> -d <staging-db> -tAc "SELECT c.code, s.year, count(*) FROM market_links ml JOIN races r ON r.id=ml.race_id JOIN events e ON e.id=r.event_id JOIN seasons s ON s.id=e.season_id JOIN competitions c ON c.id=s.competition_id WHERE ml.exchange='kalshi' AND c.code IN ('f1_wdc','nascar_cup','motogp_wc') GROUP BY 1,2 ORDER BY 1,2"
```

Check: production showed these counts at 01:45Z: f1_wdc 2025 = 1074, f1_wdc 2026 = 3256, motogp_wc 2026 = 310,
nascar_cup 2025 = 303, nascar_cup 2026 = 7910. The mirror should match or be slightly higher.

**Step 4: write the job script and the F1 grid.** The first part of this block writes the grid, the second part
writes the script, and the last line checks the script's syntax.

```
# VM (production)
sudo tee <tmp>/kalshi-resim-grid.json >/dev/null <<'EOF'
[{"job": "kalshi-11c96efd2255", "settings": {"cost":0.01,"fill":"through","seed":null,"sims":16000,"size":50.0,"skew":1.0,"kelly":null,"venue":"kalshi","widen":1.5,"max_pos":250.0,"variant":"gridq+pretrain+reset","bankroll":null,"min_edge":0.1,"info_skew":2.0,"max_stake":50.0,"ridge_team":2.0,"half_spread":0.02,"late_stages":["after FP3","after Quali"],"ridge_slope":6.0,"market_kinds":["race_win","race_podium","race_h2h","race_constructor_top","race_pole","race_top10"],"max_deployed":null,"max_disagree":0.15,"min_edge_h2h":0.05,"reset_weight":0.25,"taker_stages":["after FP1","after FP2","after FP3","after SQ","after Sprint","after Quali"],"teammate_corr":true,"driver_prior_n":3.0,"half_life_days":120.0,"min_volume_24h":50.0,"practice_prior":true,"stake_per_edge":250.0,"thin_edge_mult":null,"track_features":true,"finish_rho_scale":1.0,"min_edge_by_kind":null,"maker_min_volume_24h":null,"coherence_tol_by_kind":null}}]
EOF
sudo tee <tmp>/kalshi-resim.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
# Kalshi re-sim with --trades-out (read-only on staging's mirror). A progress line every 5 minutes.
set -uo pipefail
set -a; . <env-file>; set +a
export DATABASE_URL="${DATABASE_URL%/*}/<staging-db>"
export PYTHONUNBUFFERED=1 RACINGLINES_PROGRESS_SEC=300 PGOPTIONS="-c default_transaction_read_only=on"
cd <app>
R=.venv/bin/racinglines
OUT=data/runs/kalshi-resim-$(date -u +%Y%m%dT%H%M%SZ)
mkdir -p "$OUT"
say() { echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) $*" | tee -a "$OUT/run.log"; }
T0=$SECONDS
( while sleep 300; do say "progress: $(( (SECONDS - T0) / 60 )) min elapsed"; done ) &
BEAT=$!
trap 'kill $BEAT 2>/dev/null' EXIT
say "kalshi re-sim from $(git rev-parse --short HEAD) on ${DATABASE_URL##*/} (read-only) -> $OUT"
FAIL=0
run() { local name=$1; shift; say "start $name"; mkdir -p "$OUT/$name"
  if "$@" --trades-out "$OUT/$name" > "$OUT/$name.log" 2>&1; then say "done $name ($(( (SECONDS - T0) / 60 )) min)"
  else say "FAILED $name (see $name.log)"; FAIL=1; fi; }
run f1-2025 $R f1 sweep --year 2025 --no-fetch --grid <tmp>/kalshi-resim-grid.json
run f1-2026 $R f1 sweep --year 2026 --no-fetch --grid <tmp>/kalshi-resim-grid.json
run nascar-2025 $R strategy sweep nascar --year 2025 --venue kalshi --no-fetch
run nascar-2026 $R strategy sweep nascar --year 2026 --venue kalshi --no-fetch
run motogp-2026 $R strategy sweep motogp --year 2026 --venue kalshi --no-fetch
say "all done in $(( (SECONDS - T0) / 60 )) min, failures: $FAIL"
exit $FAIL
EOF
sudo chmod 755 <tmp>/kalshi-resim.sh && sudo chmod 644 <tmp>/kalshi-resim-grid.json && bash -n <tmp>/kalshi-resim.sh && echo script-ok
```

Check: the last line printed is `script-ok`.

**Step 5: start it as a transient unit.**

```
# VM (production)
sudo systemctl reset-failed rl-kalshi-resim 2>/dev/null; sudo systemd-run --unit=rl-kalshi-resim --uid=<app-user> <tmp>/kalshi-resim.sh && sleep 20 && journalctl -u rl-kalshi-resim --no-pager | tail -n 5
```

Check: you see `kalshi re-sim from 34b18f5 on <staging-db> (read-only)` and then `start f1-2025`. Send me
those lines. Leave the session; the unit keeps running after you disconnect.

**Step 6a: follow it live (optional).** This prints each start and done line, plus a progress line every 5
minutes. Ctrl-C stops the watching, not the job.

```
# VM (production)
sudo journalctl -fu rl-kalshi-resim
```

**Step 6 (later): check on it.**

```
# VM (production)
systemctl is-active rl-kalshi-resim; d=$(ls -1dt <app>/data/runs/kalshi-resim-* | head -1); tail -n 8 "$d/run.log"; ls -la "$d"/*/
```

Check: when it finishes, the unit shows `inactive` and run.log ends with `all done in N min, failures: 0`. Send me
the run.log tail and the parquet listing. I take the trades from there to the bucket and score them; the scoring
doesn't run on the VM.

**Rollback:** nothing to roll back, because the job only reads. To stop it early:
`sudo systemctl stop rl-kalshi-resim`. To clean up afterwards: `sudo rm <tmp>/kalshi-resim.sh <tmp>/kalshi-resim-grid.json`.

**Step 7 (only if a sport FAILED): show its log.** Run 1 (01:49Z) failed on f1-2026.

```
# VM (production)
d=<app>/data/runs/kalshi-resim-20261020T014914Z; sudo grep -n -iE "error|read-only|traceback" $d/f1-2026.log | head -n 5; sudo tail -n 25 $d/f1-2026.log
```

**Run 1 result (step 7, 01:57Z):** f1-2026 tried to INSERT a `model_runs` diagnostic for event 2026-17, which means
round 17's stage pricing isn't stored on the mirror. The published run 9637 (6 Oct) predates round 17 (11 Oct), so it
covers rounds 1 to 16 only. The read-only guard worked as intended and nothing was written.

**Step 8: re-run F1 2026 for rounds 1 to 16 only (read-only, reproduces run 9637).** Round 17 needs a writable
database and comes later as its own run. Wait for the coordinator's go before starting, because it holds the staging
merges while this runs.

```
# VM (production)
sudo sed -e 's/^run f1-2025 .*$//' -e 's/^run nascar-.*$//' -e 's/^run motogp-.*$//' -e 's/--year 2026 --no-fetch/--year 2026 --rounds 1-16 --no-fetch/' -e 's/kalshi-resim-\$(date/kalshi-resim-f1-2026-r1-16-$(date/' <tmp>/kalshi-resim.sh | sudo tee <tmp>/kalshi-resim-2.sh >/dev/null && sudo chmod 755 <tmp>/kalshi-resim-2.sh && grep -n '^run ' <tmp>/kalshi-resim-2.sh && bash -n <tmp>/kalshi-resim-2.sh && echo script-ok
```

Check: exactly one `run f1-2026 ... --year 2026 --rounds 1-16 --no-fetch ...` line, then `script-ok`.

```
# VM (production)
sudo systemctl reset-failed rl-kalshi-resim 2>/dev/null; sudo systemd-run --unit=rl-kalshi-resim --uid=<app-user> <tmp>/kalshi-resim-2.sh && sleep 60 && sudo journalctl -u rl-kalshi-resim --no-pager | tail -n 4
```

Check: you see `done f1-2026` and `all done ..., failures: 0`. If it still says running, check again with step 6a.
Then paste this, so I can compare it with run 9637:

```
# VM (production)
d=$(ls -1dt <app>/data/runs/kalshi-resim-f1-2026-r1-16-* | head -1); sudo grep -E "^(update|hold|last|early|maker)" $d/f1-2026.log; ls -la $d/f1-2026/
```

**Run 2 result (02:02Z):** F1 2026 rounds 1 to 16. All nine totals match run 9637 to the cent (update +3,968.52 on
$7,744), and the trades file was written. Step 2 has all five trade files.

**Step 9: copy the trade files to the Mac.** Run this in a Mac terminal, not on the VM. It copies about 450 KB.

```
# LOCAL (Mac)
mkdir -p <mac-folder>/kalshi-resim && cd <mac-folder>/kalshi-resim && gcloud compute scp --recurse --project <project> --zone <zone> --tunnel-through-iap "<vm>:<app>/data/runs/kalshi-resim-20261020T014914Z" "<vm>:<app>/data/runs/kalshi-resim-f1-2026-r1-16-*" . && find . -name '*.parquet' | sort
```

Check: five parquet files are listed: four in the first folder (its f1-2026 folder is empty because that run
failed) and the f1-2026 file in the `-r1-16-` folder. I read them from `<mac-folder>/kalshi-resim`.

## As run (20 Oct 2026, UTC)

| Time | Event |
|---|---|
| 01:49:14 | Run 1 started on staging commit 34b18f5, read-only |
| 01:50:03 | F1 2025 done. All nine strategy totals match the published sweep run 9636 (update +1,256.67 on $5,010) |
| 01:50:23 | F1 2026 **failed**: round 17 had no stored stage pricing, and the read-only guard refused the write |
| 01:50:36 | NASCAR 2025 done |
| 01:52:34 | NASCAR 2026 done |
| 01:52:45 | MotoGP 2026 done. Run 1 ended with 1 failure |
| 01:59:21 | **Run 2**: F1 2026 rounds 1 to 16, done. All nine totals match run 9637 (update +3,968.52 on $7,744) |

Total run time: about 4.5 minutes of wall-clock time (run 1 took 3.5 minutes, run 2 under 1 minute). The two F1 sweeps reused stored stage pricings.
