# RoadSense V2V Project Status Overview

**Last Updated:** March 4, 2026 (Run 005 training launched on EC2)
**Purpose:** Single source of truth for current project status and priorities.
**Audience:** AI agents and developers navigating this codebase.

---

## CURRENT PHASE: Phase 6.9 — Run 005 Training In Progress

```
================================================================================
                         READ THIS FIRST - PROJECT STATUS
================================================================================

  ARCHITECTURE CORRECTIONS — ALL COMPLETE (Feb 26-28, 2026)

  ✅ Phase A: Cone Filter (Python observation pipeline)
  ✅ Phase B: Continuous Action Space (Box(1,) replaces Discrete(4))
  ✅ Phase C: Mesh Relay in Emulator (multi-hop simulation)
  ✅ Phase D: Mesh in ConvoyEnv (simulate_mesh_step integration)
  ✅ Phase E: Firmware Mesh Relay (ConeFilter + MeshRelayPolicy, 18 tests)
  ✅ Phase F: Recording Strategy Update (mesh-aware ego-only protocol)
  ✅ Phase G: Real Data Collection (Recording #2 + extra driving, GO verdict)

  RUN 003 — COMPLETED, POLICY FAILED + POST-MORTEM DONE (March 2, 2026)
  ✅ Run 003: 10M steps, avg_reward=-6511, 6 root causes fixed (169 tests)

  RUN 004 — COMPLETED, FIRST SUCCESSFUL MODEL (March 3, 2026)
  ✅ Run 004: 10M steps, avg_reward=+352, 81% behavioral success
  ⚠️ 19% collision rate (ALL 38 spawn-time, steps 3-32 — not model intelligence)
  ⚠️ 0% V2V hazard reaction (model is passive, lets CF model handle it)
  ⚠️ Eval matrix NOT used (cloud script was missing flags)

  RUN 005 — IN PROGRESS ON EC2 (March 4, 2026)
  Fixes for Run 004 issues:
  ✅ Fix 1: 30-step warmup in reset() prevents spawn-time collisions
  ✅ Fix 2: Early reaction reward (+2.0) incentivizes V2V-based braking
  ✅ Fix 3: Eval matrix flags wired into cloud script (15 buckets × 10 episodes)
  ✅ Fix 4: HAZARD_PROBABILITY 0.3 → 0.5 (more training exposure)
  ✅ Fix 5: Ego route-end guard (check is_vehicle_active BEFORE get_vehicle_state)
  ✅ Fix 6: run_docker.sh array quoting (cmd="$*" → cmd=("$@"))
  ✅ Unit tests: 201 passing

  ⚠️ CRITICAL: Board Y-axis is forward (braking axis, not X).
     Always use --forward-axis y with analyze_convoy_recording.py

CURRENT STATUS:
  ✅ Run 005 training launched on EC2 (c6i.xlarge, ~8h)
  ✅ Results will land at s3://saferide-training-results/cloud_prod_005/
  🔜 H5 (post-training): Sim-to-real validation against real recordings (777s)

NEXT STEPS (after Run 005 completes):
  1. Download results from S3, analyze metrics
  2. H5: Validate trained model against real convoy recordings (offline replay)
  3. Quantization (TFLite INT8 for ESP32)
  4. Deploy on ESP32
  5. Live SUMO demo for professors (add --gui to eval script)

KEY DOCUMENTS:
  - **Run 004 Plan:** 10_PLANS_ACTIVE/RUN_004_HAZARD_TARGETING_AND_SOURCE_REACTION_EVAL_PLAN.md (v2.2)
  - **Arch Correction Progress:** 10_PLANS_ACTIVE/MESH_AND_ACTION_ARCHITECTURE_CORRECTION_PROGRESS.md
  - **Architecture:** 00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md
  - **Phase 6 Pipeline:** 10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md

DO NOT:
  - Train without mesh relay in the emulator
  - Train without cone filtering
  - Train with Discrete(4) action space — must be continuous Box(1,)
  - Train on pure synthetic data (professor rejected this)
  - Use uncalibrated emulator params for production
  - Hardcode peer slots or use fixed observation space
  - Run analyzer without --forward-axis y (board Y = forward)
  - Mix synthetic base scenarios into dataset_v3 (100% base_real only)
  - Use route randomization flags with base_real (single route, silently no-ops)
  - Use setSpeed() to "hold" current speed for action=0 — must release to CF model
  - Hardcode hazard target vehicle ID — use uniform_front_peers for training
  - Rely on collision-only success_rate — use behavioral_success_rate instead
  - Set UNSAFE_DIST > 10m (real convoy following distance is ~10m)
  - Use MAX_DECEL = 5.0 — real braking is 8.0 m/s² (measured)
  - Compute reward distance from SUMO ground truth — must use mesh-visible peers
  - Run deterministic eval matrix with hazard injection disabled
  - Skip source_reaction_summary in eval output — required for Run 004+

================================================================================
```

---

## Implementation Roadmap

```
COMPLETED (Runs 001-004 + All Architecture)        NOW: RUN 005 ON EC2
─────────────────────────────────────────────────────────────────────────────────

[Phase 1-4: ML Architecture]    [ARCH CORRECTION - COMPLETE]   [PREP - COMPLETE]
  ✅ ConvoyEnv Dict Obs            ✅ A. Cone Filter (Python)     ✅ Emulator validated
  ✅ Deep Sets Policy              ✅ B. Continuous Actions        ✅ base_real/ created
  ✅ Emulator Causality Fix        ✅ C. Emulator Mesh Relay      ✅ dataset_v3 generated
  ✅ Training Pipeline             ✅ D. ConvoyEnv Mesh            ✅ Eval n=1-5 enforced
  ✅ Run 001 (80%, n=2 only)       ✅ E. Firmware Mesh             ✅ Smoke train passed
  ✅ Run 002 (BASELINE ONLY)       ✅ F. Recording Strategy
  ✅ EC2 Training AMI              ✅ G. Real Data Collection    [Run 003: FAILED — 6 bugs fixed]
  ✅ Convoy Recording #1                                        [Run 004: FIRST SUCCESS]
  ✅ Emulator calibrated                                          ✅ avg_reward=+352, 81% success
  ✅ Recording #2 (GO) + Extra                                    ⚠️ 19% spawn collisions
                                                                  ⚠️ 0% V2V reaction
[Phase H: PRE-TRAIN — DONE]
  ✅ H0: MAX_DECEL → 8.0                                       [Run 005: IN PROGRESS]
  ✅ H4: Mesh-visible reward                                      ✅ Warmup prevents spawn collisions
  ✅ H1: Hazard source diversity                                  ✅ Early reaction reward (+2.0)
  ✅ H2: Source reaction eval                                     ✅ Eval matrix wired (15 buckets)
  ✅ H3: Eval matrix n=1..5                                       ✅ Hazard prob 0.5 (was 0.3)
  ✅ 201 unit tests pass                                          ✅ Ego route-end guard
                                                                  ► Training on EC2 (~8h)

CURRENT FOCUS: Run 005 Training → Analyze → H5 Sim-to-Real
─────────────────────────────────────────────────────────────────────────────────
  ✅ Run 004 completed: +352 avg reward, 81% behavioral success (first working model!)
  ✅ Run 005 fixes: spawn warmup, V2V reaction reward, eval matrix, hazard 50%
  ✅ Infra fixes: run_docker.sh quoting bug, ego route-end guard
  ► Run 005 training on EC2 (10M steps, dataset_v3, all corrections active)
  ○ H5: Sim-to-real validation against real recordings
  ○ Quantize INT8 for ESP32
  ○ Deploy on ESP32
  ○ Live SUMO demo for professors

  SEE: docs/10_PLANS_ACTIVE/RUN_004_HAZARD_TARGETING_AND_SOURCE_REACTION_EVAL_PLAN.md (v2.2)
```

---

## EC2 Training Procedure (PROVEN — Use This Exactly)

Every single training run (002-005) had launch issues. This procedure is the one that
actually works. Do NOT deviate. Do NOT paste run_training.sh into EC2 user-data.

### Step 1: Launch EC2 Instance (AWS Console, il-central-1)

| Setting | Value |
|---------|-------|
| AMI | My AMIs → `roadsense-training-v1` (`ami-03a3037588b0f34f2`) |
| Instance type | `c6i.xlarge` |
| Key pair | `SafeRideKey` |
| Security group | Existing (SSH port 22) |
| IAM instance profile | `SafeRide-Trainer-Role` |
| User data | **LEAVE EMPTY** |

### Step 2: SSH In

```bash
ssh -i ~/Downloads/saferide-key.pem ubuntu@<PUBLIC_IP>
```

### Step 3: Setup + Run (inside tmux)

```bash
tmux new -s train
cd /home/ubuntu/work
git config --global --add safe.directory /home/ubuntu/work
git pull origin master
chmod +x ml/run_docker.sh
```

Verify code landed (adapt grep patterns to match the current run's changes):
```bash
grep "cloud_prod_0" ml/scripts/cloud/run_training.sh   # check RUN_ID
```

Disable auto-shutdown and git-pull in the script (we already pulled manually):
```bash
sed -i 's|shutdown -h now|echo "DONE — shutdown skipped"|' ml/scripts/cloud/run_training.sh
sed -i 's|<YOUR_PAT_HERE>|SKIP|' ml/scripts/cloud/run_training.sh
sed -i 's|git pull origin master|echo "SKIP git pull — already pulled"|' ml/scripts/cloud/run_training.sh
sed -i 's|git remote set-url origin "https://${GITHUB_PAT}@|echo "SKIP remote URL change" #|' ml/scripts/cloud/run_training.sh
sed -i 's|git remote set-url origin https://github.com|echo "SKIP remote URL restore" #|' ml/scripts/cloud/run_training.sh
```

Run the script:
```bash
sudo bash ml/scripts/cloud/run_training.sh
```

(`sudo` needed because the script writes to `/var/log/training-run.log` via `exec`.)

### Step 4: Monitor (optional)

Open a second tmux pane (`Ctrl+B` then `%`):
```bash
tail -f /var/log/training-run.log
```

Detach tmux: `Ctrl+B` then `D`. Disconnect SSH. Safe to close laptop.
Training runs ~8h. S3 upload happens automatically when training finishes.

### Step 5: After Training

Results auto-upload to `s3://saferide-training-results/<RUN_ID>/`.
**Stop the instance from the AWS Console** (it does NOT auto-shutdown with the sed fix).

### Why This Works (And Previous Attempts Failed)

| Run | Failure | Root Cause |
|-----|---------|------------|
| 002 | S3 upload never ran | `set -euo pipefail` killed script before cleanup. Fixed with `trap cleanup EXIT`. |
| 003 | Training produced garbage | 6 code bugs (action release, reward thresholds, etc). Policy failures, not infra. |
| 004 | Step 5/7 SyntaxError | `run_docker.sh` used `cmd="$*"` (string flattening) + unquoted `$cmd`. Multi-line `python3 -c` arg got word-split. Fixed with `cmd=("$@")` + `"${cmd[@]}"`. |
| 005a | Instant SSH disconnect | Ran `sudo bash run_training.sh` which still had `shutdown -h now`. First error triggered cleanup trap → shutdown. Fixed by sed-disabling shutdown before running. |
| 005b | TraCIException mid-episode | V001 reached end-of-route. `step()` called `get_vehicle_state()` BEFORE checking `is_vehicle_active()`. Fixed by adding early exit guard. |

**Key principles:**
1. **Never use user-data** — SSH in and run manually so you can see errors in real-time.
2. **Pull code manually** before running the script — avoids PAT/auth issues inside the script.
3. **Disable shutdown** with sed — prevents the instance from vanishing when something fails.
4. **Use tmux** — SSH disconnect won't kill training.
5. **Always verify code landed** with grep before running — catches missed pushes.

---

## Recent Achievements

### March 4, 2026 - Run 005 Training Launched on EC2

**Run 005 fixes (addressing Run 004 issues):**
- **Fix 1 — Spawn warmup:** Added 30-step warmup in `convoy_env.py:reset()` after ego spawn. SUMO's CF model stabilizes the convoy before RL takes over. Prevents all 38 spawn-time collisions seen in Run 004.
- **Fix 2 — V2V early reaction reward:** New `_early_reaction_bonus()` in `reward_calculator.py`. Gives +2.0 reward when model brakes proactively (decel > 0.5 m/s²) while in safe zone (>10m) AND a braking peer is detected via mesh. Outweighs mild comfort penalty, making proactive V2V-based braking optimal.
- **Fix 3 — Eval matrix flags:** `run_training.sh` now passes `--eval_use_deterministic_matrix --eval_matrix_peer_counts "1,2,3,4,5" --eval_matrix_episodes_per_bucket 10`. Generates ~150 eval episodes covering all 15 (n, source_rank) buckets.
- **Fix 4 — Hazard probability:** `HAZARD_PROBABILITY` raised from 0.3 to 0.5. Model now sees hazards in 50% of training episodes (was 30%).
- **Fix 5 — Ego route-end guard:** `step()` now checks `is_vehicle_active(V001)` immediately after `sumo.step()`, BEFORE calling `get_vehicle_state()`. If V001 reached end-of-route, returns empty observation + truncated=True. Previously crashed with `TraCIException: Vehicle 'V001' is not known`.
- **Fix 6 — run_docker.sh quoting:** Changed `cmd="$*"` (string) to `cmd=("$@")` (array) and `$cmd` to `"${cmd[@]}"`. Fixes word-splitting of `python3 -c "multi-line code"` in pass-through mode. This was the Run 004 cloud script failure.
- **Unit tests:** 201/201 passing (up from 194).
- **Expected improvements over Run 004:** collision rate <5% (was 19%), hazard reaction >50% (was 0%), eval coverage 15/15 buckets (was random).
- **Results will land at:** `s3://saferide-training-results/cloud_prod_005/`

### March 3, 2026 - Run 004 Results: First Successful Model

- **Run 004 result:** avg_reward=+352, 81% behavioral success rate — a massive jump from Run 003's -6511.
- **Collision analysis:** 38 collisions (19% rate), but ALL 38 are spawn-time (steps 3-32, avg step 14.4). Zero mid-episode collisions. This is an initialization problem, not a model intelligence failure.
- **V2V reaction:** 0% reaction to hazard messages. Model survives hazards passively via SUMO's Krauss CF model. The reward structure made active braking sub-optimal (comfort penalty outweighed any benefit).
- **Eval coverage gap:** Deterministic eval matrix (H3) was not used because cloud script was missing the flags. Only ~24% of episodes received hazards (random probability).
- **Verdict:** Model works but needs three fixes before thesis-ready (spawn warmup, reaction reward, eval matrix). These became Run 005.

### March 3, 2026 - H3 Dry-Run Gate Complete (Run 004 Launch-Ready)
- Deterministic eval matrix dry-run executed in Docker on `dataset_v3/base_real`.
- Coverage result: **15/15 non-empty `(n, source-rank)` buckets** (gate requirement satisfied).
- Artifact: `ml/models/runs/rs_smoke/eval_matrix_dry_run_after_planner_fix.json`.
- Additional H3 planner fix merged: rank targets now rotate across same-peer-count scenarios, preventing scenario-lock bucket starvation.
- Unit tests updated and passing: **194/194** (`pytest tests/unit/ -q`).
- Note: strict `episodes_per_bucket=10` can still underfill far-rank buckets due dynamic convoy ordering; non-empty coverage gate is complete.

### March 2, 2026 - Phase H Complete: Run 004 Pre-Training (191/191 tests)
- **Plan review + expansion:** Original 3-phase plan (H1/H2/H3) expanded to 6 phases after architecture review identified additional gaps.
- **H0: MAX_DECEL 5.0 → 8.0 m/s²:** Real convoy recording measured -8.63 m/s² emergency braking. Model's action space now covers full physical capability. Comfort thresholds (0.5/3.0/4.5) unchanged — human-centric, independent of vehicle max.
- **H4: Reward mesh-visibility gating:** `_calculate_min_distance_and_closing_rate()` was using ALL SUMO peers (ground truth). Now uses mesh-visible peers from `received_map` only. Ego no longer penalized for not reacting to peers it cannot see.
- **H1: Hazard injector source diversity:** 4 strategies (nearest, uniform_front_peers, fixed_vehicle_id, fixed_rank_ahead). Training default: `uniform_front_peers`. Per-episode reset overrides for deterministic eval.
- **H2: Source-specific reaction eval:** `REACTION_DECEL_THRESHOLD = 0.5 m/s²`. Per-episode: hazard_source_id, reaction_time_s, hazard_message_received_by_ego. Aggregated `source_reaction_summary` keyed by (peer_count, source_rank).
- **H3: Deterministic eval matrix:** Shared `ml/eval_matrix.py` module. Generates episodes for all (n, source_rank) buckets. Coverage validator fails eval if any bucket is empty.
- **Architect review:** 5 bugs found across H2 and H3 — all fixed:
  - H2: reception check incorrectly gated on `not reaction_detected`
  - H2: reaction detection started before confirmed injection
  - H2: schema mismatch between train_convoy and evaluate_model
  - H3: missing `--no_hazard_injection` incompatibility guard
  - H3: coverage failure destroyed partial metrics before saving
- **Key decisions documented:** MAX_DECEL=8.0, cone 45° confirmed, double cone filter is correct (two purposes), SUMO CF model handles chain reaction braking, reaction metric = RL command only (not CF decel).
- **New files:** `ml/eval_matrix.py`, `ml/tests/unit/test_eval_matrix.py`
- **Test count:** 191/191 pass (up from 169 after Run 003 fixes).
- **Next:** dry-run eval matrix, push to git, launch Run 004 on EC2.
- **Plan doc:** `docs/10_PLANS_ACTIVE/RUN_004_HAZARD_TARGETING_AND_SOURCE_REACTION_EVAL_PLAN.md` (v2.2)

### March 2, 2026 - Run 003 Post-Mortem: 6 Root Causes Fixed (169/169 tests)
- **Run 003 result:** avg_reward=-6511 across 184/200 episodes. Policy trapped in penalty loop. 0 collisions but 100% truncation.
- **RC-1 (THE KILLER BUG):** `ActionApplicator.apply()` called `set_vehicle_speed(V001, current_speed)` for action=0, permanently overriding SUMO's car-following model. Ego could only decelerate, never recover. **Fix:** action <= 0.02 now calls `release_vehicle_speed()` (TraCI `setSpeed(-1)`) to hand control back to Krauss CF model.
- **RC-2:** `PENALTY_MISSED_WARNING = -3` fired every step when distance < 15m regardless of whether gap was closing or opening. Combined with `REWARD_UNSAFE = -5` = -8/step trap. **Fix:** gated on `closing_rate > 0.5 m/s` (gap must be shrinking AND ego not braking).
- **RC-3:** `success_rate = (episodes - collisions) / episodes` showed 100% with -6511 avg reward. **Fix:** added `behavioral_success_rate` (requires no collision AND reward > -1000) and `pct_time_unsafe`.
- **RC-4:** `UNSAFE_DIST = 15m` too aggressive for real convoy following. **Fix:** lowered to 10m (SAFE_MIN: 20→15m, SAFE_MAX: 40→35m).
- **RC-5:** Hazard injector hardcoded `V002` target. **Fix:** rewrote to dynamically find nearest peer.
- **RC-6:** `test_convoy_env.py` had wrong mock patch paths (`envs.convoy_env.*` vs `ml.envs.convoy_env.*`) due to `pytest.ini` dual pythonpath. 22 tests were silently broken. **Fix:** corrected all patch targets.
- **Test count:** 169/169 pass (up from 133 + 22 broken + 14 missing edge cases).
- **New edge case tests:** zero-peer observation, behind-ego cone filter, stale message exclusion, action sensitivity (release vs max brake).
- **Next:** push to git, launch Run 004 on EC2.
- **Findings doc:** `docs/10_PLANS_ACTIVE/RUN_003_FINDINGS_FOR_ARCHITECT_REVIEW.md`

### March 2, 2026 - Training Run 003 Launched on EC2
- **Dry-run validated:** SSH'd into EC2, ran every step manually before training. Verified IAM credentials, S3 upload (dummy file test), dataset generation, eval peer count enforcement.
- **Run 003 config:** 10M steps, dataset_v3 (100% real-grounded), mesh relay + continuous actions + cone filter all active, 200-episode eval (20 per scenario, hazards ON).
- **`run_training.sh` updated for Run 003:** New dataset paths, added `fix_eval_peer_counts.py` step, added dataset verification step, training log uploaded to S3 for remote debugging.
- **Run 002 S3 failure documented:** `set -euo pipefail` killed script before upload. Fixed with `trap cleanup EXIT` + `set +e` pattern. Dry-run confirmed fix works.
- **Key lesson:** First run with new script should always be dry-run (SSH + manual steps) before trusting user-data automation.
- **Training running in tmux** on c6i.xlarge, estimated ~8h based on prior runs.
- **Results will land at:** `s3://saferide-training-results/cloud_prod_003/`
- **Professor PoC plan confirmed:** Show training curve (reward over time) + live SUMO-GUI demo with trained model. Current setup supports both; only gap is wiring `--gui` flag to evaluation script (post-training task).

### March 1, 2026 - Prep Work Complete — READY FOR RUN 003
- **Step 1 (Emulator validation):** `emulator_params_measured.json` validated against Recording #2. Burst mean updated from 1.245 to ~1.6. Metadata annotated with Rec #2 cross-validation. Backup created.
- **Step 2 (Convoy → SUMO):** Real convoy data processed into `ml/scenarios/base_real/`.
  - V002/V003 trajectories extracted from V001 RX log (new feature in `analyze_convoy_recording.py`).
  - OSM exported for recording area (rural road outside Deir Hanna, near Sonol station by Nahal Tzalmon).
  - Network generated via NetConvert. Route locked to single driven edge `-635191227` (single-lane, matches recording).
  - 6 vehicles: V001 (ego, rear) + V002-V003 (real trajectories) + V004-V006 (synthetic, ahead of V003).
  - Board placement: front windshield of each car (production-realistic, good line-of-sight).
- **Step 3 (dataset_v3):** 25 train + 10 eval scenarios generated from 100% base_real (no synthetic mixing).
  - Train: `peer_drop_prob=0.4` gives natural n=1-5 distribution.
  - Eval: deterministic n=1,2,3,4,5 (exactly 2 each) via `fix_eval_peer_counts.py`.
  - Blocker resolved: eval dropout with seed=42 only produced 1x n=5. Fixed by adding `--eval_peer_drop_prob 0.0` to `gen_scenarios.py` (eval keeps all peers, fixer trims to exact counts).
  - 1000-step smoke train passed (run_20260301_025156, hazard injection ON).
  - 25 unit tests pass (gen_scenarios + fix_eval_peer_counts).
- **New scripts:** `ml/scripts/process_convoy_to_sumo.py`, `ml/scripts/fix_eval_peer_counts.py`, `ml/scripts/validate_emulator_params.py`.
- **Architect review:** dataset_v3 strategy approved. Key decisions: no synthetic merge, no route randomization (single route), 200-episode eval (10 scenarios x 20 episodes = 40 per n-value).
- **Plan:** `docs/10_PLANS_ACTIVE/PHASE_6_PREP_WORK_PLAN.md` — ALL STEPS COMPLETE.

### Feb 28, 2026 - Convoy Recording #2 + Extra Data — GO VERDICT
- **Recording #2 (hazard protocol):** 195.5s ego-only mesh recording. Hard braking -8.63 m/s² raw (0.88g). Mesh relay confirmed (953 relayed packets, max hop=2). V001 as ego.
- **Extra regular driving:** 581.6s (~10 min) regular driving without protocol. Adds distance diversity (21m vs 14m spacing).
- **Combined dataset:** ~777s, 19,883 TX + 15,860 RX rows.
- **Critical discovery: Accelerometer axis mapping.** Board Y-axis is forward (braking direction), not X. Analyzer was checking wrong axis and initially reported false NO-GO. Added `--forward-axis y` flag to `analyze_convoy_recording.py`.
- **Mesh relay validated in the field:** V003 data reaches V001 through V002 relay (hop=2). V003→V001 direct PDR is 0.752 but relay compensates.
- **All architecture correction phases (A-G) now complete.** Data collection milestone achieved.
- **Data locations:** `Convoy_recording_02282026/` (main), `Convoy_extra_data_02282026/` (extra).
- **Analysis:** `ml/data/convoy_analysis_site/` and `ml/data/convoy_analysis_extra/`.

### Feb 26-27, 2026 - Architecture Correction Phases A-F Complete
- **All six phases implemented with TDD:**
  - Phase A: Cone filter in Python observation pipeline (13 tests)
  - Phase B: Continuous action space Box(1,) (36 tests)
  - Phase C: Mesh relay in ESP-NOW emulator (9 tests)
  - Phase D: Mesh in ConvoyEnv (34 tests)
  - Phase E: Firmware mesh relay (18 tests)
  - Phase F: Recording strategy updated (ego-only mesh protocol)

### Feb 26, 2026 - CRITICAL Architecture Correction (Professor Meeting)
- **THREE fundamental issues identified:**
  1. **Mesh not implemented** — vehicles only broadcast own data, no relay. PackageManager is dead code. Emulator is point-to-point.
  2. **Action space wrong** — Discrete(4) with hardcoded values defeats purpose of RL. Must be continuous Box(1,) so model learns deceleration percentage.
  3. **Cone filtering missing** — not just observation filter but determines what each vehicle RELAYS in the mesh.
- **Run 002 reclassified as BASELINE ONLY** — trained without mesh, without cone filter, with wrong action space.
- **Implementation plan created:** `MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md` with 6 phases (A-F), full TDD, dependency graph.
- **Recording strategy corrected:** Professor says record only from ego — mesh delivers everything.
- **Augmentation clarified:** Take 3-vehicle recording, add virtual ego behind rear vehicle, ego gets rear vehicle's relayed data.

### Feb 20, 2026 - EC2 Training AMI Created + Comparison Strategy Decided
- **AMI created:** `ami-03a3037588b0f34f2` (`roadsense-training-v1`) in il-central-1.
- **Contents:** Ubuntu 22.04 + Docker + `roadsense-ml:latest` image pre-built. Launch-to-training: ~2 min.
- **Cloud scripts added:** `ml/scripts/cloud/setup_ami.sh` (AMI builder) + `run_training.sh` (runtime user-data template).
- **Execution plan updated:** Run 002 config documented (10M steps, measured emulator params, dataset_v2 with variable n).
- **Training comparison strategy decided:** Train on synthetic base (Run 002) AND real convoy base (Run 002b), compare results. Addresses professor's real-data requirement with empirical evidence.
- **Repo clarification:** Cloud scripts corrected to use `roadsense-team/roadsense-v2v` repo, `master` branch.

### Feb 20, 2026 - RTT Drive Characterization Complete (Deir Hanna → Tiberias)
- **Field log captured:** `/home/amirkhalifa/RoadSense2/rtt_log.csv` with 20k+ rows and valid GPS/IMU/mag columns.
- **Measured params generated:** `roadsense-v2v/ml/espnow_emulator/emulator_params_measured.json`.
- **Calibration method:** network stats from drive-core interval; sensor-noise stats from a clean stationary window to avoid mixed-location stop bias.
- **Representative network metrics (drive-core):** RTT mean 8.0ms, p95 12ms; one-way mean 4.0ms.
- **Representative loss metrics (drive-core):** overall 33.4%, with burst-loss behavior captured for emulator robustness.
- **n-element guardrail applied:** removed `observation.monitored_vehicles` from RTT-generated params output to avoid fixed-peer confusion with Deep Sets training.

### Feb 15, 2026 - 2-Board Mode 1 Validation Pass (Hardware Session)
- **Button workflow verified end-to-end:** both boards successfully started/stopped logging from GPIO button presses.
- **Artifacts validated:** `V001_tx_003.csv`, `V001_rx_003.csv`, `V002_tx_001.csv`, `V002_rx_001.csv` in `RXTX_test_20261402/`.
- **CSV integrity checks passed:** 16-column headers/rows, monotonic local timestamps, non-zero magnetometer columns in TX/RX files.
- **Session result:** Mode 1 RX/TX logging reliability confirmed for field workflow.
- **Carry-forward note:** RTT capture + `emulator_params_measured.json` regeneration still pending.

### Feb 12, 2026 - Field Readiness Implementation Pass (Code + Tests)
- **Production mag integration complete:** `main.cpp` now fills `msg.sensors.mag` from QMC5883L with graceful fallback to zeros on failure.
- **Reusable mag driver added:** `hardware/src/sensors/mag/QMC5883LDriver.*` with shared-`Wire` initialization policy.
- **RTT firmware refined:** sender now logs mag data; `RTTPacket` extended with mag and GPS quality fields while preserving 90-byte packet size.
- **Mode 1 logging expanded:** TX/RX CSV upgraded from 10 to 16 columns (accel + gyro + mag).
- **Analysis pipeline updated:** `analyze_rtt_characterization.py` parses mag columns and outputs `sensor_noise.mag_std_ut` + mag-derived `heading_std_deg`.
- **Test updates complete:** related RTT/DataLogger/sensor tests updated, plus new QMC driver test suite added; user reports **92 tests passing**.
- **Remaining work:** regenerated measured emulator params + field-day checklist.

### Jan 24, 2026 - Real Data Pipeline Plan Finalized
- **Two-recording strategy confirmed:** RTT (2 cars) for network params, Convoy (3 cars) for trajectory base
- **RTT enhancement planned:** Add GPS + magnetometer logging for complete sensor noise characterization
- **Clock sync analysis:** GPS NMEA time is ±100-500ms (OK for trajectory alignment, not for latency measurement)
- **Data flow clarified:** SUMO provides ground truth → `to_v2v_message()` converts → Emulator adds noise
- **Key finding:** `accel_y`, `accel_z`, `gyro_*` are hardcoded in sim, BUT observation builder only uses `accel_x` so this is OK
- **Plan created:** `PHASE_6_REAL_DATA_PIPELINE.md` with 6 sub-phases to production model
- **Diagrams:** Use case and data flow diagrams updated (user completed)

### Jan 23, 2026 - Professor Feedback Alignment (Docs + Plan)
- **Hybrid data strategy documented:** real 3-car recording is the **mandatory base** for augmentation
- **Cone filtering clarified:** V001-only step inside AI inference before Observation Builder (front FOV)
- **Use-case boundary clarified:** sensors treated as external actors (firmware boundary)
- **Next work:** update draw.io diagrams + implement cone filter in embedded + sim

### Jan 22, 2026 - Scenario Generator Enhanced for Variable Peer Counts
- **gen_scenarios.py enhanced:** Added 4 new CLI arguments for peer variation:
  - `--peer_drop_prob`: Probability of dropping each non-V001 vehicle (0.0-1.0)
  - `--min_peers`: Minimum peers to keep after dropout
  - `--route_randomize_non_ego`: Randomize route assignment for non-V001 vehicles
  - `--route_include_v001`: Include V001 in route randomization
- **New features:** `apply_peer_dropout()`, `randomize_routes()`, `AugmentationConfig` dataclass
- **Manifest enhanced:** Now includes `augmentation_config` and `peer_count_distribution`
- **Tests:** 17 new unit tests added; all tests pass
- **SCENARIO_BUILDING.txt updated:** Spec corrected for n ∈ {1,2,3,4,5} peer counts
- **Impact:** Unblocks dataset_v2 generation for Training Run 002

### Jan 19, 2026 - LR Mode Enabled + Channel 6 Standardized
- **LR mode enabled:** ESP-NOW LR mode + max TX power applied in transport + RTT firmware
- **Channel updated:** Default channel set to 6 to avoid home WiFi interference (routers often on 1)
- **Validation:** Builds + unit tests pass; 5m baseline RTT test passes
- **Pending:** Extended range tests (10m/15m/20m) before updating emulator params

### Jan 17, 2026 - CRITICAL FINDING: Fixed n=2 Training Data
- **Issue Discovered:** All 25 scenarios in `dataset_v1` had exactly 3 vehicles (V001 + 2 peers)
- **Impact:** Model trained on constant n=2 for entire 5M timesteps; behavior for n≠2 is untested
- **Resolution:**
  - Removed `monitored_vehicles` from emulator params (was misleading, not used by ConvoyEnv)
  - Added Phase 6 to N_ELEMENT_IMPLEMENTATION_PLAN.md with variable-n requirements
  - Next training run (Run 002) MUST use variable peer counts n ∈ {1,2,3,4,5}
- **NOT a catastrophic failure:** Deep Sets architecture is correct and should generalize
- **Details:** See `N_ELEMENT_IMPLEMENTATION_PLAN.md` Phase 6

### Jan 16, 2026 - Cloud Training Run 001 COMPLETE
- **Training completed:** 5M timesteps on c6i.xlarge (il-central-1)
- **Results:**
  - Reward: -478 → +478 (model learned successfully)
  - Evaluation: **80% success rate** (16/20 episodes)
  - 4/5 eval scenarios pass consistently; 1 edge case identified (late-spawn)
- **⚠️ CAVEAT:** Model only trained/evaluated with n=2 peers (see Jan 17 finding)
- **Model still improving** at 5M steps - consider extended run
- **Artifacts:** `s3://saferide-training-results/cloud_prod_001/`
- **Details:** See `20_KNOWLEDGE_BASE/CLOUD_TRAINING_RUN_001_RESULTS.md`

### Jan 15, 2026
- ConvoyEnv reset now waits for V001 spawn with a timeout to avoid TraCI "Vehicle not known" failures.
- Scenario generator sorts route files by depart time; dataset_v1 regenerated to satisfy SUMO ordering.
- Dataset-based training smoke run completed with evaluation output saved to `ml/models/runs/`.

## Key Files Location

### Emulator Parameters (USE THESE)

| File | Purpose | Status |
|------|---------|--------|
| `ml/espnow_emulator/emulator_params_measured.json` | **CURRENT** - Convoy-calibrated (Feb 24) + validated against Rec #2 (March 1) | **Active — validated, burst stats updated** |
| `ml/espnow_emulator/emulator_params_convoy.json` | Convoy-specific params (Feb 21 recording source) | Reference |
| `ml/espnow_emulator/emulator_params_measured_rtt_backup.json` | Pre-convoy RTT-only params (backup) | Archive |
| `ml/espnow_emulator/emulator_params_5m.json` | Original 5m stationary test data | Archive |
| `ml/espnow_emulator/emulator_params_pre_v3_backup.json` | Pre-validation backup (Feb 24 original) | Archive |
| `ml/data/convoy_analysis_site/convoy_emulator_params.json` | Recording #2 extracted params (Feb 28) | Reference (ego-only, inflated noise) |
| `ml/data/convoy_analysis_extra/convoy_emulator_params.json` | Extra driving extracted params (Feb 28) | Reference (ego-only, inflated noise) |

### Active Plans (CHECK THESE)

| Document | Purpose | Priority |
|----------|---------|----------|
| `docs/10_PLANS_ACTIVE/PHASE_6_PREP_WORK_PLAN.md` | **Prep work: validate + process + generate** | **COMPLETED** (Steps 1-3, March 1) |
| `docs/10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md` | **Production model path: recordings → training** | **CRITICAL - Run 003 next** |
| `docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` | n-Element problem solution | **CRITICAL - READ FIRST** |
| `docs/10_PLANS_ACTIVE/MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md` | Mesh relay + continuous actions + cone filter | **COMPLETED** (Phases A-G) |
| `docs/10_PLANS_ACTIVE/N_ELEMENT_IMPLEMENTATION_PLAN.md` | Implementation steps (Phases 1-5) | REFERENCE |
| `docs/20_KNOWLEDGE_BASE/DATA_RECORDING_STRATEGY.md` | Recording protocol + axis mapping | REFERENCE |
| `docs/10_PLANS_ACTIVE/EC2_AMI_CREATION_PLAN.md` | Reusable training AMI | **COMPLETED** (ami-03a3037588b0f34f2) |

### Completed Work (REFERENCE)

| Document | Purpose |
|----------|---------|
| `docs/90_ARCHIVE/Emulator/ESPNOW_EMULATOR_PROGRESS.md` | ESP-NOW emulator implementation (84/84 tests) |
| `docs/20_KNOWLEDGE_BASE/NETWORK_CHARACTERIZATION_GUIDE.md` | Field test lessons learned |

---

## Decision Log

### February 26, 2026: Mesh Relay + Continuous Actions + Cone Filter (Professor Correction)

**Decision:** Three architectural corrections required before any further training.

**What was wrong:**
1. "Mesh" was labeled in code (V2VMessage has hopCount/sourceMAC, PackageManager deduplicates by sourceMAC) but NO component actually relays messages. Every vehicle only broadcasts its own state. The emulator and ConvoyEnv are purely point-to-point.
2. Discrete 4-action space (MAINTAIN/CAUTION/BRAKE/EMERGENCY) with hardcoded decel values defeats the purpose of RL. The model should learn the percentage of deceleration needed — that's what PPO is for.
3. Cone filtering was never implemented. It's not just an observation filter — it determines what each vehicle RELAYS in the mesh.

**What this means for AI agents:**
- Run 002 is a BASELINE only — trained without mesh, without cone filter, with wrong action space
- DO NOT start any training run until ALL three corrections are implemented and tested
- Follow `MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md` phases A through F
- Every change MUST have tests written FIRST (TDD)
- Action space is now `Box(low=0.0, high=1.0, shape=(1,))` — model outputs decel fraction
- Emulator must simulate multi-hop relay, not point-to-point
- Recording strategy: only record from ego (mesh delivers everything)

### February 20, 2026: Synthetic vs Real-Data Comparison Training Strategy

**Decision:** Train two models (Run 002 on synthetic base, Run 002b on real convoy base) with identical augmentation and emulator params, then compare results.

**Rationale:**
1. Professor requires real-world data grounding, but practical ML impact is debatable
2. A controlled comparison provides empirical evidence rather than argument
3. Both models use the same measured emulator params (real RTT drive) and augmentation flags
4. The only variable is the base scenario source (synthetic SUMO vs real GPS recording)
5. Either outcome is a publishable thesis finding

**What this means for AI agents:**
- Run 002 uses existing synthetic base scenarios (`ml/scenarios/base_01`)
- Run 002b uses real convoy recording (after 3-car drive is completed and processed)
- SAME augmentation flags, emulator params, hyperparameters for both
- Compare on: success rate, reward, collision rate, generalization across peer counts

### January 10, 2026: Pipeline-First Approach

**Decision:** Use 5m emulator params for pipeline validation before collecting clean drive data.

**Rationale:**
1. The 5m data is valid (correct RTT, GPS, IMU characteristics)
2. Building ConvoyEnv doesn't require production-perfect data
3. Validates pipeline works before investing in field data collection
4. If architecture has bugs, we'll find them with 5m data
5. Can always retrain later with better data

**What this means for AI agents:**
- DO use `emulator_params_5m.json` for ConvoyEnv implementation
- DO NOT wait for "better" data before building the gym environment
- DO focus on getting 80/80 ConvoyEnv tests passing

### January 19, 2026: ESP-NOW LR Mode Enabled + Channel 6 Default

**Decision:** Enable WIFI_PROTOCOL_LR + max TX power and standardize on channel 6 for home testing.

**Rationale:**
1. Current PCB antenna only works at ~5m (15.7% loss)
2. LR mode expected to extend range to 10-15m
3. Software-only fix (no hardware changes)
4. Must patch all three files: EspNowTransport.cpp, sender_main.cpp, reflector_main.cpp

**What this means for AI agents:**
- DO flash LR-enabled firmware on all devices (sender/reflector/main)
- DO keep all ESP-NOW devices on the same channel (default: 6 for home tests)
- DO complete extended range tests before updating emulator params

### January 24, 2026: Two-Recording Strategy for Real Data Collection

**Decision:** Use separate recordings for network characterization (RTT) and trajectory data (3-car convoy).

**Rationale:**
1. RTT firmware measures precise round-trip time (same device timestamps both send and receive)
2. 3-car convoy recording provides trajectory data but clock sync is ±100-500ms (GPS NMEA)
3. Mixing both in one recording would compromise precision of network measurements
4. RTT recording will be enhanced with GPS + magnetometer to capture sensor noise alongside network params

**What this means for AI agents:**
- DO use RTT recording output for `emulator_params_measured.json` (latency, loss, jitter, sensor noise)
- DO use 3-car convoy recording for base augmentation scenario (trajectories only)
- DO NOT try to measure latency from convoy recording (clock sync insufficient)

### January 23, 2026: Hybrid Data Strategy + Cone Filtering Clarification

**Decision:** Ground augmentation in a real 3-car recording and apply cone filtering only on V001 **inside AI inference** before observation building.

**Rationale:**
1. Professor requires a real-data basis before augmentation
2. Cone filtering is a receiver-side perception filter; broadcasters remain dumb
3. Training and inference must match observation filtering

**What this means for AI agents:**
- DO treat the real recording as the **mandatory base** for augmentation
- DO implement cone filtering **only on V001** (front FOV) and mirror it in simulation

---

## For AI Agents: What To Do

### If returning after a break (start here)
1. **READ FIRST:** This file (PROJECT_STATUS_OVERVIEW.md) — check current phase at the top
2. **Read:** `docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` (Deep Sets architecture)
3. **Reference:** `docs/10_PLANS_ACTIVE/RUN_004_HAZARD_TARGETING_AND_SOURCE_REACTION_EVAL_PLAN.md` (v2.2)
4. **Current work:** Run 005 training on EC2 (March 4, 2026)
   - Results will be at: `s3://saferide-training-results/cloud_prod_005/`
   - 201/201 unit tests passing
   - Post-training: download results, analyze, then H5 sim-to-real validation
5. **If launching a new training run:** See "EC2 Training Procedure" section above — follow it EXACTLY

### If asked to work on ML/Training:
1. **READ FIRST:** `DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` (critical architecture)
2. **IMPLEMENT:** `N_ELEMENT_IMPLEMENTATION_PLAN.md` (step-by-step, see Phase 6 for next run)
3. **KEY RULES:**
   - Observation MUST be Dict with variable peers
   - Use DeepSetPolicy, NOT MlpPolicy
   - DO NOT hardcode V002/V003 - handle n peers dynamically
   - Use max pooling for permutation invariance
   - Ensure route files are sorted by depart time (SUMO ignores unsorted entries)
   - **Training datasets MUST vary peer count** (n ∈ {1,2,3,4,5}), not just vehicle params
   - **Run 004 target:** 10M timesteps, dataset_v3 (real-grounded), deterministic eval matrix with hazard injection ON

### If asked to work on ConvoyEnv:
1. Observation space is already `Dict`; do not revert to `Box(11,)`
2. EmulatorESPNOW supports n peers; avoid hardcoded V002/V003 logic
3. Reset now waits for V001 spawn; keep the startup wait/timeout
4. See `N_ELEMENT_IMPLEMENTATION_PLAN.md` Phase 1 and 2

### If asked to create training scenarios/datasets:
1. **User manually:** Crop real-world map from OpenStreetMap, import to SUMO (NetConvert)
2. **User manually:** Create base scenario with route file and sumocfg
3. **Script:** Use `generate_dataset.py` to augment the base scenario
4. **CRITICAL:** The generator script MUST vary the number of vehicles per scenario
   - Generate scenarios with n ∈ {1, 2, 3, 4, 5} peers (plus V001 ego)
   - Do NOT generate all scenarios with the same vehicle count
   - Vary vehicle params (decel, tau, sigma, speedFactor) AND vehicle count
5. See `N_ELEMENT_IMPLEMENTATION_PLAN.md` Phase 6 for requirements

### If asked to work on firmware:
1. ML inference must loop over n peers with shared encoder
2. Max pooling must be done in C
3. See `N_ELEMENT_IMPLEMENTATION_PLAN.md` Phase 5

### If asked about the n-element problem:
1. It's solved by Deep Sets architecture
2. φ (shared encoder) applied to each peer
3. Max pooling aggregates to fixed-size latent
4. Professor's reference: `professors_research_for_RL.pdf`

---

## Quick Reference: Test Counts

| Component | Tests | Status |
|-----------|-------|--------|
| RTT Firmware | 42/42 | ✅ Complete |
| ESP-NOW Emulator | 84/84 | ✅ Complete |
| ConvoyEnv (Dict Obs) | 74/74 | ✅ Complete (mock paths fixed) |
| Deep Sets Policy | 6/6 | ✅ Complete |
| Scenario Gen + Eval Fix | 25/25 | ✅ Complete |
| Eval Matrix | 4/4 | ✅ Complete |
| Reward Calculator | 25/25 | ✅ Complete (incl. early reaction) |
| **Total (non-SUMO `tests/unit/`)** | **201/201** | **100% passing** |

Note: The 201 count is the `tests/unit/` suite run locally without SUMO. Integration tests require SUMO via Docker (`./run_docker.sh integration`).

---

## Navigation

- **Full Index:** `docs/_MAPS/INDEX.md`
- **Architecture:** `docs/00_ARCHITECTURE/`
- **Active Plans:** `docs/10_PLANS_ACTIVE/`
- **Knowledge Base:** `docs/20_KNOWLEDGE_BASE/`
- **Archive:** `docs/90_ARCHIVE/`

---

**Document Version:** 1.9
**Maintained By:** Bookkeeper Agent
